# Git & GitHub

Git is a distributed version control system — it tracks changes to files over time so you can go back, branch off, and merge work without stepping on anyone else's toes. GitHub is a hosting service for Git repos, plus collaboration tooling (PRs, issues, Actions) built on top.

## TL;DR
- Git tracks snapshots of your project as commits, organized into branches.
- Almost everything happens locally — commits, branches, history — until you `push`/`pull` to sync with a remote like GitHub.
- Branching + merging is what makes parallel work (features, fixes, experiments) possible without chaos.
- `rebase` rewrites history for a clean line; `merge` preserves it with a merge commit. Know which one your team uses.
- Most "git disasters" are recoverable — `reflog` is your safety net.

## How it works

Git stores content as a graph of snapshots, not diffs. Three areas matter:

```
working directory  →  staging area (index)  →  repository (commits)
      (edit)              (git add)              (git commit)
```

- **Working directory**: your actual files.
- **Staging area**: what you've marked to include in the next commit (`git add`).
- **Repository**: the committed history, stored in `.git/`.

Each commit is a snapshot + metadata (author, message, parent commit hash). A **branch** is just a movable pointer to a commit. `HEAD` points to whatever branch/commit you're currently on.

## 🟢 Beginner: the everyday loop

```bash
git status                 # what's changed?
git add file.js            # stage a file
git add .                  # stage everything
git commit -m "message"    # snapshot the staged changes
git push origin main       # send commits to GitHub
git pull origin main       # fetch + merge remote changes
```

Branching for a feature:

```bash
git checkout -b feature/login   # create + switch to new branch
# ...make changes, commit...
git push -u origin feature/login
# open a Pull Request on GitHub
```

**Cloning vs forking**: `git clone` copies a repo to your machine. Forking (a GitHub feature, not a Git concept) copies a repo to *your own GitHub account* — used when you don't have write access to the original and want to contribute via PR.

## 🟡 Intermediate: branching, merging, and undoing things

### Merge vs rebase

| | Merge | Rebase |
|---|---|---|
| History | Preserves both branches' history, adds a merge commit | Replays your commits on top of target branch — linear history |
| Safety | Safe on shared branches | Never rebase a branch others are pulling from — rewrites hashes |
| Use case | Merging feature branches into main | Cleaning up a feature branch before opening a PR |

```bash
git checkout main
git merge feature/login          # merge commit created if diverged

git checkout feature/login
git rebase main                  # replay feature commits onto latest main
```

Golden rule: **rebase local/private branches, merge shared/public ones.**

### Undoing things

```bash
git restore file.js              # discard unstaged changes to a file
git restore --staged file.js     # unstage (keep the edits)
git commit --amend               # fix the last commit's message/contents
git reset --soft HEAD~1          # undo last commit, keep changes staged
git reset --hard HEAD~1          # undo last commit, DISCARD changes (careful)
git revert <commit>              # make a new commit that undoes another (safe on shared history)
```

`reset` rewrites history (dangerous on pushed/shared commits). `revert` adds a new commit — safe to use on anything, including main.

### Stashing

Need to switch branches but aren't ready to commit?

```bash
git stash            # shelve current changes
git stash pop        # bring them back
git stash list        # see all stashes
```

### Resolving conflicts

A conflict happens when Git can't auto-merge two changes to the same lines. Git marks it inline:

```
<<<<<<< HEAD
your version
=======
their version
>>>>>>> branch-name
```

Edit the file to keep what you want, remove the markers, then `git add` + `git commit` (or `git rebase --continue` if mid-rebase).

## 🔴 Advanced

- **`git reflog`**: a log of everywhere `HEAD` has pointed, even after resets/rebases. If you "lose" commits, `git reflog` almost always has them — `git reset --hard <hash-from-reflog>` recovers.
- **Interactive rebase** (`git rebase -i HEAD~5`): squash, reorder, edit, or drop commits before pushing — used to clean up messy commit history into a readable PR.
- **`git cherry-pick <commit>`**: apply one specific commit from another branch onto your current one.
- **`git bisect`**: binary-search through commit history to find which commit introduced a bug.
- **Detached HEAD**: happens when you checkout a commit directly instead of a branch — you're not "on" any branch, so new commits can get lost unless you create a branch from there.
- **`.gitignore`**: patterns for files Git should never track (`node_modules/`, `.env`, build artifacts). Already-tracked files need `git rm --cached` to actually stop being tracked.
- **Submodules vs monorepos**: submodules pin another repo at a specific commit inside yours — powerful but famously fiddly (people tend to avoid them if a monorepo or package manager works instead).

## GitHub-specific concepts

- **Pull Request (PR)**: proposes merging one branch into another, with a review/discussion UI on top of a diff.
- **Fork + PR workflow**: standard pattern for contributing to repos you don't own — fork it, branch, commit, push to your fork, open a PR back to the original.
- **GitHub Actions**: CI/CD defined in `.github/workflows/*.yml`, triggered by events (push, PR, schedule).
- **Protected branches**: require PR review / passing checks before merging to `main` — prevents force-pushes and direct commits.
- **CODEOWNERS**: auto-assigns reviewers based on which files changed.

## Common pitfalls

- Committing secrets (`.env`, API keys) — once pushed, treat the secret as compromised even if you delete it later (it's in history). Use `git filter-repo` or GitHub's secret scanning + rotate the key.
- Force-pushing (`git push --force`) to a shared branch — overwrites others' work. Use `--force-with-lease` if you must, which fails safely if someone else pushed since you last fetched.
- Committing `node_modules`/build output because `.gitignore` was added too late.
- Large binary files bloating repo size — Git doesn't diff binaries well; use Git LFS if you must track them.
- Confusing `git fetch` (downloads but doesn't merge) with `git pull` (fetch + merge in one step).

## Quick reference

```bash
git log --oneline --graph --all     # visualize branch history
git diff                            # unstaged changes
git diff --staged                   # staged changes
git branch -d branch-name           # delete local branch (safe, checks merged)
git branch -D branch-name           # force delete
git remote -v                       # list remotes
git tag v1.0.0                      # tag a release point
```

## Further reading
- [Git cheatsheet — cs.fyi](https://cs.fyi/guide/git-cheatsheet)
- [roadmap.sh: Git & GitHub](https://roadmap.sh/git-github)
- [Learn Git Branching (interactive)](https://learngitbranching.js.org/)
