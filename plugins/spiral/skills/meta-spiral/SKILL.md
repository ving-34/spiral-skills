---
name: meta-spiral
description: Orchestrate multiple /spiral runs in parallel worktrees, merge them into an integration worktree, then run a cross-cutting adversarial meta-evaluator. Loops over spirals until META-VERDICT is GREEN. For multi-feature products where the bar is "the whole must emerge", not "ship three things". Use when the user invokes /meta-spiral or asks for a multi-feature spiral build.
user_invocable: true
---

# /meta-spiral — Spirals over spirals

You orchestrate **N parallel `/spiral` runs**, each in its own git worktree, then merge them into an integration worktree and run a cross-cutting **Meta-Evaluator**. If META-VERDICT is RED, only the sub-spirals the meta-evaluator names get another spiral pass — others stay frozen. Cap: 3 meta-iterations.

This skill exists for one reason: a single spiral cannot detect seam-level failure. Three features that are each excellent can still ship as Frankenstein. The meta-evaluator's job is to fail the loop when the *whole* doesn't emerge — even if every part is GREEN.

Read `/spiral`'s SKILL.md first; this builds on it directly.

## Core design rules

Same determinism rules as `/spiral`, scaled up:

1. **Every sub-agent — including each `/spiral` instance — runs FOREGROUND.** No `run_in_background`. Parallel sub-spirals are achieved by firing multiple Agent calls in a single tool block; the runtime executes them concurrently and returns when all complete.
2. **Receipts only.** Each `/spiral` Agent returns ≤80 words: worktree path, branch, final VERDICT, path to its evaluation file. The meta-orchestrator never reads the inner spiral's docs back into context.
3. **Files are the bus.** All inter-phase data flows through `.spiral/meta-<slug>/meta-iter-N/*.md` and the individual sub-spiral directories under `.spiral/<sub-slug>/`.
4. **Worktrees are the unit of isolation.** Each sub-spiral has its own worktree (created by its own Phase 4). The integration worktree is a separate, fresh worktree the merge agent creates from `main`.
5. **Cap concurrent sub-spirals at 4.** If the decomposition produces more, batch them into waves. The Agent tool runs parallel calls concurrently, but 10 spirals × 5 phases × 4 iterations is too much fan-out for clean orchestration.
6. **Progress is visible.** `TaskCreate` one task per sub-spiral per meta-iteration plus tasks for Decompose/Merge/Meta-Eval. `mark_chapter` once per meta-iteration.

## When to invoke

- User explicitly says `/meta-spiral <slug>` or "meta-spiral on …".
- The work decomposes into ≥ 2 independent sub-features that must eventually integrate.
- Each sub-feature is non-trivial enough to deserve its own `/spiral` (multi-screen, new system, award-grade craft).
- Repo has a dev server + Playwright suite.

Do NOT use for: a single feature (use `/spiral`), refactors, polish passes, or anything where the sub-features aren't genuinely independent. If sub-features share heavy state or can't be developed in parallel without constant cross-talk, the decomposition is wrong — use one `/spiral` instead.

## Arguments

`/meta-spiral <slug> [brief sentence]`

Slug becomes `.spiral/meta-<slug>/`. The leading `meta-` prefix is the orchestrator's responsibility to add — keeps meta-spirals visually distinct from regular spirals on disk.

## Layout

```
.spiral/meta-<slug>/
  brief.md
  meta-iter-1/
    00-decomposition.md          ← lists sub-spirals + integration plan
    spirals/                      ← one subdir per sub-spiral pointing to its .spiral/<sub-slug>/
      <sub-slug-1>/  → symlink → .spiral/<sub-slug-1>/
      <sub-slug-2>/  → symlink → .spiral/<sub-slug-2>/
    01-merge.md                  ← integration worktree path + merge log
    02-meta-evaluation.md        ← cross-cutting eval, META-VERDICT line
    integration-worktree-path.txt ← absolute path to the integration worktree
  meta-iter-2/  (only if iter-1 RED)
    ...
```

## Meta-iteration model selection

| Step | Sub-agent | Model | Why |
|---|---|---|---|
| A Decompose | general-purpose | opus | Architectural framing of the whole |
| B Sub-spirals (×N) | each invokes `/spiral` | (each spiral uses its own model ladder) | Inherited from /spiral |
| C Merge | general-purpose | sonnet | Mechanical git work + typecheck; cheap |
| D Meta-Eval | general-purpose | opus | Cross-cutting adversarial depth |

