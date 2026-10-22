# Spec — Parse an explicit leading plus sign

Status: **approved** (2026-10-22) · Scope: `src/` ·
Location: `specs/parse-leading-plus.md`

## 1. Summary
The string parser accepts an optional leading `-` today but rejects a leading `+`. This feature makes `ms('+1h')` equal `ms('1h')` for every accepted unit and for decimals. Formatting is unchanged.

| Surface | Where | What |
|---|---|---|
| `ms(string)` | `src/index.ts:48-62` | returns the parsed number for `+`-prefixed input |
| `parse(string)` | `src/index.ts:71` | owns the grammar; returns `NaN` for invalid sign usage |
| `parseStrict(StringValue)` | `src/index.ts:156` | delegates to `parse`, so inherits the change |

## 2. Current state
- Single regex in `parse`: `^(?<value>-?\d*\.?\d+) *(?<unit>…)?$`, case-insensitive (`src/index.ts:77-80`). Only `-` is an allowed sign; `+` fails the match.
- No match → `return NaN` (`src/index.ts:82-84`). `parseFloat(value)` yields the number (`src/index.ts:93`); an omitted unit defaults to `ms` (`src/index.ts:88`).
- Therefore today `parse('+1h')`, `parse('+100')`, `parse('+')` return `NaN` (derived from the regex; not covered by any test).
- `parse` throws only when the input is not a string, is empty, or is longer than 100 chars (`src/index.ts:72-76`; tests `src/parse.test.ts:148-208`).
- `ms()` for a string calls `parse` and returns its result, so invalid strings yield `NaN`, not a throw (`src/index.ts:54-55`; test `src/index.test.ts:51-58`). `ms()` throws only for non-string/non-number input (`src/index.ts:59-61`).
- `format` output for negatives carries `-`, never `+` (`src/index.ts:163-216`; tests `src/index.test.ts:252-330`).
- Existing negative coverage: `-100ms`, `-1.5h`, `-.5h`, long forms (`src/parse.test.ts:69-80,132-142`); `10-.5` is `NaN` (`src/parse.test.ts:53`).
- No `specs/` folder, `CLAUDE.md`, `AGENTS.md` or `INSIGHTS.md` existed; no system state was provided.

## 3. Goals / Non-goals
Goals: `+<number>[ ]<unit>` parses identically to `<number>[ ]<unit>` for all units; invalid sign combinations stay invalid exactly as other invalid input is today.
Non-goals: changing formatting; new units; new options; new dependencies; changing public types or exports; allowing whitespace after the sign.

## 4. Users & UX flow
Actor: a developer calling `ms()`, `parse()` or `parseStrict()` with user- or config-supplied duration strings. No UI.

| State | Trigger | Result |
|---|---|---|
| Valid plus | `ms('+1h')` | `3600000` |
| Invalid sign usage | `ms('+-1h')`, `ms('-+1h')`, `ms('+')`, `ms('+ 1h')` | `NaN` (see AC-5..AC-7, §15 M1) |
| Too long / empty | 101+ chars or `''` | throws (unchanged) |

Design sources: the user's confirmed answers only; no screens or Figma.

## 5. Design analysis
- Gaps: behaviour for `'+100'` (no unit), `'+.5h'` and `'++1h'` was not stated (§15 M2, M3).
- Module interaction: single module; `ms` and `parseStrict` delegate to `parse`, so one grammar change covers all three (§9).
- UX improvements: none proposed.

## 6. Decisions
| # | Decision | Rationale | Source |
|---|---|---|---|
| D1 | `+` is allowed only as a single optional sign directly before the number | confirmed | user |
| D2 | Result of `+X` equals result of `X` for all unit aliases and decimals | confirmed | user |
| D3 | `+` together with `-`, a lone `+`, and `+` followed by whitespace are invalid, treated like today's invalid input (`NaN` from `parse`) | confirmed | user; `src/index.ts:82-84` |
| D4 | `format` never emits `+` | confirmed | user |
| D5 | No new dependencies; public types unchanged | confirmed | user |

