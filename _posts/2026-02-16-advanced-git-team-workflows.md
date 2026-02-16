---
title: "Advanced Git for Teams: Conflicts, Recovery, Stash, Bisect, and Beyond"
description: Master Git collaboration — resolve merge and rebase conflicts, recover lost work with reflog, stash and worktrees, cherry-pick, bisect bugs, squash commits, and tag releases like a pro.
author: Vaibhav Gagneja
date: 2026-02-16 23:00:00 +0530
categories: [Development, Git]
tags: [git, github, version-control, advanced, team-workflow, conflicts, rebase, stash, bisect, cherry-pick]
toc: true
image:
  path: https://images.unsplash.com/photo-1556075798-4825dfaaf498?q=80&w=870&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

You know `git add`, `git commit`, and `git push`. You can branch, merge, and rebase. Congratulations — you can work alone. But the moment you join a team of two, five, or five hundred developers, solo Git knowledge crumbles under the weight of **conflicting changes**, **lost commits**, **force-push mishaps**, and **"who broke the build?"** investigations.

This guide is the second half of the Git story — the half that lets you survive (and thrive) in a real-world engineering team. Whether you work at a scrappy startup or an enterprise behemoth, the workflows below will save you hours of frustration and earn you the quiet respect of your teammates.

> **Prerequisites:** You should be comfortable with basic Git operations — staging, committing, pushing, pulling, branching, merging, rebasing, and reading `git log`. If any of those sound unfamiliar, start with [Git Mastery: The Complete Guide](/posts/git-complete-guide/) first.

---

## 1. Forking — Your Own Copy of the World

### What Is a Fork?

A **fork** is a server-side copy of a repository. It's not a Git command — it's a feature provided by hosting platforms like GitHub, GitLab, and Bitbucket. When you fork a repository, the platform:

1. Creates a full copy of the repo under your account.
2. Records metadata linking your copy back to the original ("upstream") repo.

```
┌────────────────────────────────────────────────────┐
│                     FORKING                         │
├────────────────────────────────────────────────────┤
│                                                     │
│  bootdotdev/megacorp  ──fork──►  you/megacorp      │
│       (upstream)                   (origin)         │
│                                                     │
│  You can push to origin freely.                    │
│  You contribute to upstream via Pull Requests.     │
│                                                     │
└────────────────────────────────────────────────────┘
```

### Why Fork Instead of Clone?

| Approach | When to Use |
|----------|-------------|
| **Clone** | You have write access (your own repo or your team's repo) |
| **Fork + Clone** | You're an outside contributor (open-source, another team's repo) |

Cloning gives you a local copy. Forking gives you a **remote** copy you own — and that's where you push your branches before opening a Pull Request back to the original.

### The Fork Workflow

```bash
# 1. Fork via GitHub UI (click the Fork button)

# 2. Clone YOUR fork
git clone https://github.com/YOUR_USERNAME/megacorp.git
cd megacorp

# 3. Create a feature branch
git switch -c add_contrib

# 4. Make changes, commit, and push to YOUR fork
echo "https://github.com/YOUR_USERNAME" > contributors/YOUR_USERNAME.txt
git add .
git commit -m "add myself as a contributor"
git push origin add_contrib

# 5. Open a Pull Request from YOUR fork → upstream's main
```

> **Key Point:** You never push directly to the upstream repo. Your fork is the staging area, and Pull Requests are the gates.

### Cleaning Up After a PR

Once the PR is merged (or closed), delete the feature branch locally:

```bash
git switch main
git branch -D add_contrib   # -D = --delete --force
```

The `-D` flag force-deletes a branch even if it hasn't been merged into the current branch. Use `-d` (lowercase) for a safe delete that refuses to remove unmerged branches.

---

## 2. HEAD — Where You Are Right Now

In the Git world, **HEAD** is simply a pointer to the branch (or commit) you currently have checked out. Think of it as a "You Are Here" marker on a map.

