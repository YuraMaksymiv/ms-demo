# Spec — Fortnight unit in parsing

Status: **draft** (2026-10-22) · Scope: `src/` ·
Location: `specs/fortnight-unit.md`

## 1. Summary
The parser gains a `fortnight` unit worth 14 days: `ms('2 fortnights')` and `ms('1fn')` return multiples of 1,209,600,000 ms. Formatting is unchanged: `format()` / `ms(number)` never emit fortnights, in short or long mode. Parsing must stay as cheap as today (single regex pass, no new code path).

| Surface | Where | What |
|---|---|---|
| `ms(string)` | `src/index.ts:48-62` | returns the fortnight value; delegates to `parse` |
| `parse(string)` | `src/index.ts:71` | owns the grammar (regex) and the unit `switch`; gains the unit |
| `parseStrict(StringValue)` | `src/index.ts:156` | delegates to `parse`, inherits the change |
| `StringValue` / `Unit` types | `src/index.ts:9-32` | admit the new aliases at type level |
| `format` / `ms(number)` | `src/index.ts:163-231` | unchanged |

## 2. Current state
- Unit families are declared as string-literal types (`Years`..`Milliseconds`) joined into `Unit` (`src/index.ts:9-25`). `UnitAnyCase = Capitalize<Unit> | Uppercase<Unit> | Unit` (`src/index.ts:27`) and `StringValue` is built from it (`src/index.ts:29-32`), so a new alias in `Unit` is automatically admitted in lower, Capitalized and UPPER case.
- Time constants `s, m, h, d, w, y, mo` are defined at `src/index.ts:1-7`; there is no fortnight constant.
- `parse` uses one case-insensitive regex with an optional unit group (`src/index.ts:77-80`). Alternatives are listed per family, long form first (`milliseconds?|msecs?|ms|seconds?|secs?|s|…|years?|yrs?|y`). Plurals are expressed as `x?` or as separate aliases (`yrs`, `hrs`, `mins`, `secs`, `msecs`). The regex allows an optional leading `+`/`-` and optional spaces before the unit (`src/index.ts:78`).
- The unit is lower-cased and switched over `Lowercase<Unit>`; the `default` branch uses `matchUnit satisfies never`, so a unit added to `Unit` without a `case` fails `tsc` (`src/index.ts:95-146`).
- No match returns `NaN` (`src/index.ts:82-84`); empty, non-string or >100-char input throws (`src/index.ts:72-76`).
- `fmtShort` / `fmtLong` step through `y, mo, w, d, h, m, s` only (`src/index.ts:163-216`). 14 days therefore formats as `2w` / `2 weeks` (derived from `msAbs >= w`, `Math.round(ms / w)`).
- Tests: `src/parse.test.ts`, `src/parse-strict.test.ts`, `src/index.test.ts`, `src/format.test.ts`; type-failure cases use `@ts-expect-error`. No benchmark script or file exists (`package.json` scripts: `test`, `test:nodejs`, `test:edge`, `build`, `lint`, `typecheck`, `attw`).
- The working tree contains uncommitted leading-plus work (`specs/parse-leading-plus.md`); this spec treats it as current state. No `CLAUDE.md`, `AGENTS.md`, `INSIGHTS.md` or system state was provided.

## 3. Goals / Non-goals
Goals: G1 parse `fortnight`, `fortnights`, `fn` as 14 days each; G2 keep `format` / `ms(number)` unchanged; G3 keep parsing performance structurally unchanged.
Non-goals: a fortnight output in `format`; new options; new dependencies; other new units; changing `parse` error behaviour; changing the 100-char limit.

## 3a. Requirement coverage
| # | Mandatory requirement (source) | AC/NFR IDs | Testable? |
|---|---|---|---|
| R1 | "ms('2 fortnights') and ms('1fn') return 14 days per fortnight in milliseconds (1,209,600,000 ms)" (request 1) | AC-1, AC-2, AC-3 | yes |
| R2 | "Required: fortnight, fortnights, fn" aliases (request) | AC-1, AC-2, AC-3 | yes |
| R3 | "Formatting is unchanged: never outputs fortnights, short or long" (request 2) | AC-8, AC-9 | yes |
| R4 | "Parsing must stay fast ... no extra regex passes, the unit added to the existing single regex/switch, no meaningful performance regression" (request 3) | NFR-1, NFR-2, NFR-6 | yes: structural form (NFR-1..3) plus a before/after micro-benchmark (NFR-6); the numeric threshold in NFR-6 is a labelled proposal, see §15 M2 |
| R5 | "plural/case handling and the strict-parse/type-level implications to match other units" (request) | AC-4, AC-5, AC-7, AC-10, AC-11 | yes |
| R6 | G1 / G2 / G3 (Goals) | as R1, R3, R4 | yes |

