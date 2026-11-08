# 🐧 Linux Interview Prep — Apple IS&T Junior Application Support & SRE

> **Fork of the KodeKloud Linux Basics course notes**, turned into a single interview-prep file.
> Everything below is what you must be able to answer for the Linux round. The original course notes are linked at the [bottom](#-original-course-notes-kodekloud).

**Built from these sources (so you don't have to read them):**
- Apple SRE / DevOps interview experiences shared on r/sre and r/devops
- Google SRE-SE "Linux internals" and Meta Production Engineer prep threads (r/sre)
- r/linuxadmin "Linux engineer interview questions" (hiring managers' own lists)
- `chassing/linux-sysadmin-interview-questions`, `bregman-arie/devops-exercises` (Linux section), `trimstray/test-your-sysadmin-skills`
- SadServers "Linux & Networking screen interview" (the questions recruiters' screens use most)
- The KodeKloud Linux Basics course itself (this repo)

**Frequency legend**
| Mark | Meaning |
|---|---|
| ⭐⭐⭐ | Appears in almost every source **and** in Apple/FAANG threads. Must know cold. |
| ⭐⭐ | Common. Know it well. |
| ⭐ | Occasional. Read once. |

---

## Table of contents
0. [How the Linux round actually works (the pattern)](#0-how-the-linux-round-actually-works-the-pattern)
1. [The 30 questions that show up everywhere](#1-the-30-questions-that-show-up-everywhere)
2. [Shell, commands, redirection](#2-shell-commands-redirection)
3. [Filesystem, inodes, links, permissions](#3-filesystem-inodes-links-permissions)
4. [Processes, signals, fork/exec](#4-processes-signals-forkexec)
5. [Memory, CPU, load average](#5-memory-cpu-load-average)
6. [Kernel, boot, systemd, logs](#6-kernel-boot-systemd-logs)
7. [Storage, disks, LVM](#7-storage-disks-lvm)
8. [Networking & DNS](#8-networking--dns)
9. [Users, SSH, security](#9-users-ssh-security)
10. [Packages & cron](#10-packages--cron)
11. [One-liners & scripting tasks](#11-one-liners--scripting-tasks)
12. [Live troubleshooting scenarios (the real round)](#12-live-troubleshooting-scenarios-the-real-round)
13. [Containers are Linux + SRE mindset questions](#13-containers-are-linux--sre-mindset-questions)
14. [Behavioural questions asked in Apple SRE loops](#14-behavioural-questions-asked-in-apple-sre-loops)
15. [Last-day cheat sheet](#15-last-day-cheat-sheet)

---

## 0. How the Linux round actually works (the pattern)

Across all the sources, Linux questions come in **five shapes**. Prepare for each shape, not just for topics.

| Shape | What it looks like | Example | How to answer |
|---|---|---|---|
| **A. Screen / rapid-fire** | 10–20 short questions, 30 sec each. Often by a recruiter with an answer sheet. | "What is an inode?" "Port for SSH?" "What does `chmod 750` mean?" | One crisp sentence. Don't ramble. |
| **B. Under the hood** | "Explain how X really works." Apple threads: *"not just how you do things, how it works under the hood."* | "What happens when you run `ls`?" "What is load average made of?" "How does `kill -9` differ?" | Mechanism first (kernel, syscalls, states), then the command. |
| **C. Live debugging** | A pretend/real broken server. *"A pretend Linux server debugging session"* (Apple). | "Server is slow / disk full / app unreachable / can't SSH." | Clarify → check (say the commands) → cause → **mitigate** → fix → prevent. Think out loud. |
| **D. One-liners** | Write a command or 5-line script on CoderPad. | "Top 10 IPs in access.log." "Delete logs older than 7 days." | `grep/awk/sort/uniq/find` combos. Correct beats clever. |
| **E. Behavioural inside the technical round** | Apple interviewers open with a soft-skill question. | "Customer is pressuring you during an incident…" | STAR, 60–90 seconds. |

**Three things interviewers said they reward (from the threads):**
1. Saying **"I don't know, here's how I'd find out"** (man pages, logs, Google) instead of bluffing.
2. **Thought process in the terminal** over memorised flags.
3. An **"I'll figure it out from the logs"** mindset.

**Universal troubleshooting script (say this in every scenario):**
> "First I'd confirm the symptom and scope, check if anything changed recently, then restore service (mitigate) as fast as possible, then find root cause, then add monitoring/automation so it doesn't repeat."

---

## 1. The 30 questions that show up everywhere
If you only have one hour, do this section. Short answers; details are in later sections.

1. **What is load average, and is 8 on a 4-core box bad?** ⭐⭐⭐ Avg number of processes running/runnable **plus** in uninterruptible I/O wait over 1/5/15 min. 8 on 4 cores = 2× oversubscribed → bad; check if CPU-bound (`us`) or I/O-bound (`wa`, D-state).
2. **What is an inode?** ⭐⭐⭐ Per-file metadata structure: type, permissions, owner, size, timestamps, link count, data-block pointers. **Not the filename** (that's in the directory entry). `ls -i`, `stat`, `df -i`.
3. **Hard link vs soft link; what if the original is deleted?** ⭐⭐⭐ Hard = another name for the same inode, survives deletion, same filesystem only, not for dirs. Soft = file holding a path, breaks on deletion, crosses filesystems, can point to dirs.
4. **`df` says disk full but `du` says not. Why?** ⭐⭐⭐ A process holds a **deleted file open**; space is freed only when the FD closes. `lsof +L1` / `lsof | grep deleted` → restart the process or truncate via `/proc/<pid>/fd/<n>`.
5. **"No space left on device" but `df -h` shows space.** ⭐⭐⭐ **Inodes exhausted** (`df -i`), millions of tiny files.
6. **What is a zombie? Can you kill it?** ⭐⭐⭐ A finished child whose parent hasn't called `wait()`; only a PID-table entry. You **can't** kill it (already dead); fix/kill the **parent** so PID 1 reaps it.
7. **`kill -9` vs `kill -15`? Which signals can't be caught?** ⭐⭐⭐ 15 SIGTERM = polite, app can clean up. 9 SIGKILL = kernel ends it immediately, no cleanup. **SIGKILL and SIGSTOP cannot be caught, blocked or ignored.** Reload config = **SIGHUP** (1).
8. **Process states?** ⭐⭐⭐ R running/runnable · S interruptible sleep · **D uninterruptible sleep (I/O; can't be killed)** · Z zombie · T stopped.
9. **What happens when you type `ls` and press Enter?** ⭐⭐⭐ Shell parses → finds `/usr/bin/ls` in `$PATH` → `fork()` child → `execve()` replaces it with ls → ls syscalls (`openat`, `getdents`, `write`) → exit → parent `wait()` gets status → `$?`.
10. **`fork()` vs `exec()`?** ⭐⭐⭐ fork copies the current process (child gets PID, returns 0 in child / child PID in parent). exec replaces the process image with a new program; a successful exec never returns.
11. **What is swap? Does "free memory" exist?** ⭐⭐⭐ Swap = disk used as overflow RAM. Linux uses spare RAM for **page cache** (`buff/cache`), so "free" is low on purpose; look at **available**.
12. **What is the OOM killer and how does it choose?** ⭐⭐⭐ When RAM+swap are exhausted the kernel kills a process by `oom_score` (mostly memory usage, adjusted by `oom_score_adj`). Evidence: `dmesg -T | grep -i oom`.
13. **Explain the boot process.** ⭐⭐⭐ BIOS/UEFI POST → bootloader GRUB2 (`/boot`) loads kernel + initramfs → kernel inits hardware, mounts root → **PID 1 systemd** starts targets/services.
14. **What does `chmod 750` mean on a file vs a directory?** ⭐⭐⭐ Owner rwx, group r-x, others none. On a dir, `x` = can enter, `r` = can list, `w` = can create/delete inside.
15. **SUID / SGID / sticky bit?** ⭐⭐⭐ SUID: run as file owner (`/usr/bin/passwd`). SGID: run as group / new files inherit dir group. Sticky on dir: only owner can delete own files (`/tmp`, `drwxrwxrwt`).
16. **`>` vs `>>` vs `2>&1`?** ⭐⭐⭐ Overwrite · append · send stderr to wherever stdout goes. `cmd > out.log 2>&1`.
17. **Which process is using port 8080?** ⭐⭐⭐ `ss -tulnp | grep 8080` or `lsof -i :8080` (older: `netstat -tulnp`).
18. **What is `/proc`?** ⭐⭐⭐ Virtual FS in memory exposing kernel + process info (`/proc/<pid>/status|fd|limits|cmdline`, `/proc/meminfo`, `/proc/cpuinfo`, `/proc/sys` tunables).
19. **What happens when you type a URL in a browser?** ⭐⭐⭐ DNS (cache → `/etc/hosts` → resolver → root → TLD → authoritative) → TCP 3-way handshake :443 → TLS handshake → HTTP request → LB/CDN → app → DB → response → render.
20. **TCP 3-way handshake; TCP vs UDP?** ⭐⭐⭐ SYN → SYN-ACK → ACK. TCP reliable/ordered (HTTP, SSH, DB); UDP fire-and-forget (DNS, streaming).
21. **DNS record types?** ⭐⭐⭐ A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), NS, PTR (reverse), TXT, SOA.
22. **How does `traceroute` work?** ⭐⭐ Sends packets with TTL 1, 2, 3…; each router decrements TTL and returns ICMP "time exceeded", revealing hops.
23. **I added my key to `authorized_keys` but still get a password prompt.** ⭐⭐⭐ Wrong permissions (`~/.ssh` 700, `authorized_keys` 600, home not group-writable), wrong user/key, `PubkeyAuthentication no`, SELinux context. Debug `ssh -vvv` + server `/var/log/secure`.
24. **How do you make an app a service that restarts on failure and starts at boot?** ⭐⭐⭐ systemd unit in `/etc/systemd/system/` with `ExecStart`, `User`, `Restart=on-failure`, `WantedBy=multi-user.target` → `daemon-reload` → `enable --now`.
25. **Where are logs? How do you follow one live?** ⭐⭐⭐ `journalctl -u svc -f`, `/var/log/messages|syslog`, `/var/log/secure|auth.log`, app logs; `tail -f`.
26. **What is `/etc/fstab`?** ⭐⭐⭐ Filesystems mounted at boot: device/UUID, mount point, type, options, dump, fsck. Test with `mount -a`; a bad line can break boot.
27. **Extend a full LVM volume.** ⭐⭐⭐ `vgs` free? → `lvextend -L +10G /dev/vg/lv` → `resize2fs` (ext4) / `xfs_growfs` (xfs) → or `lvextend -r`. No VG space: `pvcreate` + `vgextend`.
28. **Install a package on RHEL and Ubuntu.** ⭐⭐⭐ `yum/dnf install -y pkg` vs `apt install pkg`; low level `rpm -ivh` / `dpkg -i` (no dependency resolution).
29. **Schedule a job every 15 minutes.** ⭐⭐⭐ `*/15 * * * * /path/job.sh >> /var/log/job.log 2>&1` via `crontab -e`.
30. **"My system is slow." What do you do?** ⭐⭐⭐ The 60-second check: `uptime`, `dmesg -T | tail`, `top`, `free -h`, `df -h; df -i`, `iostat -x 1`, `vmstat 1`, `ss -tulnp`, logs, `curl localhost`. Full version in §12.

---

## 2. Shell, commands, redirection

**Q: Internal vs external command?** ⭐ Built-ins (`cd`, `echo`, `pwd`, `export`) are part of bash; externals (`ls`, `cp`, `grep`) are binaries found via `$PATH`. Check with `type cmd`.

**Q: `X=1` vs `export X=1`?** ⭐⭐ Plain assignment is local to the shell; `export` puts it in the environment so child processes inherit it. Persist in `~/.profile` / `~/.bashrc`.

**Q: "command not found" for an installed tool (e.g. `ifconfig -a`).** ⭐⭐ Not in `$PATH` (`/sbin` often isn't for normal users), not installed (`net-tools`), or typo. `which`, `type`, `echo $PATH`, run with full path, or `export PATH=$PATH:/sbin`.

**Q: What do Ctrl-C, Ctrl-Z, Ctrl-D do?** ⭐⭐ Ctrl-C sends SIGINT (interrupt). Ctrl-Z sends SIGTSTP (stop/suspend → `bg`/`fg`). Ctrl-D sends EOF (logout / end input).

**Q: What does `&` do? `nohup`? `& disown`?** ⭐⭐ `&` runs in background (still tied to the terminal). `nohup cmd &` ignores SIGHUP so it survives logout. `disown` removes the job from the shell's job table. For real services use systemd instead.

**Q: What does `yes | apt-get install x` do and how does it stop?** ⭐ `yes` prints "y" forever into the pipe; when `apt-get` exits, the pipe closes and `yes` gets SIGPIPE and dies.

**Q: Single vs double quotes?** ⭐⭐ Double quotes expand `$VAR` and backticks; single quotes are literal.

**Q: What is an exit code? Common ones?** ⭐⭐ Integer a command returns; `$?`. 0 success, 1 general error, 2 misuse, 126 not executable, 127 not found, 128+N killed by signal N (137 = SIGKILL/OOM, 143 = SIGTERM).

**Q: `tee`?** ⭐⭐ Writes stdin to both screen and file: `./run.sh 2>&1 | tee -a run.log`.

**Q: Difference between `locate` and `find`?** ⭐⭐ `locate` searches a prebuilt DB (fast, may be stale → `updatedb`). `find` walks the filesystem live with filters (name, size, mtime, type, `-exec`).

**Q: `grep -v`, `-i`, `-r`, `-w`, `-A/-B`, `-c`, `-l`?** ⭐⭐⭐ invert, ignore case, recursive, whole word, context after/before, count, filenames only.

**Q: Globbing: `ls [XYZ]`, `[^XYZ]`, `[0-5]`, `*`, `?`** ⭐ one of X/Y/Z · not X/Y/Z · 0–5 · any string · one char.

**Q: What is in `/etc/services`?** ⭐ Standard port-name mappings (ssh 22/tcp, http 80/tcp).

**Q: vi: how to save, quit without saving, search, delete a line, undo?** ⭐⭐ `:wq`, `:q!`, `/text` (`n` next), `dd`, `u`. `i` insert, `Esc` back to command mode.

---

## 3. Filesystem, inodes, links, permissions

**Q: Describe the filesystem hierarchy.** ⭐⭐⭐ `/etc` configs · `/var` variable data (`/var/log`) · `/home` & `/root` · `/tmp` temp (sticky) · `/opt` third-party · `/bin /sbin /usr` programs · `/lib` libraries · `/dev` devices · `/proc /sys` virtual · `/boot` kernel+GRUB · `/mnt /media` mounts.

**Q: What's special about `/tmp`?** ⭐⭐ World-writable with the **sticky bit**, so users can only delete their own files; usually cleared at reboot (often tmpfs).

**Q: What is in an inode / not in an inode?** ⭐⭐⭐ In: type, mode/permissions, UID/GID, size, atime/mtime/ctime, link count, block pointers. **Not in: filename, path.**

**Q: Why can't you hard-link a directory? Why do directories have ≥2 links?** ⭐ Would create loops in the tree; `.` inside the dir is a link to itself, `..` in each child points to it.

**Q: File types (first char of `ls -l`)?** ⭐⭐ `-` regular, `d` directory, `l` symlink, `b` block device, `c` character device, `s` socket, `p` named pipe. `file <name>` identifies.

**Q: What are major and minor numbers?** ⭐ On device files: major = driver/type (8 = SCSI disks `sd*`), minor = specific device/partition. `ls -l /dev/sda*`.

**Q: What does `chmod +x file` do?** ⭐⭐⭐ Adds execute for user, group and others (subject to umask).

**Q: What is umask?** ⭐⭐ Bits removed from default perms. umask 022 → files 644, dirs 755. umask 027 → 640 / 750.

**Q: What are ACLs? When?** ⭐ Per-user/group permissions beyond owner/group/other: `setfacl -m u:bob:rw file`, `getfacl`. Use when several teams need different access to one path.

**Q: Immutable bit?** ⭐ `chattr +i file` → nobody (even root) can modify/delete until `chattr -i`. `lsattr` shows it. Classic "why can't root edit this file" trick question.

**Q: You try to create a file and it fails. Three reasons?** ⭐⭐ No write permission on the directory · disk full · **inodes full** · read-only filesystem (remounted after errors) · immutable attribute · quota.

**Q: Someone ran `chmod -x $(which chmod)`. Fix?** ⭐ Run the loader directly: `/lib64/ld-linux-x86-64.so.2 /bin/chmod +x /bin/chmod`, or `cp` the binary via `perl`/`python` `os.chmod`, or reinstall coreutils.

**Q: How do you force an fsck at next boot?** ⭐ `touch /forcefsck` (older) or `tune2fs -c 1`, or `fsck` from rescue mode. Never fsck a mounted FS.

**Q: What does the `.` or `+` after permissions mean (`-rw-r--r--.`)?** ⭐ `.` = SELinux context present; `+` = ACL present.

---

## 4. Processes, signals, fork/exec

**Q: Process vs thread?** ⭐⭐⭐ Process has its own address space; threads share the process's memory and FDs, are cheaper to create/switch. A crash in one thread can take the whole process.

**Q: Parent and child after `fork()`: what's shared, what's copied?** ⭐⭐ Child gets a copy of memory (copy-on-write), open FDs, env; new PID, PPID = parent. `fork()` returns 0 in the child and the child's PID in the parent; −1 on failure (PID or memory limit hit).

**Q: Why `fork()` + `exec()` instead of one "spawn" call?** ⭐ Between fork and exec the child can set up redirections, close FDs, change user/cwd: that's how shells implement `cmd > file`.

**Q: Why is `wait()` needed? How is the parent told the child died?** ⭐⭐ To collect the exit status and free the process-table entry; otherwise a zombie remains. Kernel sends **SIGCHLD** to the parent.

**Q: What is the init process / PID 1's job regarding orphans?** ⭐⭐ Orphaned processes are re-parented to PID 1 (systemd), which `wait()`s for them, so orphans don't become permanent zombies.

**Q: How do you kill a process in D state?** ⭐⭐⭐ You generally **can't**; it's waiting inside the kernel for I/O (disk, NFS). Fix the underlying I/O (storage, NFS server, unplug) or reboot. Many D processes = storage problem.

**Q: Default signal of `kill`? List signals you know.** ⭐⭐⭐ SIGTERM (15). SIGHUP 1 (reload), SIGINT 2 (Ctrl-C), SIGKILL 9, SIGSEGV 11, SIGPIPE 13, SIGTERM 15, SIGCHLD 17, SIGSTOP 19 / SIGCONT 18 (pause/resume), SIGTSTP 20 (Ctrl-Z).

**Q: What does `kill -0 <pid>` do? `kill 0`?** ⭐ `-0` sends no signal, just checks the PID exists / you have permission. `kill 0` sends SIGTERM to every process in the current process group (careful).

**Q: What is a trap in bash?** ⭐ `trap 'cleanup' EXIT INT TERM` runs a handler on a signal (cleanup temp files). You cannot trap SIGKILL.

**Q: How do you see which process uses the most CPU/memory?** ⭐⭐⭐ `top` (`P`/`M` to sort), `ps -eo pid,ppid,user,%cpu,%mem,rss,stat,cmd --sort=-%cpu | head`, `ps --sort=-rss`.

**Q: How much memory does a process use? RSS vs VSZ?** ⭐⭐ `ps -o rss,vsz -p <pid>`, `top`, `/proc/<pid>/status` (VmRSS). RSS = resident in RAM (what matters); VSZ = virtual address space (includes mapped libs, not all in RAM).

**Q: How does `ps` work?** ⭐⭐ It reads `/proc/<pid>/*` for every PID; no daemon involved.

**Q: Change a process's priority? Why?** ⭐⭐ `nice -n 10 cmd`, `renice -n 5 -p <pid>` (−20 highest … 19 lowest). Lower priority for batch jobs so interactive/production work isn't starved.

**Q: A process stops every couple of days. How do you investigate?** ⭐⭐ `journalctl -u svc` around the time, `dmesg` (OOM?), app logs, exit code (`systemctl status` shows it), cron jobs at that time, resource trend (memory leak), `Restart=` policy, core dumps, `strace`/`ltrace` if reproducible.

**Q: What are `strace` and `ltrace`?** ⭐⭐⭐ `strace -p <pid>` shows system calls (what a hung/erroring process is doing: `open` on a missing file, `connect` timing out). `ltrace` shows library calls. `strace -c` summarises.

**Q: You ran a binary and nothing happened. Debug?** ⭐⭐ `echo $?`, `strace ./bin`, `ldd ./bin` (missing libs), `file ./bin` (wrong arch), logs, run in foreground with verbose flag, check it daemonised (`ps`).

**Q: `ls` gives "/lib/ld-linux-armhf.so.3 no such file". Problem?** ⭐ Binary is for another architecture (ARM) / dynamic loader missing. `file`, `ldd`.

**Q: What is a daemon?** ⭐⭐ Background process detached from any terminal, usually started by systemd, often named with a trailing `d` (`sshd`, `crond`).

**Q: What is a file descriptor? 0/1/2?** ⭐⭐⭐ Integer handle for an open file/socket/pipe per process. 0 stdin, 1 stdout, 2 stderr. See `ls -l /proc/<pid>/fd`; limits `ulimit -n`, `/proc/<pid>/limits`.

**Q: Kill the process writing to a file when you only know the path.** ⭐⭐ `lsof /path/file` or `fuser -k /path/file`.

**Q: `umount` says target is busy. Who holds it?** ⭐⭐ `fuser -vm /mnt/x` or `lsof +D /mnt/x` → stop/kill → `umount` (or `umount -l` lazy as last resort).

**Q: How do you limit a process's memory/CPU?** ⭐⭐ `ulimit -v` (per shell), systemd `MemoryMax=`/`CPUQuota=` in the unit, or **cgroups** directly: same mechanism containers use.

---

## 5. Memory, CPU, load average

**Q: Explain `free -h` columns.** ⭐⭐⭐ `used` by processes · `free` untouched · `buff/cache` kernel cache (reclaimable) · **`available`** what apps can actually get. Swap row: used swap > 0 and growing = pressure.

**Q: What is virtual memory? Paging? Page fault?** ⭐⭐ Each process sees its own address space; the kernel maps pages to RAM via page tables. Minor fault = page in RAM, needs mapping; **major fault** = must read from disk (swap/file), slow. `vmstat` `si/so`, `sar -B`.

**Q: RAM usage keeps rising but throughput is flat. What's happening?** ⭐⭐ (r/linuxadmin "interview question #1") Either page cache growing (normal, reclaimable, check `available`) or a **memory leak** in a process (RSS climbing over days → restart as mitigation, heap dump/profiling as fix, add alerting on RSS).

**Q: CPU load vs CPU utilisation?** ⭐⭐⭐ Utilisation = % of time CPUs are busy *now*. Load = queue length (runnable + D-state). Load can be high with low utilisation (I/O wait) and vice versa.

**Q: What do us / sy / wa / st / id mean in `top`?** ⭐⭐⭐ user code · kernel · **I/O wait** · **steal** (hypervisor took the CPU: noisy neighbour on VMs) · idle.

**Q: Explain `iostat -x` output.** ⭐⭐ `r/s w/s` IOPS, `rkB/s wkB/s` throughput, **`await`** avg latency ms, `avgqu-sz`/`aqu-sz` queue, **`%util`** busy % (near 100 = saturated).

**Q: Explain `vmstat 1`.** ⭐⭐ `r` run queue (> cores = CPU contention), `b` blocked on I/O, `si/so` swap in/out (should be 0), `us sy id wa st`, `cs` context switches, `in` interrupts.

**Q: What is a context switch and why is too many bad?** ⭐ CPU saving one task's state and loading another's; costs cache misses and kernel time. Thousands/sec per core → too many threads/processes or lock contention.

**Q: How do you measure how long a program takes?** ⭐ `time cmd` (real/user/sys), `/usr/bin/time -v cmd` for memory too.

---

## 6. Kernel, boot, systemd, logs

**Q: What is the kernel? What does it do?** ⭐⭐⭐ Core of the OS: process scheduling, memory management, device drivers, filesystems, networking, syscalls/security. Linux is monolithic but modular.

**Q: User space vs kernel space; how does a user program read a disk?** ⭐⭐⭐ Apps can't touch hardware; they issue a **system call** (`read()`), CPU switches to kernel mode, kernel talks to the driver, copies data back, returns.

**Q: What is a kernel module? Load/unload/list? Persist on boot?** ⭐⭐ Loadable driver/feature. `lsmod`, `modprobe mod`, `modprobe -r mod`, `modinfo`; persist in `/etc/modules-load.d/*.conf`; blacklist in `/etc/modprobe.d/`.

**Q: `sysctl`: list, change, persist?** ⭐⭐ `sysctl -a`, `sysctl -w net.ipv4.ip_forward=1` (= `echo 1 > /proc/sys/net/ipv4/ip_forward`), persist in `/etc/sysctl.d/*.conf` then `sysctl -p`. Changing namespaced sysctls inside a container doesn't change the host; non-namespaced ones are read-only in containers.

**Q: What does `echo 1 > /proc/sys/net/ipv4/ip_forward` do?** ⭐⭐ Turns the box into a router (forwards packets between interfaces). Needed for NAT gateways, Docker/K8s networking.

**Q: Boot process in detail.** ⭐⭐⭐ Power → BIOS/UEFI POST → finds boot device → **GRUB2** (menu, `/boot/grub2/grub.cfg`, kernel cmdline in `/proc/cmdline`) → loads **vmlinuz** + **initramfs** → kernel decompresses, inits CPU/memory/drivers, mounts initramfs root, loads storage drivers, pivots to real `/` → starts **PID 1 systemd** → reaches `default.target` (multi-user/graphical) starting units in parallel → getty/login.

**Q: Runlevels vs targets? Single-user mode?** ⭐⭐ 3 = `multi-user.target`, 5 = `graphical.target`, 1 = `rescue.target`, emergency = `emergency.target`. `systemctl get-default`/`set-default`, `systemctl isolate rescue.target`. Boot to rescue: edit GRUB line, add `systemd.unit=rescue.target` (or `rd.break` to reset root password).

**Q: Forgot root password?** ⭐⭐ Reboot → GRUB `e` → add `rd.break` (RHEL) or `init=/bin/bash` → remount `/` rw → `passwd` → `touch /.autorelabel` (SELinux) → reboot.

**Q: What is systemd? Unit file sections?** ⭐⭐⭐ Init + service manager. `[Unit]` Description/After/Requires · `[Service]` ExecStart/User/WorkingDirectory/Restart/RestartSec/Environment/LimitNOFILE · `[Install]` WantedBy. Unit types: service, socket, timer, mount, target.

**Q: `restart` vs `reload`? `enable` vs `start`? `mask`?** ⭐⭐⭐ restart = stop+start; reload = re-read config (SIGHUP) without downtime. enable = start at boot (symlink), start = now. `mask` = link to /dev/null so it can never start.

**Q: After editing a unit file, what must you run?** ⭐⭐⭐ `systemctl daemon-reload`, then restart the service.

**Q: Where are logs? journald vs rsyslog?** ⭐⭐⭐ `journalctl` (binary journal: `-u svc`, `-f`, `-b`, `--since`, `-p err`, `-k` kernel) and text files in `/var/log` written by rsyslog (`messages`/`syslog`, `secure`/`auth.log`, `cron`, `dmesg`, app logs). Rotation by **logrotate** (`/etc/logrotate.d/`).

**Q: What is a kernel panic?** ⭐ Fatal kernel error the system can't recover from; usually hardware, bad driver/module, corrupted root FS. Look at console/`kdump` vmcore, `journalctl -k -b -1`.

**Q: What is `/boot`? What's in `/proc/cmdline`?** ⭐ Kernel images (`vmlinuz-*`), `initramfs-*`, GRUB config. `/proc/cmdline` = parameters the bootloader passed to the kernel.

**Q: Secure Boot?** ⭐ UEFI verifies bootloader/kernel signatures so only trusted code boots.

---

## 7. Storage, disks, LVM

**Q: Add a new disk end to end.** ⭐⭐⭐ `lsblk`/`dmesg` to see it → partition (`fdisk`/`gdisk`/`parted`) → `mkfs.xfs|ext4 /dev/sdb1` → `mkdir /data` → `mount` → `blkid` for UUID → `/etc/fstab` → `mount -a` to verify.

**Q: `fstab` fields?** ⭐⭐⭐ device(UUID) · mount point · fs type · options (`defaults`, `noatime`, `nofail`, `ro`) · dump · fsck pass (0 none, 1 root, 2 others).

**Q: `mount` shows nothing. How else do you see mounts?** ⭐ `findmnt`, `cat /proc/mounts`, `df -hT`, `lsblk -f`.

**Q: ext2 vs ext3 vs ext4 vs xfs?** ⭐⭐ ext3 added journaling; ext4 larger files/volumes, extents, faster fsck; xfs = RHEL default, great for large files, grows online but can't shrink.

**Q: What is a journal?** ⭐ Filesystem writes intent to a log first, so after a crash it replays/rolls back instead of a full fsck.

**Q: LVM: layers and commands?** ⭐⭐⭐ PV (`pvcreate`) → VG (`vgcreate`, `vgextend`) → LV (`lvcreate -L 10G -n data vg`) → `mkfs` → mount. Inspect `pvs vgs lvs`. Grow: `lvextend -r -L +5G /dev/vg/data`. Snapshot: `lvcreate -s`.

**Q: `df` vs `du` difference in general?** ⭐⭐⭐ `df` asks the filesystem for totals (includes space held by deleted-but-open files, reserved blocks). `du` walks files and sums them (misses deleted-open files, files under mount points).

**Q: Reserved blocks: `df` shows used+avail < size?** ⭐ ext filesystems reserve 5% for root (`tune2fs -m`).

**Q: DAS vs NAS vs SAN; block vs object storage?** ⭐⭐ DAS directly attached; NAS = file-level over network (NFS/SMB); SAN = block-level over dedicated network (FC/iSCSI). Block = raw disk you format (EBS); object = HTTP key/blob store (S3), no filesystem semantics.

**Q: NFS basics / `/etc/exports`?** ⭐ Server exports dirs to clients in `/etc/exports`, `exportfs -a`; client mounts `server:/path`. Hung NFS → D-state processes and `df` hanging.

**Q: What is RAID 0/1/5/10?** ⭐⭐ 0 stripe (speed, no redundancy) · 1 mirror · 5 stripe+parity (1 disk failure) · 10 mirrored stripes (speed + redundancy). `cat /proc/mdstat` for software RAID.

**Q: Create a file of a specific size?** ⭐ `fallocate -l 1G f` or `dd if=/dev/zero of=f bs=1M count=1024` (for disk-full testing).

---

## 8. Networking & DNS

**Q: Switch vs router vs firewall?** ⭐⭐ Switch connects hosts in one network (L2/MAC). Router connects networks (L3/IP); your default gateway. Firewall filters traffic by rules (L3/L4, or L7 for app firewalls).

**Q: What is a subnet / CIDR `/24`?** ⭐⭐⭐ Mask 255.255.255.0 → 256 addresses, 254 usable. `/16` = 65,536. `/32` = one host.

**Q: Private IP ranges? Loopback? IPv6 size/localhost?** ⭐⭐ 10/8, 172.16/12, 192.168/16. `lo` = 127.0.0.1 (traffic to self; apps bound only to 127.0.0.1 aren't reachable externally). IPv6 = 128-bit; localhost `::1`; record type AAAA.

**Q: Can a system have two default gateways?** ⭐ Only one active default route (lowest metric wins); otherwise use policy routing.

**Q: Show interfaces, IPs, routes; add an IP/route.** ⭐⭐⭐ `ip link`, `ip addr`, `ip route`; `ip addr add 192.168.1.10/24 dev eth0`; `ip route add 10.0.0.0/8 via 192.168.1.1`; bring up `ip link set eth0 up`. Persist: netplan / NetworkManager (`nmcli`) / `/etc/sysconfig/network-scripts/`.

**Q: Change DNS server via `nmcli`?** ⭐ `nmcli con mod "eth0" ipv4.dns 8.8.8.8 && nmcli con up "eth0"`.

**Q: Explain how DNS works end to end.** ⭐⭐⭐ Stub resolver checks cache and `/etc/hosts` (order from `/etc/nsswitch.conf`), asks the recursive resolver in `/etc/resolv.conf`; the resolver asks root → TLD (`.com`) → authoritative server for `apple.com`, caches the answer per **TTL**, returns the IP.

**Q: Tools to query/troubleshoot DNS?** ⭐⭐⭐ `dig name`, `dig @8.8.8.8 name` (bypass local resolver), `dig +short`, `dig -x IP` (reverse), `nslookup`, `host`, `resolvectl status`.

**Q: `dig` returns SERVFAIL / NXDOMAIN / no answer. Meaning?** ⭐⭐ NXDOMAIN = name doesn't exist (typo/zone). SERVFAIL = resolver couldn't complete (upstream broken, DNSSEC). Timeout = can't reach DNS server (network/firewall on 53).

**Q: What is split-horizon DNS?** ⭐ Same name resolves differently inside vs outside the network.

**Q: TCP flags and states you should know?** ⭐⭐ SYN, ACK, FIN, RST, PSH, URG. States: LISTEN, SYN_SENT, ESTABLISHED, **TIME_WAIT** (normal after close), **CLOSE_WAIT** (remote closed, **our app didn't close** → leak), FIN_WAIT.

**Q: How do you check open ports / active connections?** ⭐⭐⭐ `ss -tulnp` (listeners + PIDs), `ss -tan state established`, `ss -s` summary, `lsof -i`, `nmap localhost`.

**Q: How do you send an HTTP request from the shell / test a port?** ⭐⭐⭐ `curl -v http://host:8080/health`, `curl -I`, `curl -o /dev/null -s -w "%{http_code} %{time_total}\n"`; port test `nc -zv host 5432`, `telnet host 5432`, `curl -v telnet://host:5432`.

**Q: What is `tcpdump` / a packet sniffer? Example?** ⭐⭐ Captures packets on an interface: `tcpdump -i eth0 port 443 -nn`, `tcpdump -i any host 10.0.0.5 -w cap.pcap` → open in Wireshark.

**Q: Why is telnet bad? SSH vs SSL?** ⭐⭐ Telnet is plaintext (passwords visible). SSH = encrypted remote shell/tunnel protocol; SSL/TLS = encryption layer for other protocols (HTTPS). Both use public-key crypto.

**Q: HTTP vs HTTPS, briefly how HTTPS works?** ⭐⭐ TLS handshake: client hello (ciphers) → server sends certificate (public key, signed by CA) → client verifies chain → key exchange → symmetric session key → encrypted HTTP.

**Q: What is an HTTP proxy / reverse proxy / load balancer?** ⭐⭐ Forward proxy sits in front of clients (egress control/caching). Reverse proxy sits in front of servers (nginx) doing TLS, routing, caching. LB spreads traffic across backends (round-robin, least-connections) with health checks.

**Q: Status codes 401 vs 403; 502 vs 503 vs 504?** ⭐⭐⭐ 401 not authenticated, 403 authenticated but not allowed. 502 proxy got invalid/no response from backend; 503 backend unavailable/overloaded or no healthy targets; 504 backend too slow.

**Q: iptables: tables, chains, DROP vs REJECT, allow one IP on 22?** ⭐⭐ filter table chains INPUT/OUTPUT/FORWARD; nat table PREROUTING/POSTROUTING. Rules match top-down. `iptables -A INPUT -p tcp -s 10.1.1.5 --dport 22 -j ACCEPT`, then `-A INPUT -j DROP`. DROP = silent (client times out); REJECT = sends error (fails fast). Modern: nftables, `firewalld` (`firewall-cmd --add-port=8080/tcp --permanent`), `ufw`.

**Q: What is a network namespace / virtual IP / bonding / bridge?** ⭐ Namespace = isolated network stack (containers). VIP = floating IP moved between nodes for HA (keepalived). Bonding = combine NICs (active-backup, LACP). Bridge = software switch (`docker0`).

**Q: What is NTP and why does time matter?** ⭐⭐ Clock sync (`chronyd`, `chronyc tracking`, `timedatectl`). Wrong time breaks TLS certs, Kerberos/auth, log correlation, distributed DBs.

**Q: "Host key verification failed" when SSH-ing?** ⭐⭐ The server's host key changed vs `~/.ssh/known_hosts` (reinstalled server or MITM). Verify, then remove the old line (`ssh-keygen -R host`).

**Q: How would you reboot thousands of servers safely?** ⭐ (FAANG design-ish question) Batches/canaries by failure domain, drain from LB/K8s first, health check before next batch, max-unavailable limit, automation (Ansible `serial`), rollback/stop on error rate, comms + change window.

---

## 9. Users, SSH, security

**Q: Root user: name/UID? Regular users' UIDs?** ⭐⭐⭐ `root`, UID 0. Regular users typically ≥ 1000; system accounts < 1000.

**Q: `/etc/passwd` fields?** ⭐⭐⭐ `name:x:UID:GID:GECOS:home:shell`. Passwords (hashed) live in `/etc/shadow` (root-only). Groups in `/etc/group`.

**Q: Create a user who cannot log in (service account).** ⭐⭐ `useradd -r -s /usr/sbin/nologin -M svc` (or `/bin/false`).

**Q: Add/remove a user from a group.** ⭐⭐ `usermod -aG dev bob` (**-a** or you wipe other groups), `gpasswd -d bob dev`. Changes apply at next login (`newgrp` for now).

**Q: What is `/etc/skel`?** ⭐ Template copied into new users' home directories.

**Q: `grep $(whoami) /etc/passwd` returns nothing. Why?** ⭐ User comes from LDAP/AD/SSSD, not local files (`getent passwd $(whoami)` works).

**Q: What is sudo? How do you set it up safely?** ⭐⭐⭐ Run commands as root with your own password, logged. Add to `wheel`/`sudo` group or `visudo` rules; prefer specific commands over `ALL`, avoid `NOPASSWD` except for automation.

**Q: Who is logged in / login history?** ⭐⭐ `who`, `w`, `last`, `lastlog`, `lastb` (failed), `/var/log/secure`.

**Q: What is in `~/.ssh/authorized_keys` and `known_hosts`?** ⭐⭐⭐ `authorized_keys` on the **server**: public keys allowed to log in as that user. `known_hosts` on the **client**: fingerprints of servers you've connected to.

**Q: Public vs private key?** ⭐⭐⭐ Private stays with you (600, never shared); public is distributed. Server encrypts a challenge with your public key; only your private key can answer.

**Q: SSH port forwarding: local vs remote?** ⭐⭐ `-L 8080:db:5432` = local port → remote network (reach DB through bastion). `-R 9000:localhost:3000` = expose your local service on the remote host. `-D` = SOCKS proxy.

**Q: `ssh 127.0.0.1` → "connection refused".** ⭐⭐ sshd not running / not listening on that address/port (`systemctl status sshd`, `ss -tlnp | grep 22`), or firewall REJECT.

**Q: Hardening a server: what would you do?** ⭐⭐ Key-only SSH, no root login, fail2ban, firewall default-deny, minimal packages, auto security updates, SELinux enforcing, least-privilege sudo, log shipping, NTP, file integrity (AIDE).

**Q: What is SELinux? Common symptom?** ⭐⭐ Mandatory access control labelling processes/files. Symptom: service denied access despite correct Unix perms → `ausearch -m avc -ts recent`, `getenforce`, `setenforce 0` to test (never leave it), `restorecon -Rv /path`, `setsebool`.

**Q: What is a chroot jail?** ⭐ Change a process's apparent root dir so it can't see the rest of the FS. Used by sftp jails, build envs; predecessor idea to containers.

**Q: Encode/decode data quickly?** ⭐ `base64` / `base64 -d` (not encryption!).

**Q: Create a self-signed cert / CSR?** ⭐ `openssl req -new -newkey rsa:2048 -nodes -keyout site.key -out site.csr`; self-signed: add `-x509 -days 365`. Check a live cert: `openssl s_client -connect host:443`, `openssl x509 -in cert.pem -noout -dates`.

---

## 10. Packages & cron

**Q: Why package managers instead of tarballs?** ⭐⭐ Dependency resolution, versioning, verification, clean removal, repos/security updates.

**Q: Which package provides a command / owns a file?** ⭐⭐ RHEL: `yum provides */tcpdump`, `rpm -qf /usr/bin/ls`, `rpm -ql pkg` (contents). Debian: `dpkg -S /path`, `dpkg -L pkg`, `apt-file search`.

**Q: Which is newer: openssh-server-5.3p1-118.1.el6_8 or 6.6p1-1.el6?** ⭐ Version 6.6 > 5.3; the `-118` is just the packager's release number.

**Q: How do you upgrade the kernel on 100–1000 servers?** ⭐ Config management (Ansible) in batches, reboot in canary waves, verify boot + health, keep previous kernel in GRUB for rollback.

**Q: `yum`/`dnf` vs `apt` cheat.** ⭐⭐⭐ See §1 Q28. Repos: `/etc/yum.repos.d/*.repo`, `/etc/apt/sources.list(.d)`. History: `yum history`, `/var/log/dnf.log`, `/var/log/apt/history.log`.

**Q: Cron format, special strings, where it logs, why a job "doesn't run".** ⭐⭐⭐ `m h dom mon dow cmd`; `@reboot @hourly @daily`; `/var/log/cron` or syslog; failures: PATH/env minimal (use full paths), not executable, output not redirected, wrong user's crontab, `%` needs escaping, crond stopped. Systemd **timers** are the modern alternative (`systemctl list-timers`).

**Q: `at` vs cron?** ⭐ `at` runs once at a time; cron repeats.

---

## 11. One-liners & scripting tasks
*(These are asked live. Practise typing them.)*

| Task | Answer |
|---|---|
| Top 10 IPs in an access log ⭐⭐⭐ | `awk '{print $1}' access.log \| sort \| uniq -c \| sort -rn \| head -10` |
| Count HTTP status codes ⭐⭐ | `awk '{print $9}' access.log \| sort \| uniq -c \| sort -rn` |
| Count "ERROR" lines ⭐⭐⭐ | `grep -c ERROR app.log` |
| Errors in the last hour (journald) ⭐⭐ | `journalctl -u app --since "1 hour ago" -p err` |
| Files > 1 GB ⭐⭐⭐ | `find / -xdev -type f -size +1G -exec ls -lh {} + 2>/dev/null` |
| Delete logs older than 7 days ⭐⭐⭐ | `find /var/log/app -name "*.log" -type f -mtime +7 -delete` |
| Delete files older than a date ⭐ | `find /dir -type f ! -newermt "2020-01-01" -delete` |
| Biggest directories ⭐⭐⭐ | `du -xh / --max-depth=2 2>/dev/null \| sort -rh \| head -20` |
| Print 4th column ⭐⭐ | `awk '{print $4}' file` (CSV: `awk -F, '{print $4}'`) |
| Lines longer than 79 chars ⭐ | `awk 'length > 79' file` |
| Replace 1→2 in all `.yml` files ⭐⭐ | `find . -name "*.yml" -exec sed -i 's/1/2/g' {} +` |
| Split a 50-line file into 2×25 ⭐ | `split -l 25 file part_` |
| Users with bash shell ⭐ | `awk -F: '$7=="/bin/bash"{print $1}' /etc/passwd` |
| Who listens on 8080 ⭐⭐⭐ | `ss -tulnp \| grep :8080` |
| Follow a log and show only ERROR ⭐⭐ | `tail -F app.log \| grep --line-buffered ERROR` |
| Screen + file at once ⭐⭐ | `./run.sh 2>&1 \| tee -a run.log` |
| Random 7-char string ⭐ | `tr -dc A-Za-z0-9 </dev/urandom \| head -c 7` |
| Check a list of URLs ⭐⭐ | `while read u; do c=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$u"); [ "$c" != 200 ] && echo "FAIL $u $c"; done < urls.txt` |

**Script 1: alert if any filesystem > 80%** ⭐⭐⭐
```bash
#!/bin/bash
df -hP | awk 'NR>1 {gsub("%","",$5); if ($5+0 > 80) print "ALERT:", $6, "at", $5"%"}'
```

**Script 2: restart a service if it is down** ⭐⭐
```bash
#!/bin/bash
SVC=nginx
if ! systemctl is-active --quiet "$SVC"; then
  echo "$(date '+%F %T') $SVC down - restarting" >> /var/log/svc-watch.log
  systemctl restart "$SVC"
fi
```

**Script 3: parse a log for top error messages (Python)** ⭐⭐
```python
from collections import Counter
import re, sys
c = Counter()
for line in open(sys.argv[1]):
    if "ERROR" in line:
        msg = re.sub(r"\d+", "N", line.split("ERROR", 1)[1].strip())
        c[msg] += 1
for msg, n in c.most_common(5):
    print(n, msg)
```

**Bash facts asked as questions:** `$?` exit code · `$0 $1 $# $@` · `set -euo pipefail` · `[ -f ]`/`[ -d ]`/`[ -z ]` · `&&` runs next only on success (not a boolean AND of outputs) · `$(cmd)` substitution · `trap` for cleanup · `^old^new` repeats last command with substitution.

---

## 12. Live troubleshooting scenarios (the real round)

### The 60-second health check (say it in order)
```
uptime                 load vs nproc; rising or falling?
dmesg -T | tail        kernel: OOM kills, disk/NIC errors
top / htop             who eats CPU/mem; us/sy/wa/st; D or Z states
free -h                available memory, swap usage
df -h ; df -i          space AND inodes
iostat -x 1            disk saturation (%util, await)
vmstat 1               r (run queue), b (blocked), si/so (swapping), wa
ss -tulnp              is the app listening? on which IP/port?
journalctl -u app -f / tail -f /var/log/app.log    errors, stack traces
curl -v localhost:PORT/health                      does the app answer locally?
```
Then: **what changed?** (deploy, config, cron, traffic). Then **mitigate** (restart / rollback / scale / free disk), then root cause, then prevention.

### Scenarios — cause and action
| # | Scenario | What you say / do |
|---|---|---|
| 1 ⭐⭐⭐ | **"The server/app is slow."** | Clarify (what, since when, one host or all). 60-sec check. Branch: `us` high → app (Java: `top -H`, `jstack`); `wa` high → disk/NFS (`iostat`); swap → memory; many ESTABLISHED/CLOSE_WAIT → connection issue; dependency latency (DB: slow queries). Mitigate (restart/scale/rollback), RCA. |
| 2 ⭐⭐⭐ | **Disk 100% full** | `du -xh --max-depth` to find it (logs, `/tmp`, core dumps, old releases). Truncate live logs (`> file`), don't `rm` them. `journalctl --vacuum-size`, clean caches, `docker system prune`. Check `lsof +L1`. Prevent: logrotate, alert at 80%, extend LVM. |
| 3 ⭐⭐⭐ | **`df` full, `du` not** | Deleted-but-open file → `lsof +L1` → restart owner process or truncate `/proc/<pid>/fd/<n>`. Also: data hidden under a mount point. |
| 4 ⭐⭐⭐ | **Free space, can't create files** | `df -i` inodes exhausted → find dirs with most files → delete → fix generator (sessions, cache, mail queue). |
| 5 ⭐⭐⭐ | **CPU 100%** | `top` → which PID, which mode. `ps -eo ... --sort=-%cpu`. Java thread dump. `strace -c` if `sy` high. Steal → hypervisor. Kill/renice/scale, then fix code. |
| 6 ⭐⭐⭐ | **Load 20, CPU idle** | D-state / I/O wait: `ps -eo stat,pid,cmd \| awk '$1 ~ /^D/'`, `iostat -x`, hung NFS (`mount \| grep nfs`, `df` hangs). Fix storage. |
| 7 ⭐⭐⭐ | **Process killed randomly** | OOM: `dmesg -T \| grep -iE "oom\|killed process"`. Memory leak (RSS trend), bad `-Xmx`, missing limits. Containers: exit 137. |
| 8 ⭐⭐⭐ | **App up, users can't reach it** | `ss -tulnp` (bound to 127.0.0.1?) → `curl localhost:port` → firewall/iptables/security group → `nc -zv` from outside → DNS → LB health check / cert expiry. |
| 9 ⭐⭐⭐ | **Can't SSH** | `ssh -vvv`; `nc -zv host 22` (timeout = network/firewall, refused = sshd down); key perms 700/600; console: `systemctl status sshd`, `/var/log/secure`, disk full, `sshd -t`, fail2ban, `AllowUsers`. |
| 10 ⭐⭐⭐ | **Service won't start after reboot** | `systemctl status`, `journalctl -xeu svc`: not enabled, config typo, port in use, perms, dependency not ready (`After=`), mount missing, SELinux, disk full. |
| 11 ⭐⭐ | **Boot fails after fstab edit** | Rescue/emergency shell → `mount -o remount,rw /` → fix line → `mount -a` → reboot. Prevent: `mount -a` before reboot, `nofail`. |
| 12 ⭐⭐⭐ | **"Too many open files"** | `cat /proc/<pid>/limits`, `ls /proc/<pid>/fd \| wc -l`. Legit → `LimitNOFILE=` in unit / `limits.conf`. Growing → FD/socket leak (CLOSE_WAIT) → fix app. |
| 13 ⭐⭐⭐ | **DNS not resolving** | `dig host` vs `dig @8.8.8.8 host`; `/etc/resolv.conf`, `/etc/hosts`, `nsswitch.conf`; port 53 reachable; `systemd-resolved` status. |
| 14 ⭐⭐⭐ | **502 / 503 / 504** | 502 backend down/garbage → app + nginx error log. 503 no healthy backends/overload → scale/restart. 504 backend slow → DB/slow endpoints. |
| 15 ⭐⭐⭐ | **Deployed a change, app broke** | **Roll back first**, restart, then `diff` config, validate (`nginx -t`), test in staging, change process with rollback plan. |
| 16 ⭐⭐ | **Thousands of TIME_WAIT / CLOSE_WAIT** | TIME_WAIT = normal (use keep-alive/pooling). CLOSE_WAIT = app leak → restart + fix code. `ss -tan \| awk '{print $1}' \| sort \| uniq -c`. |
| 17 ⭐⭐ | **Java app hung, process alive** | `jstack` (deadlock/blocked), `jstat -gcutil` (GC storm), `strace -p`, DB pool exhausted. Capture dumps **before** restart. |
| 18 ⭐⭐ | **Web app returns 404 after deploy** | Right vhost/root path? File exists + readable by web user? Rewrite rules? Deployed to the right server/container? Access log shows the requested path; error log shows the real path tried. |
| 19 ⭐⭐ | **Time is wrong on a server** | `timedatectl`, `chronyc tracking/sources`, NTP reachable (udp/123), fix; explain impact (TLS, Kerberos, logs). |
| 20 ⭐⭐ | **Cron job didn't run** | `crond` running, syntax, full paths, executable, output redirected, correct user, `/var/log/cron`. |
| 21 ⭐⭐ | **Someone ran `chmod -R 777 /var/www`** | Security risk; some daemons refuse. Restore dirs 755 / files 644 / secrets 600, correct owner, ideally re-apply via config management. |
| 22 ⭐⭐ | **"What changed on this server recently?"** | `last`, `history`, package logs (`yum history`, `/var/log/apt/history.log`), `find /etc -mtime -1`, `journalctl --since`, change tickets, Git of config repo. |
| 23 ⭐⭐⭐ | **KodeKloud finale: app works on laptop, not on dev server** | Read the stack trace → DB service down (`systemctl status postgresql`) → wrong host/port in config (`localhost` vs `devdb01`, `ss -tlnp` for port) → `pg_hba.conf` access → file ownership (`chown -R`) → run it as a systemd service with `Restart=on-failure`. |

---

## 13. Containers are Linux + SRE mindset questions

**Q: What is a container, in Linux terms?** ⭐⭐⭐ A normal process isolated by **namespaces** (PID, NET, MNT, UTS, IPC, USER: what it can *see*) and limited by **cgroups** (CPU, memory, I/O: what it can *use*), on a layered filesystem (overlayfs). No separate kernel.

**Q: True/false lightning round (from devops-exercises):** ⭐⭐ In every PID namespace the first process is PID 1 (**true**). Parent PID ns sees child ns processes, child doesn't see parent (**true**). Two fresh network namespaces can ping each other by default (**false**, need veth/bridge). UTS ns lets processes have different hostnames (**true**). Root (UID 0) inside a child user namespace is possible (**true**, mapped to an unprivileged host UID).

**Q: Container exits with code 137 / 143?** ⭐⭐⭐ 137 = 128+9 SIGKILL (OOM-killed by cgroup limit or forced kill). 143 = 128+15 SIGTERM (graceful stop).

**Q: Do sysctl changes in a container affect the host?** ⭐ Namespaced ones (e.g. `net.*`) are per-namespace; non-namespaced (`vm.*`, `kernel.*`) are the host's and read-only in containers.

**Q: What virtualization options exist on Linux? KVM? libvirt?** ⭐ KVM = kernel hypervisor module; QEMU emulation; libvirt = management API/CLI (`virsh`). Containers vs VMs: shared kernel vs own kernel.

**Q: How do you know a system is reliable? (asked in Apple threads)** ⭐⭐⭐ Define user-facing **SLIs** (success rate, latency), set **SLOs** (99.9%), measure, use the **error budget** to decide between features and reliability, alert on SLO burn, blameless postmortems, reduce **toil** with automation.

**Q: Monitoring vs observability; golden signals?** ⭐⭐ Monitoring = known failure modes/dashboards; observability = ability to ask new questions from metrics, logs, traces. Golden signals: latency, traffic, errors, saturation.

**Q: Incident vs problem vs change (ITIL, in the JD)?** ⭐⭐ Incident = restore service fast; problem = find root cause of recurring incidents; change = planned modification with risk assessment, approval, rollback plan.

---

## 14. Behavioural questions asked in Apple SRE loops
(Reported on r/sre. Prepare STAR answers, 60–90 seconds each.)

1. During an incident the customer/business is putting a lot of pressure on you. How do you handle it? → assess impact, mitigate first, communicate at a fixed cadence, escalate early.
2. Tell me about a time you strongly disagreed with a colleague.
3. You must fix a system you don't know, and can't reach the owner, lead or manager. What do you do? → read runbooks/logs, take safe reversible actions, document, use other escalation channels.
4. Two high-priority, time-sensitive tasks; you can't deliver both. → clarify business impact, communicate early, negotiate, ask for help.
5. A recurring task from another team is poorly documented and keeps getting reworked. Would you do anything? → document, automate, propose a process fix.
6. Tell me about a production incident you handled / a mistake you made in production / something you automated.
7. Why Apple? Why SRE/support? Tell me about yourself (2 min).
8. Questions to ask: on-call rotation, biggest reliability challenges for sales/ops systems, how the team uses GenAI, what success looks like in 90 days.

---

## 15. Last-day cheat sheet

| Need | Command |
|---|---|
| Load / cores | `uptime`, `nproc` |
| Kernel msgs / OOM | `dmesg -T \| tail`, `journalctl -k`, `dmesg -T \| grep -i oom` |
| Processes | `top`, `htop`, `ps -eo pid,ppid,user,%cpu,%mem,rss,stat,cmd --sort=-%cpu \| head`, `pstree -p` |
| Kill | `kill <pid>` (15), `kill -9`, `pkill -f name`, `fuser -k file` |
| Memory | `free -h`, `vmstat 1`, `cat /proc/meminfo` |
| Disk | `df -h`, `df -i`, `du -xh --max-depth=2 \| sort -rh \| head`, `lsof +L1`, `iostat -x 1` |
| Service | `systemctl status/restart/reload/enable --now svc`, `daemon-reload`, `list-units --failed` |
| Logs | `journalctl -u svc -f --since "1 hour ago" -p err`, `tail -f /var/log/messages` |
| Ports / connections | `ss -tulnp`, `ss -tan state established`, `lsof -i :8080` |
| Network | `ip a`, `ip route`, `ping`, `traceroute`/`mtr`, `nc -zv host port`, `curl -v`, `dig @8.8.8.8 host`, `tcpdump -i eth0 port 443 -nn` |
| Firewall | `iptables -L -n --line-numbers`, `firewall-cmd --list-all`, `ufw status` |
| Perms / owner | `chmod 755`, `chown -R user:group`, `ls -ld`, `getfacl`, `lsattr` |
| Find / search | `find / -xdev -type f -size +1G`, `grep -rn text /etc`, `grep -A5 Exception log` |
| Users | `id`, `who`, `w`, `last`, `useradd`, `usermod -aG`, `passwd`, `getent passwd user` |
| Packages | `yum install/remove/provides/history`, `rpm -qa\|-qf\|-ql`, `apt update && apt upgrade`, `dpkg -l\|-S\|-L` |
| Storage | `lsblk -f`, `blkid`, `findmnt`, `mount -a`, `pvs vgs lvs`, `lvextend -r -L +5G` |
| SSH | `ssh -vvv user@host`, `ssh-copy-id`, `ssh-keygen -R host`, `scp -pr`, `rsync -avz` |
| Cron / timers | `crontab -e -l`, `systemctl list-timers` |
| Trace | `strace -p <pid>`, `strace -c cmd`, `ldd bin` |
| Kernel | `uname -r`, `lsmod`, `modprobe`, `sysctl -a`, `/proc/cmdline` |

**Say these three sentences in every troubleshooting answer**
1. "I'd first assess impact and restore service, then find the root cause."
2. "I avoid `kill -9` and deleting live logs; I use SIGTERM and truncate."
3. "After the fix I'd add monitoring/alerting and a runbook or automation so it doesn't repeat."

**Practice:** sadservers.com ("Easy" scenarios), KodeKloud Linux playground, and sttrace.com (Linux debugging challenges built for Meta PE prep).

---
---

## 📚 Original course notes (KodeKloud)

These are notes from the [Linux Basics Course](https://bit.ly/3gGnxm0) hosted on KodeKloud.

- [01-Introduction](docs/01-Introduction)
- [02-Working-With-Shell-Part-I](docs/02-Working-With-Shell-Part-I)
  - [01-Introduction-to-Shell](docs/02-Working-With-Shell-Part-I/01-Introduction-to-Shell.md)
  - [02-Basic-Commands](docs/02-Working-With-Shell-Part-I/02-Basic-Commands.md)
  - [03-Command-Line-Help](docs/02-Working-With-Shell-Part-I/03-Command-Line-Help.md)
  - [04-Lab-Working-With-Shell](docs/02-Working-With-Shell-Part-I/04-lab-working-with-shell.md)
  - [05-Bash-Shell](docs/02-Working-With-Shell-Part-I/05-Bash-Shell.md)
  - [06-Lab-Linux-Bash-Shell](docs/02-Working-With-Shell-Part-I/06-Lab-Linux-Bash-Shell.md)
- [03-Linux-Core-Concepts](docs/03-Linux-Core-Concepts)
  - [01-Bobs-First-Team-Meeting](docs/03-Linux-Core-Concepts/01-Bobs-first-team-meeting.md)
  - [02-The-Linux-Kernel](docs/03-Linux-Core-Concepts/02-The-Linux-Kernel.md)
  - [03-Working-With-Hardware](docs/03-Linux-Core-Concepts/03-Working-with-hardware.md)
  - [04-Lab-Linux-Kernel](docs/03-Linux-Core-Concepts/04-Lab-Linux-Kernel.md)
  - [05-Linux-Boot-Sequence](docs/03-Linux-Core-Concepts/05-Linux-Boot-Sequence.md)
  - [06-Run-Levels](docs/03-Linux-Core-Concepts/06-Run-Levels.md)
  - [07-File-Types](docs/03-Linux-Core-Concepts/07-File-Types.md)
  - [08-Filesystem-Hierarchy](docs/03-Linux-Core-Concepts/08-Filesystem-Hierarchy.md)
  - [09-Lab-Linux-RunLevels-Filesystem-Hierarchy](docs/03-Linux-Core-Concepts/09-Lab-Linux-RunLevels-Filesystem-Hierarchy.md)
- [04-Package-Mangement](docs/04-Package-Mangement)
  - [01-Package-Management-Distribution](docs/04-Package-Mangement/01-Package-Management-Distribution.md)
  - [02-RPM-and-YUM](docs/04-Package-Mangement/02-RPM-and-YUM.md)
  - [03-Lab-RPM-and-YUM](docs/04-Package-Mangement/03-Lab-RPM-and-YUM.md)
  - [04-DPKG-AND-APT](docs/04-Package-Mangement/04-DPKG-AND-APT.md)
  - [05-APT-vs-APT-GET](docs/04-Package-Mangement/05-APT-vs-APT-GET.md)
  - [06-Lab-DPKG-AND-APT](docs/04-Package-Mangement/06-Lab-DPKG-AND-APT.md)
- [05-Working-With-Shell-Part-II](docs/05-Working-With-Shell-Part-II)
  - [01-Behind-Schedule(story)](docs/05-Working-With-Shell-Part-II/01-Behind-Schedule(story).md)
  - [02-File-Compression-and-Archival](docs/05-Working-With-Shell-Part-II/02-File-Compression-and-Archival.md)
  - [03-Searching-For-Files-and-Patterns](docs/05-Working-With-Shell-Part-II/03-Searching-for-files-and-patterns.md)
  - [04-IO-Redirection](docs/05-Working-With-Shell-Part-II/04-IO-Redirection.md)
  - [05-Lab-Working-With-Shell-Part-II](docs/05-Working-With-Shell-Part-II/05-Lab-Working-With-Shell-Part-II.md)
  - [06-Vi-Editor](docs/05-Working-With-Shell-Part-II/06-Vi-Editor.md)
  - [07-Lab-VI-Editor](docs/05-Working-With-Shell-Part-II/07-Lab-VI-Editor.md)
- [06-Security-and-File-Permissions](docs/06-Security-and-File-Permissions)
  - [01-The-Security-Incident(story)](docs/06-Security-and-File-Permissions/01-The-Security-Incident(story).md)
  - [02-Linux-Accounts](docs/06-Security-and-File-Permissions/02-Linux-Accounts.md)
  - [03-User-Management](docs/06-Security-and-File-Permissions/03-User-Management.md)
  - [04-Access-Control-Files](docs/06-Security-and-File-Permissions/04-Access-Control-Files.md)
  - [05-File-Permissions](docs/06-Security-and-File-Permissions/05-File-Permissions.md)
  - [06-SSH-and-SCP](docs/06-Security-and-File-Permissions/06-SSH-and-SCP.md)
  - [07-IPtables](docs/06-Security-and-File-Permissions/07-IPtables.md)
  - [08-Cronjob](docs/06-Security-and-File-Permissions/08-Cronjob.md)
- [07-Networking](docs/07-Networking)
  - [01-The-Network-Issue(story)](docs/07-Networking/01-The-Network-Issue(story).md)
  - [02-DNS](docs/07-Networking/02-DNS.md)
  - [03-Networking-Basics](docs/07-Networking/03-Networking-Basics.md)
  - [04-Troubleshooting](docs/07-Networking/04-Troubleshooting.md)
- [08-Storage-in-Linux](docs/08-Storage-in-Linux)
  - [01-Where's-my-Storage](docs/08-Storage-in-Linux/01-Where's-my-Storage.md)
  - [02-Storage-Basics](docs/08-Storage-in-Linux/02-Storage-Basics.md)
  - [03-File-System-in-Linux](docs/08-Storage-in-Linux/03-File-System-in-Linux.md)
  - [04-DAS-NAS-and-SAN](docs/08-Storage-in-Linux/04-DAS-NAS-and-SAN.md)
  - [05-LVM](docs/08-Storage-in-Linux/05-LVM.md)
  - [06-Project-Status-Meeting](docs/08-Storage-in-Linux/06-Project-Status-Meeting.md)
- [09-Service-management-with-SYSTEMD](docs/09-Service-management-with-SYSTEMD)
  - [01-Working-Overtime-Story](docs/09-Service-management-with-SYSTEMD/01-Working-Overtime-Story.md)
  - [02-Creating-a-SYSTEMD-Service](docs/09-Service-management-with-SYSTEMD/02-Creating-a-SYSTEMD-Service.md)
  - [03-SYSTEMD-Tools](docs/09-Service-management-with-SYSTEMD/03-SYSTEMD-Tools.md)
- [10-The-Client-Demonstration](docs/10-The-Client-Demonstration)
  - [01-Client-Demonstration-in-Jeopardy!(story)](docs/10-The-Client-Demonstration/01-Client-Demonstration-in-Jeopardy!(story).md)
  - [02-Troubleshoot-the-Development-Environment](docs/10-The-Client-Demonstration/02-Troubleshoot-the-Development-Environment.md)
  - [03-Finale(story)](docs/10-The-Client-Demonstration/03-Finale(story).md)
