# Implementation Plan — parse-leading-plus

Spec: `specs/parse-leading-plus.md` (approved 2026-10-22) · Accepted 2026-10-22 · Execution mode: **single-agent** · M1–M3 confirmed by the user.

## Summary

**The code change is one edit.** In the `parse` regex at `src/index.ts:78`, the sign group changes from `-?` to `[+-]?`. `ms()` and `parseStrict()` both call `parse`, so they pick it up automatically. `parseFloat` already handles a leading `+`, and no types, dependencies or formatting change. Allowing only one optional sign means `'++1h'` and `'--1h'` stay invalid.

| # | Task | File | Covers |
|---|---|---|---|
| T1 | Change the regex sign group from `-?` to `[+-]?` | `src/index.ts` | AC-1–8, 10, 12, 14, NFR-1–3 |
| T2 | Parser tests: one test that loops over every unit and alias (AC-1); `+1.5h`, `+100`, `+.5h`; invalid inputs (`+-1h`, `-+1h`, `++1h`, `--1h`, `+`, `+ 1h`, `10+.5`, `1h+`); a length check with `+` at exactly 100 and 101 chars; existing results unchanged | `src/parse.test.ts` | AC-1–9, 14 |
| T3 | `ms()` returns the same number as `parse()` for `+` input and `NaN` for invalid input; `ms(number)` never outputs a leading `+` | `src/index.test.ts` | AC-10, 11, 13 |
| T4 | `parseStrict()` returns the same as `parse()` for `+` input and `NaN` for invalid input | `src/parse-strict.test.ts` | AC-11, 12 |
| T5 | `format()` never outputs a leading `+`, in short and long mode, for positive, zero and negative values | `src/format.test.ts` | AC-13 |
| T6 | Run `pnpm test` (Node and Edge), `pnpm typecheck`, `pnpm lint` and `pnpm attw`; check the diff shows no `package.json` or type changes | repo | NFR-1–5 |

Tests will be named `it('AC-n: …')`.

**Type annotations in tests (open question M4).** I worked this out from the `StringValue` definition rather than by running the compiler:
- **Valid strings** like `'+1h'` already fit the type, so tests on `ms()` and `parseStrict()` need no `@ts-expect-error` for them.
- **Invalid strings** like `'+-1h'` don't fit, so those tests need `// @ts-expect-error`, the same as the existing `'10-.5'` test.
- **`pnpm typecheck` will catch a wrong guess:** `tsconfig.json` includes `src/*.ts`, so the test files are type-checked.

**Other suggestions from planning:**
- Add `'--1h'` and `'-+1h'` as regression tests, so a too-loose regex like `[+-]*` can't slip through.
- Leave alone the error message that says "between 1 and 99" while the code allows 100. It's an existing mismatch and outside this spec.

## Please confirm (all minor; the plan already assumes these, as the spec does)
- **M1:** `ms()` and `parseStrict()` return `NaN` for invalid strings, as they do today, rather than throwing. Your original request said "throws", but the current code returns `NaN`.
- **M2:** `'+100'` is valid and equals `100`.
- **M3:** `'+.5h'` is valid, and `'++1h'` and `'--1h'` are invalid.

## Execution mode
- **Single-agent (recommended):** one pass through T1, then T2–T5, then T6, with the review steps at the end. The code change is one character, so parallel agents wouldn't save anything.
- **Multi-agent:** T1 first, then T2–T5 in parallel (each touches a different test file), then T6.

Which mode do you want, and do M1–M3 stand? Once you approve, I'll save the plan to `specs/parse-leading-plus.plan.md`. I'm on `main` and won't switch branches or commit without your go-ahead.
