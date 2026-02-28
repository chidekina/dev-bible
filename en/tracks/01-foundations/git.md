# Git

> Git is not just a backup system. It is a directed acyclic graph of snapshots, a collaboration protocol, and a debugging tool. Understanding its object model and branching strategies transforms you from someone who memorizes commands to someone who reasons about version control.

---

## 1. What & Why

Git is the world's most widely used version control system, created by Linus Torvalds in 2005 to manage the Linux kernel. Unlike older systems (SVN, CVS) that tracked differences between file versions, Git stores **complete snapshots** of the entire repository at each commit.

Understanding Git deeply lets you:
- Recover from mistakes confidently (reflog, reset, cherry-pick)
- Debug regressions efficiently (bisect)
- Collaborate without stepping on each other's changes
- Maintain a clean, navigable history (interactive rebase, conventional commits)
- Enforce quality gates automatically (hooks)
- Choose the right branching strategy for your team

---

## 2. Core Concepts

### The Git Object Model (DAG)

Git stores everything as **objects** in `.git/objects/`. Every object is identified by a SHA-1 (or SHA-256 in newer git) hash of its content. There are four object types:

```
blob     — the raw content of a single file
tree     — a directory listing (maps filenames to blob/tree SHAs)
commit   — a snapshot pointing to a tree + parent commit(s) + metadata
tag      — a named pointer to a specific commit (annotated tags)
```

A commit looks like this internally:
```
commit abc1234
  tree  def5678        ← points to the root tree snapshot
  parent bbb9876       ← previous commit (none for the first commit)
  author Alice <alice@example.com> 1704067200 +0000
  committer Alice <alice@example.com> 1704067200 +0000

  Add user authentication
```

Each tree object:
```
tree def5678
  blob aaa1111  README.md
  blob bbb2222  package.json
  tree ccc3333  src/         ← nested tree for the src/ directory
```

**Key insight:** Commits are **immutable snapshots**, not diffs. When you change one file and commit, Git creates a new tree with a new blob for the changed file and re-uses blobs for unchanged files (via SHA-based deduplication). This is what makes Git fast: checking out a branch is just changing which tree SHA is active.

**Branches and HEAD are just pointers:**
```
main       → abc1234 (commit SHA)
feature/x  → def5678 (different commit SHA)
HEAD       → main    (HEAD points to the current branch)
           → abc1234 (when detached)
```

Branches are stored as files in `.git/refs/heads/`. A branch is literally a 41-byte file containing a commit SHA. Creating a branch is O(1) — almost free.

---

### The Three Areas

```
Working Tree           Staging Area (Index)        Repository (.git/)
─────────────────      ──────────────────────      ──────────────────
Your actual files      What will go into            Committed snapshots
as you edit them       the next commit              (permanent history)

git add ──────────────────────────>
                 git commit ──────────────────────>
git checkout ───────────────────────────────────────────────────<
```

Understanding this model explains why:
- `git diff` shows working tree vs staging area
- `git diff --staged` shows staging area vs last commit
- `git reset HEAD~1` moves HEAD without touching the working tree (by default)
- `git stash` saves working tree + staging area and restores both to last commit

---

## 3. How It Works

### Branching Strategies

**Trunk-Based Development (TBD):**
```
main: A──B──C──D──E──F──G──H (everyone commits here, no long-lived branches)
               ↑         ↑
        feature flag  feature flag
            on             off

Characteristics:
- Everyone integrates to main at least once per day
- Incomplete features hidden behind feature flags
- Continuous deployment — every commit on main is deployable
- Requires excellent test coverage and CI
- Best for: experienced teams, microservices, high-deployment frequency
```

**GitHub Flow:**
```
main:      A─────────────────G──── (always deployable)
                   ↑         ↑
feature/x: B──C──D──E  (short-lived, PR to main, then deleted)
                   ↑
              Code review, CI checks, then merge

Characteristics:
- Feature branches live 1-3 days at most
- PR required to merge to main
- Simple: two "permanent" states — working and deployed
- Best for: web apps, small/medium teams, frequent deploys
```

**Git Flow:**
```
main:     A─────────────────────────E── (production releases only)
           \                       /
develop:   B──C──D──F──G──H──I──J──

           ↑ feature branches ↑    ↑ release branch ↑
feature/x: C──D (branch from develop, merge back to develop)
release/1.2:   H──I (branch from develop, merge to main + develop)
hotfix/bug:         K──L (branch from main, merge to main + develop)

Characteristics:
- Heavyweight — many branch types with strict rules
- Good for: versioned software with release cycles (mobile apps, libraries)
- Overkill for most web applications
```

