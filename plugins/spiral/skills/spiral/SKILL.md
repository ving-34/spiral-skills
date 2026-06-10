---
name: spiral
description: Five-phase adversarial development loop (design → plan → architect → implement → evaluate) that re-enters until the evaluator flags GREEN. For features where the bar is "insane quality", not "ship it". Use when the user explicitly invokes /spiral or asks for a multi-pass spiral build.
user_invocable: true
---

# /spiral — Adversarial development spiral

You orchestrate five sub-agents per iteration. The loop repeats until the **Evaluator** returns `VERDICT: GREEN`, capped at 4 iterations. Every phase writes ONE markdown file to `.spiral/<feature>/iter-N/`. The orchestrator's context stays tiny because sub-agents only return a one-line receipt; the substantive output lives in files.

## Core design rules (read first)

These exist because the previous version of this skill stalled on background agents.

1. **Every sub-agent runs in the FOREGROUND.** No `run_in_background`. Foreground agents always return a result; background agents rely on completion notifications that can hang or get lost. We trade a few seconds of orchestrator wait time for determinism.
2. **Sub-agents return ≤ 80-word receipts.** The prompt explicitly says: "Write your output to `<path>`. Reply with ONLY the verdict line plus a 1-sentence summary." The orchestrator NEVER reads the full output back into context — it reads at most the last 2 lines of a file when it needs to make a control-flow decision.
3. **Files are the bus.** Inter-phase data only travels through `.spiral/<feature>/iter-N/*.md`. Sub-agents read the files they need; the orchestrator just passes paths.
4. **The loop is a real while loop in the orchestrator's turn.** Phases run sequentially in one session. No `ScheduleWakeup`, no `Monitor`, no polling. After Phase 5, read the verdict, branch.
5. **Implementation runs inside a worktree.** Phase 4 uses `isolation: "worktree"` so a failed iteration can't leave the main tree broken. The worktree path is recorded; if the iteration ends GREEN, the orchestrator surfaces the worktree path so the user can merge.
6. **Progress is visible.** `TaskCreate` one task per phase per iteration; mark complete as they finish. `mark_chapter` once per iteration. The user can see exactly where the spiral is at any moment without you having to summarize.

## When to invoke

- User explicitly says `/spiral <feature>` or "spiral on …".
- Feature is non-trivial (multi-screen, new system) and the bar is award-grade craft.
- Repo has a dev server + Playwright suite (the Evaluator needs them).

Do NOT use for: single-component polish, copy fixes, bug triage, refactors. Those have their own skills.

## Arguments

`/spiral <feature-slug> [brief sentence]`

If slug missing, ask once via `AskUserQuestion`. Slug becomes `.spiral/<feature-slug>/`. Brief is one sentence of intent — write it to `.spiral/<feature-slug>/brief.md` and never touch it again.

## Layout

```
.spiral/<feature-slug>/
  brief.md
  iter-1/
    01-design.md
    02-plan.md
    03-architecture.md
    04-implementation.md   ← includes "Worktree: <path>" line
    05-evaluation.md       ← ends with VERDICT: GREEN | RED
    screenshots/
  iter-2/  (only if iter-1 RED)
    ...
```

## Phase models

| Phase | Model | Why |
|---|---|---|
| 1 Design | opus | Creative UX synthesis, deep |
| 2 Plan | sonnet | Structured decomposition, cheap |
| 3 Architect | opus | High-stakes, irreversible decisions |
| 4 Implementation | sonnet | Volume work, parallel children |
| 5 Evaluator | opus | Adversarial depth, blind-spot hunting |

Pass `model:` to the Agent tool to lock the tier. The orchestrator stays on whatever model the user invoked.

## Orchestrator turn structure

Inside ONE conversation turn (or a small number of turns the user is happy to leave running), do this:

```
1. Setup
   - Resolve slug, write brief.md, mkdir .spiral/<slug>/iter-1/
   - TaskCreate: one task per phase × expected iterations (start with 5 tasks for iter-1)
   - mark_chapter("Spiral iter-1: <slug>")

2. For iteration N in 1..4:
   - Run Phase 1 (foreground Agent, opus). Wait for ≤80-word receipt.
   - TaskUpdate phase 1 → completed; phase 2 → in_progress
   - Run Phase 2 (foreground Agent, sonnet). If reply contains "DESIGN-GAP", goto Phase 1 of same iter once with the gap appended.
   - TaskUpdate phase 2 → completed; phase 3 → in_progress
   - Run Phase 3 (foreground Agent, opus).
   - TaskUpdate phase 3 → completed; phase 4 → in_progress
   - Run Phase 4 (foreground Agent, sonnet, isolation: "worktree"). Receipt includes worktree path + branch.
   - TaskUpdate phase 4 → completed; phase 5 → in_progress
   - Run Phase 5 (foreground Agent, opus). Receipt is just the VERDICT line.
   - TaskUpdate phase 5 → completed.
   - If receipt contains "VERDICT: GREEN":
       - Surface 3-bullet summary + worktree path to user. STOP.
   - Else (RED):
       - mkdir iter-(N+1)/; TaskCreate 5 more tasks; mark_chapter("Spiral iter-(N+1)")
       - Continue loop.

3. If loop exits without GREEN (4 iterations RED):
   - Read just the score lines from each iter-N/05-evaluation.md (one grep, not full read).
   - Surface trend to user: which axes are stuck? Ask whether to keep iterating, pivot, or ship.
```

