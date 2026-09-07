# More Linux & Git Questions — Practical Command-Line Depth

> Sourced/topic-checked against Tecmint's Linux sysadmin list and GeeksforGeeks'/InterviewBit's Git interview question lists. Supplements `01-linux-fundamentals.md` and `03-git-version-control.md`.

---

## Linux

### 1. How do you suspend a running foreground process and move it to the background?
Press `Ctrl+Z` to suspend the current foreground process (sends `SIGTSTP`), then run `bg` to resume it running in the background, or `bg %<job-number>` if multiple jobs are suspended. `jobs` lists current background/suspended jobs with their job numbers; `fg %<job-number>` brings a background job back to the foreground. This is a common quick-and-dirty way to reclaim a terminal without killing a long-running command you started in the foreground by mistake.

### 2. What's the minimum partition setup needed to install Linux, and how do you check boot messages afterward?
At minimum, a root partition (`/`) and typically a swap partition/file (though swap is technically optional on systems with abundant RAM); a separate `/boot` partition is common but not strictly required on all setups (needed more often with specific bootloader/encryption/RAID configurations). After installation, boot messages (kernel and early userspace startup logs) are viewable via `dmesg` (kernel ring buffer) or `journalctl -b` (all logs from the current boot, on systemd systems) — useful for diagnosing why a service failed to start at boot or why a piece of hardware wasn't detected.

### 3. Which daemon tracks system events on a modern Linux system, and how do you view its logs?
`systemd-journald` collects and manages system logs (from the kernel, systemd itself, and any service using its logging API or writing to stdout/stderr under a systemd unit) into a structured, indexed binary journal. View it with `journalctl` — `journalctl -u <service>` for one service's logs, `journalctl -f` to follow in real time (like `tail -f`), `journalctl --since "1 hour ago"` for time-bounded queries, and `journalctl -p err` to filter by priority level — a significant improvement over parsing scattered plain-text files in `/var/log` for anything running as a systemd-managed service.

### 4. What should you check/do before running `fsck` on the root partition?
The root filesystem generally can't be checked while mounted read-write and in active use (risk of corrupting a filesystem being actively modified during the check) — `fsck` on root typically needs to run either from a live/rescue boot environment, in single-user/rescue mode, or is scheduled to run automatically at the *next* boot before the root filesystem is mounted read-write (many systems support `shutdown -F` or `touch /forcefsck` to force this on next reboot, or `tune2fs`-configured periodic auto-checks). Always ensure you have current backups before running an `fsck` that reports needing repairs, since automatic repair of a badly corrupted filesystem can occasionally make targeted data recovery harder than working from an unrepaired image.