## 4. Users & UX flow
Actor: a developer passing duration strings to `ms()`, `parse()` or `parseStrict()`. No UI.

| State | Trigger | Result |
|---|---|---|
| Valid fortnight | `ms('1fn')`, `ms('2 fortnights')` | `1209600000`, `2419200000` |
| Unknown alias | `ms('1fortnightly')` | `NaN` (unchanged grammar behaviour) |
| Format of 14 days | `ms(1209600000)` | `'2w'` (unchanged); long: `'2 weeks'` |

Design sources: user request text only; no screens.

## 5. Design analysis
- Gaps: whether `fns` (plural short form, like `hrs`/`mins`) is accepted was not stated (§15 M1).
- Alias collision risk: no existing alias starts with `f` (`src/index.ts:78`), so `fortnights?` and `fn` cannot be a prefix of, or prefixed by, another unit. Because the regex is anchored with `$`, alternation order cannot change which unit matches; the existing convention (long before short within a family) is kept for readability only. The only risk is a typo placing the alternative outside the unit group.
- Equivalence note: 1 fortnight equals `2 * w`; `format` keeps emitting weeks, so `ms(ms('1fn'))` returns `'2w'`, not a fortnight string (accepted by requirement 2).
- Module interaction: single module; `ms` and `parseStrict` delegate to `parse` (§9).
- UX improvements: none proposed.

## 6. Decisions
| # | Decision | Rationale | Source |
|---|---|---|---|
| D1 | One fortnight = 14 * 24 * 60 * 60 * 1000 ms = 1,209,600,000 | stated | user |
| D2 | Accepted aliases: `fortnight`, `fortnights`, `fn` (plus case variants) | stated | user |
| D3 | Alias handling follows the existing pattern: case-insensitive, optional spaces before unit, `fortnights?` for the plural | existing pattern | code `src/index.ts:78,95` |
| D4 | The unit is added to the existing regex alternation and the existing `switch`; no second regex or lookup | stated as example of the NFR | user |
| D5 | The unit joins the `Unit` type (new `Fortnights` family), so `StringValue` / `parseStrict` admit it | matches how every other unit is typed | code `src/index.ts:9-32` |
| D6 | `format` / `fmtShort` / `fmtLong` are not touched | stated | user |

## 7. Data model & contracts
No DB. Public types: `StringValue` widens to admit `${number}fn`, `${number} fortnights` etc. in lower, Capitalized and UPPER case (via `Unit`, `src/index.ts:17-32`). No signature or export change for `ms`, `parse`, `parseStrict`, `format`. No DTO or migration.

## 8. Inputs & provenance
| Input | Source | Trust | When absent or degraded | Cap |
|---|---|---|---|---|
| Duration string | caller argument (`src/index.ts:71`) | untrusted | empty / non-string / >100 chars throws; non-matching returns `NaN` | length 1..100 (`src/index.ts:72`) |

Not an LLM feature.

## 9. Module interaction
Single module `src/index.ts`. `ms(string)` → `parse`; `parseStrict` → `parse`. Failure propagation unchanged: grammar mismatch returns `NaN`, type/length violation throws. No diagram (fewer than 3 participants).