> 💡 **Choose based on your deploy frequency.** If you deploy multiple times per day, TBD or GitHub Flow. If you have quarterly releases with version numbers, Git Flow.

---

### Rebase vs Merge

**Merge** — preserves complete history, creates a merge commit:
```
Before:
main:    A──B──C
              \
feature:       D──E

After git merge feature (on main):
main:    A──B──C──────F  ← merge commit, two parents (C and E)
              \      /
feature:       D──E
```

**Rebase** — replays commits on top of target, creates linear history:
```
Before:
main:    A──B──C
              \
feature:       D──E (D,E based on B — before C was added)

After git rebase main (on feature):
main:    A──B──C
                 \
feature:          D'──E' (D' and E' are NEW commits with new SHAs)
```

Then fast-forward merge gives clean linear history:
```
After git merge --ff-only feature:
main:    A──B──C──D'──E'
```

**When to rebase:**
- Before merging a feature branch to main — clean up commits, place them on top of latest main
- To update a long-running branch with changes from main: `git rebase main`
- Interactive rebase to squash WIP commits before PR

> ⚠️ **Never rebase commits that have been pushed to a shared branch.** Rebase creates new commits (new SHAs). If your teammates have the old commits, their history diverges from yours and reconciliation is painful. The golden rule: only rebase commits that exist only on your local machine (or on branches only you use).

---

### Interactive Rebase

`git rebase -i HEAD~N` opens an editor to manipulate the last N commits:

```
pick abc1234 Add user model
pick def5678 Fix typo in user model
pick ghi9012 Add user controller
pick jkl3456 WIP: half-done feature
pick mno7890 Finish the feature
```

Commands available:
```
pick    — use commit as-is
reword  — use commit but edit its message
squash  — meld into previous commit (combine messages)
fixup   — meld into previous commit (discard this commit's message)
edit    — pause rebase here, let you amend the commit
drop    — remove this commit
reorder — drag lines to reorder commits
```

Example — clean up before PR:
```
pick abc1234 Add user model
fixup def5678 Fix typo in user model     ← folded into abc1234 silently
pick ghi9012 Add user controller
squash jkl3456 WIP: half-done feature    ← combined with mno7890
squash mno7890 Finish the feature        ← editor prompts for combined message
```

Result: a clean two-commit history ready for PR review.

---

### Git Bisect — Binary Search for Bugs

`git bisect` does a binary search through your commit history to find which commit introduced a bug. This is one of the most underused and powerful Git features.

```bash
# Start bisect
git bisect start

# Mark current commit as bad (has the bug)
git bisect bad

# Mark a commit you know was good (no bug)
git bisect good v2.0.0   # or a specific SHA: git bisect good abc1234

# Git checks out the midpoint commit
# You test it — run your test, check the bug

# If this commit has the bug:
git bisect bad

# If this commit is good:
git bisect good

# Git continues narrowing until it finds the first bad commit
# It prints: "abc1234 is the first bad commit"

# Clean up
git bisect reset

# Automate with a script
git bisect run npm test   # runs npm test; exit 0 = good, non-zero = bad
```

With 1000 commits, git bisect finds the bad commit in at most 10 steps (log2(1000) ≈ 10).

---

### Stash

```bash
git stash                          # save working tree + index to stash
git stash push -m "WIP: half-done feature"   # with descriptive message
git stash push -p                  # interactive — choose which hunks to stash
git stash push -- path/to/file.ts  # stash specific file only

git stash list                     # show all stashes
git stash show stash@{0}           # show diff of latest stash
git stash show -p stash@{1}        # show full patch

git stash pop                      # apply latest stash and remove it from list
git stash apply stash@{1}          # apply specific stash (keep in list)
git stash drop stash@{0}           # remove specific stash without applying
git stash clear                    # remove ALL stashes
git stash branch new-branch        # create branch from stash (useful if stash conflicts)
```

---

### Reflog — Recovering Lost Commits

The reflog is a local journal of where HEAD and branch tips have been. It records every time HEAD moves (checkout, reset, rebase, merge, commit, amend).

