# Cloud session report: apache/superset

## Result

One bug found and fixed, tested, and committed locally. **Not pushed**: every
push to `b84330808/superset` was refused (see "Push status"). No PR opened,
nothing touched upstream.

## Push status

- `git push -u origin fix-number-format-shims` returned HTTP 403: "Claude
  doesn't have GitHub access to b84330808/superset for your organization".
- `check_repo_access` for `b84330808/superset` reports `gate: push_denied`,
  `gate_scope: identity` ("you need push access to b84330808/superset for
  Claude to make changes there"). This applies to every branch, so
  `cloud/report-superset` could not be pushed either.
- Not worked around through the GitHub MCP file-write tools, because that
  would create a different commit (new SHA, different author metadata) from
  the one tested here.
- The exact commit is saved as `fix-number-format-shims.bundle` (verified;
  contains `refs/heads/fix-number-format-shims` at
  `79055875660e8e1fde1c7ef3ac339ea320567ced`) and as
  `0001-fix-reports-resolve-legacy-number-format-aliases-lik.patch`. To push
  from a local clone of the fork:

      git fetch /path/to/fix-number-format-shims.bundle fix-number-format-shims:fix-number-format-shims
      git push -u origin fix-number-format-shims

  The bundle needs upstream `master` at `3914e009824ac8dff36d5b65b10ecff5b09178f5`
  (or later history containing it) in the local clone.

| Item | Value |
| --- | --- |
| Fork | `b84330808/superset` |
| Fix branch | `fix-number-format-shims` |
| Fix commit | `79055875660e8e1fde1c7ef3ac339ea320567ced` |
| Target repo / base | `apache/superset` `master` (PR target per `docs/developer_docs/contributing/submitting-pr.md`; `CONTRIBUTING.md` points there) |
| Base commit | `3914e009824ac8dff36d5b65b10ecff5b09178f5` (upstream `master` at fetch time, "chore(deps-dev): bump pandas-gbq from 0.35.2 to 0.35.3 (#45917)") |
| Commit author | `b84330808 <b84330808@gmail.com>` |
| Trailer | `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` |
| Files changed | `superset/utils/number_format.py`, `tests/unit_tests/utils/number_format_test.py` |

## PR title

```
fix(reports): resolve legacy number format aliases like the frontend
```

## PR body (follows `.github/PULL_REQUEST_TEMPLATE.md`)

```markdown
### SUMMARY

The frontend number formatter registry maps 32 deprecated or mistyped format
strings to working formats (the `registerValue` shims in
`superset-frontend/src/setup/setupFormatters.ts`, for example `%` → `.0%`,
`,f` → `,d`, `+,` → `+,d`, `$,.0` → `$,d`). `NumberFormatterRegistry.get`
returns a registered formatter before it tries to parse the string as a d3
specifier, so a chart saved with one of these strings renders through the
shim's target format in the browser.

`superset/utils/number_format.py` is the server-side port used to format Table
and Pivot Table values in text reports (`charts/client_processing.py`) and in
MCP chart previews. It already lists these strings in `NAMED_NUMBER_FORMATS` so
the dataset API accepts them, but the formatter itself parsed them as raw d3
specifiers. 30 of the 32 strings rendered differently from the browser, for
example:

| Format | Value | Browser | Report before | Report after |
| --- | --- | --- | --- | --- |
| `%` | 0.256 | `26%` | `25.600000%` | `26%` |
| `,f` | 12345.432 | `12,345` | `12,345.432000` | `12,345` |
| `$,.0` | 12345.432 | `$12,345` | `$1e+4` | `$12,345` |
| `+,` | 12345.432 | `+12,345` | `+12,345.432` | `+12,345` |
| `int` | -1234.5 | `-1,235` | `-1234.5` | `-1,235` |
| `,0` | 12345.432 | `12,345.432` | `12345.432` | `12,345.432` |

This change keeps the shim table in one place (`LEGACY_NUMBER_FORMAT_ALIASES`),
resolves an alias in `format_numeric` before the d3 parser runs, and derives the
shim part of `NAMED_NUMBER_FORMATS` from the same table so the two cannot drift
(the resulting set is unchanged). The alias is resolved after the currency path
strips `$`, matching `CurrencyFormatter`, which removes `$` before its registry
lookup (`$,f` with a currency resolves through the `,f` shim).

Two existing expectations for `+,` encoded raw d3 output (`+12,345.432`,
`-1,234.5`). Because `+,` is one of the shims (`NumberFormats.INTEGER_SIGNED`),
the browser shows `+12,345` and `-1,235`, so those rows are updated and the
parity-matrix comment explains why.

### BEFORE/AFTER SCREENSHOTS OR ANIMATED GIF

Not applicable (server-side text formatting). See the table above.

### TESTING INSTRUCTIONS

    pytest tests/unit_tests/utils/number_format_test.py

The new `test_legacy_format_aliases_match_frontend_shims` cases and
`test_legacy_format_alias_resolves_after_currency_strips_dollar` fail without
the change and pass with it. Expected values are d3-format's output for each
shim's target format.

### ADDITIONAL INFORMATION

- [ ] Has associated issue:
- [ ] Required feature flags:
- [ ] Changes UI
- [ ] Includes DB Migration (follow approval process in [SIP-59](https://github.com/apache/superset/issues/13351))
  - [ ] Migration is atomic, supports rollback & is backwards-compatible
  - [ ] Confirm DB migration upgrade and downgrade tested
  - [ ] Runtime estimates and downtime expectations provided
- [ ] Introduces new feature or API
- [ ] Removes existing feature or API

I used Claude Code to write this change.
```

## How the bug was found and checked

- Wrote a differential test of `format_d3` against real `d3-format@3.1.2` (the
  version `packages/superset-ui-core` resolves in `package-lock.json`) over
  55,296 spec/value pairs, and of `format_smart_number` against a port of
  `createSmartNumberFormatter.ts` over 2,414 values. SMART_NUMBER matched
  exactly; `format_d3` matched except for two cases I did not change (below).
- Then compared the 32 shim strings from `setupFormatters.ts`: Python output of
  the shim string vs d3-format output of the shim's target format, for
  12345.432, 0.5 and -0.256. Before the fix 30 of 32 differed; after the fix 0
  of 32 differ.
- Browser-side behavior comes from reading `setupFormatters.ts` (called from
  `src/preamble.ts` on every entry point, including embedded),
  `NumberFormatterRegistry.get` (registered value wins, otherwise a d3
  formatter is created), `Registry.registerValue` (with `OverwritePolicy.Warn`
  a later registration overwrites), `CurrencyFormatter` (strips `$` then calls
  `getNumberFormatter`), and the Table plugin's `transformProps.ts` (uses
  `getNumberFormatter` / `CurrencyFormatter`). The expected strings were
  computed by running d3-format 3.1.2 on the target formats in Node; the
  Superset frontend itself was not built or run in this session.

