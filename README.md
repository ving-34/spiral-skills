# spiral-skills

Two Claude Code skills for award-grade craft:

- **`/spiral`** — adversarial 5-phase development loop (design → plan → architect → implement → evaluate). Re-iterates until a hostile evaluator returns `VERDICT: GREEN`, capped at 4 iterations. Every phase writes ONE markdown file to `.spiral/<feature>/iter-N/`; sub-agents return ≤80-word receipts; the orchestrator's context stays tiny.
- **`/meta-spiral`** — orchestrate N parallel `/spiral` runs in their own worktrees, merge them into an integration worktree, then run a cross-cutting **meta-evaluator** that fails the loop when the *whole* doesn't emerge (even if every part is GREEN). Capped at 3 meta-iterations.

Both are deterministic, foreground-only (no background-agent hangs), and use the filesystem as the inter-phase bus.

---

## Install (Claude Code plugin marketplace)

In any Claude Code session, run:

```
/plugin marketplace add ving-34/spiral-skills
/plugin install spiral@spiral-skills
```

That's it. After install, the slash commands `/spiral` and `/meta-spiral` are available in every project.

To update later: `/plugin marketplace update spiral-skills` then `/plugin install spiral@spiral-skills` again.

To remove: `/plugin uninstall spiral@spiral-skills`.

---

## What you get

### `/spiral <feature-slug> [brief]`

A 5-phase loop run inside one orchestrator turn:

| Phase | Role | Model |
|---|---|---|
| 1 Design | Senior product designer | opus |
| 2 Plan | Tech lead | sonnet |
| 3 Architect | Principal engineer | opus |
| 4 Implementation | Implementation lead (worktree-isolated) | sonnet |
| 5 Evaluator | Adversarial QA (defaults to RED) | opus |

GREEN requires: all four axes (Originality / Scalability / Usability / Functionality) ≥ 7, zero blockers, Playwright suite green, AI-slop watch empty. Anything less, the loop iterates.

### `/meta-spiral <slug> [brief]`

Decompose → fan out N `/spiral` runs in parallel worktrees → merge → meta-evaluate against cross-cutting user journeys → loop. The meta-evaluator scores Originality / Scalability / Usability / Functionality / **Coherence** and runs a "Frankenstein audit" — does it feel like one product or three?

Output lives under `.spiral/meta-<slug>/`. Sub-spiral worktrees stay on disk for forensic audit until you `git worktree remove` them.

---

## Repo prerequisites for the loops to make sense

- A git repository (worktrees are mandatory).
- A dev server and a Playwright test suite — both evaluators drive the running app.
- A `CLAUDE.md` describing repo layout, commit hygiene, and quality gates.

If your repo doesn't have Playwright or a runnable dev server, the evaluator phase has nothing to grade — use `/spiral` only after those are in place.

---

## Layout

```
spiral-skills/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── spiral/
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            ├── spiral/SKILL.md
            └── meta-spiral/SKILL.md
```

This follows the [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) format.

---

## License

MIT — see [LICENSE](./LICENSE).