```bash
git reflog                         # show HEAD reflog
git reflog show main               # show reflog for main branch

# Output:
# abc1234 HEAD@{0}: commit: Add feature X
# def5678 HEAD@{1}: checkout: moving from feature/x to main
# ghi9012 HEAD@{2}: rebase (finish): ...
# jkl3456 HEAD@{3}: commit: WIP commit that was lost

# Recover a "lost" commit after a bad reset
git checkout jkl3456               # go to the commit (detached HEAD)
git checkout -b recovery           # create a branch to preserve it
# or cherry-pick it:
git cherry-pick jkl3456
```

> 💡 Git almost never truly loses data. If you did a `git reset --hard` and lost commits, `git reflog` will almost certainly show you the SHA to recover from. Reflog entries expire after 90 days by default.

---

### Git Hooks

Hooks are scripts in `.git/hooks/` that run automatically at specific points in the Git workflow.

```
pre-commit       — before git commit creates the commit (lint, tests, formatting)
prepare-commit-msg — after the commit message file is created, before the editor
commit-msg       — validates the commit message (e.g., enforce Conventional Commits)
post-commit      — after commit is created (notifications, local automation)
pre-push         — before pushing (run full test suite, prevent pushing secrets)
post-merge       — after a successful merge (run npm install if package.json changed)
pre-rebase       — before rebasing (safety checks)
```

```bash
# Example: pre-commit hook that runs ESLint
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
set -e

# Get staged .ts/.tsx files
STAGED=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(ts|tsx)$' || true)
[ -z "$STAGED" ] && exit 0

echo "Running ESLint on staged files..."
echo "$STAGED" | xargs npx eslint --max-warnings=0
EOF
chmod +x .git/hooks/pre-commit
```

> 💡 For team-shared hooks, use **husky** (Node.js projects) or place hooks in a `.githooks/` directory and configure with `git config core.hooksPath .githooks`. Hooks in `.git/hooks/` are not committed to the repository.

```bash
# Configure shared hooks directory
git config core.hooksPath .githooks
# Now .githooks/pre-commit is tracked and shared via the repo
```

---

### Conflict Resolution

```bash
# When a merge/rebase has conflicts:
git status                    # shows conflicted files with 'both modified'

# Conflicted file looks like (git inserts these markers automatically):
# <<< HEAD
# const x = 1;               ← your change (current branch)
# ===
# const x = 2;               ← incoming change (being merged in)
# >>> feature/x

# Resolution options:
# 1. Edit the file manually — remove markers, keep what you want
# 2. git checkout --ours file.ts   — take your version
# 3. git checkout --theirs file.ts — take incoming version
# 4. git mergetool                 — open configured visual merge tool

# After resolving all conflicts:
git add resolved-file.ts
git commit                    # for merge
git rebase --continue         # for rebase (or --skip / --abort)
```

---

## 4. Code Examples

### Useful aliases
```bash
# Add to ~/.gitconfig
[alias]
  lg = log --oneline --graph --all --decorate
  s  = status -s
  d  = diff
  ds = diff --staged
  undo = reset HEAD~1
  unstage = reset HEAD --
  last = log -1 HEAD --stat
  aliases = config --get-regexp '^alias'
  contributors = shortlog -sn --no-merges
  branches = branch -vv --sort=-committerdate
```

```bash
# Quick log with graph
git log --oneline --graph --all
# * abc1234 (HEAD -> main) Add authentication
# * def5678 Refactor user service
# | * ghi9012 (feature/payments) Add payment processing
# |/
# * jkl3456 Initial commit

# See what changed in last commit
git show HEAD --stat

# See who last changed each line of a file
git blame -L 10,30 src/auth.ts   # lines 10-30 only
git blame --follow src/auth.ts   # follow renames

# Search commit messages
git log --grep="fix" --oneline

# Search all commits for when a string was introduced/removed
git log -S "secretToken" --all   # all branches

# See commits by author
git shortlog -sn --no-merges

# Compare two branches
git diff main...feature/x        # what's in feature/x but not main
git log main..feature/x          # commits in feature/x not in main
```

### .gitignore patterns
```gitignore
# Dependencies
node_modules/
vendor/
.venv/

# Build outputs
dist/
build/
*.js.map

# Environment files (NEVER commit these)
.env
.env.*
!.env.example       # exception — keep the template

# Logs
*.log
logs/

# OS files
.DS_Store
Thumbs.db

# Editor files
.idea/
.vscode/
*.swp
*.swo

# Coverage reports
coverage/
.nyc_output/

# Match any depth
**/*.env
**/node_modules/

# TypeScript (if you don't want to commit compiled output)
*.js       # careful — only if you never commit plain JS files
!jest.config.js
```