```bash
# See where HEAD points
cat .git/HEAD
# Output: ref: refs/heads/main
```

When you're on `main`, HEAD points to `main`, which in turn points to the latest commit on that branch. When you switch branches, HEAD follows.

```
┌────────────────────────────────────────────────────┐
│                    HEAD                             │
├────────────────────────────────────────────────────┤
│                                                     │
│  HEAD ──► main ──► commit abc123                   │
│                                                     │
│  After `git switch feature`:                       │
│  HEAD ──► feature ──► commit def456                │
│                                                     │
└────────────────────────────────────────────────────┘
```

### Detached HEAD

If you checkout a specific commit (not a branch), HEAD points directly to the commit instead of a branch. This is called **"detached HEAD" state**. It happens naturally during operations like `git rebase` when there are conflicts.

```bash
git checkout abc1234   # Detached HEAD
git switch main        # Back to normal
```

> **Warning:** Commits made in detached HEAD state will be garbage-collected if you don't attach them to a branch. If you realize you've been working in detached HEAD, create a branch immediately: `git switch -c rescue-branch`.

---

## 3. Reflog — Your Safety Net

The **reflog** ("reference log") records every time HEAD or a branch reference moves. While `git log` shows the commit history of a branch, `git reflog` shows the *movement history* of HEAD itself.

```bash
git reflog
```

Output looks like this:

```
abc1234 HEAD@{0}: commit: B: slander
def5678 HEAD@{1}: checkout: moving from main to slander
ghi9012 HEAD@{2}: commit: previous work
```

### Reflog Notation

| Notation | Meaning |
|----------|---------|
| `HEAD@{0}` | Where HEAD is **now** |
| `HEAD@{1}` | Where HEAD was **1 move ago** |
| `HEAD@{n}` | Where HEAD was **n moves ago** |

### Why This Matters

The reflog is your safety net. Even if you:

- Delete a branch
- Do a `git reset --hard`
- Mess up a rebase

...the **commits are still there** for a while (typically 90 days). The reflog tells you where to find them.

---

## 4. Recovering Lost Work

Deleted a branch that had a unique commit? Don't panic. The commit still exists in Git's object database — it's just "unreachable" from any branch. The reflog remembers where it was.

### Recovery via Git Internals (The Hard Way)

```bash
# 1. Find the lost commit
git reflog
# Look for the commit message, e.g., HEAD@{3}: commit: B: slander

# 2. Inspect the commit object
git cat-file -p <commit-hash>

# 3. Inspect the tree to find your file's blob
git cat-file -p <tree-hash>

# 4. Dump the file contents
git cat-file -p <blob-hash> > recovered_file.md
```

This works, but it's painful for anything beyond a single file.

### Recovery via Merge (The Smart Way)

Since `git merge` accepts a "commitish" (anything that resolves to a commit — a hash, a branch name, a tag, a reflog entry), you can do this:

```bash
# Merge the lost commit directly from the reflog
git merge HEAD@{3}
```

One command instead of three `cat-file` calls.

> **Key Point:** Almost nothing is truly lost in Git. As long as the reflog entry exists, you can recover it. The reflog is pruned after ~90 days, so don't rely on it forever.

---

## 5. Merge Conflicts — When Two Worlds Collide

A **merge conflict** happens when two commits modify the **same line(s)** of the same file independently — they don't have a parent-child relationship, so Git can't decide which change to keep.

```
    C     feature (changed line 5)
  /
A - B     main    (also changed line 5)
```

Conflicts are **not errors**. They're Git telling you: *"I found overlapping changes and I need a human to decide."*

### Anatomy of a Conflict Marker

When a conflict occurs, Git writes conflict markers directly into the file:

```
<<<<<<< HEAD
count = 1
count += 1
=======
count = 2
count++
>>>>>>> main
```

| Marker | Meaning |
|--------|---------|
| `<<<<<<< HEAD` | Start of **your** branch's version ("ours") |
| `=======` | Divider between the two versions |
| `>>>>>>> main` | End of **their** branch's version ("theirs") |

