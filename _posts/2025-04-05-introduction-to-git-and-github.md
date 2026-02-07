---
title: "A Simple Introduction to Git and GitHub for Beginners"
description: Learn about Git and GitHub - key features, workflow, and best practices
author: Vaibhav Gagneja
date: 2025-04-05 09:57:56 +0000
categories: [DevOps, Git]
tags: [git, github, version-control, beginners, tutorial]
image:
  path: https://cdn.hashnode.com/res/hashnode/image/upload/v1743846735090/7dd75171-6294-460e-bac3-0ca0e6f186de.jpeg
---

# Introduction

Git is a distributed version control system (VCS) that enables developers to track changes in their code and collaborate on projects efficiently. Created by Linus Torvalds in 2005 to manage the development of the Linux kernel, Git has since become the most widely used VCS in software development.

Unlike older version control systems that rely on a central server, Git allows each developer to have a complete copy of the entire codebase (including its history) on their local machine.

### Key Features of Git

* **Efficient Branching**
* **Merging**
* **Commit History with Cryptographic Integrity**

### What is GitHub?

GitHub is a web-based hosting platform for Git repositories. It allows developers to store, share, and collaborate on code with others.

### Checking if Git is Installed

```bash
git --version
```

If Git is not installed, follow this [guide](https://www.theodinproject.com/lessons/foundations-setting-up-git) to install it.

### Initializing a Git Repository

```bash
git init
```

#### Create or Clone a Project

* **To create a new project**: `git init`
* **To clone an existing project**: `git clone [url]`

### Git Workflow: File Stages

1. **Untracked**: The file exists but is not yet part of Git's version control.
2. **Staged**: The file has been added to Git's version control.
3. **Committed**: The changes have been committed to the repository.

```bash
git status
```

---

# Git Fundamentals

### Adding and Committing Changes

```bash
git add <file>
git commit -m "<commit message>"
git log --oneline
```

### Internal Working of Git

Git stores its data in a hidden directory called `.git`. Within this directory:

* **Commit Object**: A commit is a type of object in Git.
* **Tree Object**: Represents a directory.
* **Blob Object**: Stores the content of a file.

```bash
git cat-file -p <object_hash>
```

### Git Configuration

```bash
git config <level> <key> <value>
git config --unset <key>
git config --remove-section <section>
```

## Git Commits

When a commit is made, Git creates a commit object that includes:

* A reference to the snapshot of the staged content
* Details about who created and committed it
* The commit message
* Links to previous commits

### Branching in Git

```bash
git branch              # List all branches
git branch <name>       # Create new branch
```

### Merging in Git

```bash
git merge <branch_name>
```

* **Three-way merge**: Creates a new merge commit with two parents
* **Fast-forward merge**: Simply moves the pointer

### Visualizing Commit History

```bash
git log --oneline --decorate --graph --all
```

### Default Branch Naming

```bash
git config --global init.defaultBranch main
```

### Switching Branches

```bash
git switch <branch_name>
git switch -c <new_branch_name>
# Or older style:
git checkout -b <new_branch_name>
```

---

## Git and GitHub

### Rebasing

```bash
git rebase <target_branch>
```

> **Warning**: Never rebase a public branch like `main` that others are collaborating on.

### Fetching and Pushing

```bash
git fetch <remote_name>
git remote add <name> <url>
git push <remote_name> <branch_name>
```

### Pulling from Remote Repositories

```bash
git pull <remote_name> <branch_name>
```

### GitHub and Pull Requests

The typical workflow:

1. Creating a branch
2. Making changes
3. Pushing the branch to a remote repository
4. Creating a pull request through GitHub

### Gitignore File

The `.gitignore` file specifies intentionally untracked files that Git should ignore.

### Forking and Contributing

A **fork** is a copy of the original repository in your own account that you can modify without affecting the original.

---

# Resources

* [YouTube Video 1](https://youtu.be/q8EevlEpQ2A?si=lhrIPwnsgIKdSvqT) (Recommended for beginners)
* [YouTube Video 2](https://youtu.be/rH3zE7VlIMs?si=WUldqE-lOzTFSdfj) (For people with experience)
* [Detailed Notes](https://docs.chaicode.com/git-and-github/)
* [Pro Git Book](https://git-scm.com/book/en/v2)

Happy coding!
