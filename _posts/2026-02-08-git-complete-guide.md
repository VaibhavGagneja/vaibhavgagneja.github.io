---
title: "Git Mastery: The Complete Guide from Basics to GitHub"
description: Learn Git from scratch - repositories, commits, branches, merging, rebasing, remotes, and GitHub workflows with hands-on exercises
author: Vaibhav Gagneja
date: 2026-02-08 21:00:00 +0530
categories: [Development, Git]
tags: [git, github, version-control, tutorial]
toc: true
image:
  path: https://plus.unsplash.com/premium_photo-1678565999332-1cde462f7b24?q=80&w=870&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

Git is the most widely used version control system in the world. Whether you're a solo developer or part of a large team, understanding Git is essential. This comprehensive guide takes you from zero to confident Git user.

---

## 1. Understanding Git Repositories

### What is a Repository?

A Git **repository** (or "repo") is simply a directory that contains your project files along with a hidden `.git` folder. This hidden folder is where Git stores all its tracking and versioning information.

```
my-project/
├── .git/           ← Hidden folder (Git's brain)
├── src/
├── README.md
└── index.html
```

### Creating Your First Repository

Let's create a project called **"MovieHub"** - a movie catalog application.

```bash
# Create and enter project directory
mkdir moviehub
cd moviehub

# Initialize Git repository
git init

# Verify the .git folder exists
ls -la
```

> **🔑 Key Point:** The `git init` command creates the hidden `.git` directory, transforming a regular folder into a Git repository.

---

## 2. File States and Git Status

### The Three States of a File

| State | Description |
|-------|-------------|
| **Untracked** | Git doesn't know about this file yet |
| **Staged** | Marked to be included in the next commit |
| **Committed** | Saved permanently in Git's history |

```
┌──────────────────────────────────────────────────────────┐
│                    FILE LIFECYCLE                         │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  [Untracked] ──git add──► [Staged] ──git commit──► [Committed]
│       │                      │                           │
│       │                      │                           │
│       ◄── (new file) ────────┼──────────────────────────►│
│                              │                           │
│       ◄── (modify) ──────────┴───────────────────────────┘
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### Checking Repository Status

The `git status` command shows the current state of your repository:

```bash
git status
```

**Try This:** Create a file called `catalog.md` and run `git status` to see it listed as untracked.

```bash
echo "# Movie Catalog" > catalog.md
git status
```

---

## 3. Staging and Committing

### Staging Files

Before committing, you must **stage** files using `git add`:

```bash
# Stage a specific file
git add catalog.md

# Stage all files
git add .

# Stage files matching a pattern
git add *.md
```

### Creating Commits

A **commit** is a snapshot of your project at a specific point in time:

```bash
git commit -m "Initial commit: add movie catalog"
```

> **Best Practice:** Write clear, descriptive commit messages. They should explain *what* and *why*.

### Commit Message Format

```
<type>: <short description>

[optional body with more details]
```

**Examples:**
- `feat: add user authentication`
- `fix: resolve login redirect bug`
- `docs: update API documentation`

---

## 4. Git Log and Commit History

### Viewing Commit History

```bash
# Full log
git log

# Compact one-line format
git log --oneline

# Limit number of commits
git log -n 5

# Show with graph visualization
git log --oneline --graph --all
```

### Understanding Commit Hashes

Each commit has a unique identifier called a **SHA hash**:

```
5ba786fcc93e8092831c01e71444b9baa2228a4f
```

You can reference commits using the first 7 characters: `5ba786f`

> **Note:** Even identical changes will have different hashes because hashes also include author info, timestamp, and parent commit references.

---

## 5. Git Internals: The Plumbing

Git stores everything as **objects** in the `.git/objects` directory:

| Object Type | Description |
|-------------|-------------|
| **blob** | Stores file contents |
| **tree** | Stores directory structure |
| **commit** | Points to a tree + metadata |

### Inspecting Objects

```bash
# View commit details
git cat-file -p <commit-hash>

# View tree contents
git cat-file -p <tree-hash>

# View file (blob) contents
git cat-file -p <blob-hash>
```

**Exercise:** Run `git log --oneline -1` to get your latest commit hash, then use `git cat-file -p` to inspect its contents.

```
┌──────────────────────────────────────────────────────────┐
│                    COMMIT STRUCTURE                       │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  commit 5ba786f                                          │
│     │                                                     │
│     ├── tree 4e507fd ─────► blob a1b2c3 (catalog.md)     │
│     │                  │                                  │
│     │                  └──► blob d4e5f6 (README.md)      │
│     │                                                     │
│     ├── author: You <you@email.com>                      │
│     ├── committer: You <you@email.com>                   │
│     └── message: "Initial commit"                        │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

---

## 6. Git Configuration

### Setting Your Identity

```bash
# Set globally (for all repos)
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# Set locally (for current repo only)
git config --local user.name "Work Name"
```