### Resolving a Merge Conflict

1. **Open the file** in your editor and inspect the conflict markers.
2. **Decide what to keep**: their changes, your changes, or both.
3. **Remove all conflict markers** (`<<<<<<<`, `=======`, `>>>>>>>`).
4. **Save the file**, then stage and commit:

```bash
git add <conflicted-file>
git commit -m "Resolve merge conflict in <file>"
```

### Practical Example

Say two developers edit `customers/all.csv`:

**Your branch** (`add_customers`) adds:
```
karson,yummy,intercooler,ceo
```

**Main branch** (committed by Greg) adds:
```
jayson,gross,htmz,contributor
```

After merging main into your branch, you see:

```csv
first_name,last_name,company,title
<<<<<<< HEAD
karson,yummy,intercooler,ceo
=======
jayson,gross,htmz,contributor
>>>>>>> main
```

To keep **both** records, edit the file to:

```csv
first_name,last_name,company,title
karson,yummy,intercooler,ceo
jayson,gross,htmz,contributor
```

Then commit with a meaningful message:

```bash
git add customers/all.csv
git commit -m "E: resolve conflict — keep both customer records"
```

> **Note:** Git will let you commit conflict markers accidentally. Always double-check with `git diff --staged` or search for `<<<<<<<` before committing.

---

## 6. Ours vs. Theirs — Automated Conflict Resolution

Manually editing conflict markers is fine for one or two files, but when you have dozens of conflicting files and you *know* which side should win, Git's `--ours` and `--theirs` flags are a lifesaver.

### During a Merge

```bash
# You're on feature_branch, merging main into it
git merge main

# Conflict! Keep the version from main for one file:
git checkout --theirs customers/all.csv

# Keep YOUR version for another file:
git checkout --ours orgs/partners.txt

# Stage and commit
git add .
git commit -m "H: resolve conflicts — keep theirs for customers, ours for partners"
```

| Flag | During Merge | During Rebase |
|------|-------------|--------------|
| `--ours` | The branch you're **on** (merging into) | The branch you're rebasing **onto** |
| `--theirs` | The branch being **merged** | The branch being **rebased** (your feature branch) |

> **Warning:** Notice the semantics **flip** during a rebase! This is because rebase checks out the target branch and replays your commits on top. So "ours" becomes the target, and "theirs" becomes your feature branch. This trips up even experienced developers.

---

## 7. Rebase Conflicts — Rewrites with Responsibility