> ⚠️ If you accidentally committed `node_modules` before adding `.gitignore`:
> ```bash
> git rm -r --cached node_modules
> echo "node_modules/" >> .gitignore
> git add .gitignore
> git commit -m "chore: remove node_modules from tracking"
> ```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Committing secrets**: API keys, passwords, and tokens committed to git are compromised forever — they persist in history even after you delete them. Use `git log -S "secret"` to find where they were. Rotate the keys immediately after discovering a leak. Tools: `git-secrets`, `truffleHog`, `gitleaks` to scan for secrets. Use `.gitignore` for `.env` files and environment-based configuration.

> ⚠️ **Force-pushing to shared branches**: `git push --force` rewrites remote history. Anyone who has fetched that branch now has a diverged history. Use `git push --force-with-lease` which fails if the remote has commits you have not seen (safer). Never force-push to `main` or `develop`.

> ⚠️ **Huge commits**: Atomic commits are easier to review, easier to revert, and easier to bisect. A commit that changes 15 files across 3 different concerns is hard to review and impossible to cherry-pick. Keep commits focused on a single logical change.

> ⚠️ **Poor commit messages**: Commits are documentation. `"fix bug"` tells you nothing. A good message: `"fix(auth): handle token expiry during concurrent requests"`. Follow Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`).

> ⚠️ **Ignoring `.gitignore` until after committing `node_modules`**: The repository grows by hundreds of megabytes, `git status` becomes unusably slow, and git operations take forever. Add `.gitignore` before the first commit. Use `git rm -r --cached` to remove already-tracked files.

> ⚠️ **Rebasing shared branches**: Never rebase commits that have been pushed and that others have pulled. It creates diverged histories and forces teammates to do painful reconciliation with `git rebase --onto` or by resetting their local branches.

---

## 6. When to Use / Not Use

**Use `merge` when:**
- Integrating a feature branch into `main` (preserve the full feature branch history)
- On public/shared branches (never rebase shared branches)

**Use `rebase` when:**
- Updating your local feature branch with changes from main (before opening PR)
- Interactive rebase to clean up WIP commits before PR review
- Creating a linear history for a feature before merging

**Use `stash` when:**
- You need to quickly switch context (pull, checkout, urgent hotfix) with uncommitted changes
- You want to test something without the uncommitted changes present

**Use `cherry-pick` when:**
- You need a specific commit from another branch (backport a fix to a release branch)
- You accidentally committed to the wrong branch

**Use `bisect` when:**
- You know a bug was introduced in a commit between version X (good) and HEAD (bad)
- The bug is testable programmatically (use `git bisect run`)

**Use `reflog` when:**
- You accidentally reset, deleted a branch, or lost commits

---

## 7. Real-World Scenario

### Using git bisect to find a performance regression

**Situation:** The application was fast in v1.5.0 (released 3 months ago, 200 commits back). It is now slow. You need to find which commit introduced the regression.

```bash
# Write an automated test for the regression
cat > test-performance.sh << 'EOF'
#!/bin/bash
# Returns 0 (good) if fast, non-zero (bad) if slow
npm install --silent
DURATION=$(node -e "
  const start = Date.now();
  const result = require('./src/utils/processor').process(testData);
  console.log(Date.now() - start);
")
[ "$DURATION" -lt 100 ]  # good if under 100ms
EOF
chmod +x test-performance.sh

# Start bisect
git bisect start

# Mark current HEAD as bad
git bisect bad

# Mark v1.5.0 as good
git bisect good v1.5.0

# Automated bisect — git runs the script on each midpoint
git bisect run ./test-performance.sh

# Git output:
# Bisecting: 100 revisions left to test after this (roughly 7 steps)
# [sha1] ...
# Bisecting: 50 revisions left to test after this (roughly 6 steps)
# ...
# abc1234 is the first bad commit
# Author: Dave <dave@example.com>
# Date: Thu Jan 18 14:32:11 2024 +0000
#     refactor(processor): simplify loop logic

# Inspect the offending commit
git show abc1234

# Clean up
git bisect reset

# Now you know exactly which commit and which author introduced the regression
# git blame the affected lines, understand the change, and fix it
```

---

### Setting up a pre-commit hook with shared configuration

```bash
# Create .githooks directory (committed to repo, shared with team)
mkdir .githooks

cat > .githooks/pre-commit << 'EOF'
#!/bin/sh
set -e

STAGED_TS=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(ts|tsx)$' || true)
STAGED_ALL=$(git diff --cached --name-only --diff-filter=ACMR)

# Run ESLint on staged TypeScript files
if [ -n "$STAGED_TS" ]; then
  echo "ESLint..."
  echo "$STAGED_TS" | xargs npx eslint --max-warnings=0
fi

# Run Prettier check on all staged files
if [ -n "$STAGED_ALL" ]; then
  echo "Prettier..."
  echo "$STAGED_ALL" | xargs npx prettier --check --ignore-unknown
fi

echo "Pre-commit checks passed."
EOF
chmod +x .githooks/pre-commit

cat > .githooks/commit-msg << 'EOF'
#!/bin/sh
# Enforce Conventional Commits format
MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,100}"
if ! echo "$MSG" | grep -Eq "$PATTERN"; then
  echo "ERROR: Commit message does not follow Conventional Commits format."
  echo "Expected: type(scope): description"
  echo "Example:  feat(auth): add OAuth2 login"
  exit 1