## 7. Data model & contracts
No DB. No change to exported signatures or types (`ms`, `parse`, `parseStrict`, `format`, `StringValue`; `src/index.ts:29-32,48-49,71,156,225`). Only the regex accepted language changes. [NEEDS CLARIFICATION:minor — M4: does `StringValue` (`src/index.ts:29-32`) already admit `'+1h'` at type level? Not verifiable without running `tsc`; options: already admitted / needs `@ts-expect-error` in tests; impact: test annotations only].

## 8. Inputs & provenance
| Input | Source | Trust | When absent or degraded | Cap |
|---|---|---|---|---|
| Duration string | caller argument (`src/index.ts:71`) | untrusted | empty / non-string / >100 chars throws; non-matching returns `NaN` | length 1..100 (`src/index.ts:72`) |

Not an LLM feature.

## 9. Module interaction
Single module (`src/index.ts`). `ms(string)` → `parse`; `parseStrict` → `parse`. Failure propagation is unchanged: invalid grammar returns `NaN` from `parse` and is passed through by both wrappers; length/type violations throw from `parse`. Fewer than 3 participants, no diagram.

## 10. Edge cases
| # | Case | Expected behaviour | AC |
|---|---|---|---|
| E1 | `'+-1h'`, `'-+1h'` | `NaN` | AC-5 |
| E2 | `'+'` | `NaN` | AC-6 |
| E3 | `'+ 1h'` | `NaN` | AC-7 |
| E4 | `'+100'` (no unit) | see M2 | AC-3 |
| E5 | `'+.5h'`, `'++1h'` | see M3 | AC-4, AC-5 |
| E6 | 101-char string starting with `+` | throws as today; sign counts toward length | AC-9 |
| E7 | `''`, non-string | throws as today | AC-9 |
| E8 | `'10+.5'` (sign mid-string) | `NaN` | AC-8 |
| E9 | `'+1H'`, `'+1 Hours'` (case, space before unit) | same as without `+` | AC-1 |
| E10 | Concurrency / re-runs / partial failure | n/a — pure synchronous function | out of scope — §16 |

