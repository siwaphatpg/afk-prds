---
name: afk-prds
description: >
  Autonomously implement PRD epics and their feat: sub-issues using TDD, on a
  per-PRD branch you review and merge yourself. Use when the user says "work on
  PRDs", "implement the issues", "run the PRDs autonomously", "leave agents to
  do the issues", or wants to AFK-process the PRD/feat issues produced by
  /to-prd and /to-issues. Processes every unblocked PRD in the tracker, one at a
  time, then stops. Never auto-merges — you test locally and merge on the
  tracker. Supports a split-machine "workhorse" setup (plan on one machine,
  implement on another) via the --workhorse flag.
---

# AFK PRDs

Autonomously implement **PRD epics** and their `feat:` sub-issues using **test-driven development**, on a **per-PRD branch** that you review and merge yourself.

This skill composes two existing skills and leaves both untouched:
- [/tdd](../tdd/SKILL.md) — the red-green-refactor practice (single canonical source of TDD rules).
- [/work-on-issues](../work-on-issues/SKILL.md) — the tracker/worktree/branch mechanics this skill adapts.

**Pipeline assumption:** issues come from [/to-prd](../to-prd/SKILL.md) → [/to-issues](../to-issues/SKILL.md). A **PRD** issue (title `PRD:` or `PRD` label) is a parent epic with Implementation/Testing Decisions. Each **`feat:` issue** is a vertical tracer-bullet slice with `## What to build`, `## Acceptance criteria`, `## Blocked by`, and `## Parent` → the PRD.

## Modes

Issue state lives on the **remote tracker**, so this skill is stateless across machines — every run re-fetches from `gh issue list` / `glab issue list`. Two deployment modes:

- **Single-machine (default):** you plan (`/to-prd`, `/to-issues`), run `/afk-prds`, then test in the worktree on the same machine and merge on the tracker. The worktree stays after the MR opens so you can test in it.
- **Workhorse (`--workhorse`):** you plan on one machine (laptop), a separate **workhorse** machine runs `/afk-prds --workhorse` unattended. The workhorse removes its worktree right after push + open MR (nothing local needs it); the branch stays on the remote. You **fetch the branch on your laptop**, test there, then merge on the tracker. The workhorse deletes its local branch after the PRD is merged (detected on the next run).

Both modes share everything except **worktree cleanup timing** (see Phase 3/4).

### Arguments

```
/afk-prds                # single-machine: process all unblocked PRDs, then stop
/afk-prds 42             # single-machine: process only PRD #42
/afk-prds --workhorse    # workhorse: process all unblocked PRDs, worktree cleaned after MR
/afk-prds --workhorse 42 # workhorse: process only PRD #42
/afk-prds cleanup 42     # after PRD #42's MR is merged: remove worktree + delete local branch (single-machine)
                         # workhorse: worktree already gone; just delete local branch if present
```

### Workhorse prerequisites (in addition to the standard prerequisites)

