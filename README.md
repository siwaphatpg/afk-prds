# afk-prds

A Claude Code skill that autonomously implements **PRD epics** and their `feat:` sub-issues using **test-driven development**, on a **per-PRD branch** that you review and merge yourself.

> AFK = "away from keyboard" — you kick it off, leave it unattended, and review the results when it's done.

This skill is part of the **Matt Pocock engineering skills** ecosystem. It is **not self-contained** — see [Dependencies](#dependencies).

## What it does

Composes two existing skills and leaves both untouched:
- **`/tdd`** — the red-green-refactor practice (single canonical source of TDD rules, read live and injected into the sub-agent prompt).
- **`/work-on-issues`** — the tracker/worktree/branch mechanics this skill adapts.

**Pipeline:** this skill consumes the output of `/to-prd` → `/to-issues`. A **PRD** issue (title `PRD:` or `PRD` label) is a parent epic; each **`feat:` issue** is a vertical tracer-bullet slice with `## What to build`, `## Acceptance criteria`, `## Blocked by`, and `## Parent`.

## Flow

1. You run `/to-prd` → `/to-issues` to produce a PRD epic + `feat:` sub-issues.
2. You run `/afk-prds` and **leave it unattended**.
3. For each unblocked PRD, the skill:
   - Creates a worktree `.claude/worktrees/prd-<n>` on a new branch `prd-<n>-<slug>` off the default branch.
   - Sequences the PRD's `feat:` issues by `## Blocked by` (sequential, one at a time).
   - Per feat: derives an interface + behavior list from the PRD's Implementation/Testing Decisions + the feat's acceptance criteria, **posts it as a comment** on the feat issue.
   - Dispatches a `general-purpose` sub-agent that runs **TDD red-green-refactor** (rules read live from `/tdd/SKILL.md`) against the posted plan.
   - Commits each feat: `feat: resolve #<n> — <desc>` on the shared PRD branch.
4. Runs the full test suite once, pushes the branch, **opens an MR** (`Closes #<prd>` + `Closes #<feat-…>`). **No auto-merge, no branch deletion, worktree left intact.**
5. Moves to the next unblocked PRD; stops when none remain.
6. **You** test locally on the worktree branch, then merge the MR on GitHub/GitLab — the tracker auto-closes the PRD + feats. You tell the skill (or it detects the closed PRD) → it removes the worktree + deletes the local branch.

## Core guarantees

1. **Per-PRD branch** — all feat work for a PRD accumulates on one new branch. Never committed to the default branch.
2. **TDD is the sole quality gate** — no separate find-mismatch pass.
3. **You merge, not the skill** — never auto-merges, never deletes branches, never closes issues. Your manual merge closes them via the MR's `Closes` references.
4. **Fully autonomous** — no interactive planning dialogue; the posted plan replaces `/tdd`'s step 1.
5. **One PRD at a time** — PRDs sequential; within a PRD, feats sequential by dependency.

## Dependencies

This skill reads sibling skills via relative paths (`../<skill>/SKILL.md`) and consumes issues produced by other skills. You must install **all five** skills side-by-side in the same skills directory:

| Skill | Role | Source |
|-------|------|--------|
| `/afk-prds` | this skill — autonomous TDD implementation of PRDs | this repo (`skills/afk-prds/`) |
| `/tdd` | red-green-refactor practice; read live and injected | Matt Pocock's skills |
| `/work-on-issues` | tracker/worktree/branch mechanics | [utarn/review-skill](https://github.com/utarn/review-skill) (`skills/work-on-issues/`) |
| `/to-prd` | produces the PRD epic this skill consumes | Matt Pocock's skills |
| `/to-issues` | produces the `feat:` sub-issues this skill consumes | Matt Pocock's skills |

`/tdd`, `/to-prd`, and `/to-issues` are Matt Pocock engineering skills — obtain them from wherever you source the rest of that skill set (e.g. Matt Pocock's published skill collections). `/work-on-issues` and `find-mismatch` are also in [utarn/review-skill](https://github.com/utarn/review-skill).

## Prerequisites

- **Claude Code** installed and running.
- **The five skills above** installed side-by-side in your skills directory (the `../tdd/SKILL.md` relative paths require the siblings to be siblings on disk).
- **Per-repo setup already done** — run Matt Pocock's `/setup-matt-pocock-skills` in your project so the skills know your issue tracker, triage labels, and domain docs (`CONTEXT.md` / `docs/adr/`).
- **`gh` or `glab` CLI** installed and authenticated to your tracker (GitHub or GitLab).
- **A git repo** with a default branch, hosted on GitHub or GitLab.

## Install

This repo is compatible with [skills.sh](https://github.com/mattpocock/skills) (the `skills` npm CLI) — the same installer `utarn/review-skill` uses. Install with one command:

```
npx skills@latest add siwaphatpg/afk-prds
```

The CLI clones the repo, finds `skills/afk-prds/SKILL.md`, installs it into your skills directory, and symlinks it into Claude Code for you. No plugin manifest or npm publish needed — the `skills/<name>/SKILL.md` layout is all it requires.

Then install the sibling skills the same way:

```
npx skills@latest add mattpocock/skills       # tdd, to-prd, to-issues
npx skills@latest add utarn/review-skill      # work-on-issues (and find-mismatch)
```

After install, the five skills live side-by-side (the `../tdd/SKILL.md` relative paths require them to be siblings on disk, which the CLI handles):

```
skills/
  afk-prds/SKILL.md
  tdd/SKILL.md
  work-on-issues/SKILL.md
  to-prd/SKILL.md
  to-issues/SKILL.md
```

Then:
1. Start a Claude Code session in your project repo.
2. Run `/setup-matt-pocock-skills` once (configures issue tracker, triage labels, domain docs).
3. `/to-prd` → `/to-issues` → `/afk-prds`.

## Usage

```
/afk-prds                # process all unblocked PRDs, one at a time, then stop
/afk-prds 42             # process only PRD #42 (still autonomous)
/afk-prds cleanup 42     # after you've merged PRD #42's MR: clean up its worktree + local branch
/afk-prds --workhorse    # workhorse mode: same, but remove each worktree right after its MR opens
/afk-prds --workhorse 42 # workhorse mode, single PRD
```

Then: test on the branch, merge the MR on your tracker, and the PRD + feats auto-close.

## Single-machine vs workhorse

Issue state lives on the **remote tracker**, so the skill is stateless across machines — every run re-fetches issues from `gh`/`glab`. Two deployment modes:

**Single-machine (default):** you plan, implement, test, and merge all on one machine. The worktree stays after the MR opens so you can test in it.

**Workhorse (`--workhorse`):** split across two machines.

- **Planner machine (e.g. laptop):** runs `/to-prd` → `/to-issues` to create the PRD + `feat:` issues on the remote tracker. (It can also run `/afk-prds` in single-machine mode when you're not splitting.)
- **Workhorse machine (e.g. a beefy desktop or cloud box):** runs `/afk-prds --workhorse` unattended. It fetches the issues the planner created, builds worktrees, implements each PRD with TDD, pushes the per-PRD branch, opens an MR, then **removes the worktree** (nothing on the workhorse needs it). The branch stays on the remote.
- **You, back on the laptop:** fetch the branch, test locally, then merge the MR on the tracker:
  ```
  git fetch origin prd-<n>-<slug>
  git checkout prd-<n>-<slug>
  # run your tests, then merge the MR on GitHub/GitLab
  ```
- After merge, re-run `/afk-prds --workhorse` (or `/afk-prds cleanup <n>` on the workhorse) to delete the now-merged local branch.

**Workhorse setup gotchas:**
- `gh`/`glab` must be authenticated on the workhorse with **its own token** — the planner machine's keyring auth does not transfer.
- The workhorse needs a clone of the repo with the default branch (worktrees branch off `origin/<default>`).
- `/setup-matt-pocock-skills` config is committed to the repo (`docs/agents/*`, `CLAUDE.md`/`AGENTS.md`), so `git pull` on the workhorse gets it.
- **`.env` files are gitignored** and do NOT travel via git. The workhorse must have its own `.env`/`.env.*` (DB creds, API keys) or the TDD test suite fails on anything touching a DB/external service. One-time workhorse setup task.

## Policies

- **HITL handling:** AFK default; skip HITL-marked issues (`hitl` label / `HITL` in body); needs-clarification → skip with comment.
- **Stuck-RED:** post `blocked` comment, skip it + transitive dependents, continue with independent feats, push a partial MR listing included vs skipped.
- **Cross-PRD deps:** defer the dependent PRD until its blocker PRD is merged; skip with "blocked by unmerged PRD-X, merge and re-run."
- **Trackers:** `gh` (GitHub) and `glab` (GitLab) both supported.
- **No `find-mismatch`** — TDD is the sole quality gate.
- **Skill never closes issues** — your manual merge does, via the MR's `Closes` references.

## License

MIT
