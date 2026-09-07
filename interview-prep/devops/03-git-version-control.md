# Git & Version Control — DevOps / Backend Interview Questions

---

### 1. Explain the difference between `git merge` and `git rebase`.
`merge` combines two branch histories by creating a new merge commit with two parents, preserving the exact history of both branches (non-destructive, but the log can get "noisy" with merge commits). `rebase` replays your branch's commits one by one on top of the target branch, producing a linear history as if you'd branched off the latest commit — cleaner history, but it rewrites commit hashes, which is dangerous on shared/public branches (anyone who already pulled the old commits will get conflicts/duplicated history).

### 2. What is the "golden rule" of rebasing?
Never rebase commits that have already been pushed to a shared branch that others might have based work on. Rebasing rewrites history (new commit SHAs), so anyone with the old commits will diverge; rebase freely on your own local/feature branches before sharing them, but use `merge` (or a rebase followed by a coordinated force-push with the team's full awareness) once others depend on the branch.

### 3. Explain `git cherry-pick` and a real use case.
`git cherry-pick <sha>` applies the changes from a single specific commit onto your current branch as a new commit. Common use case: a critical bug fix was committed to `main` but you need it in a `release/2.3` branch without merging all of `main`'s other unreleased commits — cherry-pick just that fix commit onto the release branch (a classic hotfix backport).

### 4. What's the difference between `git reset --soft`, `--mixed`, and `--hard`?
`--soft` moves the branch pointer to the target commit but leaves the index (staging area) and working directory unchanged — your changes stay staged, ready to be re-committed differently. `--mixed` (default) moves the pointer and resets the index, but leaves working directory files untouched — changes become unstaged. `--hard` moves the pointer and resets both the index and working directory — all changes since the target commit are discarded, which is destructive and unrecoverable except via `git reflog`.

### 5. How do you recover a commit after an accidental `git reset --hard` or a deleted branch?
`git reflog` records every place `HEAD` has pointed, even after resets, rebases, and branch deletions (it's a local safety net, not pushed to remotes, and eventually garbage-collected after ~90 days by default). Find the SHA of the commit you lost in the reflog output, then `git checkout <sha>` or `git reset --hard <sha>` (or `git branch recovered-branch <sha>`) to get it back.

### 6. Explain `git stash` and its common flags.
`git stash` temporarily shelves uncommitted changes (both staged and unstaged, by default) so you can switch context (e.g., to fix an urgent bug on another branch) with a clean working directory, then reapply them later. `git stash pop` applies and removes the most recent stash; `git stash apply` applies without removing (useful if you might need it again); `git stash -u` also stashes untracked files; `git stash list`/`git stash show -p stash@{0}` inspect stashes; `git stash push -m "msg" -- path/` stashes selectively with a label.

### 7. What is a detached HEAD state, and is it dangerous?
Normally `HEAD` points to a branch, which points to a commit. A detached HEAD means `HEAD` points directly to a commit (e.g. after `git checkout <sha>` or `<tag>`) rather than a branch. It's not inherently dangerous, but any *new* commits made in this state aren't referenced by any branch — if you switch away without creating a branch first, those commits become unreachable (though still recoverable briefly via reflog) and eventually get garbage collected. `git switch -c newbranch` from a detached HEAD saves the work under a real branch name.

### 8. Explain `git bisect` and when you'd use it.
`git bisect` performs a binary search through commit history to find which commit introduced a bug. Start with `git bisect start`, mark a known-bad commit (`git bisect bad`) and a known-good one (`git bisect good <sha>`); git checks out the midpoint commit, you test and mark it `good`/`bad`, and it narrows the range logarithmically until it identifies the exact offending commit. `git bisect run <script>` automates this if you have a script that exits non-zero on failure.

### 9. What is the difference between `git fetch` and `git pull`?
`git fetch` downloads new commits/branches/tags from the remote into your local repo's remote-tracking branches (e.g. `origin/main`) without touching your working branches — safe, non-destructive, lets you inspect changes before integrating. `git pull` is `git fetch` followed immediately by a `merge` (or `rebase`, with `git pull --rebase`) into your current branch — convenient but can surprise you with unexpected merge commits or conflicts if you weren't expecting new remote changes.

### 10. How do you resolve a merge conflict?
Git marks conflicting sections in the affected files with `<<<<<<<`, `=======`, `>>>>>>>` markers showing both versions. Edit the file to the correct final content, remove the markers, then `git add <file>` to mark it resolved, and complete the operation with `git commit` (for a merge) or `git rebase --continue` (for a rebase). `git status` always shows which files still have unresolved conflicts; `git diff` shows the conflict markers; tools like `git mergetool` or IDE 3-way merge views make this easier for complex conflicts.

### 11. What's the difference between `.gitignore` and `git rm --cached`?
`.gitignore` prevents *untracked* files matching a pattern from being added/shown in `git status` going forward — it has no effect on files that are already tracked. `git rm --cached <file>` untracks a file that's *already* committed (removes it from the index/future commits) while leaving it on disk — typically done together with adding it to `.gitignore` when you realize a file (like `.env` or `node_modules`) was accidentally committed. Note the file remains in the repo's history unless you rewrite history (`git filter-repo`/BFG).

### 12. How do you completely remove a sensitive file (e.g., a committed secret) from git history?
Simply deleting and committing isn't enough — it still exists in earlier commits' history and is retrievable. Use `git filter-repo` (modern, recommended) or BFG Repo-Cleaner to rewrite history and strip the file/string from every commit, then force-push and have every collaborator re-clone (rewritten history is incompatible with their existing clones). Critically, the secret must also be *rotated/invalidated* — rewriting git history doesn't undo any exposure that already happened (anyone who cloned/forked before the cleanup, or crawlers, may already have a copy).

### 13. Explain Git's three "areas": working directory, staging area (index), and repository.
Working directory — your actual files on disk that you edit. Staging area/index — a snapshot of changes you've explicitly `git add`ed, which will make up the *next* commit (this decoupling lets you commit only part of your changes). Repository (`.git` directory) — the committed history, i.e. the permanent record of snapshots, refs, and objects.

### 14. What is a Git tag, and annotated vs. lightweight tags?
A tag is a named pointer to a specific commit, typically used to mark releases (`v1.2.0`). A lightweight tag is just a pointer (like a branch that doesn't move). An annotated tag (`git tag -a v1.2.0 -m "..."`) is a full Git object with its own metadata (tagger, date, message) and can be GPG-signed (`git tag -s`) — recommended for releases since it's more auditable and supports verification.

### 15. Explain trunk-based development vs. Git Flow.
**Git Flow** uses long-lived `develop` and `main` branches plus per-feature/release/hotfix branches with formal merge points — structured but can lead to long-lived branches that drift and produce large, painful merges. **Trunk-based development** has everyone commit small, frequent changes directly to (or via very short-lived branches into) a single trunk (`main`), relying on feature flags to hide incomplete work and strong CI to keep trunk always releasable — favored by teams practicing continuous delivery because it minimizes merge conflicts and integration risk.

### 16. What is a monorepo vs. a polyrepo, and what are the tradeoffs?
A monorepo holds many projects/services in a single repository (atomic cross-project commits/refactors, unified tooling/versioning, easier code sharing) but requires investment in tooling to scale (selective builds/tests via tools like Bazel/Nx/Turborepo, since naive CI would rebuild everything on every change) and access control is coarser. A polyrepo gives each project its own repo (clean ownership boundaries, independent versioning/release cadence, simpler CI per repo) but cross-repo changes (e.g. an API contract change affecting five services) require coordinating multiple PRs/releases and dependency version bumps.

### 17. What is `git blame`, and what's a limitation of it?
`git blame <file>` shows which commit and author last modified each line of a file. Limitation: it attributes the *last change* to a line, which can be misleading — e.g., a bulk reformat/rename commit will "blame" the formatter, hiding the original author. `git blame -w` ignores whitespace changes, and `git log --follow -p -- file` or `git log -L <start>,<end>:<file>` can trace a specific line's actual history through renames and reformats.

### 18. How do submodules work, and what's a common pain point?
A submodule embeds another Git repository at a specific commit inside your repo (a pointer, not the actual files, are committed). Common pain points: submodules don't auto-update on `git pull` (need `git submodule update --init --recursive`), it's easy to commit a submodule pointer referencing a commit nobody else has pushed yet, and the workflow is generally considered clunky — many teams prefer package managers, monorepos, or tools like `git subtree` instead.

### 19. Explain what a pre-commit hook is and give a practical example.
A hook is a script Git runs automatically at certain points (`pre-commit`, `pre-push`, `commit-msg`, etc.), stored in `.git/hooks/` locally (not version-controlled by default — tools like the `pre-commit` framework or Husky commit hook *configuration* to the repo and install the actual hooks on `checkout`/`install`). A practical example: a `pre-commit` hook that runs a linter/formatter and blocks the commit (or auto-fixes and re-stages) if code style violations are found, or scans the diff for accidentally-committed secrets before they ever leave your machine.

### 20. What's the difference between `git revert` and `git reset` for undoing a change, and which is safe for a shared branch?
`git revert <sha>` creates a *new* commit that applies the inverse of a previous commit's changes — history is preserved and additive, safe to use on shared/public branches since it doesn't rewrite existing commits. `git reset` moves the branch pointer backward, effectively erasing commits from that branch's history — rewriting shared history this way requires a force-push and will break other collaborators' clones, so it should be limited to local/unshared work.

### 21. How would you structure branch protection / a PR workflow for a production repo?
Protect `main`: require pull requests (no direct pushes), require at least N approving reviews, require status checks (CI: build, test, lint, security scan) to pass before merge, require branches to be up to date with base before merging (or use a merge queue) to prevent "it passed CI on an old base but breaks combined with another PR's changes," disallow force-pushes to protected branches, and optionally require signed commits and linear history (squash or rebase merge only).

### 22. What's the difference between squash merge, rebase merge, and a regular merge commit on a PR?
**Merge commit** — keeps every individual commit from the branch plus a merge commit; full fidelity but a noisier `main` history. **Squash merge** — combines all of a branch's commits into a single new commit on `main`; clean, linear, one-commit-per-feature history, but loses the intermediate commit-by-commit history (still available on the closed PR itself on most platforms). **Rebase merge** — replays each individual commit from the branch onto `main` without a merge commit, preserving granular history in a linear log; requires each individual commit to be clean/buildable since there's no merge commit to "absorb" fixups.

### 23. How do you handle a large binary file in Git without bloating the repository?
Use Git LFS (Large File Storage) — it stores a small text pointer in the actual Git history/commits while the real binary content lives in separate LFS storage, fetched on demand. Without LFS, every version of a binary you ever commit stays in the repo's `.git` history forever (binaries don't diff/compress well), causing clone times and repo size to balloon; this is hard to fix retroactively and generally requires history rewriting once discovered.

### 24. Explain semantic versioning (SemVer) and how it interacts with Git tags in a release pipeline.
SemVer format is `MAJOR.MINOR.PATCH` (e.g. `2.4.1`): increment MAJOR for breaking/incompatible API changes, MINOR for backward-compatible new functionality, PATCH for backward-compatible bug fixes. In CI/CD, a Git tag matching this pattern (`v2.4.1`) is commonly the trigger for a release pipeline (build artifact, publish package/image, generate changelog) — tools like `semantic-release` can even compute the next version automatically by parsing conventional commit messages (`feat:`, `fix:`, `BREAKING CHANGE:`) since the last tag.

### 25. What's the difference between `origin` and `upstream` in a fork-based contribution workflow?
`origin` conventionally refers to *your* fork (the remote you cloned and push your feature branches to). `upstream` conventionally refers to the original repository you forked from, added as a second remote (`git remote add upstream <url>`) so you can pull in the latest changes from the canonical project (`git fetch upstream && git merge upstream/main`) to keep your fork in sync without needing write access to the original repo.

### 26. How do you write a good commit message, and why does it matter for a DevOps team?
Convention: a short (~50 char) imperative-mood summary line ("Fix race condition in cache invalidation," not "Fixed" or "Fixes"), a blank line, then a body explaining *why* the change was made (motivation, tradeoffs, links to tickets/incidents) rather than restating *what* the diff already shows. It matters operationally because commit history is a primary tool during incident postmortems (`git log`, `git blame`) to understand why a risky-looking line of code exists, and tools that auto-generate changelogs/release notes (conventional commits) depend on structured, meaningful messages.

### 27. What happens during `git gc`, and why does a repository sometimes need it?
`git gc` (garbage collection) compresses loose objects into packfiles, removes genuinely unreachable objects (those not referenced by any branch/tag/reflog and past expiry), and optimizes internal data structures — run automatically by Git periodically, or manually (`git gc --aggressive`) when a repo has accumulated many loose objects (e.g., after heavy rebasing/history rewriting) and feels slow or bloated on disk.

### 28. In a CI pipeline, why might you use `git fetch --depth=1` (a "shallow clone")?
A shallow clone fetches only the most recent commit(s) (`--depth=1`) instead of the entire history, dramatically speeding up checkout time in CI where you typically only need the current code state to build/test, not full history. Caveat: some operations (computing a diff against an older commit, `git describe`, certain versioning tools that count commits since the last tag) don't work correctly on a shallow clone without fetching more history (`git fetch --unshallow` or `--deepen`) first.

### 29. Two engineers force-pushed conflicting rewritten history to a shared feature branch. How do you recover?
First, don't panic and don't force-push again blindly. Use `git reflog` on each affected local clone to find the pre-rewrite commit SHAs (reflog is local, so whichever machine still has the old state is the source of truth — CI logs or previous CI-checked-out SHAs can also help reconstruct it if reflog already expired locally). Reconstruct the intended combined state (often by cherry-picking or creating a merge of the two divergent tips), verify with the team, then push once, communicating clearly so nobody else force-pushes over the fix. Long-term: enable branch protection / require PRs even on shared feature branches for critical work, and educate on the "don't rebase shared history" rule.

### 30. Explain how `git log --graph --oneline --all` helps you and what you're looking for.
It renders an ASCII graph of commit history across all branches/refs in one compact line per commit — useful for visually understanding branch topology: where a feature branch diverged from `main`, whether it's been merged, whether there are unexpected extra merge commits, or whether a branch has drifted far behind `main` (many commits on `main` not present in the feature branch) before attempting a rebase or merge.
