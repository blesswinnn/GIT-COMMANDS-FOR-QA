# Git for QA Engineers: A Practical Reference

A quick reference to the Git commands QA and test automation engineers use most, with an explanation of what each one does and when to use it.

> **Who this is for:** manual testers moving into automation, SDETs, and anyone who needs to check out builds, review what changed, or track down regressions using Git.

---

## Table of Contents

1. [Key Concepts](#1-key-concepts)
2. [Initial Setup](#2-initial-setup)
3. [Getting the Code](#3-getting-the-code)
4. [Staying in Sync with the Team](#4-staying-in-sync-with-the-team)
5. [Working with Branches](#5-working-with-branches)
6. [Saving and Sharing Your Test Code](#6-saving-and-sharing-your-test-code)
7. [Investigating Changes (Test Scoping)](#7-investigating-changes-test-scoping)
8. [Finding the Commit That Broke Something: `git bisect`](#8-finding-the-commit-that-broke-something-git-bisect)
9. [Testing Specific Releases and Tags](#9-testing-specific-releases-and-tags)
10. [Stashing Work in Progress](#10-stashing-work-in-progress)
11. [Undoing Mistakes](#11-undoing-mistakes)
12. [Cherry-Picking a Single Fix](#12-cherry-picking-a-single-fix)
13. [Searching the Codebase and History](#13-searching-the-codebase-and-history)
14. [A Typical QA Workflow](#14-a-typical-qa-workflow)
15. [Good Practices](#15-good-practices)
16. [Quick Reference Table](#16-quick-reference-table)

---

## 1. Key Concepts

| Term | Meaning |
|------|---------|
| **Repository (repo)** | A project folder tracked by Git, including its full history. |
| **Commit** | A saved snapshot of changes, with a unique ID (hash) such as `a1b2c3d`. |
| **Branch** | A separate line of development, such as `main` or `feature/login-fix`. |
| **Remote** | The shared copy of the repo on a server such as GitHub. It's usually named `origin`. |
| **Working directory** | The files you're currently editing. |
| **Staging area (index)** | Changes you've marked with `git add` to go into the next commit. |
| **HEAD** | The commit you currently have checked out. |
| **Tag** | A fixed label on a commit, usually marking a release (for example `v2.4.0`). |

**The basic flow:**

```
Working directory --git add--> Staging area --git commit--> Local repo --git push--> Remote (GitHub)
                                                  ^                                       |
                                                  +--------- git fetch / git pull --------+
```

---

## 2. Initial Setup

Set your identity once per machine. It's recorded on every commit you make.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@company.com"
```

Optional but useful:

```bash
# Use 'main' as the default branch name for new repos
git config --global init.defaultBranch main

# Show colored output
git config --global color.ui auto

# Check your current settings
git config --list
```

---

## 3. Getting the Code

```bash
# Download a repository to your machine
git clone https://github.com/<org>/<repo>.git

# Check which remote server(s) the repo is connected to
git remote -v
```

**When to use:** the first time you start testing or automating a project.

---

## 4. Staying in Sync with the Team

```bash
# Download the latest changes from the remote without changing your files
git fetch --all --prune

# Download the changes AND merge them into your current branch
git pull

# Pull, then replay your local commits on top (keeps history linear)
git pull --rebase origin main
```

| Command | What it does |
|---------|--------------|
| `fetch` | Updates your view of the remote. It's safe and never changes your files. |
| `pull` | `fetch` plus `merge`. It updates your current branch. |
| `--prune` | Removes local references to branches that were deleted on the remote. |

**Tip:** run `git fetch` before you start testing so you know you're on the latest build.

---

## 5. Working with Branches

```bash
# List all branches, local and remote
git branch -a

# Switch to an existing branch (for example, the feature a developer wants tested)
git switch feature/login-fix

# Create a new branch and switch to it (for your test code)
git switch -c test/login-automation

# Delete a local branch you no longer need
git branch -d test/old-branch

# Copy one folder or file from another branch without switching branches
git checkout origin/release/2.4 -- tests/
```

**Branch naming ideas for QA:**

- `test/<feature>`: new automated tests
- `qa/<ticket-id>`: work tied to a ticket, such as `qa/JIRA-1234`
- `fix/test-<name>`: fixing flaky or broken tests

> `git switch` is the newer, clearer command for changing branches. The older `git checkout <branch>` does the same thing.

---

## 6. Saving and Sharing Your Test Code

```bash
# See which files are changed, staged or untracked
git status

# Stage a specific file
git add tests/login_test.py

# Stage all changes in the current folder
git add .

# Commit the staged changes with a message
git commit -m "Add negative test cases for login page"

# Push your branch to GitHub (-u is only needed the first time)
git push -u origin test/login-automation

# After that, a plain push is enough
git push
```

After pushing, open a **Pull Request (PR)** on GitHub so others can review your tests.

**Writing good commit messages:**

- ✅ `Add API tests for order cancellation endpoint`
- ✅ `Fix flaky wait in checkout E2E test`
- ❌ `changes`
- ❌ `update`

---

## 7. Investigating Changes (Test Scoping)

**This is the most valuable section for QA.** These commands show you *what changed*, so you can decide *what to test*.

### View history

```bash
# Compact history with a branch graph (last 20 commits)
git log --oneline --graph -20

# Commits from the last 2 days by a specific developer
git log --since="2 days ago" --author="dev-name"

# Full history of changes to a specific folder, with the code changes
git log -p -- src/payment/
```

### Compare branches and releases

```bash
# Summary of which files changed and how much
git diff main..feature/login-fix --stat

# Only the names of the files that changed between two releases
git diff release/2.3..release/2.4 --name-only

# Full code differences between two branches
git diff main..feature/login-fix
```

**Use case:** before a regression cycle, run `git diff --name-only` between the last release and the current one. The list of changed files shows you which modules need the most attention.

### Inspect a single commit

```bash
# Show the message, author and code changes of one commit
git show <commit-hash>
```

### Find who changed a line

```bash
# Show who last modified each line of a file, and in which commit
git blame src/login.js
```

**Use case:** when you find a bug, `blame` helps you find the developer and commit to reference in the bug report.

---

## 8. Finding the Commit That Broke Something: `git bisect`

`git bisect` runs a **binary search** through your commit history to find the exact commit that introduced a bug. Even across hundreds of commits, it usually takes only a handful of steps.

### Manual bisect

```bash
git bisect start
git bisect bad                  # The current version has the bug
git bisect good v2.3.0          # This older version worked
```

Git checks out a commit halfway between the two. Test it, then tell Git the result:

```bash
git bisect good    # The bug is NOT present in this commit
# or
git bisect bad     # The bug IS present in this commit
```

Repeat until Git reports:

```
<hash> is the first bad commit
```

Then return to where you started:

```bash
git bisect reset
```

### Automated bisect

If you have a test or script that **exits 0 on pass and non-zero on fail**, Git can run it for you:

```bash
git bisect start
git bisect bad
git bisect good v2.3.0
git bisect run npm test
# or, for example: git bisect run pytest tests/test_checkout.py
git bisect reset
```

**Use case:** attach the "first bad commit" hash to your bug ticket. It saves developers a lot of debugging time.

---

## 9. Testing Specific Releases and Tags

```bash
# List all tags (releases)
git tag

# Check out the code exactly as it was at a release
git checkout v2.4.0

# List the commits included in a release (what's new since the previous one)
git log v2.3.0..v2.4.0 --oneline
```

> Checking out a tag puts you in a **"detached HEAD"** state. That's fine for testing. If you want to make changes from there, create a branch first: `git switch -c test/v2.4-hotfix`.

**Use case:** reproducing a bug reported in production on the exact version that's deployed.

---

## 10. Stashing Work in Progress

Stashing temporarily shelves your uncommitted changes so you can switch tasks.

```bash
# Save the current changes with a label
git stash push -m "wip: checkout tests"

# See all stashes
git stash list

# Re-apply the most recent stash and remove it from the list
git stash pop

# Re-apply a specific stash but keep it in the list
git stash apply stash@{1}
```

**Use case:** you're halfway through writing a test when an urgent build needs a smoke test. Stash, switch, test, switch back, then pop.

---

## 11. Undoing Mistakes

| Situation | Command | Notes |
|-----------|---------|-------|
| Discard local edits to a file | `git restore <file>` | ⚠️ The changes are lost permanently. |
| Unstage a file (keep the edits) | `git restore --staged <file>` | Undoes `git add`. |
| Fix the last commit message | `git commit --amend -m "New message"` | Only if you **haven't pushed** yet. |
| Undo the last commit but keep the changes | `git reset --soft HEAD~1` | The changes stay staged. |
| Undo a commit that's already pushed | `git revert <commit-hash>` | ✅ Safe: adds a new commit that reverses it. |

> **Rule of thumb:** if a commit is already pushed and shared, use `revert`, not `reset`. Rewriting shared history causes problems for everyone else.

---

## 12. Cherry-Picking a Single Fix

```bash
# Apply one specific commit from another branch onto your current branch
git cherry-pick <commit-hash>
```

**Use case:** a developer fixed a bug on their branch and you need just that fix in your test branch, without merging everything else.

---

## 13. Searching the Codebase and History

```bash
# Search the current files for text (for example, a test locator)
git grep 'data-testid="submit"'

# Find commits that added or removed a specific piece of text
git log -S "calculateTax" --oneline

# Find commits whose message mentions a ticket ID
git log --grep="JIRA-1234" --oneline
```

**Use case:** finding where a UI element is defined, or which commit changed the function behind a failing test.

---

## 14. A Typical QA Workflow

```bash
# 1. Get the latest code and switch to the feature under test
git fetch --all --prune
git switch feature/XYZ
git pull

# 2. See what changed, to plan your test scope
git diff main..feature/XYZ --stat

# 3. Create your own branch for automation work
git switch -c test/XYZ-automation

# 4. Write your tests, then commit and push
git add tests/
git commit -m "Add E2E tests for XYZ feature"
git push -u origin test/XYZ-automation

# 5. Open a Pull Request on GitHub for review

# 6. If a regression appears, find the commit that caused it
git bisect start
git bisect bad
git bisect good <last-known-good-tag>
git bisect run <your-test-command>
git bisect reset
```

---

## 15. Good Practices

- **Pull before you start.** Always test against the latest code.
- **Never commit secrets.** Keep passwords, tokens and API keys out of test code. Use environment variables or a `.env` file listed in `.gitignore`.
- **Use a `.gitignore`.** Exclude test reports, screenshots, logs and `node_modules/` or `venv/` folders.
- **Keep commits small and focused.** One logical change per commit makes review and reverting easier.
- **Don't push straight to `main`.** Use branches and Pull Requests.
- **Reference tickets.** Include the ticket ID in branch names or commit messages, for example `qa/JIRA-1234`.
- **Don't rewrite shared history.** Avoid `reset --hard` and `push --force` on branches other people use.

Example `.gitignore` entries for test projects:

```gitignore
# Test output
reports/
allure-results/
screenshots/
videos/
*.log

# Dependencies
node_modules/
venv/
.venv/

# Secrets and local configuration
.env
*.local.json

# IDE files
.idea/
.vscode/
```

---

## 16. Quick Reference Table

| Task | Command |
|------|---------|
| Clone a repo | `git clone <url>` |
| Check status | `git status` |
| Get the latest changes | `git pull` |
| List branches | `git branch -a` |
| Switch branch | `git switch <branch>` |
| Create a branch | `git switch -c <branch>` |
| Stage changes | `git add <file>` |
| Commit | `git commit -m "message"` |
| Push | `git push` |
| View history | `git log --oneline --graph` |
| Compare branches | `git diff main..<branch> --stat` |
| Files changed between releases | `git diff v1..v2 --name-only` |
| Inspect a commit | `git show <hash>` |
| Who changed a line | `git blame <file>` |
| Find the breaking commit | `git bisect start` / `good` / `bad` / `reset` |
| Check out a release | `git checkout <tag>` |
| Shelve work | `git stash` / `git stash pop` |
| Undo a pushed commit | `git revert <hash>` |
| Apply one commit | `git cherry-pick <hash>` |
| Search code | `git grep "text"` |

---

## Further Reading

- [Official Git documentation](https://git-scm.com/doc)
- [Pro Git book (free)](https://git-scm.com/book/en/v2)
- [GitHub Docs: Git basics](https://docs.github.com/en/get-started/using-git)
- [Interactive Git branching tutorial](https://learngitbranching.js.org/)

---

*Contributions are welcome. Feel free to open a PR to add commands or examples that have helped you in QA work.*