### Viewing Configuration

```bash
# List all config
git config --list

# Get specific value
git config user.name

# View config file directly
cat ~/.gitconfig      # Global
cat .git/config       # Local
```

### Useful Configuration Options

```bash
# Set default branch to main
git config --global init.defaultBranch main

# Set default editor
git config --global core.editor "code --wait"

# Enable colorful output
git config --global color.ui auto
```

---

## 7. Branching

### What is a Branch?

A **branch** is simply a pointer to a specific commit. The default branch is typically called `main` (or `master` in older repos).

```
┌──────────────────────────────────────────────────────────┐
│                    BRANCH = POINTER                       │
├──────────────────────────────────────────────────────────┤
│                                                           │
│   A ─── B ─── C                                          │
│               ↑                                           │
│             main                                          │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### Branch Commands

```bash
# List branches
git branch

# Create new branch
git branch feature-login

# Create and switch to new branch
git switch -c feature-login

# Switch to existing branch
git switch main

# Delete branch
git branch -d feature-login
```

### Branch Workflow Example

```bash
# Start from main
git switch main

# Create feature branch
git switch -c add-reviews

# Make changes and commit
echo "# Reviews" > reviews.md
git add reviews.md
git commit -m "Add reviews page"

# Switch back to main
git switch main
```

**Resulting Structure:**

```
          D    add-reviews
         /
A - B - C      main
```

---

## 8. Merging

### What is Merging?

Merging combines changes from different branches into one.

### Fast-Forward Merge

When the target branch has no new commits, Git simply moves the pointer forward:

**Before:**
```
      C - D    feature
     /
A - B          main
```

**After `git merge feature`:**
```
A - B - C - D    main, feature
```

### Three-Way Merge

When both branches have new commits, Git creates a **merge commit**:

**Before:**
```
A - B - C        main
   \
    D - E        feature
```

**After merge:**
```
A - B - C - F    main
   \     /
    D - E        feature
```

### Merge Commands

```bash
# Switch to target branch
git switch main

# Merge feature branch
git merge feature-branch

# Merge with custom message
git merge feature-branch -m "Merge feature into main"
```

---

## 9. Rebasing

### Merge vs Rebase

| Aspect | Merge | Rebase |
|--------|-------|--------|
| **History** | Preserves true history | Creates linear history |
| **Merge commits** | Creates merge commit | No merge commit |
| **Use case** | Team collaboration | Cleaning up local work |

### How Rebase Works

**Before:**
```
A - B - C          main
   \
    D - E          feature
```

**After `git rebase main` (from feature):**
```
A - B - C              main
         \
          D' - E'      feature
```

The commits are **replayed** on top of main.

### Rebase Commands

```bash
# While on feature branch
git switch feature
git rebase main

# Interactive rebase (edit commits)
git rebase -i HEAD~3
```

> ⚠️ **Warning:** Never rebase public branches that others are using. It rewrites history and will cause conflicts for your team.

### Golden Rule of Rebasing

```
┌──────────────────────────────────────────────────────────┐
│                    REBASE RULES                           │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ✅ DO: Rebase your local branches onto main             │
│  ✅ DO: Rebase before pushing to clean history           │
│                                                           │
│  ❌ DON'T: Rebase main onto anything                      │
│  ❌ DON'T: Rebase branches others are using               │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

---

## 10. Undoing Changes

### Git Reset

`git reset` moves the branch pointer to a different commit:

| Mode | Working Directory | Staging Area |
|------|-------------------|--------------|
| `--soft` | Unchanged | Unchanged (changes staged) |
| `--mixed` (default) | Unchanged | Cleared |
| `--hard` | Cleared | Cleared |

```bash
# Undo last commit, keep changes staged
git reset --soft HEAD~1

# Undo last commit, unstage changes
git reset HEAD~1

# Completely discard last commit and changes
git reset --hard HEAD~1
```

> ⚠️ **Danger:** `--hard` permanently deletes uncommitted changes!

### Other Undo Methods

```bash
# Discard changes in working directory
git checkout -- filename.txt

# Unstage a file
git restore --staged filename.txt

# Create a new commit that undoes a previous commit
git revert <commit-hash>
```

---

## 11. Working with Remotes

### What is a Remote?

A **remote** is a version of your repository hosted elsewhere (like GitHub, GitLab, or another server).

```bash
# Add a remote
git remote add origin https://github.com/username/repo.git

# List remotes
git remote -v

# Remove a remote
git remote remove origin
```

### Fetch vs Pull

| Command | What it does |
|---------|--------------|
| `git fetch` | Downloads changes but doesn't merge |
| `git pull` | Downloads AND merges changes |

```bash
# Fetch remote changes
git fetch origin

# View remote branches
git branch -r

# Pull changes
git pull origin main
```

### Push

Send your local commits to the remote:

