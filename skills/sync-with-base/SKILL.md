---
name: sync-with-base
description: "Merge a PR's base branch into the current feature branch, resolve conflicts without losing work, type-check, then push the current branch only. Use on 'sync my branch with its base', 'catch my branch up', 'my PR is behind its base', 'merge main/dev into my branch', or a branch that fell behind. Detects the base from the branch's PR; never pushes to it."
---

# sync-with-base

Bring the latest changes from the current feature branch's **base branch** into it, resolve conflicts without dropping anyone's work, verify the result compiles, then commit and push **the current branch only**.

The base branch is whatever this branch's PR targets (often `main`, but also `dev`, `staging`, or a release branch). This skill works out which one, rather than assuming.

## The one rule that can never break

**Never push to the base branch.** This skill fetches the base and merges it *into* the current branch. It pushes *the current branch* to its own upstream. It does not push to, force-push to, or otherwise mutate the shared base branch. If you ever find yourself about to run `git push ... <base>`, stop. That is not this skill.

The second rule: **lose no work.** A merge is done only when the intent of *both* sides survives. Textual conflicts are the visible risk; delete/modify conflicts and semantic breakage are the silent ones. Treat all three as data-loss risks.

## Workflow

### 1. Preflight: refuse unsafe starting states

Run these checks before touching anything. If any fails, stop and tell the user what's wrong; don't try to fix it silently.

```bash
git rev-parse --abbrev-ref HEAD        # current branch
git status --porcelain                 # working-tree cleanliness
git remote -v                          # confirm origin exists
```

- **Not a git repo, or no `origin` remote** → stop, report.
- **Working tree is not clean** (`git status --porcelain` prints anything) → **stop and report.** List the uncommitted / untracked files and ask the user to commit or stash them first. Do not merge on top of a dirty tree. A conflict resolution could clobber work git isn't tracking, and that work is unrecoverable. Do not auto-stash unless the user explicitly asks you to.

Record the current branch name. You'll need it for the push, and it's your guard against ever pushing to the base.

### 2. Determine the base branch

Resolve the base in this order and **report which one you picked and how**:

1. **User named one.** If the user said "merge `main`", "sync with `staging`", etc., use that branch. Done.
2. **The branch's PR base.** Ask the PR that this branch is the head of:

   ```bash
   gh pr view --json baseRefName -q .baseRefName
   ```

   A non-empty result is the base. This is the normal path and the whole point of the skill. It merges what the PR actually targets.
3. **No PR yet (or `gh` is unavailable / unauthenticated).** Fall back to the repo's default branch and say so:

   ```bash
   git symbolic-ref --quiet refs/remotes/origin/HEAD   # e.g. refs/remotes/origin/main
   # or, if that is unset:
   gh repo view --json defaultBranchRef -q .defaultBranchRef.name
   ```

   Strip to the bare branch name.

**Never silently assume `dev` (or any fixed name).** If every method above comes up empty, stop and ask the user which branch to merge.

Now guard against a degenerate target:

- **The current branch *is* the resolved base** (you're on `main`/`dev`/`staging` itself) → stop. There is nothing to merge into, and pushing from here would target the shared branch. Ask the user which feature branch they meant.

Call the resolved base `<base>` for the rest of this workflow.

### 3. Fetch the latest base branch

```bash
git fetch origin <base>
```

Fetch only. Do not pull-with-rebase, do not `git pull` (which can pull the wrong tracking branch). You want `origin/<base>` up to date so the merge sees the real latest.

### 4. Merge the base into the current branch

```bash
git merge origin/<base>
```

Three outcomes:

- **"Already up to date"** → the branch already has everything in the base. Nothing to commit, nothing to push. Tell the user and stop.
- **Clean merge (fast-forward or auto-merged)** → git created/moved the commit for you. Skip to step 6 (verify).
- **Conflicts** → git halts mid-merge. Go to step 5. **Never `git merge --abort`** to escape. Resolve it.

### 5. Resolve conflicts without losing data

Invoke the **resolving-merge-conflicts** skill and follow it. It holds the conflict-resolution discipline (read both sides' history and intent, merge semantically, preserve both where possible). This skill adds the data-loss gate on top:

**Preserve both intents by default.** When both sides changed nearby but compatible things, the resolution keeps both. That is not data loss. It is the normal, correct outcome, and you do not need to ask.

**Ask before discarding. This is the user's explicit requirement.** Some conflicts force a genuine either/or where keeping one side means dropping the other's work:

- The two sides changed the *same* lines in incompatible ways and there is no union that preserves both meanings.
- A **delete/modify** conflict: one side deleted a file/function the other side modified. Silently taking "deleted" throws away the modifications; silently taking "modified" resurrects something the other side intentionally removed.
- A rename/edit conflict where honoring one side orphans the other's edits.

For each of these, **do not pick for the user.** Show them, in plain language:
- which file and what the conflict is about,
- what `origin/<base>` did and why (commit message / PR intent),
- what their branch did and why,
- that the two cannot both survive, and which one your resolution would keep and which it would drop.

Then ask which to keep. Wait for the answer before writing the resolution. Losing work is exactly what the user asked to be consulted on.

After resolving, before staging, **show a short summary of every conflicted file and how it was resolved**, so the user can confirm nothing was dropped. Stage resolved files with `git add`.

### 6. Verify the merge compiles

A merge with zero textual conflicts can still break the branch. The base may have renamed or removed a symbol the current branch still uses. Catch that now, not after pushing.

```bash
pnpm type-check
```

- **Green** → continue.
- **Red** → the merge introduced breakage. Fix it at the root (usually adapting the current branch's code to the base's new shape, read the actual change, don't paper over it with `as any` or a null check). Re-run until green. If it stays red after a genuine attempt and you don't understand the type model, stop and hand it back to the user rather than pushing a broken branch.

(If the user asked for the fuller `/done` pipeline instead, run that here.)

### 7. Commit the merge

- If step 4 auto-committed (clean/fast-forward merge), there's nothing to commit, so go to step 8.
- If you resolved conflicts, the merge is staged but not committed. Commit it:

```bash
git commit --no-edit
```

`--no-edit` keeps git's default merge message (`Merge remote-tracking branch 'origin/<base>' into <branch>`), which is the right, conventional message here. If any post-merge fixes from step 6 aren't yet staged, `git add` them first so they land in the same merge commit.

**Commit as the repo's own git identity.** Never add a `Co-Authored-By: Claude` trailer and never mention Claude / AI in the message.

### 8. Push the current branch

Push **the branch you recorded in step 1**, to its own upstream, never the base.

```bash
git push origin HEAD
```

`HEAD` pushes the current branch to the matching remote branch; it cannot accidentally target the base unless you're standing on it, which step 2 already refused. Do not use `--force` / `--force-with-lease`. A merge only adds history, so a normal push fast-forwards. If the remote rejects the push as non-fast-forward, someone else pushed to this branch meanwhile: stop and tell the user rather than forcing.

### 9. Report

Tell the user, concretely:
- which base branch was merged and how it was resolved (PR base, explicit, or default-branch fallback),
- what merged (how many commits from the base, or "fast-forward"),
- which files conflicted and how each was resolved,
- anything you asked them to decide and what they chose,
- that type-check passed,
- that the current branch was pushed (name it), and that the base was untouched.

Per the project's honesty rule: if you skipped a check or only inspected something without running it, say so.

## Quick reference

```
git rev-parse --abbrev-ref HEAD              # 1. record current branch
git status --porcelain                       # 1. must be empty, else STOP
gh pr view --json baseRefName -q .baseRefName  # 2. resolve <base> (PR target)
#   empty -> repo default branch; still empty -> ask. current branch must NOT be <base>
git fetch origin <base>                      # 3. fetch only
git merge origin/<base>                      # 4. merge into current branch
#   conflicts -> resolving-merge-conflicts skill + data-loss gate (5)
pnpm type-check                              # 6. must be green
git commit --no-edit                         # 7. only if conflicts were resolved
git push origin HEAD                         # 8. current branch only, never <base>
```