On the workhorse machine:
- The five skills installed, `gh`/`glab` authenticated (its own token — Machine A's keyring auth does not transfer).
- A clone of the repo with the default branch (worktrees branch off it).
- `/setup-matt-pocock-skills` config — travels via git (committed `docs/agents/*` + `CLAUDE.md`/`AGENTS.md`), so a `git pull` on the workhorse gets it.
- **`.env` files** — gitignored, so they do NOT travel via git. The workhorse must have its own `.env`/`.env.*` for the TDD test suite to run (DB creds, API keys). Without them, red-green fails on anything touching a DB/external service. This is a one-time workhorse setup task.

On your laptop (for the test step): a clone of the repo, `gh`/`glab` authed, and the five skills if you also plan to run `/to-prd`/`/to-issues` there.

## Core guarantees (do not violate)

1. **Per-PRD branch.** All `feat:` issues under one PRD accumulate as commits on a single new branch. One branch + one MR per PRD. Never commit feat work to the default branch.
2. **TDD is the sole quality gate.** Every feat is implemented via red-green-refactor (vertical slices, behavior tests, refactor only while GREEN). There is no separate find-mismatch pass.
3. **You merge, not the skill.** The skill pushes the branch, opens the MR, and stops. It never merges, never deletes the branch before you've merged, and never closes issues — your manual merge closes the PRD + feats via the MR's `Closes` references. Worktree removal is mode-dependent: kept after the MR in single-machine mode (it's your local test checkout); removed after the MR in workhorse mode (you test on your laptop by fetching the pushed branch).
4. **Fully autonomous.** No interactive planning dialogue. The TDD plan (interface + behaviors) is derived from the PRD + feat issue and posted as a comment, replacing /tdd's interactive step 1.
5. **One PRD at a time.** PRDs are processed strictly sequentially; within a PRD, feat issues are processed strictly sequentially by dependency. No parallel branches.

## Detecting the Tracker

Determine the tracker at session start from remotes:

```bash
git remote -v
```

| Remote host | Tracker | CLI | Command prefix |
|-------------|---------|-----|----------------|
| `github.com` | GitHub Issues | `gh` | `gh issue ...` / `gh pr ...` |
| `gitlab.com` | GitLab Issues | `glab` | `glab issue ...` / `glab mr ...` |

Store the resolved CLI alias (`gh` or `glab`) as **`$TRACKER`**. GitLab uses `mr`/`note` where GitHub uses `pr`/`comment`; use platform-specific forms where subcommands differ.

## Detecting the default branch

Do not hardcode `main`. Detect the default branch:

```bash
git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
# fallback:
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's@origin/@@'
```

Branch every PRD off this default branch.

## Workflow

### Phase 1: Fetch & group

1. **List open issues** (machine-readable):

   ```bash
   # GitHub
   gh issue list --repo <repo> --state open --json number,title,labels,body
   # GitLab
   glab issue list -O json
   ```

2. **Identify PRDs** — issues whose title starts with `PRD:` OR carry the `PRD` label. These are parent epics. For each PRD, discover its `feat:` sub-issues via the `## Parent` reference in feat issue bodies (pointing back to the PRD) and/or the PRD body listing `#<n>` references. Build a map:

   ```
   PRDS = {
     #10 (PRD: Auth): { feats: [#11, #12, #13], ... }
     #20 (PRD: Reports): { feats: [#21, #22], ... }
   }
   ```

   Issues that are neither a PRD nor a `feat:` referenced by a PRD are ignored (not presented as pickable work).

3. **Read each PRD** body for Implementation Decisions (modules, interfaces, API contracts, schema) and Testing Decisions (what makes a good test, which modules to test, prior art). These feed the TDD plan derivation.

4. **Parse `## Blocked by`** within each PRD's feat set to build a per-PRD `DEPS` map for ordering. Detect cross-PRD blockers (a feat blocked by a feat belonging to a different PRD) — see Cross-PRD Dependencies.

5. **Parse arguments to set mode + scope:**
   - `--workhorse` present → workhorse mode (affects cleanup timing in Phase 3/4). Default (absent) → single-machine mode.
   - Scope: a PRD number argument → process only that one PRD. No number → process **all unblocked PRDs**, one at a time, then stop.
   - `cleanup <n>` → skip to Phase 4 cleanup for PRD #<n> (do not implement).

### Phase 2: Implement a PRD (one at a time)

For each selected/unblocked PRD, in order:

6. **Create the worktree + per-PRD branch** off the latest remote default branch (fetch first, so a workhorse builds on the planner's latest commits rather than a stale local ref):

   ```bash
   git fetch origin
   mkdir -p .claude/worktrees
   git worktree add .claude/worktrees/prd-<n> -b prd-<n>-<slug> origin/<default>
   ```

   `<slug>` is a kebab slug of the PRD title.

   **Copy environment files into the worktree** (tests need them; a worktree does not inherit `.env`):

   ```bash
   for f in .env .env.*; do [ -f "$f" ] && cp "$f" .claude/worktrees/prd-<n>/; done
   ```

7. **Order the PRD's feat issues** by `DEPS`: repeatedly pick a feat whose blockers (within this PRD) are all done. Skip closed feats and HITL-marked feats (see HITL handling).

8. **Per feat issue** (sequential, one at a time):

   a. **Derive the TDD plan** — from the PRD's Implementation/Testing Decisions + this feat's `## Acceptance criteria`, produce:
      - The **public interface** to build/modify (module boundaries, signatures, API contract, schema).
      - A prioritized **behavior list** (the things to test), drawn from the acceptance criteria, named with the project's domain glossary.

   b. **Validate feasibility.** If the acceptance criteria are ambiguous and a coherent interface + behavior list cannot be derived, **skip** this feat: post a `needs clarification` comment on the feat issue and move to the next independent feat. Do not guess.

   c. **Post the plan as a comment** on the feat issue (your audit trail). Example:

      ```markdown
      ## AFK TDD plan
      Interface: <module/sig/API/schema>
      Behaviors to test:
      1. <behavior>
      2. <behavior>
      ...
      Testing notes: <from PRD Testing Decisions + prior art>
      ```

   d. **Set labels** — remove `needs-triage`/`ready-for-agent`, add `in-progress` (create label first if missing).

   e. **Dispatch a sub-agent** (Agent tool, `subagent_type: "general-purpose"`) into the worktree directory with:
      - The feat issue body + acceptance criteria.
      - The posted TDD plan (interface + behaviors) as the **pre-approved plan**.
      - The **live TDD rules**: the orchestrator reads `../tdd/SKILL.md` and injects its content into the prompt, AND instructs the sub-agent to invoke the `/tdd` skill via the Skill tool if available. The sub-agent **skips /tdd's interactive step 1** (the plan is supplied) and runs steps 2–4: tracer-bullet → incremental red-green-refactor → refactor while GREEN.
      - Working directory = `.claude/worktrees/prd-<n>`, branch already checked out.
      - Constraints: only modify files within this feat's scope; run tests after each cycle; never refactor while RED; keep tests to observable behavior through public interfaces.

      Sub-agent prompt skeleton:

      ```markdown
      You are implementing feat issue #<n>: <title>, part of PRD #<prd>.

      ## Working Directory
      .claude/worktrees/prd-<n>  (branch prd-<n>-<slug> is checked out)

      ## Pre-approved TDD plan (skip /tdd step 1 — plan is supplied)
      Interface: <...>
      Behaviors to test:
      1. <...>
      2. <...>
      Testing notes: <...>

      ## TDD rules (from /tdd/SKILL.md)
      <inject live content of ../tdd/SKILL.md here>
      Also invoke the /tdd skill via the Skill tool if it is available to you.

      ## Requirements
      <feat issue body + acceptance criteria>

      ## Stuck policy
      If you cannot reach GREEN after genuine effort, STOP. Do not commit broken
      code. Return: what you tried, the failing test output, where you're stuck.
      Do not commit partial/failing work.

      ## Output
      Return: what you implemented, tests written, test results, files changed,
      and whether you reached GREEN.
      ```

   f. **Verify** — after the sub-agent returns, confirm GREEN yourself by running the relevant tests in the worktree. Do not trust the claim.

   g. **Handle stuck-RED** (see Stuck-RED policy): if the feat can't reach GREEN, post a `blocked` comment on the feat issue with the failing output, leave it uncommitted, skip it and its transitive dependents, continue with the next independent feat.

   h. **Commit** (only on GREEN) inside the worktree:

      ```bash
      git commit -m "feat: resolve #<n> — <short description>"
      ```

9. **After all processable feats** for the PRD are committed or skipped, **run the full test suite once** in the worktree. If it fails, record the failure in the MR body (do not silently push broken state — investigate; if a skipped feat's absence is the cause, note it).

### Phase 3: Submit (you merge)

10. **Push the branch:**

    ```bash
    git push -u origin prd-<n>-<slug>
    ```

11. **Open the MR/PR**, closing the PRD + all successfully-committed feat issues:

    ```bash
    # GitHub
    gh pr create --repo <repo> --title "PRD #<n>: <title>" --body "<body>"
    # GitLab
    glab mr create --title "PRD #<n>: <title>" --description "<body>"
    ```

    MR body template:

    ```markdown
    Closes #<prd>
    Closes #<feat-1>
    Closes #<feat-2>
    ...
    Closes #<feat-k>

    ## Status
    Implemented: #<feat-1>, #<feat-2>, ...
    Skipped (blocked/needs-clarification): #<feat-x>, ... — see comments on those issues.

    ## Notes
    Tests: <full-suite result>. Review and test locally on branch `prd-<n>-<slug>`.
    ```

    Only list feats that were actually committed in `Closes`. Skipped feats stay open (their `needs clarification`/`blocked` comments explain why).

12. **Set labels** on committed feat issues: remove `in-progress`, add `ready-for-review`. Create labels first if missing.

13. **Do NOT merge. Do NOT delete the branch.** Branch cleanup is deferred to Phase 4 (after you merge). Worktree cleanup depends on mode:

    - **Single-machine mode:** leave `.claude/worktrees/prd-<n>` intact — it is your local test checkout. (You test in the worktree on the same machine.)
    - **Workhorse mode:** remove the worktree now — nothing on the workhorse needs it (you test on your laptop by fetching the pushed branch, not here):
      ```bash
      git worktree remove .claude/worktrees/prd-<n>
      ```
      The local branch `prd-<n>-<slug>` stays until Phase 4 deletes it after merge.

    In both modes, print to the user the laptop-side fetch/test/merge instructions for this PRD:
    ```bash
    # On your laptop:
    git fetch origin prd-<n>-<slug>
    git checkout prd-<n>-<slug>
    # run your tests, then merge the MR on the tracker
    ```

14. **Move to the next unblocked PRD** (Phase 2). When no unblocked PRDs remain, stop and report a summary table:

    | PRD | Branch | MR | Feats done | Feats skipped | Status |
    |-----|--------|----|------------|---------------|--------|
    | #10 | prd-10-auth | !42 | #11,#12,#13 | — | ✅ MR open |
    | #20 | prd-20-reports | !43 | #21 | #22 (blocked) | ⚠️ partial |

### Phase 4: Cleanup (after you merge)

15. The skill does **not** auto-clean during implementation. Cleanup happens when:
    - You tell the skill a PRD's MR is merged (`/afk-prds cleanup <n>`), OR
    - You re-run the skill and it detects the PRD issue is now `closed` on the tracker.

    First verify the PRD issue is actually `closed` on the tracker before cleaning. Then, for that PRD:
    - Propagate any changed `.env`/`.env.*` back to the main working tree (worktrees diverge). Skip this in workhorse mode if the worktree is already gone.

      ```bash
      for f in .env .env.*; do [ -f ".claude/worktrees/prd-<n>/$f" ] && ! diff -q "$f" ".claude/worktrees/prd-<n>/$f" >/dev/null 2>&1 && cp ".claude/worktrees/prd-<n>/$f" "$f"; done
      ```

    - Remove the worktree (single-machine mode only — in workhorse mode it was already removed in Phase 3) and delete the local branch:

      ```bash
      git worktree remove .claude/worktrees/prd-<n> 2>/dev/null  # no-op if already gone (workhorse)
      git branch -d prd-<n>-<slug>                            # fails safely if the worktree still holds it; retry after worktree remove
      ```

    Never delete a local branch whose MR is not yet merged — you may still need to fetch/test/fix it. In single-machine mode, never remove the worktree before merge either — it is your local test checkout.

## Policies

### HITL handling

Treat all feat issues as AFK by default. **Skip** (do not dispatch) any feat issue that carries a HITL marker:
- a `hitl` label, or
- `HITL` / `Type: HITL` in the issue body.

Post a `needs human (HITL)` comment and leave it open. As a fallback, any feat the orchestrator can't autonomously plan is skipped via the `needs clarification` path. Nothing forces a human interaction mid-run.

> Note: `/to-issues` doesn't currently persist the HITL/AFK tag onto the issue, so proactive HITL skipping relies on a `hitl` label or a `HITL`/`Type: HITL` marker in the body. If you want reliable proactive skipping, add a `hitl` label in your `/to-issues` output.

### Stuck-RED policy

When a feat can't reach GREEN:
1. Post a `blocked` comment on the feat issue with the failing test output and what was tried.
2. **Do not commit** partial or failing work.
3. Skip that feat AND every feat transitively blocked by it (re-evaluate `DEPS`).
4. Continue with the next independent feat on the same branch.
5. The PRD's MR lists included vs skipped feats (see MR body template).

If zero feats in a PRD succeed, do not open an empty MR — report the PRD as fully blocked with a summary of the stuck feats.

### Cross-PRD dependencies

A feat in PRD-B that is `## Blocked by` a feat in PRD-A is a cross-PRD dependency. Since PRD-A's branch hasn't merged yet (you merge manually), PRD-B can't build on it:

- If PRD-A is not yet merged on the tracker → **skip the entire PRD-B** with a note: `blocked by unmerged PRD-A; merge PRD-A's MR and re-run`. Process other unblocked PRDs, then stop.
- If PRD-A is merged (its PRD issue is closed) → PRD-B's feat can build off the default branch normally; proceed.

Re-running the skill after you merge PRD-A picks up the now-unblocked PRD-B.

### Sub-agent & TDD delivery

- Sub-agent type: `general-purpose` (reads/writes files, runs shell, commits; can invoke skills if available).
- TDD rules are delivered belt-and-suspenders: the orchestrator reads `../tdd/SKILL.md` **live** and injects its content into the sub-agent prompt, AND tells the sub-agent to invoke `/tdd` via the Skill tool if available. This guarantees the practice reaches the sub-agent regardless of its skill access, with `/tdd` as the single canonical source (no hardcoded duplication).
- The sub-agent uses the posted plan as its pre-approved spec and skips /tdd's interactive step 1.

## Label conventions

| Label | Meaning | Default color |
|-------|---------|---------------|
| `PRD` | Parent epic (tracks sub-issues) | `#0075CA` |
| `in-progress` | Currently being implemented | `#E4E669` |
| `ready-for-agent` | Ready for autonomous agent | `#0E8A16` |
| `ready-for-review` | MR open, awaiting human merge | `#0E8A16` |
| `hitl` | Needs human interaction (skip autonomously) | `#D93F0B` |

`ai-agent-closed` is **not used** — the skill never closes issues; your manual merge does.

Auto-create any missing label before applying (try → on "not found", create with the color above, retry).

## Edge cases

- **No tracker CLI installed** — fall back to API calls or ask the user to install `gh`/`glab`.
- **No PRDs found** — report "No PRD issues in the tracker. Run /to-prd and /to-issues first." Stop.
- **PRD has no feat issues** — skip the PRD, report it has no implementable sub-issues.
- **All of a PRD's feats are HITL/blocked** — do not open an empty MR; report the PRD as fully blocked.
- **Worktree/branch already exists** for a PRD (interrupted run) — resume in place rather than recreating; check for existing commits before continuing.
- **Workhorse mode, no `.env` on the workhorse** — red-green fails on DB/external-service tests. Detect missing `.env` before dispatching the first sub-agent and halt with a clear message: "workhorse missing .env — copy your project's .env to the repo root and re-run" rather than letting every feat fail.
- **Workhorse mode, default branch is behind remote** — `git worktree add -b` off a stale default branch produces a branch missing the planner's recent commits. Before creating the worktree, `git fetch origin` and branch off `origin/<default>` (not the local default ref) so the workhorse builds on the latest remote state.
- **Branch already merged before cleanup** — if you merged the MR on the tracker and the remote branch was deleted, the local branch may still exist; Phase 4's `git branch -d` cleans it up on next run.
- **`.env` files gitignored** — they must be propagated back before worktree removal so secrets/config persist; never commit them.
- **User merges before skill finishes** — fine; the skill detects closed PRD issues and skips/cleans them on re-run.
