# Linux Fundamentals — DevOps Interview Questions

> Junior → Senior. Covers filesystem, processes, permissions, networking basics, performance, and troubleshooting.

---

### 1. What is the difference between a process and a thread?
A process is an independent execution unit with its own memory space, file descriptors, and PID; threads are lightweight execution units *within* a process that share the same memory space and file descriptors but have their own stack and register set. Threads are cheaper to create/switch but a crash in one thread can corrupt the whole process; processes are isolated but heavier (separate address space, IPC needed to communicate).

### 2. Explain the Linux boot process.
BIOS/UEFI (POST, hardware init) → bootloader (GRUB) loads the kernel + initramfs → kernel initializes drivers, mounts the real root filesystem → kernel starts `/sbin/init` (PID 1, systemd on most modern distros) → systemd reads unit files and brings the system to the configured target/runlevel (multi-user.target, graphical.target, etc.), starting services in dependency order.

### 3. What is systemd and how do you manage services with it?
systemd is the init system and service manager on most modern Linux distros. Key commands: `systemctl start/stop/restart/status <svc>`, `systemctl enable/disable <svc>` (controls boot-time startup via symlinks in `/etc/systemd/system/`), `systemctl daemon-reload` (reload unit files after edits), `journalctl -u <svc>` (view logs). Units can be `.service`, `.socket`, `.timer` (cron replacement), `.mount`, `.target` (grouping).

### 4. What's the difference between a hard link and a symbolic (soft) link?
A hard link is a second directory entry pointing to the *same inode* — it shares data blocks, has no concept of "original vs copy," survives deletion of the original name, and cannot cross filesystems or link to directories. A symlink is a separate small file that stores a *path* to the target; it can cross filesystems and link directories, but breaks (dangling link) if the target is moved/deleted.

### 5. Explain Linux file permissions (rwx, octal notation) and special bits.
`r` (4) = read, `w` (2) = write, `x` (1) = execute, for three classes: owner/group/other, e.g. `rwxr-xr--` = 754. Special bits: **SUID** (4000) — execute as file owner (e.g. `passwd`); **SGID** (2000) — execute as group owner, or on a directory, new files inherit the directory's group; **sticky bit** (1000) — on a directory (e.g. `/tmp`), only the file owner/root can delete/rename files inside even if others have write access.

### 6. How do you change ownership and permissions?
`chmod 755 file` or `chmod u+x,go-w file`; `chown user:group file`; `chgrp group file`; add `-R` for recursive. `umask` sets the default permission mask for newly created files (e.g. umask 022 → files default to 644, dirs to 755).

### 7. What is an inode? What happens when you run `rm` on a file that's still open by a process?
An inode stores a file's metadata (permissions, owner, size, timestamps, pointers to data blocks) but not its name — names live in directory entries. `rm` removes the directory entry and decrements the link count; if a process still has the file open (fd table holds a reference), the data blocks aren't freed until the last file descriptor is closed — this is why a deleted log file can still keep growing and consuming disk space until the writing process is restarted (`lsof | grep deleted` finds these).

### 8. Difference between `>`, `>>`, `2>`, `2>&1`, and `&>`.
`>` overwrites stdout to a file; `>>` appends stdout; `2>` redirects stderr; `2>&1` redirects stderr to wherever stdout currently points (must come *after* the stdout redirect to work as expected); `&>file` (bash) redirects both stdout and stderr to a file in one go.

### 9. How do you find which process is using a given port or file?
`sudo lsof -i :8080` or `sudo ss -tulnp | grep 8080` for ports; `lsof /path/to/file` or `fuser /path/to/file` for open files; `lsof -p <pid>` lists everything a process has open.

### 10. Explain `nice` and `renice`. What is process priority?
Linux process priority ranges from -20 (highest) to 19 (lowest); default is 0. `nice -n 10 command` starts a process with lower priority; `renice -n -5 -p <pid>` changes priority of a running process (negative values require root). Lower niceness = more CPU time relative to other processes when the system is under contention.