fi
EOF
chmod +x .githooks/commit-msg

# Configure git to use .githooks/ for all team members
# (add this to README or onboarding script)
git config core.hooksPath .githooks
```

---

## 8. Interview Questions

**Q1: What is git rebase and when should you use it?**

A: Rebase replays commits from one branch on top of another, creating new commits with new SHAs. The result is a linear history as if the branch was created from the target's current tip, not from where it was originally branched. Use rebase: (1) Before merging a feature branch to main — `git rebase main` on your feature branch places your commits after the latest main commits; (2) Interactive rebase to clean up WIP commits (squash, reword, drop) before a PR; (3) Updating a long-running feature branch with changes from main. Never rebase commits already pushed to a shared branch — it rewrites history and forces teammates to reconcile diverged histories.

---

**Q2: How do you recover a deleted branch?**

A: Use `git reflog` to find the SHA of the latest commit on the deleted branch. Reflog records every move of HEAD, including when you checked out the branch and when it was deleted. `git reflog | grep "branch-name"` shows relevant entries. Then `git checkout -b recovered-branch <SHA>` recreates the branch from that commit. This works because deleting a branch only removes the pointer file in `.git/refs/heads/` — the commit objects remain until garbage collection (which does not run on reachable reflog entries, which expire after 90 days by default).

---

**Q3: What is git bisect?**

A: Git bisect is a binary search through commit history to find the commit that introduced a bug. You tell git `bisect start`, mark the current state as bad (`bisect bad`) and a known-good state as good (`bisect good v1.5.0`). Git checks out the midpoint commit. You test it and tell git `bisect good` or `bisect bad`. Git halves the search space each time. With N commits, it finds the culprit in log₂(N) steps. The power: `git bisect run <script>` automates the testing — git runs your script on each midpoint, using its exit code to determine good/bad, requiring zero human interaction.

---

**Q4: Explain the trade-offs between merge and rebase.**

A: **Merge** preserves the true history — when branches diverged, how they were integrated, who worked on what. The downside is merge commits that can clutter the log in a busy repository. **Rebase** produces a clean, linear history that is easier to read and reason about. The downside is that it rewrites history (new SHAs), which is dangerous for shared branches, and it loses the information about when work was done concurrently. Merge is safer and correct; rebase is cleaner. Most teams rebase local feature branches before merging to main, but merge when integrating to keep traceability.

---

**Q5: What is a detached HEAD state?**

A: Normally, HEAD points to a branch name (e.g., `ref: refs/heads/main`), which in turn points to the latest commit on that branch. When you `git checkout <SHA>` or `git checkout v1.5.0`, HEAD points directly to a commit SHA rather than to a branch. This is "detached HEAD". You can make commits in this state, but they have no branch tracking them. When you checkout another branch, those commits appear to be lost (they are reachable via reflog but no branch points to them). Fix: `git checkout -b new-branch` while in detached HEAD to create a branch that captures your work.

---

**Q6: How do you undo the last commit?**

A: It depends on what you want to undo and whether the commit has been pushed. For a local, unpushed commit: (1) `git reset HEAD~1` — moves HEAD back one commit, keeps changes in working tree (staged or not depending on `--mixed`/`--soft`/`--hard`). (2) `git reset --soft HEAD~1` — keeps changes staged. (3) `git reset --hard HEAD~1` — discards all changes (destructive). For a pushed commit: (4) `git revert HEAD` — creates a new commit that undoes the previous one (safe for shared branches, preserves history). Never use `reset --hard` on pushed commits unless you intend to force-push, which is dangerous on shared branches.

---

**Q7: What is a git hook?**

A: A git hook is a script that runs automatically at a specific point in the Git workflow. Hooks live in `.git/hooks/` (or a configured `core.hooksPath` directory). Key hooks: `pre-commit` (run linting/formatting before the commit is created), `commit-msg` (validate commit message format), `pre-push` (run tests before pushing). Hooks are not committed with the repository by default (`.git/` is not tracked). To share hooks with the team, store them in a committed directory (e.g., `.githooks/`) and configure `git config core.hooksPath .githooks`. Or use husky in Node.js projects.

---

**Q8: Explain the git object model.**

A: Git stores four object types: blobs (file content), trees (directories mapping names to blob/tree SHAs), commits (snapshots pointing to a tree + parent commit + metadata), and annotated tags (named pointers to commits). All objects are identified by a SHA hash of their content — identical content produces the same SHA, providing automatic deduplication. Branches and HEAD are just pointers (text files containing a SHA). Creating a branch is O(1). Commits are immutable — you cannot change a commit, only create new ones. `git rebase` creates new commits (new SHAs) with new parents; the old commits remain in the object store until garbage-collected.

---

## 9. Exercises

**Exercise 1 — Git bisect to find a bug:**

1. Create a repository with 20+ commits, one of which introduces a bug (e.g., a function returns wrong output)
2. Practice `git bisect start`, mark current as bad, mark commit 1 as good
3. Manually identify the bad commit step by step
4. Reset and do it again with `git bisect run` using a test script

Hint: Create a test script that `exit 0` on success and `exit 1` on failure. `git bisect run sh test.sh`.

---

**Exercise 2 — Pre-commit hook for ESLint:**

Set up a pre-commit hook that:
- Identifies staged `.ts` and `.tsx` files
- Runs `eslint --max-warnings=0` on them
- Fails the commit if ESLint reports any errors or warnings
- Skips the check if no TypeScript files are staged

Place it in `.githooks/pre-commit`, make it executable, and configure `git config core.hooksPath .githooks`.

---

**Exercise 3 — Git log alias:**

Create a git alias `lg` that shows:
- One line per commit
- ASCII graph of branches and merges
- All branches (local and remote)
- Colored output with branch/tag names
- Relative dates

```bash
git config --global alias.lg "log --oneline --graph --all --decorate --color"
# Enhance with:
git config --global alias.lg "log --graph --format='%C(yellow)%h%C(reset) %C(green)(%ar)%C(reset) %C(bold blue)%an%C(reset) %C(white)%s%C(reset)%C(red)%d%C(reset)' --all"
```

Test by creating a few branches and commits to see the graph in action.

---

**Exercise 4 — Resolve a merge conflict deliberately:**

1. Create a branch from main
2. On main: change line 5 of a file and commit
3. On the branch: change the same line 5 differently and commit
4. Try to merge the branch into main — it will conflict
5. Open the conflicted file and resolve manually (keep one version)
6. Stage the resolved file and complete the merge
7. Use `git log --graph --oneline` to verify the merge commit appears

Bonus: repeat with `git rebase main` instead of merge, resolve the rebase conflict, and compare the resulting histories.

---

## 10. Further Reading

- **Pro Git** (free online): https://git-scm.com/book/en/v2 — the official comprehensive Git book by Scott Chacon
- **"Git Internals"** (from Pro Git, Chapter 10): https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain — how git works under the hood
- **"A Successful Git Branching Model"** by Vincent Driessen (Git Flow): https://nvie.com/posts/a-successful-git-branching-model/
- **"Please stop recommending Git Flow"** (Trunk-Based Development case): https://georgestocker.com/2020/03/04/please-stop-recommending-git-flow/
- **Trunk-Based Development**: https://trunkbaseddevelopment.com — comprehensive guide
- **Conventional Commits specification**: https://www.conventionalcommits.org/ — commit message standard
- **git-scm.com reference**: https://git-scm.com/docs — official man pages
- **Oh Shit, Git!**: https://ohshitgit.com — plain-English guide to undoing common mistakes
- **Learn Git Branching** (interactive): https://learngitbranching.js.org — visual, interactive tutorial
- **Husky** (Git hooks for Node.js): https://typicode.github.io/husky/