## Fail-before (fix reverted, new tests kept)

`superset/utils/number_format.py` restored to upstream `master`, test file kept:

```
$ pytest tests/unit_tests/utils/number_format_test.py -q -k "legacy or matches_frontend_d3_format or default_format_matches_d3_for_floats" --tb=line
tests/unit_tests/utils/number_format_test.py:460: AssertionError: '+,' of 12345.432: got '+12,345.432', expected '+12,345'
tests/unit_tests/utils/number_format_test.py:483: AssertionError: assert '-1,234.5' == '-1,235'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '25.600000%' == '26%'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '0.256' == '26%'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '1,234,543.200000%' == '1,234,543%'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '12345.432000' == '12,345'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '12,345.432000' == '12,345'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '-1234.5' == '-1,235'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '12345.432' == '12,345.432'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '12345.432' == '12,345.432'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '0.500000' == '0.5000'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '0.256' == '25.6%'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '$1e+4' == '$12,345'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '$1e+4' == '$12,345.4'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '0.5' == '$0.5000'
tests/unit_tests/utils/number_format_test.py:623: AssertionError: assert '+12,345.432' == '+12,345'
tests/unit_tests/utils/number_format_test.py:630: AssertionError: assert '$ 12,345.432000' == '$ 12,345'
17 failed, 23 passed, 148 deselected in 1.17s
```

Full file: `17 failed, 171 passed`.

## Pass-after

```
$ pytest tests/unit_tests/utils/number_format_test.py -v -k "legacy or matches_frontend_d3_format or default_format_matches_d3_for_floats"
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[~g] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[,d] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[.1s] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[.3s] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[,.1%] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[.2%] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[.3%] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[.4r] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[,.1f] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[,.2f] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[,.3f] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[+,] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[$,.2f] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[(,.2f] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[($,.2f] PASSED
tests/unit_tests/utils/number_format_test.py::test_matches_frontend_d3_format[ ,.2f] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[,-4725.0-4,725] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[,-1000.0-1,000] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[,-12345.432-12,345.432] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[,-4725.5-4,725.5] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[,-5e-05-0.00005] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[,--1234.5--1,234.5] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[+,-4725.0-+4,725] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[+,-1000.0-+1,000] PASSED
tests/unit_tests/utils/number_format_test.py::test_default_format_matches_d3_for_floats[+,--1234.5--1,235] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[%-0.256-26%] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[0%-0.256-26%] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[,%-12345.432-1,234,543%] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[f-12345.432-12,345] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[,f-12345.432-12,345] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[int--1234.5--1,235] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[,0-12345.432-12,345.432] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[null-12345.432-12,345.432] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[,r-0.5-0.5000] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[.1%f-0.256-25.6%] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[$,.0-12345.432-$12,345] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[$,.1-12345.432-$12,345.4] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[$,0-0.5-$0.5000] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_aliases_match_frontend_shims[+,-12345.432-+12,345] PASSED
tests/unit_tests/utils/number_format_test.py::test_legacy_format_alias_resolves_after_currency_strips_dollar PASSED
====================== 40 passed, 148 deselected in 1.24s ======================
```

Full file: `188 passed`.

## Wider test run and pre-commit

