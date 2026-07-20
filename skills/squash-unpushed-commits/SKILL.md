---
name: squash-unpushed-commits
description: "Squash every local commit that's ahead of the remote tracking branch into a single commit with a given message — no rebase, no force-push, since nothing has been pushed yet. Trigger when the user says things like \"squash my commits into one\", \"combine today's commits\", or \"we haven't pushed these, make them one commit\"."
argument-hint: <COMMIT_MESSAGE>
allowed-tools: Bash
disable-model-invocation: true
---

# Squash Unpushed Commits

Collapse a run of local-only commits sitting on top of the remote tracking branch into one commit. This is the simple case of history cleanup: since none of the commits have been pushed yet, there's no force-push and no shared-history risk — just a soft reset and a single new commit.

If the branch also needs rebasing onto a fresh base, or has already been pushed and needs a force-push, use `clean-branch-history` instead — that skill covers the full destructive-git-surgery workflow with its confirmation gates.

## Task

$ARGUMENTS

If a commit message is given, use it. If not, ask the user for one before committing.

**Never report this skill as complete without having actually run every command below in this turn.** Conversation memory, an earlier session's summary, or a prior report that "the squash already happened" is not evidence of the *current* repo state — always re-verify from scratch, even if you believe you already did this recently.

## Step 1: Verify the working tree is clean

```bash
git status
```

If there are uncommitted changes (staged or unstaged), stop and tell the user — a soft reset would pull those into the squashed commit's diff along with everything else, which is very likely not intended. Ask them to commit or stash first.

## Step 2: Find what's actually unpushed

```bash
git rev-parse --abbrev-ref HEAD
git status -sb   # shows "ahead N" if there's an upstream
git log --oneline @{u}..HEAD
```

If there's no upstream branch configured, ask the user which remote branch to compare against (commonly `origin/main` or `origin/<branch-name>`) and use that in place of `@{u}` for the rest of this workflow.

If the list is empty or has only one commit, tell the user there's nothing to squash and stop.

## Step 3: Squash

```bash
git reset --soft @{u}
git commit -m "<message>"
```

`--soft` rewinds the branch pointer to the upstream commit without touching the working tree or index, so every change from the squashed commits lands staged and ready — then the single new commit captures all of it at once.

## Step 4: Verify the squash actually landed

```bash
git log --oneline @{u}..HEAD
git status -sb
```

Confirm this shows exactly one commit ahead of upstream — the new commit you just made, and nothing else. If it shows more than one, or the commit message/hash don't match what you just committed, stop and investigate before reporting anything to the user — do not report success on the assumption that the commands above worked.

## Step 5: Report

Quote the actual commit hash and "ahead N" count from Step 4's output (not a generic confirmation), and note that since these commits were never pushed, this was a local-only history rewrite — no force-push was needed or performed.
