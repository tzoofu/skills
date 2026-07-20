---
name: clean-branch-history
description: Untangle a contaminated or stale feature branch into a single clean commit on a fresh base — diagnoses commit pollution, squashes history, rebuilds on the latest base branch, and verifies the diff before a safe force-push. Trigger when the user says "this branch is a mess", "clean up my commits", "squash this branch", or notices unrelated changes mixed into a commit.
argument-hint: [branch-name] [base-branch]
allowed-tools: Bash, Read, Edit, AskUserQuestion
disable-model-invocation: true
---

# Clean Branch History

Turn a messy branch — one with unrelated changes accidentally mixed into a commit, or one that's gone stale relative to its base — into a single clean commit sitting on a fresh base. This is git surgery: every step re-verifies before proceeding, and the two destructive operations (hard reset, force-push) require explicit user confirmation first.

## Task

$ARGUMENTS

If a branch name is given, operate on it (checking it out first if not already current). If a base branch is given, use it as the target (default: `dev`, falling back to `main`/`master` if `dev` doesn't exist). If no arguments are given, operate on the current branch against its likely base.

## Step 1: Establish the true scope

Don't trust `git diff <base>..HEAD` blindly if the branch might be stale — a stale base makes an unrelated multi-commit gap look like part of your diff. First check divergence:

```bash
git log --oneline HEAD..origin/<base> | wc -l   # how far behind
git log --oneline origin/<base>..HEAD | wc -l   # how far ahead — should match your real commit count
```

If "ahead" doesn't match the number of commits you actually intended to make, something's already mixed in. Identify the commit(s) that make up your actual work:

```bash
git log --oneline -10
```

## Step 2: Diagnose contamination

For each commit that's yours, diff it against its **true parent** (not the stale base):

```bash
git show <commit-sha> --stat
git show <commit-sha> -- <suspect-file>
```

Look for hunks that don't match what you actually changed — renamed keys, removed lines, files you never touched. Cross-check against the target base to see what's actually there:

```bash
git show origin/<base>:<file> | grep -n "<suspect-symbol>"
```

This tells you definitively whether content in your commit belongs to your fix or leaked in from other work sitting in the same working tree.

## Step 3: Fix content surgically

Edit the contaminated files directly in the working tree to remove what doesn't belong — restore any values your commit shouldn't have touched, keep only your actual additions. After editing, re-verify with a grep for the contaminating symbol:

```bash
git diff | grep -i "<contaminating-symbol>"
```

Repeat until empty. Watch for a subtler issue: restoring a removed line at the *wrong position* in the file still shows as a spurious delete+add pair in diffs even though the content is identical — move it back to its exact original position (check line-adjacency against the base) to get a fully silent diff.

## Step 4: Protect other in-progress work

Before any destructive git operation, check for uncommitted changes that are NOT part of what you're cleaning up:

```bash
git status --short
```

If there's unrelated WIP sitting in the tree (common when multiple pieces of work share a working directory), stash it — **with `-u`** to catch untracked files too:

```bash
git stash push -u -m "unrelated WIP — preserved before branch cleanup"
```

Never proceed to Step 5 or 6 with unrelated uncommitted changes still in the tree.

## Step 5: Squash into one clean commit

If your fix is spread across multiple commits (the original + fixup commits), collapse them:

```bash
git reset --soft HEAD~<N>     # N = number of commits to squash
git commit -m "<accurate message>"
```

`--soft` moves the branch pointer back without touching the working tree or index — all the combined changes land in staging, ready to be re-committed as one. Verify the result matches intent:

```bash
git show --stat HEAD
git diff HEAD^..HEAD | grep -i "<contaminating-symbol>"   # should be empty
```

## Step 6: Rebuild on a fresh base (if the branch was stale)

**Ask the user for explicit confirmation before this step** — it rewrites the branch's commit history.

```bash
git fetch origin <base>
git reset --hard origin/<base>
git cherry-pick <clean-commit-sha>
```

`reset --hard` snaps the branch to match the base exactly (safe now — Step 4 already protected anything uncommitted). The cherry-pick replays your one clean commit on top; if the base picked up unrelated changes to the same files since your branch was cut, cherry-pick auto-merges them as long as your commit's diff is genuinely self-contained.

## Step 7: Verify against the fresh base

```bash
git diff origin/<base>..HEAD --stat                              # should list only your intended files
git diff origin/<base>..HEAD | grep -iE "<contaminating-symbol>" # should be empty
```

If either check surfaces anything unexpected, go back to Step 2 — don't push.

## Step 8: Restore protected work

```bash
git stash pop
```

Confirm it applied cleanly with `git status --short`.

## Step 9: Push safely

**Ask the user for explicit confirmation before this step** if the branch was already pushed — it force-rewrites shared history.

```bash
git push --force-with-lease origin <branch-name>
```

Always `--force-with-lease`, never bare `--force` — it refuses to push if someone else updated the remote branch since your last fetch, so you can't silently clobber someone else's work.

## Step 10: Report

Summarize what was cleaned:
- What was contaminated/stale, and how it was identified
- The final commit SHA and what it contains (file count, one-line description)
- Confirmation that the diff against the base is exactly the intended change
- Whether anything was stashed/restored, and whether the push happened