### 11. What do load average numbers mean (`uptime`, `top`)?
The three numbers are the average number of processes in the *runnable or uninterruptible* (D-state, usually I/O wait) queue over the last 1, 5, and 15 minutes. A load average of 4 on a 4-core machine means the CPUs are, on average, fully utilized with no queue; on an 8-core machine the same number means 50% utilization. Rising load with low CPU% often points to I/O wait (`vmstat`, `iostat` confirm).

### 12. How do you diagnose high CPU usage on a Linux host?
`top`/`htop` sorted by CPU to find the offending PID; `top -H -p <pid>` to see per-thread CPU inside that process; for a specific hot thread, `cat /proc/<pid>/task/<tid>/stack` (kernel stack) or attach `perf top -p <pid>` / `perf record -g -p <pid>` and `perf report` for a flamegraph of user+kernel call stacks; for JVM apps, a thread dump (`jstack`) correlated with the hot native thread ID (converted to hex) pinpoints the exact Java method.

### 13. How do you diagnose high memory usage / an OOM kill?
`free -h` for overall memory/swap; `ps aux --sort=-%mem | head`; `smem` or `/proc/<pid>/status` (VmRSS) for per-process RSS; `dmesg -T | grep -i "killed process"` or `journalctl -k | grep -i oom` to confirm the OOM killer fired and which process/score triggered it (`oom_score_adj` can protect a process). `cat /proc/meminfo` shows cache vs. actually-used memory — Linux uses free RAM aggressively for page cache, so "available" (not "free") is the number that matters.

### 14. Explain `iostat`, `vmstat`, and how you'd find a disk I/O bottleneck.
`iostat -xz 1` shows per-device `%util`, `await` (avg wait time incl. queue), `r/s`/`w/s` — `%util` near 100% with high `await` indicates the disk is the bottleneck. `vmstat 1` shows `b` (processes blocked on I/O), `wa` (%CPU time waiting on I/O), and swap in/out (`si`/`so`) — nonzero `si/so` under memory pressure means active swapping, a major performance killer. `iotop` shows which process is generating the I/O.

### 15. What is swap and when is swapping a problem?
Swap is disk space used as an overflow for RAM when physical memory is exhausted. Some swap usage for cold/inactive pages is normal and harmless; *active* swapping (constant page-in/page-out, high `si`/`so` in `vmstat`) is a severe performance problem because disk is orders of magnitude slower than RAM — the fix is adding memory, reducing memory usage, or tuning `vm.swappiness`.

### 16. Explain the Linux filesystem hierarchy (`/etc`, `/var`, `/usr`, `/proc`, `/tmp`, `/opt`).
`/etc` — system-wide configuration; `/var` — variable data (logs `/var/log`, spool, databases); `/usr` — user-space programs/libraries (the bulk of installed software); `/opt` — optional/third-party self-contained packages; `/tmp` — temporary files, often cleared on reboot; `/proc` — virtual filesystem exposing kernel/process state in real time (not real files on disk); `/sys` — virtual filesystem exposing kernel objects/device tree; `/home` — user home directories.