## 10. Edge cases
| # | Case | Expected behaviour | AC |
|---|---|---|---|
| E1 | `'1FN'`, `'1 Fortnight'`, `'1FORTNIGHTS'` | same as lower case | AC-4 |
| E2 | `'1.5fn'`, `'.5fn'` | decimal multiples of 14 days | AC-5 |
| E3 | `'-1fn'`, `'+1fn'` | negative / same as `'1fn'` (leading-plus behaviour) | AC-5 |
| E4 | `'1fortnightly'`, `'1fnn'`, `'1 f'`, `'fn'` (no number) | `NaN` | AC-6 |
| E5 | `'1 fn'`, `'1  fn'` (spaces before unit) | same as `'1fn'` | AC-4 |
| E6 | `'1fns'` | see M1 | AC-6 [NEEDS CLARIFICATION:minor — M1: `fns` accepted? — options: NaN (not required, no plural-s on short aliases `w`,`d`,`h`) / accept as alias (like `hrs`); impact: regex alternative + one AC] |
| E7 | Existing units adjacent in regex (`m`, `mo`, `ms`, `s`, `w`, `d`, `y`) | results unchanged | AC-9 |
| E8 | `format` of 14 days, 28 days, 1,209,600,000 | `2w` / `4w` (short); `2 weeks` / `4 weeks` (long) | AC-8 |
| E9 | Over-100-char string, empty, non-string | throws as today | AC-12 |
| E10 | Concurrency / re-runs / partial failure | n/a — pure synchronous function; out of scope — §16 | §16 |

## 11. Acceptance criteria (EARS)
| ID | Pattern | Requirement | Cat |
|---|---|---|---|
| AC-1 | Ubiquitous | `parse()` shall return 1,209,600,000 for `'1fn'`. | C3 |
| AC-2 | Ubiquitous | `parse()` shall return 2,419,200,000 for `'2 fortnights'`. | C3 |
| AC-3 | Ubiquitous | `parse()` shall return 1,209,600,000 for `'1 fortnight'` and for `'1fortnights'` (singular and plural spellings are interchangeable regardless of count). | C3 |
| AC-4 | Ubiquitous | `parse()` shall return the same value for any letter case of `fortnight`, `fortnights` and `fn` (e.g. `'1FN'`, `'1 Fortnight'`, `'1FORTNIGHTS'`) and with zero or more spaces before the unit. | C3 |
| AC-5 | Ubiquitous | `parse()` shall return `n * 1,209,600,000` for decimal, negative and `+`-prefixed `n` (e.g. `'1.5fn'` = 1,814,400,000; `'-1fn'` = -1,209,600,000; `'+1fn'` = 1,209,600,000). | C3 |
| AC-6 | Unwanted | If the unit text is not one of the accepted aliases (`'1fortnightly'`, `'1fnn'`, `'1 f'`) or the number is missing (`'fn'`), then `parse()` shall return `NaN`. | C5 |
| AC-7 | Event-driven | When `ms()` or `parseStrict()` receives a fortnight string, it shall return the same number as `parse()`. | C4 |
| AC-8 | Ubiquitous | `format()` and `ms(number)` shall never output the text `fortnight` or `fn`, in short and `long` modes, for any finite input (including 1,209,600,000 and its multiples, which stay `2w`-style week output). | C1 |
| AC-9 | Ubiquitous | `parse()` shall return the same results as before for every input that does not use a fortnight alias (existing test suites pass unchanged). | C1 |
| AC-10 | Ubiquitous | The `StringValue` type shall admit `'1fn'`, `'1 fortnight'`, `'1 fortnights'` and their Capitalized and UPPERCASE forms, so `parseStrict('1fn')` and `ms('1fn')` compile without `@ts-expect-error`. | C3 |
| AC-11 | Unwanted | If a unit alias is in the `Unit` type but has no `case` in the `parse` switch, then `tsc` shall fail (existing `satisfies never` guard kept). | C3 |
| AC-12 | Ubiquitous | `parse()` shall throw for empty, non-string and over-100-character input exactly as before. | C5 |

## 12. Non-functional requirements
| ID | Requirement | Measure / threshold |
|---|---|---|
| NFR-1 | `parse()` shall perform exactly one regex execution per call. | `parse` body contains one `exec`; no added `match`/`test`/`replace`/second regex (review plus code diff) |
| NFR-2 | The fortnight aliases shall be added to the existing regex alternation and existing `switch`, with no new lookup table, pre-check or loop. | diff touches only the alternation (`src/index.ts:78`), a constant, a `case` group, and the `Unit` types |
| NFR-3 | The regex shall add no nested or ambiguous quantifier. | review: new alternatives are `fortnights?` and `fn` only |
| NFR-4 | The package shall add no dependency and no new export. | `package.json` dependency fields unchanged; `pnpm attw` passes |
| NFR-5 | The change shall pass in Node and Edge Runtime, lint and typecheck. | `pnpm test`, `pnpm lint`, `pnpm typecheck` exit 0 |
| NFR-6 | `parse()` shall show no meaningful throughput regression for inputs that use existing units and for fortnight inputs. | Proposal (M2): on one machine, same Node version, run a fixed corpus (one valid string per existing unit family, one `'1fn'`, one `'2 fortnights'`, one non-matching string) for >= 1,000,000 calls each, before and after the change, median of >= 5 runs; after is no more than 10% slower than before. The 10% figure is not user-confirmed. |