### 5. How do you copy an entire directory tree while preserving permissions, ownership, and timestamps?
`cp -a src/ dest/` (`-a`/archive mode bundles `-dR --preserve=all`, preserving symlinks, permissions, ownership, and timestamps recursively) for a local copy; `rsync -a src/ dest/` for the same guarantee with resumability and delta-transfer benefits, especially valuable for large trees or over a network (see the main Linux file's `rsync` question) — plain `cp -r` alone does *not* preserve all of these attributes by default, which is a common gotcha when someone expects an exact clone and instead gets a copy with reset permissions/timestamps.

### 6. How do you find the person/job that scheduled a specific one-time job with `at`?
`atq` lists pending jobs queued via `at`, showing job numbers, scheduled times, and the queue letter, but doesn't directly show the submitting user in its default output on every system — combined with checking `/var/spool/cron/atjobs/` (or the equivalent spool directory) and its file ownership, or checking `at -c <job-number>` to view a specific job's actual script content (which often reveals context about who/what scheduled it based on the commands/environment captured within it), you can usually reconstruct both who scheduled it and what it will do.

### 7. How do you view the contents of a tar archive without extracting it?
`tar -tvf archive.tar` (or `-tzvf` for a gzip-compressed `.tar.gz`) lists the archive's contents (file names, permissions, sizes) without extracting anything to disk — useful for quickly checking what's inside a large archive, verifying an expected file is present, or checking for path-traversal-risk entries (files with `../` in their paths) before deciding whether it's safe to extract at all.

### 8. What is a page fault, and when does a "major" vs. "minor" page fault happen?
A page fault occurs when a process accesses a memory page that isn't currently mapped into physical RAM, requiring the kernel to intervene. A **minor** page fault happens when the page exists somewhere accessible quickly (e.g., already in the page cache, or a copy-on-write page that just needs a fresh physical page allocated) — resolved fast, without disk I/O. A **major** page fault requires the kernel to actually read the page in from disk (e.g., swapped-out memory, or a memory-mapped file page not yet loaded) — orders of magnitude slower, and a high rate of major faults is a strong signal of memory pressure/thrashing (see the `vmstat`/swap discussion in the main Linux file).

### 9. What are return codes (exit codes), and how do you use them in scripting?
Every command/process returns a numeric exit status when it finishes — `0` conventionally means success, any non-zero value indicates some kind of failure (with specific non-zero values sometimes carrying specific meaning per-tool, like 137 for a SIGKILL-terminated process, discussed in the Docker file). Immediately after a command, `$?` in bash holds its exit code, which scripts use for conditional logic (`if command; then ... fi` implicitly checks the exit code, since `if` evaluates truthiness based on it) — this is the foundation of `set -e`'s fail-fast behavior discussed in the scripting-automation file.

### 10. What's the difference between `kill -9` and `kill -15`, precisely?
`kill -15` (the default signal `kill` sends, `SIGTERM`) asks the process to terminate, giving it a chance to catch the signal and clean up gracefully (close files, flush buffers, release resources) before exiting. `kill -9` (`SIGKILL`) cannot be caught, blocked, or ignored by the process at all — the kernel terminates it immediately and unconditionally, without giving it any chance to run its own cleanup code, which can leave temp files, locks, or partially-written data in an inconsistent state. Always try `-15` first and reserve `-9` for a process that's genuinely unresponsive/hung and not terminating after a reasonable wait on `-15`.

### 11. How do you check which ports are open and which services are listening on them?
`ss -tulnp` (modern, preferred) or the older `netstat -tulnp` — `-t`/`-u` for TCP/UDP, `-l` for listening sockets only, `-n` for numeric (skip slow DNS reverse-lookups), `-p` to show the owning process name/PID (requires root/sudo to see other users' processes). For checking connectivity to a *remote* host's port rather than what's listening locally, use `nc -zv host port` or `telnet host port` (see the networking file's troubleshooting question).

### 12. What is the OOM Killer, and how does it decide what process to kill?
(See also the main Linux file's OOM question.) When the system is critically low on memory with no way to reclaim more, the kernel's OOM killer selects a process to terminate to free memory rather than letting the whole system deadlock/crash. It computes an `oom_score` for each process (influenced heavily by memory usage — larger memory consumers score higher/riskier by default) adjustable per-process via `/proc/<pid>/oom_score_adj` (a very negative value can effectively protect a critical process from being chosen, though setting it to the extreme `-1000` disables OOM-killing for that process entirely, which should be done sparingly and deliberately, not as a blanket workaround for genuine memory pressure).

### 13. How do you find files modified in the last 24 hours?
`find /path -type f -mtime -1` (`-mtime -1` means modified less than 1 day ago; note the surprising sign convention — `-1` means "less than," `+1` means "more than," no sign means "exactly"). For finer granularity than whole days, `find /path -type f -mmin -60` finds files modified in the last 60 minutes. This is a routine first step when investigating "what changed" during an incident investigation (combined with checking against a known-good deployment timestamp) or when auditing unexpected filesystem changes.

---

## Git

### 14. What does `git ls-tree` do, and when would you use it over `git status`/`git log`?
`git ls-tree <tree-ish>` lists the contents (files and subdirectories, with their mode, type, and object hash) of a specific tree object (a commit, branch, or tag) as Git actually stored it — useful for low-level inspection of exactly what a specific commit/tree contains, independent of your current working directory state, which `git status` (compares working directory to the index) and `git log` (shows commit history/messages) don't directly expose.

### 15. What does a Git commit object actually contain?
A commit object records: a pointer to the tree object representing the complete project snapshot at that commit, one or more parent commit pointers (none for the first commit, two+ for a merge commit), author and committer information (name, email, timestamp — these can differ, e.g., when a commit is cherry-picked or rebased, "author" stays the original creator while "committer" becomes whoever performed that operation), and the commit message — the commit's own hash is a SHA computed over all of this content, which is why any change (even just rebasing without altering the actual code diff) produces a new hash.

### 16. Explain the three levels of Git configuration (system, global, local) and their precedence.
`--system` (`/etc/gitconfig` or equivalent) — applies to every user on the machine, lowest precedence. `--global` (`~/.gitconfig`) — applies to the current user across all their repositories. `--local` (`.git/config` within a specific repository, the default scope when no flag is given) — applies only to that one repository, highest precedence, overriding global/system settings for anything it sets. This layering lets you set sensible personal defaults globally (name, email, default editor) while overriding specific settings (e.g., a different email for a work-specific repo) locally per-project.

### 17. How would you recover a branch that was pushed to the shared/central repository but then accidentally deleted from everyone's local machines?
If it was ever pushed, the central remote (GitHub/GitLab/etc.) almost certainly still has it reachable via its reflog or through the platform's own deleted-branch recovery UI/API for a period after deletion — check the remote platform's own branch-recovery feature first. Failing that, if you know the last commit SHA that branch pointed to (from CI logs, a PR page showing the merge commit, team chat history, or anyone's local reflog who had recently fetched it), you can recreate the branch directly from that SHA (`git branch recovered-branch <sha>`) and push it — this is exactly why understanding `git reflog` (see the main Git file) and knowing that "deleted" in Git rarely means truly gone immediately matters in a real recovery scenario.

### 18. What is the practical advice on fixing a broken commit: create an additional commit, or amend the existing one?
If the broken commit hasn't been pushed/shared yet, amending (`git commit --amend`) is fine and keeps history clean, since nothing else depends on that commit's exact SHA yet. If it has already been pushed and others may have already pulled/based work on it, prefer a new commit (or `git revert` for a full undo) instead of amending — amending rewrites the commit's SHA, and force-pushing an amended already-shared commit causes the same disruption as any other history rewrite of shared work (see the "golden rule of rebasing" question in the main Git file, which applies identically here).

### 19. What is the difference between a "pull request" and a "branch," and why isn't a pull request called a "push request"?
A branch is a Git-native concept — simply a movable pointer to a commit, existing entirely within Git itself, with no concept of review/discussion attached. A "pull request" (or "merge request") is a *platform-level* feature (GitHub/GitLab/Bitbucket, not Git itself) built around requesting that someone review and merge your branch — it's called a "pull" request because, conceptually, you're asking the maintainer of the target repository to *pull* your changes into their branch (echoing Git's underlying `git pull` model of fetching+merging from another party's repository), rather than you having write access to *push* directly into their protected branch yourself.
