# Git Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## Setup & Config

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "vim"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global rebase.autoStash true
git config --list --show-origin       # show all config with source
git config --global alias.lg "log --oneline --graph --decorate --all"
```

---

## Daily Workflow

```bash
# Status & info
git status
git status -s                         # short format
git diff                              # unstaged changes
git diff --staged                     # staged changes
git diff HEAD                         # all changes since last commit
git diff main..feature                # between branches

# Staging
git add file.ts                       # stage specific file
git add src/                          # stage directory
git add -p                            # interactive hunk staging
git add -u                            # stage all tracked changes
git add .                             # stage everything (use carefully)
git restore --staged file.ts          # unstage
git restore file.ts                   # discard working dir changes

# Committing
git commit -m "feat: add user auth"
git commit --amend                    # modify last commit (not pushed)
git commit --amend --no-edit          # amend without changing message
git commit -a -m "fix: quick patch"   # stage tracked + commit

# Pushing
git push origin feature/my-branch
git push -u origin feature/my-branch  # set upstream
git push --force-with-lease           # safe force push
git push --tags                       # push all tags
```

---

## Branching

```bash
git branch                            # list local branches
git branch -a                         # list all (including remote)
git branch feature/new-thing          # create branch
git switch feature/new-thing          # switch to branch
git switch -c feature/new-thing       # create and switch
git switch -c feature/x origin/feature/x  # track remote branch

git branch -d feature/done            # delete (safe — merged only)
git branch -D feature/wip             # force delete
git branch -m old-name new-name       # rename

git push origin --delete feature/done # delete remote branch

# Compare
git log main..feature                 # commits in feature not in main
git diff main...feature               # diff from common ancestor
```

---

## Merging

```bash
git merge feature/my-branch           # merge into current branch
git merge --no-ff feature/my-branch   # always create merge commit
git merge --squash feature/my-branch  # squash to single commit
git merge --abort                     # abort in-progress merge

# After conflict:
# 1. Edit files to resolve conflicts
# 2. git add <resolved-files>
# 3. git commit
```

---

## Rebasing

```bash
git rebase main                       # rebase current branch onto main
git rebase -i HEAD~3                  # interactive rebase last 3 commits
git rebase -i origin/main             # interactive from divergence point
git rebase --abort                    # abort rebase
git rebase --continue                 # continue after resolving conflict
git rebase --skip                     # skip current conflicting commit

# Interactive rebase commands (in editor):
# pick   = use commit as-is
# reword = change commit message
# edit   = pause to amend commit
# squash = meld into previous commit (keep all messages)
# fixup  = meld into previous (discard this message)
# drop   = remove commit entirely
```

---

## Remote Operations

```bash
git remote -v                         # list remotes
git remote add origin git@github.com:user/repo.git
git remote remove origin
git remote rename origin upstream

git fetch                             # fetch all remotes
git fetch origin                      # fetch specific remote
git fetch --prune                     # also remove deleted remote branches

git pull                              # fetch + merge
git pull --rebase                     # fetch + rebase (cleaner history)
git pull origin main                  # pull specific branch

# Tracking
git branch -u origin/main main        # set upstream
git branch -vv                        # show tracking info
```

---

## Stash

```bash
git stash                             # stash working dir + index
git stash push -m "WIP: half-done feature"
git stash push -p                     # interactive (choose hunks)
git stash push --include-untracked    # also stash untracked files

git stash list                        # list all stashes
git stash show stash@{0}              # show diff of stash
git stash show -p stash@{1}           # full patch

git stash pop                         # apply + drop top stash
git stash apply stash@{1}             # apply without dropping
git stash drop stash@{0}              # remove a stash
git stash clear                       # remove all stashes
git stash branch feature/x stash@{0} # create branch from stash
```

---

## History & Log

```bash
git log
git log --oneline                     # one line per commit
git log --oneline --graph --all       # visual branch graph
git log --stat                        # show changed files
git log -p                            # show full patch
git log -n 10                         # last 10 commits
git log --author="Alice"
git log --since="2024-01-01" --until="2024-12-31"
git log --grep="fix:"                 # search commit messages
git log -S "functionName"             # search added/removed string
git log -- path/to/file               # commits touching a file
git log main..HEAD                    # commits not in main

git show HEAD                         # show last commit + diff
git show abc1234                      # show specific commit
git show HEAD:src/app.ts              # show file at commit