### 17. What is `/proc` and give three practically useful things in it.
`/proc` is a pseudo-filesystem that exposes kernel and process information as "files." Useful ones: `/proc/cpuinfo` and `/proc/meminfo` (hardware/memory info), `/proc/<pid>/environ` (a running process's environment variables), `/proc/<pid>/fd/` (its open file descriptors, useful for finding what a mystery process has open), `/proc/sys/` (tunable kernel parameters, same as `sysctl`), `/proc/loadavg`.

### 18. How does DNS resolution order work on a Linux host?
Historically controlled by `/etc/nsswitch.conf` (`hosts:` line, e.g. `files dns`) — `/etc/hosts` is checked first, then whatever resolvers `/etc/resolv.conf` lists (usually `systemd-resolved` at 127.0.0.53, or the resolvers pushed by DHCP/cloud metadata). `systemd-resolved` adds its own caching layer; `resolvectl status` shows the effective resolvers per interface.

### 19. What's the difference between TCP and UDP, and when do you use each?
TCP is connection-oriented, guarantees ordered/reliable delivery via acknowledgements and retransmission, and has flow/congestion control — used for HTTP(S), databases, SSH, anything needing correctness. UDP is connectionless, has no delivery guarantee or ordering, and lower overhead/latency — used for DNS, video/audio streaming, gaming, and protocols that implement their own reliability on top (QUIC/HTTP-3 runs over UDP).

### 20. Explain the TCP three-way handshake and four-way termination.
Handshake: client sends `SYN` → server replies `SYN-ACK` → client replies `ACK`; connection is now established. Termination: either side can initiate — sends `FIN` → other side `ACK`s it (and can keep sending data) → when it's also done, sends its own `FIN` → original side `ACK`s — hence "four-way," though it can collapse to three if `FIN`+`ACK` are combined. The side that sends the final `ACK` enters `TIME_WAIT` for 2×MSL to handle delayed duplicate packets.

### 21. What causes a socket to sit in `TIME_WAIT`, and why does a large number of them matter?
`TIME_WAIT` is held by whichever side actively closed the connection, for 2×MSL (often ~60s), to absorb any delayed packets from the old connection and prevent them from being misread by a new connection reusing the same 4-tuple. Large numbers of `TIME_WAIT` sockets (common on high-throughput servers making many short-lived outbound connections) can exhaust the local ephemeral port range; mitigations: connection pooling/keep-alive, `net.ipv4.tcp_tw_reuse=1`, widening the ephemeral port range.

### 22. What does `SO_REUSEADDR` / `SO_REUSEPORT` do?
`SO_REUSEADDR` lets a socket bind to a port that's in `TIME_WAIT` from a previous instance of the same service (common after a quick restart). `SO_REUSEPORT` allows *multiple* independent sockets/processes to bind the exact same address:port simultaneously, with the kernel load-balancing incoming connections/packets across them — used by multi-process servers (nginx, many Go/Node clusters) to scale across cores without a shared accept-queue bottleneck.

### 23. Explain how you'd troubleshoot "the server is unreachable."
Layered check: `ping` (ICMP reachability/latency — but may be blocked by firewalls, so a failure isn't conclusive), `traceroute`/`mtr` (find where in the path packets are dropped), `telnet host port` or `nc -zv host port` (is the TCP port actually open/listening), `curl -v` (application-layer response, TLS handshake details, HTTP status), check the host's own firewall (`iptables -L`, `nftables`, security groups/NACLs in cloud), check the service is actually running/listening (`ss -tulnp`), check DNS resolves to the expected IP.

### 24. What is a zombie process and an orphan process?
A zombie (`Z` state) is a process that has terminated but whose exit status hasn't been reaped by its parent via `wait()` — it holds only a slot in the process table, no real resources, and disappears once the parent reaps it or the parent itself dies (then `init`/systemd adopts and reaps it). An orphan is a process whose parent has died before it — it gets reparented to PID 1, which will reap it when it eventually exits. Many zombies indicate a parent process with a bug in its child-reaping logic.

### 25. Explain Linux signals: SIGTERM vs SIGKILL vs SIGHUP vs SIGINT.
`SIGTERM` (15) — polite request to terminate; the process can catch it and clean up (close connections, flush buffers) before exiting — this is what `kill` sends by default and what orchestrators send before force-killing. `SIGKILL` (9) — cannot be caught, blocked, or ignored; the kernel terminates the process immediately, no cleanup — used as a last resort since it can leave resources/files in an inconsistent state. `SIGHUP` (1) — historically "terminal hung up," often repurposed by daemons to mean "reload your config." `SIGINT` (2) — sent on Ctrl+C, default action is terminate but is commonly caught for graceful shutdown.

### 26. How do containers achieve isolation on Linux (namespaces & cgroups)?
**Namespaces** partition what a process can *see*: PID namespace (own process tree), network namespace (own interfaces/routing table/ports), mount namespace (own filesystem view), UTS (hostname), IPC, user namespace (UID/GID remapping). **cgroups** (control groups) limit and account for what a process can *use*: CPU shares/quota, memory limits (with OOM behavior scoped to the cgroup), block I/O bandwidth, PIDs count. A container is essentially a process (or process tree) running inside a set of namespaces with cgroup limits applied — there's no hypervisor or separate kernel.

### 27. What is the difference between a hardware virtual machine and a container?
A VM virtualizes hardware and runs a full guest OS with its own kernel on top of a hypervisor — strong isolation, higher overhead (each VM has its own kernel/memory footprint, slower boot). A container shares the host kernel and is isolated via namespaces/cgroups — much lighter weight (starts in milliseconds, higher density per host), but isolation is weaker (a kernel-level exploit can potentially escape a container, whereas escaping a hypervisor is much harder).

### 28. What's the difference between `kill -9` and a graceful shutdown, in the context of Kubernetes/systemd?
Orchestrators send `SIGTERM` first and give the process a grace period (Kubernetes' `terminationGracePeriodSeconds`, default 30s) to shut down cleanly — finish in-flight requests, close DB connections, deregister from load balancers/service discovery. If the process hasn't exited by the end of the grace period, `SIGKILL` is sent, which cannot be caught — any cleanup code won't run. Applications should implement a `SIGTERM` handler that stops accepting new work but finishes existing work before exiting.

### 29. How do you check and tune open file descriptor limits?
`ulimit -n` shows the current shell's soft limit; `ulimit -Hn` shows the hard limit; persistent changes go in `/etc/security/limits.conf` (`* soft nofile 65536`) or, for systemd services, `LimitNOFILE=` in the unit file (`/etc/security/limits.conf` is not read by systemd services). Running out of file descriptors ("too many open files") is a common production incident for servers handling many concurrent connections and shows up as `EMFILE` errors in application logs.

### 30. Explain `strace` and `ltrace` — when would you use them?
`strace` traces the system calls a process makes (file opens, network calls, signals) and their return values/errno — invaluable for diagnosing "why does this binary fail with no useful error," permission issues, or figuring out what config files a program actually reads (`strace -f -e trace=open,openat,connect <cmd>`). `ltrace` traces library calls instead. Both add overhead, so use judiciously in production (or on a copy of the workload).

### 31. What is the difference between `apt`/`yum`/`dnf` package management and why does it matter for DevOps?
`apt` (Debian/Ubuntu) and `yum`/`dnf` (RHEL/CentOS/Fedora) are package managers that resolve dependencies and install/upgrade/remove software from repositories. It matters for reproducible infrastructure: pinning exact versions (`apt-get install pkg=1.2.3-1`), understanding how base images differ (Alpine uses `apk` and musl libc instead of glibc, which occasionally breaks compiled binaries), and knowing how to mirror/cache repos for air-gapped or faster CI builds.

### 32. What is a cron job, and what's the syntax `* * * * *`?
Fields, left to right: minute (0-59), hour (0-23), day of month (1-31), month (1-12), day of week (0-6, Sun=0). `0 2 * * *` = every day at 2:00 AM. Managed via `crontab -e` (per-user) or `/etc/cron.d/`. In modern systemd-based DevOps environments, systemd timers (`.timer` units) are often preferred — they integrate with journald logging, support more precise scheduling (`OnCalendar=`), and can express dependencies.

### 33. How would you copy a large file/directory efficiently between two servers?
`rsync -avz --progress src/ user@host:dest/` is generally preferred over `scp` because it's resumable, only transfers changed blocks on subsequent syncs (delta transfer), and preserves permissions/timestamps. For a first-time huge transfer, compressing (`tar czf - dir | ssh host "tar xzf - -C dest"`), using parallel transfer tools, or, in cloud environments, using object storage as an intermediary (upload once, download once) or a snapshot/volume copy can be faster than a single-stream copy.

### 34. Explain `SELinux`/`AppArmor` at a high level.
Both are Linux Mandatory Access Control (MAC) systems that enforce policies beyond standard discretionary Unix permissions — e.g., "this process, even running as root, may only write to these specific paths." SELinux (RHEL-family) uses labels/contexts on every file and process and is policy-driven and very granular but has a reputation for being hard to debug (`ausearch`, `audit2allow` help). AppArmor (Ubuntu/SUSE) is path-based and generally considered simpler to author profiles for. Both are commonly a source of mysterious "permission denied" errors that don't show up in normal `ls -l` output — checking `getenforce`/`dmesg | grep -i denied` (SELinux) or `/var/log/audit/audit.log` is a standard troubleshooting step.

### 35. What does `df -h` vs `du -sh` tell you, and why might they disagree?
`df` reports filesystem-level free/used space as the kernel sees it; `du` walks a directory tree summing file sizes. They can disagree when a file is deleted but still held open by a process (space isn't returned to the filesystem until the fd closes, so `df` shows it used while `du` — which only sees named files — does not), or with sparse files, hard links counted once by `du` per directory tree scanned but occupying one block set, or bind mounts/overlay filesystems that `du` traverses differently than `df` accounts for.

### 36. How do you set up passwordless SSH login, and why is it useful for automation?
Generate a key pair (`ssh-keygen -t ed25519`), copy the public key to the target's `~/.ssh/authorized_keys` (`ssh-copy-id user@host`), ensure `~/.ssh` is `700` and `authorized_keys` is `600` (SSH refuses to use keys if permissions are too open). It's essential for automation (CI runners, Ansible, deployment scripts) so scripts don't need to prompt for or embed passwords, and keys can be scoped/rotated/revoked independently of user passwords.

### 37. What's the difference between a shell script running in `sh` vs `bash`, and why does `#!/bin/sh` sometimes break things?
`sh` on many modern Linux distros is a symlink to `dash` (a minimal, POSIX-focused shell), not `bash`. Bash-specific syntax — arrays, `[[ ]]` conditionals, string manipulation like `${var//search/replace}`, `local` in certain contexts, `function` keyword — will fail or behave differently under `dash`. Always match the shebang to the actual syntax used, and prefer `#!/usr/bin/env bash` when you rely on bashisms.

### 38. What is the difference between environment variables set in `~/.bashrc`, `~/.bash_profile`, and `/etc/environment`?
`/etc/environment` is system-wide, read at login by PAM (not a shell script — no shell syntax like `export` or `$VAR` expansion). `~/.bash_profile`/`~/.profile` runs for login shells (SSH sessions, or a fresh terminal that spawns a login shell). `~/.bashrc` runs for every new *interactive non-login* shell (e.g., a new terminal tab, `bash` invoked inside a script). This distinction is a classic source of "it works when I SSH in but not in my cron job/systemd service," because cron and systemd services don't source any of these by default — env vars must be set explicitly in the crontab or unit file (`Environment=`/`EnvironmentFile=`).

### 39. How do you monitor and rotate log files so disks don't fill up?
`logrotate` (config in `/etc/logrotate.d/`) rotates, compresses, and eventually deletes old logs based on size/age policies (`daily`, `rotate 7`, `compress`, `size 100M`). Applications should log to stdout/stderr in containerized setups and let the container runtime/log driver or a sidecar (Fluent Bit, Filebeat) handle shipping and rotation instead of writing to local files inside the container, since container filesystems are ephemeral.

### 40. Walk through diagnosing "disk is full" and freeing space safely.
`df -h` to confirm which filesystem is full → `du -sh /* 2>/dev/null | sort -rh | head` (or `ncdu` interactively) to find the largest directories → common culprits: log files (`/var/log`), old kernels (`/boot`), Docker images/volumes/build cache (`docker system df`, `docker system prune`), core dumps, package manager caches (`apt clean`, `yum clean all`). Check for deleted-but-open files with `lsof +L1` or `lsof | grep deleted` before assuming space is unrecoverable without a service restart. Always confirm what a file is before deleting, and prefer truncating (`> file.log`) over deleting a log that's actively being written to, to avoid the "deleted but still open" scenario.
