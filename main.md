# Git Commands: reset, restore & checkout — Complete Reference Guide

---

## Table of Contents
1. [Overview](#overview)
2. [git reset](#git-reset)
3. [git restore](#git-restore)
4. [git checkout](#git-checkout)
5. [Quick Comparison](#quick-comparison)
6. [Decision Flowchart](#decision-flowchart)
7. [Common Scenarios & Solutions](#common-scenarios)
8. [Safety Tips & Warnings](#safety-tips)

---

## 1. Overview <a name="overview"></a>

| Command | Moves HEAD? | Rewrites History? | Risk Level | Git Version |
|---------|-------------|-------------------|------------|-------------|
| `git reset` | ✅ Yes | ✅ Yes | 🔴 High | All versions |
| `git restore` | ❌ No | ❌ No | 🟡 Medium | 2.23+ (Aug 2019) |
| `git checkout` | ✅ Yes (branches) | ❌ No | 🟡 Medium | All versions |

**Modern Git Philosophy (2.23+):**
- Use `git switch` for branch operations (replaces `checkout` for branches)
- Use `git restore` for file operations (replaces `checkout --` for files)
- Use `git reset` only when you need to rewrite commit history

---

## 2. git reset <a name="git-reset"></a>

### 2.1 What It Does
`git reset` moves the current branch HEAD to a specified state and optionally modifies the staging area (index) and working directory. It is primarily used to **undo commits or unstage changes**.

**Core Concept:** `reset` manipulates the three Git trees:
- HEAD (commit pointer)
- Index/Staging Area
- Working Directory

### 2.2 The Three Modes

#### 🔶 `--soft` — Move HEAD Only
```bash
git reset --soft HEAD~1
```
- **HEAD:** Moves to specified commit
- **Index (Staging):** Unchanged — changes remain staged
- **Working Directory:** Unchanged
- **Commits:** "Undone" commits become staged changes

**When to use:**
- You want to undo the last commit but keep all changes staged
- You want to combine the last commit with new changes and re-commit
- You committed too early and want to add more changes

**Example:**
```bash
# You committed but forgot to add a file
git reset --soft HEAD~1        # Undo commit, keep changes staged
git add forgotten-file.txt     # Add the missing file
git commit -m "Complete feature" # Re-commit everything
```

---

#### 🔶 `--mixed` (DEFAULT) — Move HEAD + Unstage
```bash
git reset HEAD~1
# or explicitly:
git reset --mixed HEAD~1
```
- **HEAD:** Moves to specified commit
- **Index (Staging):** Cleared — all changes unstaged
- **Working Directory:** Unchanged — changes remain in files
- **Commits:** "Undone" commits become unstaged changes

**When to use:**
- You want to undo commits and reorganize what gets staged
- You want to split a large commit into smaller ones
- Most common "undo" scenario

**Example:**
```bash
# You made a commit with too many unrelated changes
git reset HEAD~1               # Undo commit, unstage everything
# Now selectively stage and commit in logical chunks:
git add src/feature-a.js
git commit -m "Add feature A"
git add src/feature-b.js
git commit -m "Add feature B"
```

---

#### 🔴 `--hard` — Move HEAD + Unstage + DELETE Working Changes
```bash
git reset --hard HEAD~1
```
- **HEAD:** Moves to specified commit
- **Index (Staging):** Cleared
- **Working Directory:** **DESTRUCTIVELY RESET** — all uncommitted changes are lost forever
- **Commits:** "Undone" commits are completely erased

**When to use:**
- You want to completely discard all changes and go back to a specific commit
- You're sure you don't need any uncommitted work
- Cleaning up a messy working directory

**⚠️ DANGER ZONE:**
```bash
# This DESTROYS all uncommitted changes permanently
git reset --hard HEAD

# This DESTROYS the last commit AND any uncommitted changes
git reset --hard HEAD~1
```

**Recovery (if you acted fast):**
```bash
# Use reflog to find the commit before reset
git reflog
# Then restore it:
git reset --hard HEAD@{1}   # Or the specific commit hash
```

---

### 2.3 Reset to Specific Commit
```bash
# Reset to a specific commit hash
git reset --hard abc1234

# Reset to origin's version (discard local commits)
git reset --hard origin/main

# Reset a specific file to HEAD version (unstage changes to that file)
git reset HEAD <file>       # Same as: git restore --staged <file>
```

---

### 2.4 Visual Diagram: reset --soft vs --mixed vs --hard

```
BEFORE (3 commits, some staged and unstaged changes):
    [Commit A] --- [Commit B] --- [Commit C] <-- HEAD/main
                                      |
                                (staging: file1, file2)
                                (working: file3, file4)

AFTER git reset --soft HEAD~1:
    [Commit A] --- [Commit B] <-- HEAD/main
                      |
                (staging: file1, file2, + all of Commit C's changes)
                (working: file3, file4)

AFTER git reset --mixed HEAD~1:
    [Commit A] --- [Commit B] <-- HEAD/main
                      |
                (staging: empty)
                (working: file1, file2, file3, file4, + all of Commit C's changes)

AFTER git reset --hard HEAD~1:
    [Commit A] --- [Commit B] <-- HEAD/main
                      |
                (staging: empty)
                (working: exactly as Commit B — file3, file4 LOST!)
                (Commit C is GONE from history)
```

---

## 3. git restore <a name="git-restore"></a>

### 3.1 What It Does
`git restore` modifies files in the working directory or staging area **without moving HEAD or changing commit history**. It was introduced in Git 2.23 to separate file operations from branch operations.

**Core Concept:** `restore` copies content from one Git tree to another:
- From HEAD to working directory (discard changes)
- From HEAD to staging area (unstage)
- From any commit to working directory or staging area

### 3.2 Restore Working Directory (Discard Local Changes)

```bash
# Discard all uncommitted changes in a file (restore to HEAD version)
git restore <file>

# Discard all uncommitted changes in entire working directory
# ⚠️ DANGER: Irreversible if not stashed first
git restore .

# Discard changes in a directory
git restore src/
```

**What happens:**
- Working directory file is overwritten with the version from HEAD
- Staging area is not affected
- HEAD does not move
- **Changes are lost** (unless stashed)

**Example:**
```bash
# You edited config.js but want to start over
cat config.js
# shows your experimental changes

git restore config.js
# config.js is now exactly as it was in the last commit
# your experimental changes are gone
```

---

### 3.3 Restore Staging Area (Unstage Files)

```bash
# Unstage a file (keep changes in working directory)
git restore --staged <file>

# Unstage all files
git restore --staged .
```

**What happens:**
- File is removed from staging area (index)
- Working directory changes are preserved
- HEAD does not move
- Commit history unchanged

**Example:**
```bash
# You accidentally staged a file you don't want in the next commit
git add huge-log-file.txt
git status
# shows huge-log-file.txt as staged

git restore --staged huge-log-file.txt
git status
# huge-log-file.txt is now unstaged (but still modified in working directory)
# add it to .gitignore to prevent future accidents
```

---

### 3.4 Restore From Specific Source

```bash
# Restore file to version from a specific commit
git restore --source=abc1234 <file>

# Restore file to version from another branch
git restore --source=feature-branch <file>

# Restore file to version from 3 commits ago
git restore --source=HEAD~3 <file>

# Restore to staging area from a specific source
git restore --source=abc1234 --staged <file>

# Restore to both staging area and working directory
git restore --source=abc1234 --staged --worktree <file>
# (or shorthand:)
git restore --source=abc1234 -SW <file>
```

**Example:**
```bash
# You want to see what a file looked like 5 commits ago
git restore --source=HEAD~5 app.js
# app.js in working directory is now from 5 commits ago
# (current version is not lost — it's still in HEAD)

# To go back to current version:
git restore app.js
```

---

### 3.5 Restore vs Checkout (Legacy Comparison)

| Operation | Old Way (checkout) | Modern Way (restore) |
|-----------|-------------------|---------------------|
| Discard working changes | `git checkout -- <file>` | `git restore <file>` |
| Unstage a file | `git reset HEAD <file>` | `git restore --staged <file>` |
| Restore from commit | `git checkout abc1234 -- <file>` | `git restore --source=abc1234 <file>` |
| Restore to staging | `git checkout -- <file>` then `git add` | `git restore --staged <file>` |

---

### 3.6 Visual Diagram: restore Operations

```
BEFORE:
    [Commit A] <-- HEAD
         |
    (staging: file1-modified-staged, file2-new-staged)
    (working: file1-modified-staged-and-more, file3-untracked)

AFTER git restore file1:
    [Commit A] <-- HEAD
         |
    (staging: file1-modified-staged, file2-new-staged)  ← unchanged!
    (working: file1-as-in-HEAD, file3-untracked)       ← file1 reverted

AFTER git restore --staged file1:
    [Commit A] <-- HEAD
         |
    (staging: file2-new-staged)                          ← file1 removed from staging
    (working: file1-modified-staged-and-more, file3-untracked) ← file1 changes preserved

AFTER git restore --source=HEAD~2 file1:
    [Commit A] <-- HEAD  ← HEAD doesn't move!
         |
    (staging: unchanged)
    (working: file1-as-in-HEAD~2, file3-untracked)       ← file1 from older commit
```

---

## 4. git checkout <a name="git-checkout"></a>

### 4.1 What It Does
`git checkout` is the legacy command that serves two distinct purposes:
1. **Switching branches** — moves HEAD to a different branch or commit
2. **Restoring files** — copies file content from a commit to working directory

Since Git 2.23, these have been split into `git switch` and `git restore`, but `checkout` remains widely used.

### 4.2 Branch Operations (Use `switch` Instead)

#### Switch to Existing Branch
```bash
# Legacy:
git checkout feature-branch

# Modern equivalent:
git switch feature-branch
```

**What happens:**
- HEAD moves to the branch pointer
- Working directory updates to match branch's commit
- Uncommitted changes that conflict will block the switch (or can be merged with `-m`)

---

#### Create and Switch to New Branch
```bash
# Legacy:
git checkout -b new-feature

# Modern equivalent:
git switch -c new-feature
# or: git switch --create new-feature
```

---

#### Switch to Specific Commit (Detached HEAD)
```bash
# Look at a specific commit without creating a branch
git checkout abc1234

# Modern equivalent:
git switch --detach abc1234
```

**⚠️ Warning:** In "detached HEAD" state, new commits won't belong to any branch. To save them, create a branch:
```bash
git checkout -b rescue-branch
```

---

#### Switch to Previous Branch
```bash
git checkout -
# Same as:
git checkout @{-1}
```

---

### 4.3 File Operations (Use `restore` Instead)

#### Discard Working Directory Changes
```bash
# Legacy:
git checkout -- <file>

# Modern equivalent:
git restore <file>
```

**What happens:**
- Overwrites `<file>` in working directory with version from HEAD
- Does NOT move HEAD
- Does NOT affect staging area (unless file was staged)
- **Changes are lost**

**Example:**
```bash
# You made experimental changes to main.js
git checkout -- main.js
# main.js is restored to last committed version
# (same as: git restore main.js)
```

---

#### Restore File From Specific Commit
```bash
# Legacy:
git checkout abc1234 -- <file>

# Modern equivalent:
git restore --source=abc1234 <file>
```

**What happens:**
- Copies `<file>` from commit `abc1234` to both staging area AND working directory
- Does NOT move HEAD
- File becomes staged (ready to commit)

**Example:**
```bash
# You accidentally deleted a file in a previous commit and want it back
git log --oneline
# find the commit BEFORE deletion: abc1234

git checkout abc1234 -- deleted-file.txt
# File is restored and staged
git commit -m "Restore accidentally deleted file"
```

---

#### Restore File From Another Branch
```bash
# Get a file from another branch without switching branches
git checkout feature-branch -- config/settings.json

# Modern equivalent:
git restore --source=feature-branch config/settings.json
```

---

### 4.4 Checkout with Merge (`-m`)

When switching branches with uncommitted changes:
```bash
# Attempt to merge local changes into the branch you're switching to
git checkout -m feature-branch
```

This performs a three-way merge between:
- Current branch's HEAD
- Target branch's HEAD
- Your local changes

---

### 4.5 Visual Diagram: checkout Operations

```
BRANCH SWITCHING:
    Before:                    After git checkout feature:

    main: [A]-[B]-[C]          main: [A]-[B]-[C]
              ↑                         ↑
            HEAD                      HEAD (still points to C)

    feature: [A]-[B]-[D]       feature: [A]-[B]-[D]
                                    ↑
                                  HEAD (moved here!)

FILE RESTORATION:
    Before:                    After git checkout -- file.txt:

    [Commit C] <-- HEAD        [Commit C] <-- HEAD  ← HEAD doesn't move!
         |                            |
    Working: file.txt (modified)  Working: file.txt (as in Commit C)
    Staging: (empty)              Staging: (empty)     ← staging unaffected

FILE FROM COMMIT:
    Before:                    After git checkout abc1234 -- file.txt:

    [Commit C] <-- HEAD        [Commit C] <-- HEAD  ← HEAD doesn't move!
         |                            |
    Working: file.txt (current)  Working: file.txt (from abc1234)
    Staging: (empty)              Staging: file.txt (staged!)
```

---

## 5. Quick Comparison <a name="quick-comparison"></a>

### 5.1 Feature Matrix

| Task | `git reset` | `git restore` | `git checkout` |
|------|-------------|---------------|----------------|
| Move HEAD / switch branches | ✅ Yes | ❌ No | ✅ Yes |
| Rewrite commit history | ✅ Yes | ❌ No | ❌ No |
| Unstage files | ✅ `--mixed` | ✅ `--staged` | ❌ No |
| Discard working changes | ✅ `--hard` | ✅ Default | ✅ `-- <file>` |
| Restore from specific commit | ✅ (indirectly) | ✅ `--source` | ✅ `<commit> --` |
| Affects staging area | ✅ Yes | ✅ Yes | ✅ (file mode) |
| Risk of data loss | 🔴 High | 🟡 Medium | 🟡 Medium |
| Modern replacement | N/A | `restore`/`switch` | `restore`/`switch` |

### 5.2 What Gets Modified

```
┌─────────────────┬─────────────┬───────────────┬─────────────────┐
│     Command     │    HEAD     │     Index     │  Working Tree   │
├─────────────────┼─────────────┼───────────────┼─────────────────┤
│ reset --soft    │   Moves     │   Unchanged   │    Unchanged    │
│ reset --mixed   │   Moves     │   Cleared     │    Unchanged    │
│ reset --hard    │   Moves     │   Cleared     │    Reset        │
│ restore         │  Stays put  │   Optional    │    Optional     │
│ restore --staged│  Stays put  │   Cleared     │    Unchanged    │
│ checkout branch │   Moves     │   Updated     │    Updated      │
│ checkout -- file│  Stays put  │   Unchanged   │    Reset        │
│ checkout commit │   Moves     │   Updated     │    Updated      │
└─────────────────┴─────────────┴───────────────┴─────────────────┘
```

---

## 6. Decision Flowchart <a name="decision-flowchart"></a>

```
START: What do you want to do?
│
├─► Undo a commit?
│   ├─► Keep changes staged? ───────► git reset --soft HEAD~1
│   ├─► Keep changes unstaged? ─────► git reset --mixed HEAD~1 (or just git reset HEAD~1)
│   └─► Discard everything? ────────► git reset --hard HEAD~1 ⚠️
│
├─► Fix a file (discard local changes)?
│   ├─► Modern Git (2.23+) ─────────► git restore <file>
│   └─► Legacy Git ─────────────────► git checkout -- <file>
│
├─► Unstage a file?
│   ├─► Modern Git (2.23+) ─────────► git restore --staged <file>
│   └─► Legacy Git ─────────────────► git reset HEAD <file>
│
├─► Get a file from another commit/branch?
│   ├─► Modern Git (2.23+) ─────────► git restore --source=<commit> <file>
│   └─► Legacy Git ─────────────────► git checkout <commit> -- <file>
│
├─► Switch branches?
│   ├─► Modern Git (2.23+) ─────────► git switch <branch>
│   └─► Legacy Git ─────────────────► git checkout <branch>
│
└─► Look at old commit temporarily?
    ├─► Modern Git (2.23+) ─────────► git switch --detach <commit>
    └─► Legacy Git ─────────────────► git checkout <commit>
```

---

## 7. Common Scenarios & Solutions <a name="common-scenarios"></a>

### Scenario 1: "I committed to the wrong branch"
```bash
# You committed to main instead of feature-branch
# Step 1: Save the commit (soft reset keeps it staged)
git reset --soft HEAD~1

# Step 2: Switch to correct branch
git switch feature-branch    # or: git checkout feature-branch

# Step 3: Commit there
git commit -m "Your commit message"
```

### Scenario 2: "I accidentally staged a huge file"
```bash
# Modern:
git restore --staged huge-file.zip

# Legacy:
git reset HEAD huge-file.zip

# Add to .gitignore to prevent recurrence
echo "*.zip" >> .gitignore
git add .gitignore
git commit -m "Ignore zip files"
```

### Scenario 3: "I want to undo all changes since last commit"
```bash
# ⚠️ DESTRUCTIVE — all uncommitted work will be lost!

# Modern approach (safer — review first):
git status                    # See what will be lost
git restore .                 # Discard all working changes
git restore --staged .        # Unstage everything

# Legacy approach:
git reset --hard HEAD         # Nuclear option
```

### Scenario 4: "I want to split my last commit into multiple commits"
```bash
# Undo the commit but keep all changes unstaged
git reset HEAD~1              # Same as --mixed

# Now selectively stage and commit:
git add src/part1.js
git commit -m "Add part 1"

git add src/part2.js
git commit -m "Add part 2"

git add tests/
git commit -m "Add tests"
```

### Scenario 5: "I deleted a file and committed the deletion — want it back"
```bash
# Find the commit before deletion
git log --oneline --follow -- deleted-file.txt
# Output:
# abc1234 Delete file
# def5678 Last version with file ← this one!

# Restore from that commit:
# Modern:
git restore --source=def5678 deleted-file.txt

# Legacy:
git checkout def5678 -- deleted-file.txt

# Commit the restoration:
git commit -m "Restore accidentally deleted file"
```

### Scenario 6: "I want to see what the project looked like 10 commits ago"
```bash
# Modern (detached HEAD):
git switch --detach HEAD~10

# Legacy:
git checkout HEAD~10

# Look around, test, etc.
# When done, go back:
git switch main              # or: git checkout main
```

### Scenario 7: "I made a mess and want to start fresh from remote"
```bash
# ⚠️ DESTRUCTIVE — local commits and changes will be lost!
git fetch origin
git reset --hard origin/main
```

### Scenario 8: "I want to unstage part of a file"
```bash
# Git doesn't support partial unstaging with restore
# Use interactive reset:
git reset --patch HEAD -- <file>
# or:
git checkout --patch <file>
```

---

## 8. Safety Tips & Warnings <a name="safety-tips"></a>

### 🚨 Before Any Destructive Operation

```bash
# 1. Check your status
git status

# 2. Stash important changes (just in case)
git stash push -m "Backup before reset"

# 3. See what will be affected
git log --oneline -5          # For reset operations
git diff                      # For restore operations

# 4. If you have a safety net (stash), proceed
# 5. If things go wrong, recover:
git stash pop                 # Get your stashed changes back
```

### 🛡️ The Reflog — Your Safety Net

Git keeps a log of all HEAD movements for 30+ days:

```bash
# See all recent HEAD movements
git reflog

# Example output:
# abc1234 HEAD@{0}: reset: moving to HEAD~1
# def5678 HEAD@{1}: commit: Add feature X
# 9012345 HEAD@{2}: checkout: moving from main to feature

# Recover from a bad reset:
git reset --hard HEAD@{1}     # Go back to before the reset
# or:
git reset --hard def5678      # Use the specific commit hash
```

### ⚠️ Critical Warnings

| Danger Level | Command | Why It's Dangerous |
|--------------|---------|-------------------|
| 🔴 **CRITICAL** | `git reset --hard` | Permanently deletes uncommitted changes and commits |
| 🔴 **CRITICAL** | `git checkout -- .` | Discards ALL uncommitted changes in current directory |
| 🟡 **HIGH** | `git restore .` | Discards all working changes without confirmation |
| 🟡 **HIGH** | `git checkout <commit> -- <file>` | Overwrites file and auto-stages it |
| 🟢 **LOW** | `git reset --soft` | Safe — nothing is lost |
| 🟢 **LOW** | `git restore --staged` | Safe — changes remain in working directory |
| 🟢 **LOW** | `git switch <branch>` | Safe if no conflicting changes |

### 📋 Pre-Operation Checklist

Before running any command that might lose data:

- [ ] Run `git status` to see current state
- [ ] Run `git diff` to review changes
- [ ] Consider `git stash` to save work temporarily
- [ ] For `reset --hard`, verify commit hash with `git log`
- [ ] Remember: `git reflog` can recover recent changes

---

## Appendix: Command Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                      GIT RESET                                  │
├─────────────────────────────────────────────────────────────────┤
│ git reset --soft HEAD~1    │ Undo commit, keep staged           │
│ git reset HEAD~1           │ Undo commit, unstage (default)     │
│ git reset --hard HEAD~1    │ Undo commit, DELETE changes ⚠️     │
│ git reset HEAD <file>      │ Unstage file (legacy)              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     GIT RESTORE (2.23+)                         │
├─────────────────────────────────────────────────────────────────┤
│ git restore <file>           │ Discard working changes          │
│ git restore --staged <file>  │ Unstage file                     │
│ git restore --source=C <file>│ Restore from commit C              │
│ git restore .                │ Discard all working changes ⚠️   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    GIT CHECKOUT (Legacy)                        │
├─────────────────────────────────────────────────────────────────┤
│ git checkout <branch>        │ Switch branch                    │
│ git checkout -b <branch>     │ Create & switch branch           │
│ git checkout -- <file>       │ Discard working changes          │
│ git checkout <commit> -- <f>│ Restore file from commit         │
│ git checkout <commit>        │ Detached HEAD (look around)      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              MODERN ALTERNATIVES (2.23+, Recommended)           │
├─────────────────────────────────────────────────────────────────┤
│ git switch <branch>          │ Instead of checkout <branch>     │
│ git switch -c <branch>       │ Instead of checkout -b          │
│ git switch --detach <commit> │ Instead of checkout <commit>     │
│ git restore <file>           │ Instead of checkout -- <file>    │
│ git restore --staged <file>  │ Instead of reset HEAD <file>     │
└─────────────────────────────────────────────────────────────────┘
```

---

*Generated: 2026-05-01*
*Git Version Reference: Commands valid for Git 2.23+*
*For older Git versions, use legacy `checkout` and `reset HEAD` syntax*