# Who changed this line?
git blame file.ts
git blame -L 10,25 file.ts            # specific line range
git log -p -S "suspectFunction" file.ts  # when was it introduced?
```

---

## Undoing Changes

```bash
# Undo working dir changes (destructive)
git restore file.ts                   # discard unstaged changes
git restore .                         # discard all unstaged changes
git clean -fd                         # remove untracked files/dirs
git clean -fdn                        # dry run first

# Undo staged changes
git restore --staged file.ts

# Undo commits (safe — adds new commit)
git revert HEAD                       # revert last commit
git revert abc1234                    # revert specific commit
git revert HEAD~3..HEAD               # revert range

# Reset (moves branch pointer — dangerous if pushed)
git reset --soft HEAD~1               # undo commit, keep staged
git reset --mixed HEAD~1              # undo commit + unstage (default)
git reset --hard HEAD~1               # undo commit + discard changes

# Undo a bad merge
git reset --hard ORIG_HEAD            # after merge (before other commits)
```

---

## Reflog — Your Safety Net

```bash
git reflog                            # every HEAD movement ever recorded
git reflog show feature/my-branch     # reflog for specific branch

# Recover deleted branch
git reflog                            # find the SHA before deletion
git switch -c recovered-branch abc1234

# Recover after bad reset
git reset --hard HEAD@{3}             # go back to 3 moves ago

# Reflog expires (default 90 days for reachable, 30 for unreachable)
git reflog expire --expire=now --all  # manually expire (use with care)
```

---

## Bisect — Find the Commit That Broke Things

```bash
git bisect start
git bisect bad                        # current commit is broken
git bisect good v2.0.0                # last known good commit/tag
# Git checks out midpoint — test it, then:
git bisect good                       # this one works
git bisect bad                        # this one is broken
# Repeat until git prints the bad commit
git bisect reset                      # return to original HEAD

# Automated bisect
git bisect run npm test               # run test suite automatically
```

---

## Tags

```bash
git tag                               # list tags
git tag v1.0.0                        # lightweight tag
git tag -a v1.0.0 -m "Release 1.0"   # annotated tag (use this)
git tag -a v1.0.0 abc1234 -m "Tag old commit"
git push origin v1.0.0                # push single tag
git push origin --tags                # push all tags
git tag -d v1.0.0                     # delete local tag
git push origin --delete v1.0.0      # delete remote tag
git show v1.0.0                       # show tag info
```

---

## Cherry-Pick

```bash
git cherry-pick abc1234               # apply single commit
git cherry-pick abc1234..def5678      # apply range (exclusive start)
git cherry-pick abc1234^..def5678     # apply range (inclusive start)
git cherry-pick -n abc1234            # apply without committing
git cherry-pick --abort
git cherry-pick --continue
```

---

## Submodules

```bash
git submodule add https://github.com/user/lib.git libs/lib
git submodule update --init --recursive   # init + fetch submodules
git submodule update --remote             # update to latest remote
git clone --recurse-submodules <url>      # clone with submodules
```

---

## Useful Aliases

```bash
git config --global alias.st "status -s"
git config --global alias.co "switch"
git config --global alias.br "branch"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.unstage "restore --staged"
git config --global alias.visual "!gitk"
git config --global alias.aliases "config --get-regexp alias"
```

---

## Git Flow Quick Reference

```
main        — production-ready code
develop     — integration branch
feature/*   — new features (branch from develop)
release/*   — release prep (branch from develop, merge to main + develop)
hotfix/*    — urgent fixes (branch from main, merge to main + develop)
```

```bash
# Feature workflow
git switch -c feature/user-auth develop
# ... work ...
git switch develop && git merge --no-ff feature/user-auth
git branch -d feature/user-auth

# Hotfix workflow
git switch -c hotfix/1.0.1 main
# ... fix ...
git switch main && git merge --no-ff hotfix/1.0.1
git tag -a v1.0.1 -m "Hotfix 1.0.1"
git switch develop && git merge --no-ff hotfix/1.0.1
git branch -d hotfix/1.0.1
```

---

## .gitignore Patterns

```gitignore
# Directories
node_modules/
dist/
.cache/

# Files
.env
.env.local
*.log
*.tmp

# Negation (un-ignore)
!.env.example

# Wildcards
**/*.test.js     # any depth
src/**/*.snap    # inside src at any depth

# OS
.DS_Store
Thumbs.db
```