The orchestrator NEVER reads a sub-agent's full output. It either reads the agent's receipt text directly, or runs `tail -n 5 .spiral/<slug>/iter-N/05-evaluation.md` to grab the verdict.

## Sub-agent prompt template

Every phase's Agent call follows this shape — copy and fill in:

```
Role: <one sentence>.

Read in this order:
1. <path>
2. <path>
[3. Prior iter-(N-1)/05-evaluation.md — treat findings as REQUIRED corrections]

Write your output to <output-path>. Cap: <N> words.

Required sections (in order):
- ...

Output rubric:
- Be specific. Cite file:line for non-obvious claims.
- No filler ("delightful", "seamless", "modern").

When done, reply with ONLY:
- A 1-sentence summary
- The literal line `WROTE: <output-path>`
[- For Phase 5 also: the literal line `VERDICT: GREEN` or `VERDICT: RED`]

Do NOT include the file's contents in your reply.
```

That last rule is the load-bearing one. It keeps your context from blowing up.

## Phase 1 — Design (opus, foreground)

Role: senior product designer + UX researcher.

Reads: `brief.md`, prior `iter-(N-1)/05-evaluation.md` if exists, `CLAUDE.md`, the closest existing screen (orchestrator names the path).

Writes `iter-N/01-design.md` (≤ 1500 words):
- User flows (every path including failure & empty states)
- States table: idle / loading / empty / partial / success / error-recoverable / error-terminal / disabled
- UX principles (3-5 bullets, specific to this feature)
- UI direction (typography, color, motion, spacing; cite tokens; flag new ones)
- Copy register + ~10 candidate strings (EN + FR if repo is bilingual)
- Open questions (gaps for Planner to resolve)

## Phase 2 — Planner (sonnet, foreground)

Role: tech lead doing work-wave decomposition.

Reads: `iter-N/01-design.md`, prior eval if exists, repo layout from `CLAUDE.md`.

Writes `iter-N/02-plan.md` (≤ 1000 words):
- Design readiness check (per section of 01-design). If under-specified, write `DESIGN-GAP: <what>` at the top and reply with that line. Orchestrator re-runs Phase 1 with the gap appended (max one redo per iteration).
- Work waves (ordered, ≤ 6h each, each ends in a typecheckable + testable state)
- Surface inventory (files created vs modified, paths only)
- Wave dependencies
- Risk list (3-5, ranked, with mitigation)

## Phase 3 — Architect (opus, foreground)

Role: principal engineer choosing patterns for production-grade code.

Reads: `iter-N/01-design.md`, `iter-N/02-plan.md`, prior eval, `CLAUDE.md`, files named in plan's surface inventory.

Writes `iter-N/03-architecture.md` (≤ 1500 words):
- State shape (stores, reducers, persisted state; discriminated unions over flag bags)
- API contract (endpoints touched: method, path, request, response, error shape; Zod at HTTP edges)
- Module boundaries (deep modules vs thin routes; where coupling is acceptable)
- Pattern choices (3-7 non-obvious decisions, each justified in one sentence)
- Failure & resilience (timeouts, retries, idempotency, propagate vs catch)
- Scalability notes (only bits that materially matter)
- Test surfaces (unit / integration / E2E split)

## Phase 4 — Implementation (sonnet, foreground, worktree)

Role: implementation lead. May spawn its own parallel children for independent waves.

Agent call MUST pass `isolation: "worktree"`. The worktree path + branch come back in the agent's result message — capture them and write them into the receipt header.

Reads (this agent and its children): `iter-N/02-plan.md`, `iter-N/03-architecture.md`, `CLAUDE.md`, source files named in plan.

Per wave:
- Independent wave → spawn child Agent (general-purpose, sonnet) with surface list + architecture subsection + hard "don't touch files outside this list" rule. **Children are foreground too.** Run independent children as parallel Agent calls in the same message.
- Dependent wave → run after prior completes.
- After every wave: typecheck the affected package (`cd <pkg> && pnpm exec tsc --noEmit`). Fix RED before moving on. Never `--no-verify`.

Writes `iter-N/04-implementation.md` (≤ 800 words):
- `Worktree: <absolute path>` and `Branch: <branch-name>` on the first two lines
- Waves completed (`[x]` checklist)
- `git diff --stat HEAD` output
- Typecheck status per package
- Deviations from architecture (with reason; empty if none)
- Known follow-ups (out of scope)
- Hand-off notes for evaluator (≤ 5 sentences: how to exercise the feature, what state to seed)

