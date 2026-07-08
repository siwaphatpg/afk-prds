# afk-prds

A Claude Code skill that autonomously implements **PRD epics** and their `feat:` sub-issues using **test-driven development**, on a **per-PRD branch** that you review and merge yourself.

> AFK = "away from keyboard" — you kick it off, leave it unattended, and review the results when it's done.

## What it does

Composes two existing skills and leaves both untouched:
- **`/tdd`** — the red-green-refactor practice (single canonical source of TDD rules).
- **`/work-on-issues`** — the tracker/worktree/branch mechanics this adapts.

**Pipeline:** this skill consumes the output of `/to-prd` -> `/to-issues`. A **PRD** issue (title `PRD:` or `PRD` label) is a parent epic; each **`feat:` issue** is a vertical tracer-bullet slice with `## What to build`, `## Acceptance criteria`, `## Blocked by`, and `## Parent`.

## Flow

1. You run `/to-prd` -> `/to-issues` to produce a PRD epic + `feat:` sub-issues.
2. You run `/afk-prds` and **leave it unattended**.
3. For each unblocked PRD, the skill:
   - Creates a worktree `.claude/worktrees/prd-<n>` on a new branch `prd-<n>-<slug>` off the default branch.
   - Sequences the PRD's `feat:` issues by `## Blocked by` (sequential, one at a time).
   - Per feat: derives an interface + behavior list from the PRD's Implementation/Testing Decisions + the feat's acceptance criteria, **posts it as a comment** on the feat issue.
   - Dispatches a `general-purpose` sub-agent that runs **TDD red-green-refactor** (rules read live from `/tdd/SKILL.md`) against the posted plan.
   - Commits each feat: `feat: resolve #<n> — <desc>` on the shared PRD branch.
4. Runs the full test suite once, pushes the branch, **opens an MR** (`Closes #<prd>` + `Closes #<feat-…>`). **No auto-merge, no branch deletion, worktree left intact.**
5. Moves to the next unblocked PRD; stops when none remain.
6. **You** test locally on the worktree branch, then merge the MR on GitHub/GitLab — the tracker auto-closes the PRD + feats. You tell the skill (or it detects the closed PRD) -> it removes the worktree + deletes the local branch.

## Core guarantees

1. **Per-PRD branch** — all feat work for a PRD accumulates on one new branch. Never committed to the default branch.
2. **TDD is the sole quality gate** — no separate find-mismatch pass.
3. **You merge, not the skill** — never auto-merges, never deletes branches, never closes issues. Your manual merge closes them via the MR's `Closes` references.
4. **Fully autonomous** — no interactive planning dialogue; the posted plan replaces `/tdd`'s step 1.
5. **One PRD at a time** — PRDs sequential; within a PRD, feats sequential by dependency.

## Policies

- **HITL handling:** AFK default; skip HITL-marked issues (`hitl` label / `HITL` in body); needs-clarification -> skip with comment.
- **Stuck-RED:** post `blocked` comment, skip it + transitive dependents, continue with independent feats, push a partial MR listing included vs skipped.
- **Cross-PRD deps:** defer the dependent PRD until its blocker PRD is merged; skip with "blocked by unmerged PRD-X, merge and re-run."
- **Trackers:** `gh` (GitHub) and `glab` (GitLab) both supported.

## Install

Copy `skills/afk-prds/` into your Claude Code skills directory (e.g. `~/.claude/skills/afk-prds/` or `~/.agents/skills/afk-prds/`), alongside a `/tdd` and `/work-on-issues` skill (the skill references `../tdd/SKILL.md` and `../work-on-issues/SKILL.md`).

Requires `/to-prd` and `/to-issues` to produce the PRD + feat issues it consumes.

## License

MIT
