---
title: "Git Conflict Resolution Exercises: A Poetic Journey"
description: Master Git conflict resolution through wolf-themed poetic exercises
author: Vaibhav Gagneja
date: 2025-04-10 12:10:53 +0000
categories: [DevOps, Git]
tags: [git, version-control, conflict-resolution, tutorial]
image:
  path: https://cdn.hashnode.com/res/hashnode/image/upload/v1744286268529/e614f545-7628-41d3-999f-314b6f6ae61a.png
---

### Setup

**Initialize Repository**:

```bash
mkdir git-poem-exercise
cd git-poem-exercise
git init  
echo -e "The sun dips low in twilight's glow,\nShadows stretch across the snow.\nA lone wolf howls a mournful tune,\nBeneath the watchful eye of the moon." > poem.txt  
git add poem.txt && git commit -m "Initial poem"
```

### Scenario 1: Adjacent-Line Conflict

**Objective**: Resolve conflicts in adjacent lines (e.g., line 2 and 3).

**Setup**:

1. In `branch1`, edit line 2:
    
    ```bash
    git checkout -b branch1  
    # Change line 2 to emphasize darkness:  
    sed -i '2s/.*/Darkness creeps where shadows grow,/' poem.txt  
    git commit -am "branch1: Rephrase line 2"  
    ```
    
2. In `branch2`, edit line 3:
    
    ```bash
    git checkout main  
    git checkout -b branch2  
    # Add environmental details to line 3:  
    sed -i '3s/.*/A lone wolf howls a mournful tune in the frosty air,/' poem.txt  
    git commit -am "branch2: Expand line 3"
    ```

**Conflict**

Merge `branch2` into `main` after merging `branch1`:

```bash
git checkout main  
git merge branch1  
git merge branch2
```

Git may flag adjacent changes if the modifications are close enough to overlap in context.

**Resolution**

1. **Identify Changes**
2. **Adjust Changes**
3. **Commit the Merge**:
    ```bash
    git add poem.txt  
    git commit -m "Merge branch2 into main: Resolve adjacent-line conflict"
    ```

---

### Scenario 2: Binary File Conflict

**Objective**: Resolve conflicts in binary files (e.g., `image.png`).

**Setup**:

1. **Add** `image_v1.png` to `main`:
    ```bash
    cp ~/Downloads/image_v1.png image.png
    git add image.png
    git commit -m "Add image_v1.png"
    ```

2. **In** `branch1`, replace with `image_v2.png`:
    ```bash
    git checkout -b branch1
    cp ~/Downloads/image_v2.png image.png
    git commit -am "Update image to v2"
    ```

3. **In** `branch2`, replace with `image_v3.png`:
    ```bash
    git checkout main
    git checkout -b branch2
    cp ~/Downloads/image_v3.png image.png
    git commit -am "Update image to v3"
    ```

**Conflict**:

```bash
git merge branch1
```

**Resolution**:

1. **Choose which version to keep**:
    ```bash
    git checkout branch1 -- image.png  # Keep v2
    # OR
    git checkout branch2 -- image.png  # Keep v3
    ```

2. **Commit the resolved file**:
    ```bash
    git commit -am "Resolve binary conflict"
    ```

---

### Scenario 3: Merge vs. Rebase Conflict

**Objective**: Compare merge and rebase workflows.

**Setup**:

1. In `feature`, edit line 5:
    ```bash
    git checkout -b feature
    # Change "frankincense" to "myrrh"
    git commit -am "feature: use myrrh"
    ```

2. In `main`, edit line 5:
    ```bash
    # Change "frankincense" to "lavender"
    git commit -am "main: use lavender"
    ```

**Conflict**:

* **Merge**: `git merge feature`
* **Rebase**: `git checkout feature && git rebase main`

**Resolution**:

* For merge: Resolve conflict and commit.
* For rebase: Resolve conflict, then `git rebase --continue`.

---

### Scenario 4: Cherry-Pick Conflict

**Objective**: Resolve conflicts when cherry-picking a commit.

**Setup**:

1. **Create** `branch1` and modify line 4:
    ```bash
    git checkout -b branch1
    # Edit line 4 to add "lilies"
    git commit -am "branch1: add lilies to line 4"
    ```

2. **Switch to** `main`, modify the same line:
    ```bash
    git checkout main
    # Edit line 4 to replace "leaves" with "petals"
    git commit -am "main: replace leaves with petals"
    ```

**Conflict**:

```bash
git cherry-pick <commit-hash>
```

**Resolution**:

1. **Manually resolve the conflict** in `poem.txt`
2. **Complete the cherry-pick**:
    ```bash
    git add poem.txt  
    git cherry-pick --continue
    ```

---

### Scenario 5: Diverged Branches Conflict

**Objective**: Resolve merge conflicts between two branches that modified overlapping lines.

**Setup**:

1. **Create** `branch1` with 3 commits modifying lines 1, 3, and 4
2. **Create** `branch2` with 2 commits modifying lines 3 and 4

**Conflict**:

```bash
git checkout branch1  
git merge branch2
```

**Resolution**:

1. Use `git diff branch1..branch2` to identify changes
2. Edit `poem.txt` to combine changes
3. Commit the merge

---

### Scenario 6: Renamed File Conflicts

**Objective**: Resolve conflicts when a file is renamed in one branch and edited in another.

**Setup**:

1. **Create** `branch1` (rename file):
    ```bash
    git checkout -b branch1  
    git mv poem.txt elegy.txt  
    git commit -m "Rename poem.txt to elegy.txt"
    ```

2. **Create** `branch2` (edit original file):
    ```bash
    git checkout main  
    git checkout -b branch2  
    echo "The night embraces all in quiet grace." >> poem.txt  
    git commit -am "branch2: Add new line to poem.txt"
    ```

**Resolution**:

1. Extract changes from `branch2`: `git show branch2:poem.txt > temp.txt`
2. Apply to renamed file: `echo "The night embraces all in quiet grace." >> elegy.txt`
3. Resolve: `git rm poem.txt && git add elegy.txt && git commit -m "Resolve rename/edit conflict"`

---

### Scenario 7: Deleted File Conflicts

**Objective**: Resolve conflicts when a file is deleted in one branch and modified in another.

**Setup**:

1. **Create** `branch1` (delete file):
    ```bash
    git checkout -b branch1  
    git rm poem.txt  
    git commit -m "Delete poem.txt"
    ```

2. **Create** `branch2` (edit file):
    ```bash
    git checkout main  
    git checkout -b branch2  
    echo "The night whispers secrets to the stars." >> poem.txt  
    git commit -am "branch2: Add new line"
    ```

**Resolution**:

**Option 1**: Keep the file with changes:
```bash
git checkout branch2 -- poem.txt
git add poem.txt  
git commit -m "Restore poem.txt with changes"
```

**Option 2**: Permanently delete:
```bash
git rm poem.txt  
git commit -m "Delete poem.txt"
```

---

## Key Takeaways

* Git flags conflicts when a file is modified in one branch and deleted/renamed in another
* Use `git checkout <branch> -- <file>` to recover changes from a specific branch
* Use `git rm` to finalize deletion if the file is no longer needed
* Always verify changes with `git status` and `git diff` before committing