Standalone /spiral: never commits, leaves changes in the worktree for the user to review.
SUB-SPIRAL MODE (running inside /meta-spiral): the OPPOSITE — every iteration's final state MUST be committed on the worktree branch before the receipt goes back. The meta-merge consumes the BRANCH TIP via `git merge`; uncommitted worktree changes are silently dropped (this shipped stubs/stale work to main three times). Commit message: `spiral(<slug>): iter-N implementation`. Exclude `.spiral/` files.

## Phase 5 — Evaluator (opus, foreground)

Role: adversarial critic + QA engineer. Defaults to RED unless evidence proves GREEN.

Reads: `iter-N/01-design.md` … `iter-N/04-implementation.md`, source diffs in the worktree, prior `iter-(N-1)/05-evaluation.md` if exists (verify previously-flagged issues are resolved).

Required actions:
1. Switch to the worktree path from `04-implementation.md`.
2. Start the dev server (use `/run` skill or repo's documented command).
3. Use Playwright MCP to exercise every state from the design's state table — golden path + at least 2 failure paths.
4. Save screenshots to `.spiral/<feature>/iter-N/screenshots/`.
5. Write or extend a Playwright spec under `tests/e2e/specs/`. Run `pnpm test:e2e`.
6. Adversarial critique across four axes — score 0-10 with evidence:
   - **Originality** — one thing no competitor does, or AI-slop pastiche?
   - **Scalability** — at 100× load, what breaks first? file + reason.
   - **Usability** — wet-hands / one-handed / distracted user can recover from every error state?
   - **Functionality** — every state from design actually reachable? Dead ends? Blind spots?

Writes `iter-N/05-evaluation.md` (≤ 2000 words):
- Scores (4 axes, evidence per score)
- Blockers (numbered, each `file:line` + what's wrong + what GREEN looks like)
- AI-slop watch (quote generic / unfelt elements)
- Playwright results (pass/fail per spec; link spec path)
- Screenshots referenced (one bullet per state)
- Final line, alone: `VERDICT: GREEN` or `VERDICT: RED`

GREEN requires: all four axes ≥ 7, zero blockers, Playwright suite green, AI-slop watch empty. Anything less is RED.

Reply to orchestrator with ONLY:
- 1-sentence summary
- `WROTE: .spiral/<slug>/iter-N/05-evaluation.md`
- `VERDICT: GREEN` or `VERDICT: RED`

## Worktree handling on completion

- **GREEN**: surface the worktree path. Tell the user: "Changes are in `<worktree-path>` on branch `<branch>`. Review with `git -C <path> diff main` and merge when ready." Do not auto-merge.
- **RED, continuing**: leave the worktree in place; the next iteration's Phase 4 gets a fresh worktree (a new isolation: worktree call). Prior worktrees stay on disk for the user to inspect — DON'T clean them up automatically; mention them in the final summary.
- **4× RED, stopping**: surface all worktree paths + the score trend.

## Out-of-scope findings → `spawn_task`

When the evaluator catches improvements that are out of scope (dead code, stale docs, security issues in unrelated files), it MAY include a `SPAWN_TASKS:` block at the end of its reply listing them. The orchestrator then calls `mcp__ccd_session__spawn_task` once per item to fire them as side tasks. This keeps the spiral focused while not losing the finds.

## Hard rules

- One file per phase. If a sub-agent writes more or returns the file contents in its reply, re-issue with a tighter prompt — don't accept it.
- Orchestrator never edits source. Only writes `.spiral/<slug>/brief.md` and reads receipts/verdicts.
- No commits unless the user explicitly asks. CLAUDE.md governs commit hygiene. EXCEPTION: in sub-spiral mode (under /meta-spiral) the worktree branch MUST carry committed work — see Phase 4.
- Stage files explicitly when committing — never `git add -A`. (Sub-spiral-mode worktree commits may `add -A` then unstage `.spiral/` — the worktree contains only this spiral's work.)
- Never `--no-verify` past hooks.
- If two consecutive iterations score the same axis < 5, the bet is wrong, not the execution. Stop and surface to the user.
- If a foreground sub-agent appears to hang (no return after a reasonable wall-clock for its task type), don't wait indefinitely — surface it to the user. Foreground SHOULD return; if it doesn't, that's a real failure to escalate, not poll around.

## Definition of done

- `.spiral/<feature>/iter-N/05-evaluation.md` ends with `VERDICT: GREEN`.
- Playwright suite green from a clean run inside the worktree.
- Screenshots for every state in `iter-N/screenshots/`.
- Worktree path surfaced to user; main tree untouched.
- TaskList shows all phase tasks completed.
- User can open `.spiral/<feature>/` and audit every decision back to the brief.