## Orchestrator turn structure

```
1. Setup
   - Resolve slug. Write brief.md. mkdir .spiral/meta-<slug>/meta-iter-1/
   - TaskCreate: Decompose, then placeholder tasks (will refine after decomposition).
   - mark_chapter("Meta-spiral iter-1: <slug>")

2. For meta-iter M in 1..3:

   A. Decompose (only on M=1, OR on M>1 if prior meta-eval includes "RE-DECOMPOSE")
      - Foreground Agent, opus.
      - Writes meta-iter-M/00-decomposition.md.
      - Receipt: list of sub-spiral slugs + one-line each.
      - TaskCreate one task per sub-spiral (so user sees what's coming).

   B. Sub-spirals (parallel)
      - Read sub-spiral list from decomposition.
      - For each sub-spiral that is NOT marked "frozen" in the decomposition's status table:
          - Fire ONE foreground Agent (general-purpose, default model) per sub-spiral, IN PARALLEL
            (single tool block, multiple Agent calls).
          - Each agent's prompt is just: `Invoke the /spiral skill with args "<sub-slug> <brief>".
            Read .spiral/meta-<slug>/meta-iter-M/00-decomposition.md first for context.
            Reply with only: worktree path, branch, final VERDICT, path to final 05-evaluation.md.`
      - Cap concurrent at 4. If decomposition has >4 active sub-spirals, run in waves.
      - Each sub-spiral returns its own GREEN/RED. If any returns RED after exhausting its 4
        iterations, escalate to the user — do NOT proceed to merge. Meta-spiral cannot integrate
        a broken sub-feature.

   C. Merge
      - Foreground Agent, sonnet.
      - Reads decomposition + each sub-spiral's 04-implementation.md (for worktree paths and
        branches).
      - Creates fresh integration worktree from main:
        `git worktree add .git-worktrees/meta-<slug>-iter-M main`
      - Checks each sub-spiral worktree for uncommitted work (commits it on the sub-branch) and
        spot-checks branch tips against 04-implementation.md claims BEFORE merging.
      - Cherry-picks or merges each sub-spiral branch into it, in the order from decomposition.
      - On conflict: tries algorithmic resolution per architectural notes in 00-decomposition.md.
        Escalates conflicts that need product judgment.
      - Runs `pnpm -r exec tsc --noEmit` and `pnpm test` in integration worktree.
      - Writes integration-worktree-path.txt and 01-merge.md.
      - Receipt: integration worktree path + build status.

   D. Meta-Evaluate
      - Foreground Agent, opus.
      - cd into the integration worktree.
      - Reads brief.md, 00-decomposition.md, each sub-spiral's final 05-evaluation.md (for context
        on what each sub-spiral promised), 01-merge.md.
      - Starts dev server in integration worktree.
      - Drives Playwright across CROSS-CUTTING flows (journeys that span multiple sub-features).
      - Writes 02-meta-evaluation.md ending with META-VERDICT: GREEN or RED.
      - Receipt: META-VERDICT line + (if RED) `RE-SPIRAL: <slug>, <slug>` line + optional
        `RE-DECOMPOSE` flag.

   E. Branch on verdict
      - META-VERDICT: GREEN → Surface integration worktree path + 3-bullet summary. STOP.
      - META-VERDICT: RED:
          - Parse RE-SPIRAL list from receipt. These sub-spirals will re-run in next meta-iter.
          - Unlisted sub-spirals are "frozen" — they keep their iter-M worktree/branch for next merge.
          - If RE-DECOMPOSE flag present, next meta-iter re-runs phase A.
          - mkdir meta-iter-(M+1)/; TaskCreate; mark_chapter("Meta-spiral iter-(M+1)").
          - Continue loop.

3. If loop exits without GREEN (3 meta-iterations RED):
   - Read just the META-VERDICT + cross-cutting score lines from each meta-iter-M/02-meta-evaluation.md.
   - Surface trend. Ask user: keep iterating, ship partial, or pivot decomposition.
```

The orchestrator never reads a full sub-spiral output, never reads a full meta-evaluation, never edits source. It reads receipts and verdict lines.

## Phase A — Decomposer (opus)

