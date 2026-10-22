# Workflow retro ledger

An append-only record of `/sdd-engineering:workflow-retro` runs in this repository. Each section covers one session: its metrics, what was learned about each touched module, and the proposals made. Earlier sections are never rewritten. When a proposal is applied, the next section's trend line says so.

## 2026-10-22 · parse-leading-plus · session 39fbfc12 · deep

| Agents | Peak ∥ | Tokens | Cost | Cache hit | Orchestrator share | Signals |
|---|---|---|---|---|---|---|
| 5 (implementer, test-runner, plan-verifier ×2, architecture-reviewer) | 2 | 885k | $0.70 (2 Haiku agents unpriced) | 84% / 74% / 82% / 41% / 60%; orchestrator 96% | 42% of tokens, 47% of priced cost | test-runner#2: 4 tool errors |

**Trend vs previous run**: first run

**Insights by module**
- `src/`: `pnpm test` runs jest twice, in the Node and Edge environments (183 tests each, 100% coverage). `pnpm attw` runs a `tsdown` build first and writes `dist/`, which is gitignored. Type-level tests use `// @ts-expect-error`, and `pnpm typecheck` flags any unused ones. The parse regex is at `src/index.ts:78` and the 100-char input limit is at `:72`. The Bash permission checker blocks `sed -i`, heredoc appends, `( … )` subshell groups and `; echo "$?"` chains, so use Edit and single commands. The repo has no `CLAUDE.md` and no `INSIGHTS.md`, so the agents found the check commands from the plan and `package.json`.

**Proposals**
1. `CLAUDE.md` (new, repo root) → list the check commands (`pnpm typecheck`, `pnpm test` = Node + Edge jest, `pnpm lint` = biome, `pnpm attw` = build + attw) and state that the package is a single module (`src/index.ts`) with no layering rules — status: open
2. sdd-engineering `plan-verifier` (plugin issue) → in `tasks` mode, use the test-runner report and don't re-run typecheck/lint/test — status: open
3. sdd-engineering `test-runner` (plugin issue) → run each documented command as a plain single command, with no `(time …)` wrapper or `; echo "$?"` suffix — status: open
4. sdd-engineering `implementer` (plugin issue) → read each target file once with Read instead of probing it with `sed -n`/`tail` in Bash first, and use Edit, not `sed -i`/heredoc — status: open
5. sdd-engineering `workflow-retro/scripts/prices.json` (plugin issue) → add a `claude-haiku-5-5` entry — status: open