## 11. Acceptance criteria (EARS)
| ID | Pattern | Requirement | Cat |
|---|---|---|---|
| AC-1 | Ubiquitous | `parse()` shall return, for `+<n><unit>` (optional spaces before the unit, any letter case), the same number as for `<n><unit>` for every unit alias the parser accepts (`src/index.ts:78`). | C3 |
| AC-2 | Ubiquitous | `parse()` shall return, for `+<d>.<d><unit>` (e.g. `'+1.5h'`), the same number as for `<d>.<d><unit>`. | C3 |
| AC-3 | Ubiquitous | `parse()` shall return, for `+<n>` with no unit, the same number as for `<n>` [NEEDS CLARIFICATION:minor — M2: confirm `'+100'` is `100` — options: valid (consistent with `'100'`) / invalid; impact: one AC and the regex's optional-unit handling]. | C3 |
| AC-4 | Ubiquitous | `parse()` shall return, for `+.<d><unit>`, the same number as for `.<d><unit>` [NEEDS CLARIFICATION:minor — M3: confirm `'+.5h'` is `1800000` — options: valid (consistent with `'-.5h'`) / invalid; impact: regex shape]. | C3 |
| AC-5 | Unwanted | If the string has two or more sign characters before the number (`'+-1h'`, `'-+1h'`, and `'++1h'` [NEEDS CLARIFICATION:minor — M3: confirm `'++1h'`, `'--1h'` invalid]), then `parse()` shall return `NaN`. | C5 |
| AC-6 | Unwanted | If the string is `'+'` alone, then `parse()` shall return `NaN`. | C5 |
| AC-7 | Unwanted | If whitespace separates a sign from the number (`'+ 1h'`), then `parse()` shall return `NaN`. | C5 |
| AC-8 | Unwanted | If `+` appears anywhere other than as the leading character (`'10+.5'`, `'1h+'`), then `parse()` shall return `NaN`. | C5 |
| AC-9 | Ubiquitous | `parse()` shall throw for empty, non-string and over-100-character inputs exactly as before, counting a leading `+` toward the length. | C5 |
| AC-10 | Event-driven | When `ms()` receives a `+`-prefixed string that `parse()` accepts, `ms()` shall return the same number as `parse()`. | C4 |
| AC-11 | Unwanted | If `ms()` or `parseStrict()` receives a string that `parse()` rejects as in AC-5..AC-8, then it shall return `NaN` like `parse()` [NEEDS CLARIFICATION:minor — M1: the request says the public `ms()` "throws the same error it already throws for invalid strings", but `ms('☃')` returns `NaN` today (`src/index.test.ts:51-58`); confirm `NaN` — options: NaN (today's behaviour) / throw (new behaviour); impact: AC-11, E1-E3 tests]. | C5 |
| AC-12 | Ubiquitous | `parseStrict()` shall return, for `+`-prefixed input, the same number as `parse()`. | C4 |
| AC-13 | Ubiquitous | `format()` and `ms(number)` shall never output a leading `+`, for short and `long` modes, for positive, zero and negative input. | C1 |
| AC-14 | Ubiquitous | `parse()` shall keep today's results for every input without a leading `+` (including `-`-prefixed values, bare numbers and `'10-.5'`). | C1 |

## 12. Non-functional requirements
| ID | Requirement | Measure / threshold |
|---|---|---|
| NFR-1 | The package shall add no runtime or dev dependency. | `package.json` dependency fields unchanged |
| NFR-2 | The public API types shall not change. | `dist/index.d.ts` exports and signatures identical (`pnpm attw`/`pnpm typecheck` pass) |
| NFR-3 | The parser shall run in time linear in input length for inputs up to 100 chars. | no nested/ambiguous quantifier added to the regex; review check |
| NFR-4 | The change shall pass in Node and Edge Runtime. | `pnpm test` green (both envs) |
| NFR-5 | The change shall pass lint and typecheck. | `pnpm lint`, `pnpm typecheck` exit 0 |

## 13. Traceability
| Req | Goal / Decision | Module | Area | Verification |
|---|---|---|---|---|
| AC-1..4 | D1, D2 | `src/` | regex `src/index.ts:77-80` | unit, `parse.test.ts`, `parse-strict.test.ts` |
| AC-5..8 | D3 | `src/` | same regex | unit, `parse.test.ts` |
| AC-9 | non-regression | `src/` | `src/index.ts:72-76` | unit, `parse.test.ts` |
| AC-10, AC-11 | D2, D3 | `src/` | `ms` `src/index.ts:54` | unit, `index.test.ts` |
| AC-12 | D2 | `src/` | `src/index.ts:156` | unit, `parse-strict.test.ts` |
| AC-13 | D4 | `src/` | `fmtShort`/`fmtLong` | unit, `format.test.ts` / `index.test.ts` |
| AC-14 | non-regression | `src/` | existing tests | existing unit suites |
| NFR-1..5 | D5 | repo | `package.json`, `src/` | commands in §14, review |

## 14. Verification hints
Documented commands (`package.json:29-37`): `pnpm test` (Node + Edge), `pnpm test:nodejs`, `pnpm typecheck`, `pnpm lint`, `pnpm attw`. No other documentation exists.
- AC-1..AC-14: unit tests in the existing Jest files, named `it('AC-n: …')`. AC-1 should iterate all 8 unit families and their aliases (`src/index.ts:9-16`).
- NFR-2: `pnpm typecheck` plus `pnpm attw`; NFR-3: code review, no automated test.
- Whether a `ms('+1h')` test needs `@ts-expect-error` depends on M4; `implementation-planner` must check with `pnpm typecheck`.

## 15. Open questions
| # | Severity | Question |
|---|---|---|
| M1 | minor | `ms()` on invalid strings: `NaN` (code today) or throw (request wording)? Spec assumes `NaN` per "same as today". |
| M2 | minor | Is `'+100'` (no unit) valid and equal to `100`? |
| M3 | minor | `'+.5h'` valid? `'++1h'` and `'--1h'` invalid? |
| M4 | minor | Does `StringValue` (`src/index.ts:29-32`) already admit `'+1h'` at type level? Not verifiable without running `tsc`; the request says types do not change. |

## 16. Out of scope
Formatting changes; new units; whitespace after sign; concurrency/re-run concerns (pure function); type-level validation of `+` strings.

## 17. Changelog
| Date | Change | Reason / source |
|---|---|---|
| 2026-10-22 | Initial draft | user request with confirmed answers; code read of `src/index.ts` and tests |