Role: principal product architect deciding the cut lines between sub-features.

Reads:
- `.spiral/meta-<slug>/brief.md`
- Prior `meta-iter-(M-1)/02-meta-evaluation.md` if exists (treat as required corrections).
- `CLAUDE.md`
- Repo layout (skim `apps/` and `packages/`).

Writes `meta-iter-M/00-decomposition.md` (≤ 1500 words):
- **The meta-bet** — the single thing that makes the integrated whole > the sum of parts. One paragraph.
- **Sub-spiral list** — table: slug, one-sentence intent, owner surface (which app/dir), depends-on (other sub-spirals that must finish first or `none`), status (`active` or `frozen` — only set `frozen` on M>1 for sub-spirals the prior meta-eval did NOT name in RE-SPIRAL).
- **Cross-cutting contracts** — shared types, shared API endpoints, shared design tokens that multiple sub-spirals will touch. Explicit ownership: which sub-spiral OWNS the contract; others must consume.
- **Integration plan** — merge order, expected conflict zones, resolution policy per zone.
- **Definition of META-GREEN** — list the 3-5 cross-cutting user journeys that must work end-to-end after merge. The meta-evaluator will Playwright these.

Reply with ONLY: 1-sentence summary + `WROTE: .spiral/meta-<slug>/meta-iter-M/00-decomposition.md` + bullet list of sub-spiral slugs (no descriptions).

## Phase B — Parallel sub-spirals

The orchestrator fires N foreground Agent calls in a single tool block. Each agent's full prompt:

```
You are a sub-orchestrator for ONE sub-spiral inside a meta-spiral.

1. Read .spiral/meta-<META_SLUG>/meta-iter-<M>/00-decomposition.md for full context, especially:
   - Your sub-spiral's row in the table (slug: <SUB_SLUG>)
   - Cross-cutting contracts you must honor
   - Whether you OWN any contract (build it first) or CONSUME one (read it from another sub-spiral's docs)

2. Invoke the /spiral skill with args: "<SUB_SLUG> <one-line intent from the table>"

3. Let /spiral run to completion (up to 4 iterations). This is SUB-SPIRAL MODE: the final
   state MUST be committed on the worktree branch (`spiral(<SUB_SLUG>): iter-N implementation`,
   no `.spiral/` files). The meta-merge consumes the branch TIP — uncommitted worktree changes
   are silently lost. Before replying, verify `git -C <worktree> status --porcelain` is clean.

4. Reply with ONLY:
   - Worktree path
   - Branch name
   - Final VERDICT (GREEN or EXHAUSTED-RED)
   - Path to final 05-evaluation.md
   - Tip check: CLEAN (or what you had to commit)

Do NOT summarize the work. Do NOT include scores. The meta-evaluator will read the files directly.
```

Hard rules:
- Sub-spiral Agent calls run with no `isolation` flag at the meta level — `/spiral` itself uses `isolation: "worktree"` in its Phase 4, which is where the actual file changes land.
- The meta-orchestrator does NOT inspect a sub-spiral's intermediate iterations. It only sees the final receipt.
- If a sub-spiral returns EXHAUSTED-RED (hit its 4-iteration cap without GREEN), STOP the meta-spiral before merging. Surface the failing sub-spiral's evaluation path to the user and ask whether to lower the bar, redecompose, or abandon.

## Phase C — Merge (sonnet)

Role: integration engineer doing mechanical git work.

Reads:
- `00-decomposition.md` (especially: integration plan, merge order, conflict policy)
- Each sub-spiral's `04-implementation.md` (for worktree path + branch — the meta-orchestrator passes paths in the prompt)