```bash
# Push to remote
git push origin main

# Push new branch
git push -u origin feature-branch

# Force push (dangerous!)
git push --force origin main
```

---

## 12. GitHub Workflow

### Typical Team Workflow

```
┌──────────────────────────────────────────────────────────┐
│                    GITHUB WORKFLOW                        │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  1. git pull origin main        (Get latest changes)     │
│                    ↓                                      │
│  2. git switch -c feature       (Create feature branch)  │
│                    ↓                                      │
│  3. Make changes + commit       (Work on feature)        │
│                    ↓                                      │
│  4. git push origin feature     (Push to remote)         │
│                    ↓                                      │
│  5. Open Pull Request           (Request merge on GitHub)│
│                    ↓                                      │
│  6. Code Review                 (Team reviews changes)   │
│                    ↓                                      │
│  7. Merge PR                    (Merge into main)        │
│                    ↓                                      │
│  8. Delete feature branch       (Cleanup)                │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### Pull Requests

A **Pull Request (PR)** is a GitHub feature to propose and discuss changes before merging:

1. Push your branch to GitHub
2. Click "New Pull Request"
3. Select base (main) and compare (your branch)
4. Add description and request reviewers
5. After approval, merge the PR

---

## 13. Gitignore

### What to Ignore

The `.gitignore` file tells Git which files to ignore:

```gitignore
# Dependencies
node_modules/
vendor/
venv/

# Build outputs
dist/
build/
*.class
*.pyc

# Environment files
.env
.env.local
*.secret

# Editor files
.vscode/
.idea/
*.swp

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/
```

### Pattern Syntax

| Pattern | Meaning |
|---------|---------|
| `*.log` | All .log files anywhere |
| `/temp` | temp folder in root only |
| `temp/` | Any folder named temp |
| `!important.log` | Exception - don't ignore this |
| `**/cache` | cache folder at any depth |

### Example .gitignore for Web Project

```gitignore
# Dependencies
node_modules/

# Build
dist/
.next/

# Environment
.env
.env*.local

# Testing
coverage/

# IDE
.vscode/
.idea/

# OS
.DS_Store
*.swp
```

---

## 14. Quick Reference

### Essential Commands

```bash
# Setup
git init                    # Initialize repo
git clone <url>             # Clone remote repo

# Daily workflow
git status                  # Check status
git add .                   # Stage all changes
git commit -m "message"     # Commit
git push origin main        # Push to remote

# Branching
git branch                  # List branches
git switch -c feature       # Create & switch
git switch main             # Switch branch
git merge feature           # Merge branch

# History
git log --oneline           # View commits
git diff                    # View changes

# Undo
git reset --soft HEAD~1     # Undo commit, keep changes
git reset --hard HEAD~1     # Undo commit, discard changes

# Remote
git pull                    # Fetch + merge
git push                    # Upload commits
git fetch                   # Download without merge
```

### Useful Aliases

Add these to your global config:

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all"
```

---

## Summary

### Key Takeaways

1. ✅ **Repository** = Project folder + `.git` directory
2. ✅ **Commit** = Snapshot of your project at a point in time
3. ✅ **Branch** = Pointer to a commit, cheap to create
4. ✅ **Merge** = Combine branches, may create merge commit
5. ✅ **Rebase** = Replay commits for linear history
6. ✅ **Remote** = External copy of your repo (e.g., GitHub)
7. ✅ **Pull Request** = Propose and review changes before merging
8. ✅ **.gitignore** = Tell Git which files to ignore

### Best Practices

```
┌────────────────────────────────────────────────────────────┐
│                    GIT BEST PRACTICES                       │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ DO:                                                     │
│  • Commit often with meaningful messages                    │
│  • Use branches for features and bug fixes                  │
│  • Pull before you push                                     │
│  • Review your changes before committing                    │
│  • Use .gitignore for generated/sensitive files            │
│                                                             │
│  ❌ DON'T:                                                   │
│  • Don't commit sensitive data (passwords, API keys)        │
│  • Don't force push to shared branches                     │
│  • Don't rebase public branches                            │
│  • Don't commit large binary files                         │
│  • Don't leave uncommitted changes for too long            │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

---

## Practice Exercises

### Exercise 1: Basic Workflow
1. Create a new directory called `my-portfolio`
2. Initialize a Git repository
3. Create `index.html` with basic HTML
4. Stage and commit with message "Initial portfolio setup"
5. Run `git log` to verify

### Exercise 2: Branching
1. Create a branch called `add-about-page`
2. Add an `about.html` file
3. Commit the changes
4. Switch back to main
5. Merge the branch and delete it

### Exercise 3: GitHub Workflow
1. Create a new repo on GitHub
2. Add it as a remote to your local repo
3. Push your main branch
4. Create a feature branch locally
5. Push it and open a Pull Request
6. Merge the PR on GitHub
7. Pull the changes locally

---

*Happy Version Controlling! 🎉*