Rebasing rewrites Git history. That makes it powerful (linear history, no merge commits) and dangerous (you can lose work if you're not careful). Conflicts during a rebase follow a different workflow than merge conflicts.

### How Rebase Conflicts Work

```
    I     banned (your changes)
   /
A - J     main   (someone else's changes)
```

When you run `git rebase main` from `banned`:

1. Git checks out `main` (this is now "ours").
2. Git replays `I` on top of `main` (this is "theirs").
3. If `I` and `J` touch the same lines — conflict.

```bash
$ git rebase main
CONFLICT (add/add): Merge conflict in customers/banned.csv
error: could not apply abc123... I

$ git branch
* (no branch, rebasing banned)    # ← Detached HEAD!
  banned
  main
```

### Resolving Rebase Conflicts

Unlike merges, you **do not commit** to resolve a rebase conflict. Instead:

```bash
# 1. Fix the conflict (edit files or use checkout --ours/--theirs)
git checkout --ours customers/banned.csv

# 2. Stage the resolution
git add customers/banned.csv

# 3. Continue the rebase (NOT commit!)
git rebase --continue
```

### What If the Commit Becomes Empty?

If your resolution discards **all changes** from a commit, Git drops that commit entirely. This is expected behavior — Git sees no reason to keep an empty commit when rewriting history.

### The Accidental Commit Trap

If you accidentally run `git commit` instead of `git rebase --continue`:

```bash
# Undo the commit but keep the changes staged
git reset --soft HEAD~1

# Now continue the rebase properly
git rebase --continue
```

### Aborting a Rebase

If things go sideways and you want to start over:

```bash
git rebase --abort
```

This returns you to the exact state before the rebase started. No harm done.

---

## 8. RERERE — Reusing Recorded Resolutions

**RERERE** stands for **"Reuse Recorded Resolution."** It's one of Git's best-kept secrets.

### The Problem

Imagine you have two feature branches that both conflict with main in the same way. Without RERERE, you'd resolve the same conflict twice — once per branch.

### The Solution

Enable RERERE, and Git will:

1. **Record** how you resolve a conflict the first time.
2. **Automatically apply** the same resolution the next time it sees the same conflict.

```bash
# Enable RERERE
git config set --local rerere.enabled true
```

### RERERE in Action

```bash
# Branch 1: Rebase onto main, resolve conflict
$ git switch favs
$ git rebase main
Recorded preimage for 'customers/favs.md'   # ← Recording!
# ... resolve conflict, stage, continue ...
Recorded resolution for 'customers/favs.md' # ← Saved!

# Branch 2: Rebase onto main, conflict auto-resolved
$ git switch favs2
$ git rebase main
Resolved 'customers/favs.md' using previous resolution. # ← Magic!
# Just stage and continue — no manual editing needed
git add .
git rebase --continue
```

### Clearing the RERERE Cache

If you made a mistake in your recorded resolution:

```bash
rm -rf .git/rr-cache
```

> **Key Point:** RERERE is especially useful when you maintain long-running branches or when multiple feature branches conflict with main in similar ways. Enable it once and forget about it.

---

## 9. Squashing Commits — Clean History for Pull Requests

Every team has different commit hygiene standards, but the common expectation is: **your Pull Request should contain a clean, reviewable commit history.** Squashing collapses multiple commits into one.

### Before and After

```
# Before squashing:
A - B - C - D     (4 commits, incremental work)

# After squashing into 1:
ABCD              (1 commit, final result)
```

### How to Squash with Interactive Rebase

```bash
# Squash the last 3 commits
git rebase -i HEAD~3
```

Git opens your editor with something like:

```
pick abc1234 K: add credit card scanner
pick def5678 L: add SSN scanner
pick ghi9012 M: add phone number scanner
```

Change `pick` to `squash` (or `s`) for all but the **first** commit:

```
pick abc1234 K: add credit card scanner
squash def5678 L: add SSN scanner
squash ghi9012 M: add phone number scanner
```

Save and close. Git opens another editor for the combined commit message. Edit it to something clean:

```
K: add security scanners (credit cards, SSNs, phone numbers)
```

### The PR Squash Workflow

```
┌────────────────────────────────────────────────────────────┐
│              SQUASH + PR WORKFLOW                            │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  1. git switch -c feature        Create feature branch      │
│  2. Work, commit, work, commit   Multiple WIP commits       │
│  3. git rebase -i HEAD~N         Squash into 1 commit       │
│  4. git push origin feature      Push clean branch          │
│  5. Open Pull Request            Request review              │
│  6. Merge PR                     Clean single commit in main│
│                                                              │
└────────────────────────────────────────────────────────────┘
```

> **Warning:** Squashing is a destructive operation — it rewrites history. Never squash commits that have already been pushed to a shared branch that others are working on. Do it on **your own feature branch** before the PR.

### Force Pushing After a Squash

Since squashing rewrites commits, your local branch diverges from the remote. You'll need to force push:

```bash
git push origin feature-branch --force
```

The `--force` flag tells the remote: **"Overwrite whatever you have with what I'm giving you."** Use it with care.

---

## 10. Git Stash — Pausing Work Without Committing

You're halfway through a feature when your boss tells you to drop everything and fix a critical bug. You have uncommitted changes that you're not ready to commit. What do you do?

**`git stash`** saves your working directory and staging area to a stack, then reverts your files to match the last commit.

### Basic Usage

```bash
# Save current changes
git stash

# List stashed changes
git stash list

# Restore most recent stash (and remove from stack)
git stash pop

# Restore most recent stash (keep in stack)
git stash apply
```

### Stash with a Message

```bash
git stash -m "halfway through login feature"
```

### Managing Multiple Stashes

Stashes are stored in a **stack** (LIFO — Last In, First Out):

```
stash@{0}: WIP on main: abc1234 bad marketing    ← most recent
stash@{1}: WIP on main: abc1234 good marketing   ← older
```

```bash
# Apply a specific stash by index
git stash apply stash@{1}

# Drop a specific stash without applying
git stash drop stash@{1}
```

### Stash Conflicts

Yes, popping a stash can cause conflicts! If your working directory has changed since you stashed, and the stash touches the same lines, Git will mark the conflict just like a merge conflict. Resolve it the same way.

### When to Use Stash vs. Branch

| Situation | Use |
|-----------|-----|
| Quick context switch (minutes to hours) | `git stash` |
| Longer parallel work (hours to days) | New branch |
| Working on two features simultaneously | Worktrees (see Section 14) |

> **Key Point:** Stash is a clipboard, not a filing cabinet. If you find yourself with more than 2-3 stash entries, you should probably be using branches or worktrees instead.

---

## 11. Git Revert — Surgical Undo

While `git reset` is a sledgehammer that removes commits from history, `git revert` is a **scalpel** that creates a **new commit** doing the exact opposite of a previous commit. The original commit stays in history — the undo is transparent.

### Usage

```bash
# Revert a specific commit
git revert <commit-hash>

# Git opens your editor for a commit message
# Default: "Revert '<original message>'"
```

### Reset vs. Revert

```
┌────────────────────────────────────────────────────────────┐
│              reset --hard vs. revert                        │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  reset --hard HEAD~1:                                       │
│    A - B - C   →   A - B      (C is gone from history)     │
│                                                              │
│  revert C:                                                  │
│    A - B - C   →   A - B - C - C'  (C' undoes C)           │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

| Command | History | Use When |
|---------|---------|----------|
| `git reset --soft HEAD~1` | Undo commit, keep staged | Cleaning up your own branch |
| `git reset --hard HEAD~1` | Undo commit, discard changes | Throwing away local work |
| `git revert <hash>` | New commit that undoes old one | Undoing on shared branches |

### When to Use Each

- **`reset`**: You're on your own branch, cleaning up before a PR. No one else cares about these commits.
- **`revert`**: The commit is on `main` (or any shared branch). You need to preserve history and not break your teammates' repos.

> **Note:** Always prefer `revert` on shared branches. It's the only safe way to undo changes without rewriting history that others depend on.

---

## 12. Git Diff — Seeing Exactly What Changed

`git diff` is perhaps the most underused tool in the everyday Git workflow. It shows you line-by-line differences between states of your code.

### Common Diff Commands

```bash
# Unstaged changes (working directory vs. last commit)
git diff

# Staged changes (index vs. last commit)
git diff --staged

# Between two commits
git diff abc1234 def5678

# Between the last two commits
git diff HEAD~1 HEAD

# Between two branches
git diff main feature-branch

# Just file names (no content)
git diff --name-only main feature-branch
```

### Reading Diff Output

```diff
diff --git a/README.md b/README.md
index abc1234..def5678 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,3 @@
-# megacorp | good marketing example
+# megacorp | bad marketing example

 MegaCorp™ is _the_ enterprise CRM software.
```

| Symbol | Meaning |
|--------|---------|
| `-` (red) | Line removed |
| `+` (green) | Line added |
| `@@` | Line range (hunk header) |

> **Key Point:** Make `git diff --staged` a habit before every commit. It's your last chance to catch mistakes before they become part of history.

---

## 13. Cherry-Pick — Plucking Individual Commits

Sometimes you want **one specific commit** from another branch without merging or rebasing the entire thing. That's `cherry-pick`.

```bash
# Copy a commit from any branch onto your current branch
git cherry-pick <commit-hash>
```

### When to Cherry-Pick

- A bug fix exists on a feature branch but you need it on `main` urgently.
- You want one commit from a branch that has 20 commits, most of irrelevant.
- A colleague fixed something on their branch and you need it before they merge.

### Example

```bash
# You're on main, want commit O from add_partners
git log add_partners --oneline
# abc1234 P: add WorldBanc
# def5678 O: add ClosedML        ← want this one

git cherry-pick def5678

# Verify
git log --oneline -3
cat orgs/partners.txt
```

### Cherry-Pick Conflicts

Cherry-pick can cause conflicts too (if the commit's context differs from your current branch). Resolve them the same way as merge conflicts:

```bash
# After resolving
git add <file>
git cherry-pick --continue

# Or abort
git cherry-pick --abort
```

> **Warning:** Cherry-picking creates a **new commit** with a **different hash**. If you later merge the original branch, you may end up with duplicate changes. Use cherry-pick sparingly.

---

## 14. Git Bisect — Binary Search Your Bugs

You know the bug exists at HEAD. You know it didn't exist at some earlier commit. **Where did it sneak in?**

Checking every commit takes O(n) time. `git bisect` does it in **O(log n)** — binary search applied to your commit history.

| Total Commits | Max Checks |
|---------------|------------|
| 10 | 4 |
| 100 | 7 |
| 1,000 | 10 |
| 10,000 | 14 |

### Manual Bisect

```bash
# 1. Start bisecting
git bisect start

# 2. Mark the current commit as bad
git bisect bad HEAD

# 3. Mark a known-good commit
git bisect good <old-commit-hash>

# 4. Git checks out a commit in the middle
#    Test if the bug exists, then:
git bisect good   # Bug not present → search upper half
# or
git bisect bad    # Bug present → search lower half

# 5. Repeat step 4 until Git reports the first bad commit

# 6. Exit bisect mode
git bisect reset
```

### Automated Bisect

If you have a script that can test for the bug (exit code 0 = good, 1 = bad):

```bash
#!/bin/bash
# bisect.sh — exit 1 if the bug is present
if grep -q "SCANNING" "scripts/scan.sh"; then
    exit 1  # bad — the unwanted code is present
else
    exit 0  # good — the code isn't here yet
fi
```

```bash
git bisect start
git bisect bad HEAD
git bisect good <first-commit-hash>
git bisect run ./scripts/bisect.sh
# Git runs the script at each step and finds the bad commit automatically!
```

> **Key Point:** `git bisect run` is criminally underused. If you can write a test for the regression, you can automate the entire search. This is especially valuable in large codebases where "was this line present?" isn't the right question — instead you need to actually *run* something.

### After Finding the Bad Commit

Once bisect identifies the culprit:

```bash
# See what the commit changed
git show <bad-commit-hash>

# Option 1: Revert it (preserve history)
git revert <bad-commit-hash>

# Option 2: Fix forward (write new code)
# ... make changes, commit ...
```

---

## 15. Worktrees — Parallel Workspaces Without Cloning

### The Problem

You're deep into a feature branch when a P0 bug drops. You could:

1. **Stash** your changes → switch to main → fix → switch back → pop. (Annoying for large changes.)
2. **Clone** the repo again. (Wastes disk space and re-clones entire history.)
3. **Use a worktree.** (This is the right answer.)

### What Is a Worktree?

A **worktree** (working tree) is the directory where your tracked files live. Your main worktree is your repo root (where `.git/` lives). A **linked worktree** is a separate directory that shares the same `.git` data but has its own checked-out branch.

```
┌──────────────────────────────────────────────────────────┐
│                    WORKTREES                               │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ~/projects/megacorp/           ← Main worktree          │
│    └── .git/                    ← Full Git data          │
│                                                           │
│  ~/projects/megacorp-hotfix/    ← Linked worktree        │
│    └── .git (file, not dir)     ← Points to main .git   │
│                                                           │
│  Same repo, different branches, zero re-cloning.         │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### Key Commands

```bash
# Create a linked worktree
git worktree add ../megacorp-hotfix hotfix-branch

# List all worktrees
git worktree list

# Remove a worktree
git worktree remove megacorp-hotfix

# Clean up references to deleted worktree directories
git worktree prune
```

### Rules and Gotchas

- A branch can only be checked out in **one worktree at a time**. You can't have `main` checked out in both the main worktree and a linked worktree.
- Changes made in a linked worktree are reflected in the main worktree's reflog, branches, etc. — they share the same `.git` data.
- Linked worktrees are **lightweight** (no re-clone), making them ideal for quick context switches.
- Deleting a worktree does **not** delete the branch. It only removes the working directory and reference.

### When to Use Worktrees vs. Stash vs. Clone

| Approach | Disk Cost | Speed | Isolation |
|----------|-----------|-------|-----------|
| **Stash** | None | Instant | None (same working dir) |
| **Worktree** | Small (only working files) | Fast | Full (separate directory) |
| **Clone** | Large (full .git copy) | Slow | Full (separate repo) |

> **Note:** If you find yourself `git stash`-ing more than once a day, consider using worktrees instead.

---

## 16. Tags and Releases — Pinning Points in History

A **tag** is a named pointer to a specific commit — similar to a branch, but it **doesn't move**. Tags are typically used to mark releases.

### Creating Tags

```bash
# Annotated tag (recommended — includes metadata)
git tag -a v1.0.0 -m "First stable release"

# Lightweight tag (just a pointer, no metadata)
git tag v1.0.0-beta

# Tag a specific commit (not just HEAD)
git tag -a v0.9.0 -m "Pre-release" abc1234
```

### Viewing Tags

```bash
# List all tags
git tag

# See tag details
git show v1.0.0

# Tags appear in git log
git log --oneline --decorate
```

### Pushing Tags

Tags are **not pushed** by default. You must push them explicitly:

```bash
# Push a specific tag
git push origin v1.0.0

# Push all tags
git push origin --tags
```

### Semantic Versioning (SemVer)

Most projects use **Semantic Versioning** for their tags:

```
v MAJOR . MINOR . PATCH
  │        │       │
  │        │       └── Bug fixes (backward-compatible)
  │        └────────── New features (backward-compatible)
  └─────────────────── Breaking changes
```

| Version Bump | When |
|-------------|------|
| `1.0.0` → `1.0.1` | Bug fix |
| `1.0.1` → `1.1.0` | New feature, no breaking changes |
| `1.1.0` → `2.0.0` | Breaking API change |

> **Key Point:** Tags are "commitish" — anywhere you can use a commit hash, you can use a tag name. This makes them perfect for `git diff v1.0.0 v2.0.0` comparisons, `git checkout v1.0.0` for debugging, and more.

---

## Quick Reference Cheat Sheet

### Conflict Resolution

```bash
# Merge conflict resolution
git merge main
# ... edit conflicted files ...
git add .
git commit -m "Resolve merge conflicts"

# Rebase conflict resolution
git rebase main
# ... edit conflicted files ...
git add .
git rebase --continue          # NOT commit!

# Automated resolution
git checkout --ours <file>     # Keep current branch's version
git checkout --theirs <file>   # Keep incoming branch's version
```

### Recovery & Undo

```bash
# View reference history
git reflog

# Recover a deleted branch's commit
git merge HEAD@{N}

# Undo on your own branch
git reset --soft HEAD~1        # Keep changes staged
git reset --hard HEAD~1        # Discard everything

# Undo on shared branches
git revert <commit-hash>       # Safe, creates anti-commit
```

### Stash

```bash
git stash                      # Save changes
git stash -m "description"     # Save with message
git stash list                 # View stash stack
git stash pop                  # Apply + remove
git stash apply stash@{N}     # Apply without removing
git stash drop stash@{N}      # Remove without applying
```

### Advanced Operations

```bash
# Cherry-pick
git cherry-pick <commit-hash>

# Squash last N commits
git rebase -i HEAD~N

# Bisect
git bisect start
git bisect bad HEAD
git bisect good <hash>
git bisect run ./test.sh       # Automated!
git bisect reset

# Worktrees
git worktree add <path> <branch>
git worktree list
git worktree remove <name>

# Tags
git tag -a v1.0.0 -m "Release"
git push origin --tags
```

---

## Best Practices for Team Git Workflows

```
┌────────────────────────────────────────────────────────────┐
│                    TEAM GIT RULES                            │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  DO:                                                         │
│  • Rebase your feature branch onto main frequently          │
│  • Squash commits before opening a PR                       │
│  • Use descriptive commit messages                          │
│  • Enable RERERE for recurring conflicts                    │
│  • Use git revert on shared branches                        │
│  • Use worktrees for quick parallel work                    │
│  • Tag releases with semantic versioning                    │
│                                                              │
│  DON'T:                                                      │
│  • Force push to shared branches (main, develop)            │
│  • Rebase public branches other people are using            │
│  • Leave stale branches lying around                        │
│  • Commit conflict markers (lint for <<<<<<< in CI)         │
│  • Cherry-pick excessively (prefer merge/rebase)            │
│  • Ignore .gitignore (no node_modules, no .env files!)      │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

---

## Decision Flowchart

Not sure which Git operation to use? Here's a quick guide:

```
"I need to undo something"
  ├── On my own branch? → git reset
  └── On a shared branch? → git revert

"I need to resolve a conflict"
  ├── During merge? → Edit file → git add → git commit
  └── During rebase? → Edit file → git add → git rebase --continue

"I need to pause my work"
  ├── Quick switch? → git stash
  └── Long parallel work? → git worktree add

"I need changes from another branch"
  ├── All changes? → git merge (or git rebase)
  └── One specific commit? → git cherry-pick

"I need to find which commit broke something"
  ├── Can I script the test? → git bisect run ./test.sh
  └── Manual testing? → git bisect (good/bad loop)
```

---

## Summary

| Concept | Purpose |
|---------|---------|
| **Fork** | Your own remote copy for contributing to others' projects |
| **HEAD** | Pointer to your current branch/commit |
| **Reflog** | Safety net — records every move HEAD makes |
| **Merge Conflicts** | Happen when same lines change independently |
| **Ours/Theirs** | Automated conflict resolution for known outcomes |
| **Rebase Conflicts** | Same as merge, but `--continue` instead of `commit` |
| **RERERE** | Auto-resolve recurring conflicts |
| **Squash** | Collapse multiple commits into one for clean history |
| **Stash** | Temporary storage for uncommitted changes |
| **Revert** | Safe undo via anti-commit (preserves history) |
| **Diff** | Line-by-line comparison between states |
| **Cherry-Pick** | Copy a single commit between branches |
| **Bisect** | Binary search to find which commit introduced a bug |
| **Worktrees** | Parallel working directories sharing one `.git` |
| **Tags** | Immutable named pointers to commits (releases) |

Git is one of those tools where the basics take an afternoon to learn, but the advanced features take years to appreciate. The workflows in this guide represent the 20% of advanced Git that covers 80% of what you'll encounter in team-based development. Practice these patterns, and you'll never be the person who says *"I just re-cloned the repo"* again.

---

## Resources

This post is based on my notes from the [Learn Git 2](https://www.boot.dev/courses/learn-git-2) course on Boot.dev. If you're looking for a hands-on, structured way to learn these concepts, I highly recommend starting there — it's the first guide you should look at.

I also found this video walkthrough incredibly helpful:

- [Git for Professionals (YouTube)](https://youtu.be/rH3zE7VlIMs?si=MVgawUvmlvPiz_PH)

---

*Happy collaborating!*