Actions:
0. **Tip-integrity check (MANDATORY — three real incidents shipped stale tips).** For EACH sub-spiral worktree, run `git -C <worktree> status --porcelain`. `git merge` consumes the BRANCH TIP; anything uncommitted in the worktree is silently dropped. If dirty: commit everything on that worktree's branch first — `git -C <worktree> add -A` (then unstage any `.spiral/` files), `git -C <worktree> commit -m "spiral(<sub-slug>): iter-N implementation"`. Then spot-check the tip against the claims: pick 2-3 files named in that sub-spiral's `04-implementation.md` and confirm they exist on the branch (`git -C <worktree> show <branch>:<path>` succeeds, and stubs the doc says were replaced are actually gone). A mismatch means the sub-spiral lied or forgot to land work — STOP and report `MERGE-BLOCKED: <sub-slug> branch tip missing claimed work` rather than merging a stale tip.
1. Create integration worktree: `git worktree add .git-worktrees/meta-<slug>-iter-M main` (add `-b meta-<slug>-iter-M` — main is usually checked out in the primary worktree, so the integration worktree needs its own branch).
2. cd into it.
3. For each sub-spiral in merge order: `git merge <sub-branch> --no-ff -m "meta-spiral: merge <sub-slug>"`. Use `--no-ff` so the integration history shows the structure.
4. On conflict:
   - Check `00-decomposition.md` Cross-cutting contracts table — if the conflict is in a file the decomposition assigned to a specific owner, take the owner's version.
   - For other conflicts: attempt 3-way resolution using the architectural notes from each sub-spiral's `03-architecture.md`.
   - If a conflict requires product judgment (e.g., two sub-spirals introduce competing UX patterns for the same surface), abort the merge and write `01-merge.md` with `MERGE-BLOCKED: <reason>` and STOP. The meta-orchestrator escalates to the user.
5. After all merges: run `pnpm -r exec tsc --noEmit` and `pnpm test`. Fix import collisions and obvious post-merge breakage; do NOT add new features.
6. **Commit every post-merge fix on the integration branch before finishing.** The integration worktree's value IS its branch — the user (or a later session) merges `meta-<slug>-iter-M` into main, and uncommitted fixes evaporate at that point (real incident: stub→real-package swaps done in the integration worktree but never committed shipped stubs to main). Final check: `git status --porcelain` in the integration worktree must be empty (ignoring `.spiral/`).
7. Write absolute integration worktree path to `integration-worktree-path.txt` (single line, no other content).

Writes `01-merge.md` (≤ 600 words):
- Worktree path + branch
- Tip-integrity log (one bullet per sub-spiral: tip clean / committed N leftover files / BLOCKED-missing-work)
- Merge log (one bullet per sub-spiral: clean / N conflicts resolved)
- Conflict resolutions (file:line + which side won + why)
- Typecheck status per package
- Test status
- Post-merge fixes applied (be honest; if you wrote new code, list it)

Reply with ONLY: 1-sentence summary + `WROTE: .spiral/meta-<slug>/meta-iter-M/01-merge.md` + the integration worktree path.

## Phase D — Meta-Evaluator (opus)

Role: adversarial critic for the integrated whole. Defaults to RED unless the cross-cutting bar is met.

Reads:
- `brief.md`
- `meta-iter-M/00-decomposition.md` (especially the META-GREEN journeys + meta-bet)
- Each sub-spiral's `iter-<final>/05-evaluation.md` (for what each sub-spiral promised)
- `01-merge.md`
- `integration-worktree-path.txt`

Required actions:
1. cd into integration worktree.
2. Start dev server. Run Playwright against each META-GREEN journey from the decomposition.
3. Take cross-cutting screenshots to `.spiral/meta-<slug>/meta-iter-M/screenshots/`.
4. Adversarial critique on FIVE axes — score 0-10 with evidence (4 from /spiral + 1 cross-cutting):
   - **Originality (whole)** — does the integrated product do one thing no competitor does? Is the meta-bet from decomposition actually realized?
   - **Scalability (integration)** — at 100× load, do the seams break first?
   - **Usability (cross-feature)** — a user moving across sub-features: do they feel one product or three?
   - **Functionality (end-to-end)** — every META-GREEN journey actually works on a clean install?
   - **Coherence** — design language, copy register, motion language consistent across surfaces? Cite specific drift.

Writes `02-meta-evaluation.md` (≤ 2500 words):
- Scores (5 axes, evidence each)
- Meta-bet realization check (the decomposition claimed X — did we deliver X?)
- Cross-cutting blockers (numbered, file:line, what GREEN looks like)
- Frankenstein audit — explicit list of seams that show. Quote them.
- Playwright results for META-GREEN journeys (pass/fail per journey)
- Screenshots referenced (one bullet per journey)
- `RE-SPIRAL: <sub-slug>[, <sub-slug>]` line if RED — only the sub-spirals that need another pass. Each gets a one-paragraph focus brief written into their own `.spiral/<sub-slug>/feedback-from-meta-iter-M.md`.
- Optional `RE-DECOMPOSE` flag if the cut itself was wrong.
- Final line, alone: `META-VERDICT: GREEN` or `META-VERDICT: RED`