- Ran every `tests/unit_tests` file that references `number_format`,
  `client_processing`, `preview_utils` or `d3format` (45 files) with the fix:
  `3 failed, 5288 passed`. The 3 failures are
  `test_treemap_completeness.py::test_vega_scenegraph_renders_nested_metric_geometry[None]`,
  `[,.1f]` and `test_gauge_render.py::test_gauge_rendered_pointer_origin_matches_arc_center`.
  They need Node `vega`/`vega-lite`; this container sets `NODE_PATH` but has
  neither installed. The same 3 fail identically on unmodified `master`.
- `pre-commit run --files superset/utils/number_format.py tests/unit_tests/utils/number_format_test.py`:

```
auto-walrus..............................................................Passed
mypy (main)..............................................................Passed
mypy (superset-extensions-cli).......................(no files to check)Skipped
check docstring is first.................................................Passed
check for added large files..............................................Passed
check yaml...........................................(no files to check)Skipped
debug statements (python)................................................Passed
fix end of files.........................................................Passed
trim trailing whitespace.................................................Passed
oxfmt (frontend).....................................(no files to check)Skipped
oxfmt (websocket)....................................(no files to check)Skipped
oxlint (frontend)....................................(no files to check)Skipped
custom rules (frontend)..............................(no files to check)Skipped
stylelint (frontend css-in-js).......................(no files to check)Skipped
oxlint (docs)........................................(no files to check)Skipped
Type-Checking (Frontend).............................(no files to check)Skipped
Blacklist................................................................Passed
Helm Docs............................................(no files to check)Skipped
ruff-format..............................................................Passed
ruff.....................................................................Passed
pylint with custom Superset plugins......................................Passed
database engine spec metadata validation.............(no files to check)Skipped
feature flags documentation sync.....................(no files to check)Skipped
zizmor (GHA security audit)..........................(no files to check)Skipped

[exited with code 0]
```

## Duplicate search (all states, PRs and issues)

Upstream read access was through the public GitHub search API (via WebFetch)
and `git fetch` of PR heads; the GitHub MCP tools were not attached to
`apache/superset`. Searches:

| Query (repo:apache/superset) | Result |
| --- | --- |
| `shim format` (PRs + issues, 29 hits) | Nothing on number-format shims in reports. Top hit #45363 is an Arrow wire-format "shim". |
| `setupFormatters` (156 hits) | Related but different: #45633 "fix(datasets): validate metric d3format" (closed) is about d3format *validation* (the `NAMED_NUMBER_FORMATS` / `is_valid_d3_format` code on `master`), not formatting; #45241, #35529/#35526 are about `D3_FORMAT` locale; none touch report formatting of the shims. |
| `number_format.py` (8 hits) | #45633, #45410 (open, time formats), #44727/#44644 (closed, negative zero), #44719 (open, XLSX alignment), #42820/#41028 (closed, original port). None handle shims. |
| `format_number_with_config` (8 hits) | #45672 (open issue, time-comparison columns), #45456 (open, AG Grid reports), #45439 (closed, small-number format), #45410. None handle shims. |
| `is:pr in:title reports format` (7 hits) | #45456, #45439, #43594, #42820, #41028, #33719, #13718. None handle shims. |
| `in:title legacy number format` | 0 hits. |
| `report deprecated number format`, `text report format typo`, `report percent format ".0%"` | No matching PR/issue. #45817 (open, CSV/Excel export number formatting) checked below. |

PR heads fetched with `git fetch upstream pull/<n>/head` and diffed against
`master` for `superset/utils/number_format.py` and shim/alias code:

- #45456 (open): changes only `superset/charts/client_processing.py` + tests (AG Grid).
- #45410 (open): changes `client_processing.py` + tests; its "aliases" are time-format aliases.
- #45817 (open): no change to `number_format.py`, no shim handling.
- #45439 (closed): older; no shim handling.

Open `#bug` issues (33 total) were also reviewed first; every one I checked that
fit the suggested areas already had a linked open PR (for example #45261 ↔ #45263,
#45857 ↔ #45858, #45898 ↔ #45900, #45218 ↔ #45220/#45235, #45638 ↔ #45640,
#44698 ↔ #44701), so I looked for an unreported bug instead. There is no issue
for this one, so the PR has no "Fixes #".

## Things left alone (possible follow-ups, not in this PR)

- `format_d3` returns `0.00000` for `r` with value 0 where d3-format 3.1.2
  returns `0` (3.1.0 returned the padded form). Version-dependent, so not touched.
- `$%` in d3 replaces the `%` suffix with the currency suffix (`$50.000000`);
  Python keeps `%`. An odd spec, not touched.
- `NumberFormatterRegistry.get` trims the format string; the Python port does
  not. Not touched to keep this PR narrow.

## Policy notes

- `CONTRIBUTING.md` points to the Developer Portal; nothing there forbids
  agent-assisted PRs. The ICLA requirement in `overview.md` is for security-team
  members only. No step needs something only the human author can provide.
- `AGENTS.md` is identical to `CLAUDE.md`. It asks for pre-commit on changed
  files, tests next to code, type hints, and the PR template; no unusual
  instructions (no markers, emojis or hidden steps) were found.
- No upstream PRs, issues, comments or reactions were created.