## 13. Traceability
| Req | Goal / Decision | Module | Area | Verification |
|---|---|---|---|---|
| AC-1..5 | G1, D1-D3 | `src/` | regex `src/index.ts:78`, switch `:98-146`, new constant near `:1-7` | unit, `parse.test.ts`, `index.test.ts` |
| AC-6 | D2, D3 | `src/` | regex | unit, `parse.test.ts` |
| AC-7 | D5 | `src/` | `src/index.ts:54,156` | unit, `index.test.ts`, `parse-strict.test.ts` |
| AC-8 | G2, D6 | `src/` | `fmtShort`/`fmtLong` | unit, `format.test.ts`, `index.test.ts` |
| AC-9, AC-12 | non-regression | `src/` | existing tests | existing suites |
| AC-10, AC-11 | D5 | `src/` | `src/index.ts:9-32,141-142` | `pnpm typecheck` with non-`@ts-expect-error` test lines |
| NFR-1..3 | G3, D4 | `src/` | `parse` | code review of diff |
| NFR-6 | G3 | `src/` | `parse` regex `src/index.ts:77-80` | one-off micro-benchmark before/after (script kept outside `src/`, not committed unless the user asks) |
| NFR-4, NFR-5 | non-regression | repo | `package.json`, `src/` | commands in §14 |

## 14. Verification hints
Documented commands (`package.json`): `pnpm test` (Node + Edge), `pnpm test:nodejs`, `pnpm typecheck`, `pnpm lint`, `pnpm attw`. No benchmark command is documented — `implementation-planner` must ask where a benchmark script may live (M2); it must not add a dependency (NFR-4).
- Test names: `parse.test.ts` and `parse-strict.test.ts` already hold `AC-n` tests from `parse-leading-plus.md` (e.g. `src/parse.test.ts:250`); name fortnight tests `it('fortnight AC-n: …')` or use a separate `describe` so `plan-verifier` can tell the two specs apart. The existing leading-plus unit-alias list (`src/parse.test.ts:213-248`) does not include fortnight aliases; extending it is optional.
- NFR-6: before/after micro-benchmark as described in the NFR; record both medians in the plan's verification notes.
- AC-1..AC-9, AC-12: unit tests named `it('AC-n: …')` in the existing Jest files, in the style of the existing `describe` blocks.
- AC-8: assert on short and long output for 1,209,600,000, 2,419,200,000 and 14 days +/- 1 ms.
- AC-10: type-level check by the compiler (`pnpm typecheck`); AC-11: temporary removal of a `case` is a manual check, no automated test.
- NFR-1..NFR-3: review of the diff.

## 15. Open questions
| # | Severity | Question |
|---|---|---|
| M1 | minor | Is `fns` (short plural, like `hrs`/`mins`) accepted? Spec assumes not: only `fortnight`, `fortnights`, `fn` as required. |
| M2 | minor | [NEEDS CLARIFICATION:minor — M2: confirm the NFR-6 threshold (proposed: no more than 10% slower, median of 5 runs) — options: 10% / 5% / structural-only (NFR-1..3); impact: NFR-6 number only]. No benchmark exists in the repo (`package.json:29-40`). |

## 16. Out of scope
Fortnight output in `format`; `fns` unless M1 resolves otherwise; other new units; concurrency / re-run concerns (pure function); README / docs updates (not checked, not in write scope).

## 17. Changelog
| Date | Change | Reason / source |
|---|---|---|
| 2026-10-22 | Initial draft | user request; code read of `src/index.ts` and tests, `specs/parse-leading-plus.md` conventions |
| 2026-10-22 | Added NFR-6 (before/after micro-benchmark, proposed 10% threshold), updated R4, §13, §14, M2; added test-naming hint | request 3 requires an NFR with a verification hint; second pass over existing draft |