META-GREEN requires: all five axes ≥ 7, zero cross-cutting blockers, Frankenstein audit empty, all META-GREEN journeys Playwright-green. Anything less is RED.

Reply with ONLY:
- 1-sentence summary
- `WROTE: .spiral/meta-<slug>/meta-iter-M/02-meta-evaluation.md`
- `META-VERDICT: GREEN` or `META-VERDICT: RED`
- (If RED) `RE-SPIRAL: <slugs>` line, and optional `RE-DECOMPOSE` line

## Re-spiral on RED (how sub-spirals consume meta-feedback)

When the meta-evaluator names a sub-spiral in `RE-SPIRAL:`, the meta-orchestrator:

1. Writes the focus brief to `.spiral/<sub-slug>/feedback-from-meta-iter-M.md`.
2. In the next meta-iteration's Phase B, the sub-spiral Agent's prompt gains one line:
   `Before invoking /spiral, read .spiral/<sub-slug>/feedback-from-meta-iter-M.md. /spiral's Phase 1 will treat this as required input alongside any prior 05-evaluation.md.`
3. `/spiral`'s Phase 1 (Design) is already designed to read corrections — the feedback file slots in as another input.

Unnamed sub-spirals are "frozen": their worktree from the prior meta-iter is reused at merge time. They do not re-spiral.

## Worktree hygiene

- Each sub-spiral's Phase 4 creates a worktree under `.git-worktrees/spiral-<sub-slug>-iter-<n>/`.
- The integration worktree lives at `.git-worktrees/meta-<slug>-iter-<M>/`.
- DO NOT auto-clean worktrees between meta-iterations. They are forensic evidence. Surface their paths in the final summary so the user can `git worktree list` and prune at will.
- On final GREEN: surface only the integration worktree path. Tell the user: "Review with `git -C <path> diff main`, then merge to main when satisfied. Sub-spiral worktrees stay on disk for audit — prune with `git worktree remove` when done."

## Out-of-scope findings

Both `/spiral` evaluators and the meta-evaluator can include `SPAWN_TASKS:` blocks. The meta-orchestrator collects them all (from its own meta-eval + by reading the final-line `SPAWN_TASKS:` of each sub-spiral's evaluation file via `tail`) and fires them via `mcp__ccd_session__spawn_task` after the final summary, NOT during the loop.

## Hard rules

- One file per phase. If a sub-agent returns file contents in its reply, re-issue with a tighter prompt.
- Meta-orchestrator never edits source. Only writes `.spiral/meta-<slug>/brief.md` and reads receipts/verdicts/paths.
- No commits to main (or any user branch) unless the user explicitly asks. CLAUDE.md governs commit hygiene. EXCEPTION — spiral worktree branches and the integration branch REQUIRE commits: `git merge` consumes branch tips, so work left uncommitted in a worktree silently vanishes at merge time (this shipped stubs to main three times). Phase C's tip-integrity check (step 0) and clean-tree check (step 6) are mandatory, not optional hygiene.
- Never `--no-verify` past hooks.
- Cap parallel sub-spirals at 4 per wave. Cap meta-iterations at 3.
- If a sub-spiral exhausts its 4-iteration cap with RED, STOP the meta-spiral — do not merge a broken sub-feature.
- If two consecutive meta-iterations score the same cross-cutting axis < 5, the decomposition is wrong, not the execution. Stop and surface to the user with a recommendation to `RE-DECOMPOSE` or change the brief.
- If a foreground Agent appears to hang past a reasonable wall-clock for its task type, surface it. Foreground SHOULD return.

## Definition of done

- `.spiral/meta-<slug>/meta-iter-M/02-meta-evaluation.md` ends with `META-VERDICT: GREEN`.
- All META-GREEN journeys from `00-decomposition.md` pass Playwright in the integration worktree from a clean run.
- Integration worktree path surfaced to user; main tree untouched.
- All sub-spiral final evaluations also ended GREEN.
- TaskList shows all meta-phase and sub-spiral tasks completed.
- User can `cd <integration-worktree-path>` and see the merged code; user can open `.spiral/meta-<slug>/` and audit every cross-cutting decision back to the brief.
