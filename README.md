# 🐧 Linux Interview Prep — Apple IS&T Junior Application Support & SRE

> **Who this is for:** a candidate with 1–2 years of experience who is new to Linux.
> **How it's written:** every answer first **teaches the idea in simple words**, then shows **how it works**, then gives you a **ready-to-say interview answer**. You should be able to understand a question even if you have never heard of the concept before.
> **Scope:** only what a junior Application Support / SRE at Apple is realistically asked. No advanced kernel topics.
> **Source of the questions:** Apple SRE interview experiences on Reddit (r/sre, r/devops), hiring managers' question lists (r/linuxadmin), the most popular Linux interview question repos on GitHub, SadServers' screening-question list, and the KodeKloud Linux Basics course (this repo).

**How to use this file**
1. Read Part A once, slowly. Don't memorise, understand.
2. For each question, cover the answer and say your own version out loud. Then compare with the *"Say this in the interview"* line.
3. Do Part B (troubleshooting) by talking through each case like you are on a call.
4. The night before, read only Part D.

---

## Table of contents
- [Part A — Concepts you must understand (45 questions)](#part-a--concepts-you-must-understand)
  - [A1. Linux basics](#a1-linux-basics)
  - [A2. Files, permissions, users](#a2-files-permissions-users)
  - [A3. Processes](#a3-processes)
  - [A4. Memory and CPU](#a4-memory-and-cpu)
  - [A5. Boot, services, logs](#a5-boot-services-logs)
  - [A6. Disk and storage](#a6-disk-and-storage)
  - [A7. Networking](#a7-networking)
  - [A8. SSH, packages, cron](#a8-ssh-packages-cron)
  - [A9. Shell and text tools](#a9-shell-and-text-tools)
- [Part B — Troubleshooting scenarios (12 cases)](#part-b--troubleshooting-scenarios)
- [Part C — One-liners you may be asked to type](#part-c--one-liners-you-may-be-asked-to-type)
- [Part D — Last-day cheat sheet](#part-d--last-day-cheat-sheet)
- [Original course notes](#-original-course-notes-kodekloud)

---

# Part A — Concepts you must understand

## A1. Linux basics

### Q1. What is Linux? What is the kernel? What is a shell?
**In simple words:** Linux is an operating system, like Windows, but used on almost every server in the world. Inside it there are three layers you should be able to name:

1. **Hardware**: CPU, memory (RAM), disk, network card.
2. **Kernel**: the core program that controls the hardware. No normal program talks to hardware directly; everything goes through the kernel. Think of the kernel as the **manager of a warehouse**: you don't walk into the warehouse yourself, you ask the manager and he brings things to you.
3. **User space**: everything else: your Java/Python apps, `nginx`, databases, and the **shell**.

A **shell** (usually **bash**) is the program that reads what you type in the terminal and runs it. When you type `ls` and press Enter, bash is the program that understands that.

**How it works:** when a program needs something from hardware (read a file, send data on the network, get more memory), it asks the kernel through a **system call** (for example `open()`, `read()`, `write()`). The kernel does the privileged work and returns the result. This is why a crashing program doesn't crash the whole machine: programs are kept away from the hardware.

**Commands:** `uname -r` (kernel version), `cat /etc/os-release` (which Linux distribution, e.g. RHEL 8, Ubuntu 22.04), `echo $SHELL` (your shell).

**Say this in the interview:** "Linux is the OS; the kernel is its core that manages CPU, memory, devices and processes. Applications run in user space and request kernel services through system calls. The shell, usually bash, is the command-line program I use to interact with the system."

---

### Q2. Describe the Linux directory structure. Where are configs, logs, and apps?
**In simple words:** Linux has **one tree** that starts at `/` (called *root*). There are no C: or D: drives. Even extra disks are attached ("mounted") into folders of this tree. You need to know what lives where because during troubleshooting you go straight to the right folder.

| Folder | What's inside | When you'll use it |
|---|---|---|
| `/etc` | **Configuration files** for the system and apps (`/etc/ssh/sshd_config`, `/etc/hosts`, `/etc/fstab`) | "Where is the config?" → here |
| `/var/log` | **Log files** (`messages` or `syslog`, `secure` or `auth.log`, app logs) | First place when something breaks |
| `/var` | Data that changes: logs, caches, mail queues, DB data | Disks usually fill up here |
| `/home` | Users' personal folders (`/home/om`) | |
| `/root` | The root (admin) user's home | |
| `/tmp` | Temporary files, cleared on reboot | |
| `/opt` | Third-party / company applications | Apps are often installed here |
| `/bin`, `/sbin`, `/usr/bin` | The programs themselves (`ls`, `cp`, `systemctl`) | |
| `/lib`, `/lib64` | Shared libraries programs need | |
| `/dev` | Device files: disks appear here as `/dev/sda`, `/dev/sdb` | |
| `/proc` | A **virtual** folder: live information about running processes and the kernel. Not real files on disk. | `cat /proc/meminfo`, `/proc/<pid>/` |
| `/boot` | Kernel and bootloader files | |
| `/mnt`, `/media` | Places to attach extra disks/USB | |

**Say this in the interview:** "`/etc` holds configuration, `/var/log` holds logs, `/opt` is for third-party apps, `/home` for users, `/proc` is a virtual view of running processes, and `/dev` holds device files. Everything hangs off a single root `/`."

---

### Q3. What is `/proc`?
**In simple words:** `/proc` looks like a folder with files, but nothing in it is stored on disk. The kernel *generates* the contents the moment you read them. It's a **live window into the kernel and every running process**.

**How it works:** every running process has a folder `/proc/<PID>/`. Inside: `status` (state, memory used), `cmdline` (the exact command that started it), `fd/` (every file and network connection it has open), `limits` (its resource limits). System-wide files: `/proc/meminfo` (memory), `/proc/cpuinfo` (CPU), `/proc/loadavg` (load). Tools like `ps`, `top` and `free` simply read `/proc` and format it nicely.

**Say this in the interview:** "`/proc` is a virtual filesystem the kernel creates in memory. It exposes process and system information as files. Tools like `ps` and `top` read it. I use `/proc/<pid>/fd` to see open files of a process and `/proc/<pid>/limits` to check its limits."

---

## A2. Files, permissions, users

### Q4. Explain file permissions. What does `chmod 755` mean?
**In simple words:** every file has an **owner** (a user) and a **group**, and three sets of permissions: for the **owner**, for the **group**, and for **others** (everyone else). Each set can have **r** (read), **w** (write), **x** (execute).

```
-rwxr-x---  1  om  dev  app.sh
 └┬┘└┬┘└┬┘
  │  │  └── others:  ---  nothing
  │  └───── group:   r-x  read + execute
  └──────── owner:   rwx  read + write + execute
```

**Numbers:** r = 4, w = 2, x = 1. Add them up for each set.
- `rwx` = 7, `r-x` = 5, `rw-` = 6, `r--` = 4, `---` = 0
- `chmod 755 file` → owner 7 (rwx), group 5 (r-x), others 5 (r-x). Typical for scripts and folders.
- `chmod 644 file` → owner rw, everyone else read only. Typical for normal files.
- `chmod 600 file` → only the owner can read/write. Required for SSH private keys and secrets.
- `chmod 777` → everyone can do everything. **Never on a production server** (security risk).

**For a directory the letters mean something different:** `r` = can list files inside, `w` = can create/delete files inside, `x` = can enter it (`cd`). A folder with `r` but no `x` can't be entered.

**Commands:** `ls -l` (see permissions), `chmod 750 file`, `chmod +x script.sh` (add execute), `chown om:dev file` (change owner and group), `chown -R om /opt/app` (recursively).

**Say this in the interview:** "Permissions are read, write and execute for owner, group and others. 755 means the owner has full rights and group and others can read and execute. On directories, execute means you can enter the directory. I'd set 644 for files, 755 for scripts and directories, 600 for secrets, and never 777 in production."

---

### Q5. What are SUID, SGID and the sticky bit?
**In simple words:** three special permission flags. You'll mostly be asked to define them.

- **SUID (set user ID)** on a program: when anyone runs it, it runs **with the file owner's permissions**, not the person running it. Example: `/usr/bin/passwd` is owned by root and has SUID, so a normal user can change their own password, which requires writing to a root-only file. Shown as `s` in the owner's execute position: `-rwsr-xr-x`.
- **SGID (set group ID)** on a folder: new files created inside automatically get the **folder's group**. Used for shared team folders so everyone in the team can access each other's files.
- **Sticky bit** on a folder: everyone can create files, but you can **only delete your own**. `/tmp` has it (`drwxrwxrwt`, note the `t`), so one user can't delete another user's temp files.

**Say this in the interview:** "SUID makes a program run as its owner, like `passwd` running as root. SGID on a directory makes new files inherit the directory's group. The sticky bit on a directory like `/tmp` means users can only delete their own files."

---

### Q6. What is root? What is `sudo`? Where are users and passwords stored?
**In simple words:** **root** is the administrator account (user ID 0). It can do anything. Because that's dangerous, you normally log in as a normal user and use **`sudo`** to run a *single command* as root: `sudo systemctl restart nginx`. `sudo` asks for **your own** password and **logs** who ran what, which is good for audits.

`su -` is different: it switches you fully into another user's shell and needs **that user's** password. Companies prefer `sudo` because it is granular and logged.

**Files:**
- `/etc/passwd`: the list of users. One line per user: `name:x:UID:GID:comment:home:shell`. Despite the name it holds **no passwords** (the `x` means "look in shadow"). Anyone can read it.
- `/etc/shadow`: the **hashed passwords** and password expiry rules. Only root can read it. Passwords were moved here so normal users can't try to crack them.
- `/etc/group`: groups and their members.
- `/etc/sudoers`: who is allowed to use sudo (edit only with `visudo`, which checks syntax). Usually you just add the user to the `wheel` group (RHEL) or `sudo` group (Ubuntu): `usermod -aG wheel om`.

**Useful commands:** `id om` (UID, groups), `whoami`, `who` (who is logged in now), `last` (login history), `useradd -m om`, `passwd om`, `usermod -aG dev om` (add to a group; the `-a` is important, otherwise other groups get removed).

**Say this in the interview:** "Root is UID 0 with unlimited rights. I use `sudo` instead of logging in as root because it's granular and audited. Users are listed in `/etc/passwd`, password hashes in `/etc/shadow` which only root can read, and sudo rights are in `/etc/sudoers` or via the wheel/sudo group."

---

### Q7. What is an inode?
**In simple words:** when Linux stores a file, it stores two separate things: the **data** (the content) and a small record **about** the file. That record is the **inode**. It contains: file type, permissions, owner, size, timestamps, how many names point to it, and the addresses of the data blocks on disk.

**The surprising part:** the **filename is not in the inode**. The folder (directory) is just a list of `name → inode number` pairs. That's why renaming is instant (only the name changes) and why one file can have two names (see hard links).

**Why an SRE cares:** every filesystem is created with a **fixed number of inodes**. If an application creates millions of tiny files (sessions, cache, mail), you can run out of inodes while the disk still has free space. The error says "No space left on device" and `df -h` looks fine. Check `df -i`.

**Commands:** `ls -i file` (inode number), `stat file` (everything in the inode), `df -i` (inode usage).

**Say this in the interview:** "An inode is the metadata record of a file: permissions, owner, size, timestamps and pointers to the data blocks. It doesn't contain the filename; directories map names to inode numbers. Filesystems have a fixed number of inodes, so a server can report 'no space left' when inodes are exhausted even if `df -h` shows free space. I check that with `df -i`."

---

### Q8. What is the difference between a hard link and a soft (symbolic) link?
**In simple words:** both are ways to have a "shortcut" to a file, but they work differently.

- **Soft link (symlink)**: a tiny separate file that just **contains a path** to the target, like a Windows shortcut. `ln -s /opt/app/v2 /opt/app/current`. If you delete the target, the link **breaks** (points to nothing). It can point to a directory and across different disks. This is the one used in real life, for example to switch application versions.
- **Hard link**: a **second name for the same inode**. `ln file1 file2`. Both names are equal; the data stays as long as **at least one** name exists. Deleting the "original" changes nothing. Limitations: only inside the same filesystem, and not allowed for directories.

**How to tell them apart:** `ls -l` shows `l` and an arrow for symlinks (`current -> /opt/app/v2`); hard links show a link count > 1 in the second column and the same inode number with `ls -i`.

**Say this in the interview:** "A soft link is a separate file that stores a path, like a shortcut. It breaks if the target is deleted but can cross filesystems and point to directories. A hard link is another name for the same inode, so the file survives until the last name is removed, but it only works within one filesystem and not for directories."

---

## A3. Processes

### Q9. What is a process? What are PID, PPID and PID 1?
**In simple words:** a **program** is a file on disk (`/usr/bin/java`). A **process** is that program *running*: it has memory, open files, and a number called the **PID** (process ID). Every process is started by another process, its **parent**, whose ID is the **PPID**.

Follow the chain upwards and you always reach **PID 1**, which is **systemd** (older systems: `init`). The kernel starts PID 1 first at boot, and PID 1 starts everything else (services, login screens). It is also the "adopter": if a parent process dies, its children are handed to PID 1.

A **thread** is a smaller unit *inside* a process. Threads share the process's memory. A Java app is one process with many threads.

**Commands:** `ps -ef` (all processes with PID and PPID), `ps aux` (with CPU and memory), `top` (live view), `pstree -p` (tree), `pgrep -a java` (find PIDs by name).

**Say this in the interview:** "A process is a running instance of a program with its own PID and memory. Every process has a parent; PID 1 is systemd, the first process started by the kernel, which starts all services and adopts orphaned processes. Threads live inside a process and share its memory."

---

### Q10. What are the process states? What is a zombie process and can you kill it?
**In simple words:** `ps` and `top` show a state letter for each process:

| State | Meaning | What it tells you |
|---|---|---|
| **R** | Running or ready to run | Normal, using CPU |
| **S** | Sleeping, waiting for something (input, a timer, a network reply) | Normal; most processes are here |
| **D** | **Uninterruptible sleep**: waiting for disk or network storage inside the kernel | Can't be killed even with `kill -9`. Many D processes = a **storage problem** (slow disk, hung NFS) |
| **Z** | **Zombie**: finished, but not yet cleaned up | Explained below |
| **T** | Stopped (paused with Ctrl-Z) | |

**Zombie explained:** when a child process finishes, it leaves a tiny note (its exit code) for its parent and waits for the parent to read it with a call named `wait()`. Until the parent reads it, the child's entry stays in the process table as a **zombie**. It uses **no CPU and no memory**, only a PID slot. The zombie is *already dead*, so `kill -9` does nothing to it. The fix is to make the **parent** collect it: restart or kill the parent; then PID 1 adopts the zombie and cleans it up. A few zombies are harmless; thousands mean a buggy parent and could use up all PIDs.

**Orphan** (often confused with zombie): a child whose parent died first. It is still alive, gets adopted by PID 1, and is completely normal.

**Say this in the interview:** "Processes are running, sleeping, uninterruptible sleep, stopped or zombie. A zombie is a process that has exited but whose parent hasn't collected its exit status, so it's just a process-table entry. You can't kill it because it's already dead; you fix or restart the parent, and systemd reaps it. D-state processes are waiting on I/O and usually point to a storage issue."

---

### Q11. What is the difference between `kill -9` and `kill -15`?
**In simple words:** `kill` doesn't necessarily kill. It **sends a signal**, which is a small message to a process. Different numbers mean different messages.

- **`kill <pid>` = `kill -15` = SIGTERM ("please terminate")**: the process receives the message and can **clean up**: finish the current request, close files, flush data to disk, then exit. This is the polite, safe way. Try it first.
- **`kill -9` = SIGKILL ("die now")**: the process never sees this; the **kernel** removes it immediately. No cleanup, so you can get corrupted files, half-written data, or leftover lock files. Use only when -15 didn't work after a few seconds.
- **`kill -1` = SIGHUP**: many services (nginx, sshd) use this to **reload their configuration** without stopping.
- **Ctrl-C** sends SIGINT (interrupt), similar to TERM for interactive programs.

Two signals can never be caught or ignored by a program: **SIGKILL (9)** and **SIGSTOP (19)**.

**Commands:** `kill 1234`, `kill -9 1234`, `pkill -f myapp` (by name), `kill -HUP $(pidof nginx)`.

**Say this in the interview:** "`kill` sends a signal. 15, SIGTERM, asks the process to shut down gracefully so it can clean up. 9, SIGKILL, is handled by the kernel and ends the process instantly with no cleanup, so it can leave corrupted state. I always try 15 first and use 9 only as a last resort. SIGHUP is commonly used to reload a service's configuration."

---

### Q12. What happens when you type `ls` and press Enter?
**In simple words:** this classic question checks whether you understand how programs start. Here is the story, step by step:

1. **bash reads the line** and splits it into the command `ls` and any options.
2. bash checks if `ls` is a built-in command (like `cd`). It isn't, so bash **searches the folders in `$PATH`** (`/usr/bin`, `/bin`, …) and finds `/usr/bin/ls`.
3. bash calls **`fork()`**: the kernel makes a **copy** of the bash process. Now there are two bash processes: the parent and a child.
4. The child calls **`exec()`** (`execve`): the kernel **replaces** the child's program with `/usr/bin/ls`. Same PID, new program.
5. `ls` runs. To read the folder it asks the kernel through system calls: `openat()` to open the directory, `getdents()` to read entries, `write()` to print to the screen (file descriptor 1, stdout).
6. `ls` finishes with **exit code 0** (success).
7. The parent bash, which was waiting with **`wait()`**, receives the exit code, stores it in `$?`, and shows the prompt again.

Why fork then exec instead of one step? Because between fork and exec, the child can set things up: for example if you typed `ls > out.txt`, the child points its output to the file *before* exec. That is how redirection works.

**Say this in the interview:** "Bash parses the command, finds `ls` in `$PATH`, forks a child process, the child execs `/usr/bin/ls`, which uses system calls like `openat`, `getdents` and `write` to list the directory, then exits with a status code that the parent collects with `wait()` and exposes as `$?`."

---

### Q13. What is a file descriptor? What does "Too many open files" mean?
**In simple words:** every time a process opens a file, a network connection, or a pipe, the kernel hands it a number called a **file descriptor (FD)**. The process uses that number to read and write. Three are always there: **0 = standard input (keyboard)**, **1 = standard output (screen)**, **2 = standard error (error messages)**. New ones get 3, 4, 5…

Each process has a **limit** on how many FDs it can hold at once (often 1024 by default; `ulimit -n`). When it hits the limit, you see the famous error **"Too many open files"**: the app can't accept new connections or open logs.

**Two reasons it happens:** (1) the limit is genuinely too low for a busy server, so raise it (`LimitNOFILE=65535` in the systemd unit file, or `/etc/security/limits.conf`); (2) the app has a **leak**: it opens connections/files and never closes them, so the count grows until it hits the limit. Then raising the limit only delays the crash; the app must be fixed.

**Commands:** `ls /proc/<pid>/fd | wc -l` (how many open now), `cat /proc/<pid>/limits` (its limit), `lsof -p <pid>` (list them).

**Say this in the interview:** "A file descriptor is the handle the kernel gives a process for each open file or socket; 0, 1 and 2 are stdin, stdout and stderr. 'Too many open files' means the process reached its FD limit. I'd compare `/proc/<pid>/fd` count with `/proc/<pid>/limits`; if the usage is legitimate I raise `LimitNOFILE` in the unit file, if it keeps growing it's a leak in the application."

---

## A4. Memory and CPU

### Q14. `free -h` shows very little free memory. Is the server in trouble?
**In simple words:** usually **no**. Linux deliberately uses any spare RAM to **cache files** from disk, because reading from RAM is far faster than from disk. That cache is counted under `buff/cache`, and Linux gives it back *instantly* when an application needs memory. So "free" is low on purpose. There is a saying: *unused RAM is wasted RAM*.

```
$ free -h
          total   used   free   shared  buff/cache  available
Mem:       15Gi  6.0Gi  1.0Gi   200Mi       8.0Gi       8.5Gi
Swap:     2.0Gi    0B  2.0Gi
```
- **used**: memory applications are really using.
- **free**: completely untouched memory.
- **buff/cache**: disk cache, reclaimable.
- **available**: the number to look at. It is what applications can actually get (free + reclaimable cache).

**When to worry:** `available` is small **and** swap usage is growing, or `dmesg` shows the OOM killer (next question).

**What is swap?** A piece of disk used as emergency overflow when RAM is full. It works but is very slow, so a server actively swapping feels extremely slow. `vmstat 1` columns `si`/`so` (swap in/out) should be 0 on a healthy server.

**Say this in the interview:** "Low free memory is normal because Linux uses spare RAM as page cache, which is released when needed. I look at the `available` column, not `free`. I'd worry only if available is low, swap is being actively used, or there are OOM kills in `dmesg`."

---

### Q15. What is the OOM killer?
**In simple words:** OOM = Out Of Memory. When the machine runs out of RAM **and** swap, the kernel has to do something or the whole system freezes. So it picks a process and kills it. That is the **OOM killer**. It chooses the process with the highest "badness score", which is mostly **the one using the most memory** (often your Java app or database).

**How you notice it:** an application "randomly died" with no error in its own log. The proof is in the kernel log: `dmesg -T | grep -i -E "oom|killed process"` or `journalctl -k | grep -i oom`. In containers/Kubernetes the same event appears as **exit code 137** (OOMKilled).

**Fixes:** find why memory ran out (memory leak: the process's RSS keeps growing; too many processes; a Java heap `-Xmx` set bigger than the box; no limits), then add memory, fix the leak, or set proper limits. You can also protect a critical process by lowering its `oom_score_adj`.

**Say this in the interview:** "The OOM killer is the kernel's last resort when RAM and swap are exhausted: it kills the process with the highest memory score. If a process disappears without an application error, I check `dmesg` for OOM messages, then look at the memory trend of that process to decide whether it's a leak, a wrong heap setting, or genuinely too little RAM."

---

### Q16. What is load average? Is a load of 8 on a 4-core server bad?
**In simple words:** `uptime` prints three numbers, e.g. `load average: 3.50, 2.10, 1.20`. They are the average number of processes that **wanted to run** over the last **1, 5 and 15 minutes**. "Wanted to run" means: running on a CPU, waiting in the queue for a CPU, **or waiting for disk I/O** (D state). That last part is important and often forgotten.

**How to read it:** compare with the number of CPU cores (`nproc`).
- 4 cores, load 4.0 → every core busy, nobody waiting. Fully used.
- 4 cores, load 8.0 → on average 4 processes are waiting in line. **Overloaded.**
- 4 cores, load 1.0 → 25% busy, relaxed.
- If the 1-minute number is higher than the 15-minute number, load is **rising** right now.

**The trap:** high load but `top` shows CPU mostly idle → the waiting processes are stuck on **disk/NFS I/O**, not CPU. Check `top`'s `wa` (I/O wait) and `iostat -x 1`.

**Say this in the interview:** "Load average is the average number of processes running or waiting, including those waiting on disk I/O, over 1, 5 and 15 minutes. You compare it to the core count: 8 on a 4-core box means the system is about twice oversubscribed. I'd then check in `top` whether it's CPU-bound, high `us`, or I/O-bound, high `wa` and D-state processes."

---

### Q17. In `top`, what do `us`, `sy`, `wa`, `id` and `st` mean, and how do they guide troubleshooting?
**In simple words:** the CPU line in `top` splits CPU time into categories. Each one points to a different kind of problem.

`%Cpu(s): 60.0 us, 5.0 sy, 0.0 ni, 30.0 id, 5.0 wa, 0.0 hi, 0.0 si, 0.0 st`

| Field | Meaning | If it's high, suspect… |
|---|---|---|
| **us** (user) | time running application code | the application itself: heavy computation, infinite loop, too much traffic. For Java, look at busy threads (`top -H -p <pid>`, `jstack`) |
| **sy** (system) | time inside the kernel doing work for apps | too many system calls, heavy network/disk activity |
| **wa** (I/O wait) | CPU idle *because* it's waiting for disk | a slow or saturated disk, or a hung network filesystem (`iostat -x 1`) |
| **id** (idle) | doing nothing | normal |
| **st** (steal) | on a virtual machine, time the hypervisor gave to *another* VM | a "noisy neighbour" on the cloud host; nothing to fix inside your VM |

**Inside top:** press `P` to sort by CPU, `M` by memory, `1` to see each core, `k` to kill, `q` to quit.

**Say this in the interview:** "`us` is application CPU, `sy` is kernel CPU, `wa` is time waiting on disk, `st` is CPU stolen by the hypervisor. High `us` means I look at the app, high `wa` means I look at the disk with `iostat`, high `st` means the VM host is overloaded."

---

## A5. Boot, services, logs

### Q18. Explain the Linux boot process.
**In simple words:** what happens between pressing the power button and getting a login prompt. Four stages:

1. **BIOS/UEFI (firmware)**: runs a hardware self-test (POST) and finds the disk to boot from.
2. **Bootloader (GRUB2)**: a small program at the start of the disk, with files in `/boot`. It shows the boot menu, loads the **kernel** and a temporary mini-filesystem (**initramfs**) into memory, and hands over control.
3. **Kernel**: initialises CPU, memory and drivers, uses the drivers from initramfs to find the real root disk, mounts `/`.
4. **systemd (PID 1)**: the kernel starts systemd, which mounts the other filesystems from `/etc/fstab` and starts all services **in parallel** until it reaches the default **target** (`multi-user.target` = server without GUI, `graphical.target` = desktop). Then you get the login prompt.

**Why SREs care:** when a server doesn't come back after a reboot, you need to know which stage failed: no boot menu (GRUB/disk), kernel panic (kernel/driver), stuck in emergency mode (usually a bad `/etc/fstab` line), or services not starting (systemd logs).

**Commands:** `systemctl get-default`, `journalctl -b` (logs of this boot), `journalctl -b -1` (previous boot).

**Say this in the interview:** "BIOS or UEFI runs POST and finds the boot disk, GRUB loads the kernel and initramfs, the kernel initialises hardware and mounts the root filesystem, then starts systemd as PID 1, which mounts filesystems and starts services in parallel until the default target is reached."

---

### Q19. What is systemd? How do you manage a service and make it start at boot?
**In simple words:** systemd is the service manager on every modern Linux (RHEL 7+, Ubuntu 16+). A **service** is a program that runs in the background (nginx, postgresql, your Java app). systemd starts them at boot, restarts them if they crash, and collects their logs. You control it with **`systemctl`**.

| Command | What it does |
|---|---|
| `systemctl status nginx` | Is it running? Shows state, PID, and the last log lines. Always run this first. |
| `systemctl start / stop / restart nginx` | restart = stop + start (short downtime) |
| `systemctl reload nginx` | Re-read the config **without** stopping (only if the service supports it) |
| `systemctl enable nginx` | Start automatically at every boot. **`start` alone does not survive a reboot.** `enable --now` does both. |
| `systemctl disable nginx` | Don't start at boot |
| `systemctl list-units --failed` | Which services failed |
| `systemctl daemon-reload` | Run after you create or edit a unit file |

**Unit file:** a small text file that tells systemd how to run your app. Here is the one from the course's final lab:
```ini
# /etc/systemd/system/mercury.service
[Unit]
Description=Project Mercury Web Application
After=network.target postgresql.service   # start after these

[Service]
ExecStart=/usr/bin/python3 manage.py runserver 0.0.0.0:8000
WorkingDirectory=/opt/caleston-code/mercuryProject/
User=mercury                 # never run apps as root
Restart=on-failure           # auto-restart if it crashes
RestartSec=10

[Install]
WantedBy=multi-user.target   # which target "enable" attaches it to
```
Then: `systemctl daemon-reload` → `systemctl enable --now mercury` → `systemctl status mercury`.

**Say this in the interview:** "systemd is the init system and service manager. I use `systemctl status`, `start`, `stop`, `restart`, `reload` and `enable`. Enable makes a service start at boot; start only runs it now. To run an application as a service I write a unit file with ExecStart, User, Restart=on-failure and WantedBy=multi-user.target, then daemon-reload and enable it."

---

### Q20. Where are the logs and how do you read them?
**In simple words:** logs are your main evidence during an incident. There are two places:

1. **The journal** (collected by systemd): read with `journalctl`.
   - `journalctl -u nginx` → logs of one service
   - `journalctl -u nginx -f` → follow live (like `tail -f`)
   - `journalctl -u nginx --since "1 hour ago"` → time window
   - `journalctl -p err -b` → only errors since boot
   - `journalctl -k` or `dmesg -T` → **kernel** messages: hardware errors, disk errors, **OOM kills**
2. **Text files in `/var/log`**:
   - `/var/log/messages` (RHEL) or `/var/log/syslog` (Ubuntu): general system log
   - `/var/log/secure` (RHEL) or `/var/log/auth.log` (Ubuntu): logins, sudo, SSH failures
   - `/var/log/cron`: cron jobs
   - application logs, e.g. `/var/log/nginx/error.log`, `/opt/app/logs/app.log`

**Reading tricks:** `tail -f app.log` (follow), `tail -n 200 app.log`, `grep -i error app.log`, `grep -A5 Exception app.log` (show 5 lines after each match: great for Java stack traces), `less app.log` then `/ERROR` to search.

**Log rotation:** `logrotate` (config in `/etc/logrotate.d/`) compresses and deletes old logs so the disk doesn't fill up. If a disk fills with logs, rotation is probably misconfigured.

**Say this in the interview:** "I use `journalctl -u <service>` for systemd services, `dmesg` or `journalctl -k` for kernel messages like OOM kills, and the files under `/var/log` such as messages, secure and the application's own logs. I follow logs with `tail -f` or `journalctl -f` and grep for errors with context."

---

### Q21. A service does not start after a reboot. How do you troubleshoot?
**In simple words:** this is one of the most common real-life tasks. Work through it in order:

1. **`systemctl status svc`**: is it `inactive`, `failed`, or did it never start? Read the last log lines shown.
2. **`journalctl -xeu svc`**: the full error. Usually tells you directly.
3. Check the common causes:
   - **Not enabled**: `systemctl is-enabled svc` → `disabled` means nobody told it to start at boot. Fix: `systemctl enable svc`.
   - **Config error**: a typo in the config file. Many services can self-check: `nginx -t`, `sshd -t`, `apachectl configtest`.
   - **Port already in use**: another process took the port: `ss -tulnp | grep :8080`.
   - **Permissions**: the service user can't read its files or write its log/pid directory (`ls -l`, ownership).
   - **Dependency not ready**: it needs the database or network that wasn't up yet. Fix with `After=` / `Requires=` in the unit file.
   - **Disk full**: `df -h`. Services can't write their pid/log files.
   - **A mount missing**: the app lives on `/data` which didn't mount.
4. Fix, `systemctl daemon-reload` if you edited the unit, `systemctl restart svc`, confirm with `status` and by testing the app (`curl localhost:port`).

**Say this in the interview:** "I'd start with `systemctl status` and `journalctl -xeu` for the exact error, then check whether the service is enabled, whether its config is valid, whether its port is already taken, permissions on its files, dependencies like the database or network, and disk space. After fixing I restart it and verify the application actually responds."

---

## A6. Disk and storage

### Q22. The disk is 100% full. What do you do?
**In simple words:** a full disk breaks almost everything (apps can't write logs, databases stop, even SSH can fail). Act fast but carefully.

1. **Find what's using space:** `df -h` (which filesystem), then `du -xh /var --max-depth=2 | sort -rh | head -20` to drill into the biggest folders. Usual suspects: `/var/log` (logs not rotated), `/tmp`, old application releases, core dumps, Docker images.
2. **Free space safely:**
   - **Do not `rm` a log file that is currently being written** (see Q23 for why). Instead **truncate** it: `> /var/log/app/huge.log` (empties it in place).
   - Delete old *rotated* logs (`*.gz`, `*.1`), clear caches (`yum clean all` / `apt clean`), `journalctl --vacuum-size=500M`, `docker system prune`.
3. **Check for the hidden case:** `lsof +L1` shows deleted files still held open (Q23).
4. **Confirm** with `df -h` and check the application recovered.
5. **Prevent:** fix `logrotate`, add a disk-usage alert at 80%, extend the volume (LVM, Q25), or move logs to their own partition.

**Say this in the interview:** "I'd find the biggest directories with `du`, free space safely by truncating active logs rather than deleting them, removing rotated logs and caches, and check `lsof +L1` for deleted-but-open files. Then I'd fix the root cause, usually log rotation, and set an alert so we catch it at 80% next time."

---

### Q23. `df` says the disk is full, but `du` adds up to much less. Why?
**In simple words:** this is a favourite interview question because it tests whether you understand how Linux handles files.

When a process has a file **open** and someone **deletes** that file, Linux removes the *name* from the directory, but the **data stays on disk until the process closes the file**. `du` counts files by walking the directory names, so it can't see the deleted file anymore. `df` asks the filesystem for its real usage, which still includes that data. Result: `df` full, `du` small.

**Classic cause:** someone deleted a huge log file while the application was still writing to it. The app keeps writing into a file that has no name, and the space is never freed.

**How to find it:** `lsof +L1` (or `lsof | grep deleted`) lists processes holding deleted files, with their PID and FD number.

**How to fix:** restart that process (it closes the file, space is released), or if you can't restart, empty the file through the process's FD: `> /proc/<PID>/fd/<FD>`.

**Lesson:** never delete active logs; **truncate** them instead (`> file`).

**Say this in the interview:** "A process is still holding a deleted file open, so the data is still allocated. `du` only sees files that have names, `df` sees real usage. I'd find it with `lsof +L1` and either restart the process or truncate the file via `/proc/<pid>/fd`. That's also why I truncate live logs instead of deleting them."

---

### Q24. What is `/etc/fstab`, and what are the steps to add a new disk?
**In simple words:** `/etc/fstab` is the list of disks that should be attached (mounted) automatically at boot: which device, where in the tree, what filesystem type, and options. If you mount a disk by hand with `mount`, it disappears after a reboot unless you add it here.

A line: `UUID=3f2a...  /data  xfs  defaults  0 0` → device (by UUID, from `blkid`), mount point, type, options, dump flag, fsck order.

**Adding a new disk, end to end:**
1. `lsblk` → see the new disk, e.g. `/dev/sdb`.
2. Partition it: `fdisk /dev/sdb` (or `gdisk` for big disks) → `/dev/sdb1`.
3. Create a filesystem: `mkfs.xfs /dev/sdb1` (RHEL default) or `mkfs.ext4 /dev/sdb1`.
4. `mkdir /data` and `mount /dev/sdb1 /data`. Check with `df -h`.
5. Get the UUID with `blkid /dev/sdb1` and add the line to `/etc/fstab`.
6. **Test with `mount -a` before rebooting.** A typo in fstab can stop the server from booting (it drops into emergency mode). Add the `nofail` option for non-critical disks so the boot continues even if the disk is missing.

**Say this in the interview:** "fstab defines what gets mounted at boot. To add a disk I'd find it with `lsblk`, partition it, create a filesystem with `mkfs`, mount it, then add a UUID-based entry to `/etc/fstab` and validate with `mount -a`, because a bad fstab entry can break the next boot."

---

### Q25. What is LVM, and how do you extend a volume that is full?
**In simple words:** without LVM, a partition has a fixed size and growing it is painful. **LVM (Logical Volume Manager)** adds a flexible layer: you pool one or more disks into a **Volume Group** and carve **Logical Volumes** out of it. You can grow a logical volume **while it is mounted and in use**, with no downtime. That is why enterprise servers use it.

Three layers, bottom to top:
- **PV (Physical Volume)**: a real disk or partition prepared for LVM: `pvcreate /dev/sdb`
- **VG (Volume Group)**: a pool made of PVs: `vgcreate datavg /dev/sdb`
- **LV (Logical Volume)**: a "virtual partition" from the pool: `lvcreate -L 10G -n datalv datavg` → then `mkfs` and mount it like any disk.

**Extending a full volume (the practical question):**
1. `vgs` → does the volume group have free space?
2. `lvextend -L +10G /dev/datavg/datalv` → make the LV bigger.
3. Grow the filesystem inside it: `xfs_growfs /data` (xfs) or `resize2fs /dev/datavg/datalv` (ext4). Shortcut: `lvextend -r -L +10G ...` does both steps.
4. `df -h` → confirm.
5. If the VG has no free space: add a disk → `pvcreate /dev/sdc` → `vgextend datavg /dev/sdc` → then steps 2–3.

**Say this in the interview:** "LVM pools disks into volume groups and carves logical volumes that can be resized online. To extend a full volume I check free space with `vgs`, run `lvextend`, then grow the filesystem with `xfs_growfs` or `resize2fs`, or `lvextend -r` to do both. If the group is out of space I add a disk with `pvcreate` and `vgextend` first."

---

## A7. Networking

### Q26. Explain the networking basics: IP address, subnet, gateway, port, DNS.
**In simple words:** think of a server like a house.
- **IP address** = the house's street address (`192.168.1.10`). Every machine on a network has one. `ip a` shows yours.
- **Subnet** = the neighbourhood. `192.168.1.0/24` means "all addresses starting with 192.168.1" (the `/24` says the first 24 bits are fixed, leaving 256 addresses). Machines in the same subnet talk directly through a **switch**.
- **Gateway (router)** = the exit of the neighbourhood. To reach a different subnet or the internet, traffic goes to the gateway. `ip route` shows the **default route** (`default via 192.168.1.1`), meaning "anything I don't know, send here."
- **Port** = a door number inside the house. One server can run many services; each listens on a port. Common ones: **22 SSH, 53 DNS, 80 HTTP, 443 HTTPS, 3306 MySQL, 5432 PostgreSQL, 1521 Oracle, 8080 app servers**. Ports below 1024 need root to open.
- **DNS** = the phone book: turns names (`apple.com`) into IPs, because humans remember names and computers need numbers.
- **localhost / 127.0.0.1** = "this machine itself". A service bound only to 127.0.0.1 can't be reached from other machines.

**Say this in the interview:** "An IP identifies a machine, the subnet mask defines the local network, the default gateway is where traffic for other networks goes, a port identifies a service on a machine, and DNS resolves names to IPs. I check these with `ip a`, `ip route`, `ss -tulnp` and `dig`."

---

### Q27. What is the difference between TCP and UDP? What is the 3-way handshake?
**In simple words:** both carry data over IP, but with different promises.

- **TCP** is like a **phone call**: first both sides connect, then data flows, and every piece is confirmed. If something is lost it is resent, and everything arrives in order. Slower but reliable. Used by HTTP/HTTPS, SSH, databases, email.
- **UDP** is like **sending postcards**: no connection, no confirmation, no ordering. Fast and light. Used by DNS lookups, video/voice streaming, NTP (time).

**The TCP 3-way handshake** is how the "phone call" is set up:
1. Client → Server: **SYN** ("can we talk?")
2. Server → Client: **SYN-ACK** ("yes, can you hear me?")
3. Client → Server: **ACK** ("yes") → connection established.

**Connection states you'll see in `ss`:** `LISTEN` (server waiting), `ESTABLISHED` (talking), `TIME_WAIT` (just closed, normal, lasts a minute), `CLOSE_WAIT` (the other side closed but **our app hasn't closed its side**; many of these = a bug/leak in the app).

**Say this in the interview:** "TCP is connection-oriented and reliable: it uses a SYN, SYN-ACK, ACK handshake, acknowledges data and retransmits losses, so it's used for HTTP, SSH and databases. UDP is connectionless and faster with no guarantees, used for DNS and streaming."

---

### Q28. How does DNS resolution work on a Linux server? Which files are involved?
**In simple words:** when a program on the server asks "what is the IP of `db01.example.com`?", Linux checks in this order:

1. **`/etc/hosts`**: a local text file of `IP  name` pairs. Checked first. Handy for testing or overriding a name.
2. **`/etc/resolv.conf`**: lists the **DNS servers** to ask (`nameserver 10.0.0.2`) and search domains.
3. The DNS server either knows the answer (cached) or walks the DNS tree: **root servers → `.com` servers → `example.com`'s own server**, gets the IP, caches it for the **TTL** (time to live), and returns it.

(The order "hosts first, then DNS" is defined in `/etc/nsswitch.conf`: `hosts: files dns`.)

**Record types:** **A** = name → IPv4, **AAAA** = name → IPv6, **CNAME** = alias to another name, **MX** = mail server, **PTR** = reverse (IP → name), **NS** = which servers are authoritative, **TXT** = free text (verification).

**Tools:** `dig db01.example.com` (detailed), `dig +short name`, `dig @8.8.8.8 name` (ask a specific DNS server: if this works but the default doesn't, your DNS server is the problem), `nslookup name`.

**Say this in the interview:** "The resolver checks `/etc/hosts` first, then asks the nameservers in `/etc/resolv.conf`, which recursively query root, TLD and authoritative servers and cache the answer for its TTL. I troubleshoot with `dig`, and `dig @8.8.8.8` to tell whether the problem is my DNS server or the record itself. Common records are A, AAAA, CNAME, MX and PTR."

---

### Q29. What happens when you type `https://www.apple.com` in a browser?
**In simple words:** this question is asked everywhere. It checks whether you can connect DNS, TCP, TLS and HTTP into one story. Tell it as steps:

1. **DNS**: the browser needs the IP. It checks its cache, the OS checks `/etc/hosts`, then asks the DNS server (Q28) → gets an IP.
2. **TCP connection**: the browser opens a connection to that IP on **port 443** using the 3-way handshake (Q27).
3. **TLS handshake** (the "S" in HTTPS): the server sends its **certificate**; the browser checks it was signed by a trusted authority and matches the name; they agree on encryption keys. From now on everything is encrypted.
4. **HTTP request**: the browser sends `GET / HTTP/1.1` with headers (host, cookies, user agent).
5. **Server side**: the request hits a **load balancer / CDN**, which forwards it to one of many **web/app servers**; the app may query a **database** or cache.
6. **HTTP response**: status code (**200 OK**), headers, and the HTML body.
7. **Rendering**: the browser parses the HTML and makes more requests for CSS, JavaScript and images, then draws the page.

**Say this in the interview:** "DNS resolves the name to an IP, the browser does a TCP handshake to port 443, then a TLS handshake to verify the certificate and set up encryption, sends an HTTP GET, the request goes through a load balancer to an application server which may hit a database, and the response comes back with a status code and HTML which the browser renders, fetching further assets."

---

### Q30. How do you find which process is using a port, and whether a service is listening?
**In simple words:** when "the app is not reachable," the first question is: *is anything actually listening on that port?* And when "port already in use," you need to know *who* has it.

**Command:** `ss -tulnp` (older systems: `netstat -tulnp`). The letters: **t** TCP, **u** UDP, **l** listening only, **n** numeric (don't resolve names), **p** show the process.

```
$ ss -tulnp | grep 8080
tcp  LISTEN 0 128  0.0.0.0:8080  0.0.0.0:*  users:(("java",pid=2231,fd=45))
```
This says: Java with PID 2231 is listening on port 8080 on **all interfaces** (`0.0.0.0`).

**Important detail:** if the address shows **`127.0.0.1:8080`** instead of `0.0.0.0:8080`, the app only accepts connections **from the same machine**. Users from outside will get "connection refused" even though the app is running. This is a very common misconfiguration.

Other ways: `lsof -i :8080`, `fuser 8080/tcp`.

**Say this in the interview:** "`ss -tulnp` shows listening ports with their process. I check that the service is listening on the expected port and on 0.0.0.0 rather than only 127.0.0.1, which would make it unreachable from outside. `lsof -i :port` gives the same information."

---

### Q31. Which tools do you use to troubleshoot "I can't connect to server X"?
**In simple words:** you test each layer from near to far, so you know exactly where it breaks.

| Step | Tool | What it proves |
|---|---|---|
| 1. Is my own network up? | `ip a`, `ip route` | I have an IP and a default route |
| 2. Does the name resolve? | `dig X` / `nslookup X` | DNS works, I have the right IP |
| 3. Can I reach the machine at all? | `ping X` | Basic connectivity (note: some firewalls block ping, so no reply isn't always "down") |
| 4. Where does the path break? | `traceroute X` (or `mtr X`) | Shows each router hop; the hop where replies stop is where the problem is |
| 5. Is the specific port open? | `nc -zv X 5432` or `telnet X 5432` | "succeeded" = reachable; "connection refused" = nothing listening; "timed out" = firewall or routing |
| 6. Does the service answer properly? | `curl -v http://X:8080/health` | Shows the full HTTP conversation and status code |
| 7. On the server itself | `ss -tulnp`, `systemctl status`, firewall rules | Is it listening, running, allowed? |

**How traceroute works** (sometimes asked): it sends packets with a "time to live" of 1, then 2, then 3… Each router reduces TTL by one and, when it hits zero, sends back an error. That error reveals the router's address, so you see the path hop by hop.

**Say this in the interview:** "I go layer by layer: `ip a` and `ip route` for my own config, `dig` for DNS, `ping` for reachability, `traceroute` to find where the path breaks, `nc -zv` to test the specific port, and `curl -v` to test the application. Connection refused means nothing is listening; a timeout usually means a firewall."

---

### Q32. What do HTTP status codes 401, 403, 404, 500, 502, 503 and 504 mean?
**In simple words:** as application support you will read these in logs and dashboards all day. The first digit tells the family: **2xx success, 3xx redirect, 4xx client's fault, 5xx server's fault.**

| Code | Meaning | What it usually tells you |
|---|---|---|
| **200** | OK | Fine |
| **301/302** | Redirect | Normal |
| **400** | Bad request | Client sent something malformed |
| **401** | Unauthorized | Not logged in / token missing or expired |
| **403** | Forbidden | Logged in, but **not allowed** (permissions) |
| **404** | Not found | Wrong URL, or the file/route doesn't exist on this server (wrong deployment?) |
| **429** | Too many requests | Rate limiting |
| **500** | Internal server error | **A bug or exception in the application.** Look in the app log for the stack trace |
| **502** | Bad gateway | The proxy/load balancer **couldn't get a valid response from the backend**: backend down, crashed, or restarting |
| **503** | Service unavailable | Backend overloaded, in maintenance, or **no healthy instances** behind the load balancer |
| **504** | Gateway timeout | The backend is **too slow** to respond. Often a slow database query or a hung dependency |

**Say this in the interview:** "4xx means the client's request is the issue, 401 unauthenticated, 403 forbidden, 404 not found. 5xx means our side: 500 is an application error, 502 means the load balancer got no valid response from the backend, 503 means no healthy or available backend, and 504 means the backend timed out, which usually points to a slow dependency like the database."

---

### Q33. What is a firewall on Linux? What's the difference between DROP and REJECT?
**In simple words:** a firewall decides which network traffic is allowed in or out of the server. On Linux the engine is **netfilter**, controlled by `iptables` (classic) or newer front-ends: **firewalld** on RHEL (`firewall-cmd`) and **ufw** on Ubuntu. In the cloud there is also an external layer, **security groups**, which can block traffic before it even reaches your server. When something isn't reachable, check **both**.

**Mental model of iptables:** rules are grouped in chains: **INPUT** (traffic coming to this server), **OUTPUT** (traffic leaving), **FORWARD** (passing through). Rules are checked **top to bottom, first match wins**.

```bash
iptables -L -n --line-numbers                              # list rules
iptables -A INPUT -p tcp -s 10.1.1.5 --dport 22 -j ACCEPT  # allow SSH from one IP
iptables -A INPUT -p tcp --dport 80 -j ACCEPT              # allow HTTP from anyone
iptables -A INPUT -j DROP                                  # then drop everything else
firewall-cmd --add-port=8080/tcp --permanent && firewall-cmd --reload   # firewalld way
```

**DROP vs REJECT:** both block. **DROP** silently throws the packet away: the client waits and eventually **times out**. **REJECT** sends back an error so the client fails **immediately** ("connection refused"). DROP hides the server from scanners; REJECT is friendlier for internal services because people get a clear error fast.

**Say this in the interview:** "Linux filters traffic with netfilter via iptables, firewalld or ufw, plus cloud security groups outside the host. Rules are evaluated top-down. DROP silently discards packets so the client times out, while REJECT returns an error so the client fails immediately. When a port isn't reachable I check the local firewall and the security group."

---

## A8. SSH, packages, cron

### Q34. How does SSH key-based login work, and why does it still ask for a password sometimes?
**In simple words:** SSH is how you log in to a remote Linux server securely (port 22). Instead of typing a password, you can use a **key pair**:
- a **private key** (`~/.ssh/id_rsa` or `id_ed25519`), which stays on your laptop and is never shared;
- a **public key** (`id_rsa.pub`), which you copy to the server into the user's `~/.ssh/authorized_keys` file.

When you connect, the server sends a challenge that only your private key can answer. No password travels over the network. Setup: `ssh-keygen -t ed25519` → `ssh-copy-id user@server` → `ssh user@server`.

**"I added my key but it still asks for a password"** (a very common interview question). The usual causes:
1. **Wrong permissions.** SSH refuses keys if files are too open: `~/.ssh` must be **700**, `authorized_keys` **600**, your private key **600**, and your home directory must not be writable by others.
2. The public key is in the **wrong user's** `authorized_keys`, or you're connecting as the wrong user.
3. The server disallows it: `PubkeyAuthentication no` in `/etc/ssh/sshd_config`.
4. On RHEL, SELinux labels: `restorecon -Rv ~/.ssh`.

**How to debug:** `ssh -vvv user@server` on the client shows exactly which key is offered and why it's rejected; on the server, `/var/log/secure` (RHEL) or `/var/log/auth.log` (Ubuntu).

**Related:** "Host key verification failed" means the server's identity changed compared to what's saved in your `~/.ssh/known_hosts` (server reinstalled, or something suspicious). Verify, then `ssh-keygen -R server` to remove the old entry.

**Say this in the interview:** "SSH key auth uses a private key on the client and the public key in the server's authorized_keys. If it still prompts for a password, the usual cause is permissions: .ssh must be 700 and authorized_keys 600. I'd debug with `ssh -vvv` on the client and the auth log on the server, and check sshd_config allows public key authentication."

---

### Q35. How do you install software on RHEL vs Ubuntu? What's the difference between `rpm` and `yum`?
**In simple words:** software comes as **packages** (a compressed bundle of program files plus a list of other packages it depends on). Each Linux family has its own format and tools:

| | RHEL / CentOS / Rocky / Oracle Linux | Ubuntu / Debian |
|---|---|---|
| Package file | `.rpm` | `.deb` |
| Low-level tool (installs one file, **doesn't fetch dependencies**) | `rpm -ivh pkg.rpm` | `dpkg -i pkg.deb` |
| High-level tool (downloads from repositories and **resolves dependencies**) | `yum install -y httpd` (or `dnf`) | `apt install nginx` |
| Remove | `yum remove httpd` | `apt remove nginx` |
| Update everything | `yum update` | `apt update` (refresh list) then `apt upgrade` |
| Is it installed? | `rpm -qa \| grep httpd` | `dpkg -l \| grep nginx` |
| Which package gives this command? | `yum provides */tcpdump` | `apt-file search tcpdump` |
| Repository config | `/etc/yum.repos.d/*.repo` | `/etc/apt/sources.list` |

**`rpm` vs `yum`:** `rpm` just installs the file you give it; if a dependency is missing it fails. `yum`/`dnf` sits on top of rpm, looks up the configured repositories, downloads the package **and all dependencies**, and installs them in the right order. That's why you almost always use `yum`.

**Say this in the interview:** "RHEL uses rpm packages with yum or dnf, Ubuntu uses deb packages with apt. The low-level tools rpm and dpkg install a single file without resolving dependencies, while yum and apt download from repositories and resolve dependencies automatically, so I use those in practice."

---

### Q36. How does cron work? Schedule a job every 15 minutes. Why might a cron job not run?
**In simple words:** `cron` is the built-in scheduler. You give it a timetable (a **crontab**) and it runs commands at those times: backups, cleanups, health checks.

Edit your crontab with `crontab -e`, list it with `crontab -l`. Each line has 5 time fields then the command:
```
┌ minute (0-59)
│  ┌ hour (0-23)
│  │  ┌ day of month (1-31)
│  │  │  ┌ month (1-12)
│  │  │  │  ┌ day of week (0-6, Sunday = 0)
*  *  *  *  *  /path/to/command
```
Examples: `*/15 * * * *` every 15 minutes · `0 2 * * *` every day at 02:00 · `0 9 * * 1-5` 09:00 Monday to Friday · `@reboot` once at boot.

Always redirect output to a log so you can see failures: `*/15 * * * * /opt/scripts/check.sh >> /var/log/check.log 2>&1`

**Why a cron job "didn't run" (the classic follow-up):**
1. Cron runs with a **tiny environment**: almost no `PATH`. Commands that work in your terminal fail in cron. **Use full paths** (`/usr/bin/python3`, `/opt/scripts/x.sh`).
2. The script isn't executable (`chmod +x`).
3. It ran but failed silently because output wasn't logged. Add `>> log 2>&1`.
4. It's in a different user's crontab (`crontab -u bob -l`), or `crond` isn't running (`systemctl status crond`).
5. Check the log: `/var/log/cron` (RHEL) or `grep CRON /var/log/syslog` (Ubuntu).

**Say this in the interview:** "Cron runs scheduled commands from a crontab with minute, hour, day, month and weekday fields; `*/15 * * * *` runs every 15 minutes. When a job doesn't run I check `/var/log/cron`, make sure the script uses full paths because cron has a minimal environment, that it's executable, that output is redirected to a log, and that it's in the right user's crontab."

---

## A9. Shell and text tools

### Q37. What do `>`, `>>`, `2>`, `2>&1` and `|` do?
**In simple words:** every command has three "streams": **stdin (0)** where input comes from, **stdout (1)** where normal output goes, **stderr (2)** where error messages go. Redirection changes where those go.

| Symbol | Meaning | Example |
|---|---|---|
| `>` | send stdout to a file, **overwriting** it | `ls > files.txt` |
| `>>` | send stdout to a file, **appending** | `echo "done" >> run.log` |
| `2>` | send **errors** to a file | `cmd 2> errors.txt` |
| `2>/dev/null` | throw errors away (`/dev/null` is a black hole) | `find / -name x 2>/dev/null` |
| `> out.log 2>&1` | send stdout to the file **and** errors to the same place | `backup.sh > backup.log 2>&1` ← the one you'll use in cron |
| `\|` (pipe) | send the output of one command **as input** to the next | `ps aux \| grep java` |
| `\| tee -a file` | show on screen **and** save to file | `./deploy.sh 2>&1 \| tee -a deploy.log` |

**Why `2>&1` reads strangely:** it means "point stream 2 (errors) to wherever stream 1 (stdout) is currently pointing." The order matters: `> out.log 2>&1` works; `2>&1 > out.log` sends errors to the screen.

**Say this in the interview:** "`>` overwrites, `>>` appends, `2>` redirects errors, and `2>&1` sends errors to the same place as standard output. A pipe sends one command's output into the next command's input, which is how I chain tools like `ps`, `grep`, `sort` and `uniq`."

---

### Q38. How do you use `grep`, `find`, `awk` and `sed` in day-to-day support?
**In simple words:** these four tools are your daily toolkit for digging through logs and files.

**`grep`: search text inside files.**
```bash
grep "ERROR" app.log            # lines containing ERROR
grep -i "timeout" app.log       # ignore case
grep -c "ERROR" app.log         # count matching lines
grep -v "DEBUG" app.log         # lines NOT containing DEBUG
grep -r "db.host" /opt/app/conf # search recursively in a folder
grep -n "Exception" app.log     # show line numbers
grep -A5 -B2 "Exception" app.log   # 5 lines After and 2 Before each match (stack traces!)
grep -E "ERROR|FATAL" app.log   # either word
```

**`find`: search for files by name, size, age.**
```bash
find /var/log -name "*.log"               # by name
find / -type f -size +1G 2>/dev/null      # files bigger than 1 GB
find /tmp -type f -mtime +7 -delete       # delete files older than 7 days
find /opt -name "*.conf" -exec grep -l "8080" {} \;   # which conf files mention 8080
```

**`awk`: work with columns** (splits each line on spaces by default; `$1` is column 1).
```bash
awk '{print $1}' access.log               # first column (the client IP)
awk -F: '{print $1}' /etc/passwd          # use ":" as separator → usernames
awk '$9 == 500' access.log                # lines where column 9 equals 500
```

**`sed`: find and replace in a stream or file.**
```bash
sed 's/8080/9090/g' app.conf              # show the file with 8080 replaced by 9090
sed -i 's/8080/9090/g' app.conf           # -i edits the file in place
sed -n '100,120p' app.log                 # print lines 100–120
```

**The most-asked one-liner**, "top 10 IPs hitting the web server":
```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```
Read it as: take column 1 (IP) → sort so duplicates are together → `uniq -c` counts each → sort by count, biggest first → first 10.

**Say this in the interview:** "I use grep to search logs, with -i, -c, -v and -A/-B for context around exceptions; find to locate files by name, size or age; awk to pull out columns like IPs or status codes; and sed for in-place replacements. A typical one-liner is awk, sort, uniq -c, sort -rn, head to get the top IPs in an access log."

---

### Q39. Which variables and basics of Bash scripting should I know?
**In simple words:** you won't be asked to write big programs, but you may be asked to write a 5–10 line script or explain one. Know these pieces:

```bash
#!/bin/bash                 # first line: which interpreter runs this script
set -euo pipefail           # stop on any error, on unset variables, or a failed pipe (safer scripts)

NAME="om"                   # variable (no spaces around =)
echo "Hello $NAME"          # use it with $
TODAY=$(date +%F)           # store a command's output

echo "$0"   # script name     "$1"  # first argument     "$#"  # number of arguments
echo "$?"   # exit code of the last command: 0 = success, anything else = failure

if [ -f /etc/hosts ]; then echo "exists"; fi        # -f file exists, -d directory, -z string empty
if [ "$1" == "start" ]; then echo start; else echo usage; fi

for s in web1 web2 web3; do                          # loop over a list
  ping -c1 "$s" > /dev/null && echo "$s up" || echo "$s DOWN"
done

while read -r line; do echo "$line"; done < servers.txt   # loop over lines of a file
```

**Two scripts to be able to write from memory:**
```bash
# 1) alert if any filesystem is above 80%
df -hP | awk 'NR>1 {gsub("%","",$5); if ($5+0 > 80) print "ALERT:", $6, "at", $5"%"}'

# 2) restart a service if it is down
if ! systemctl is-active --quiet nginx; then
  echo "$(date) nginx down, restarting" >> /var/log/svc-watch.log
  systemctl restart nginx
fi
```

**Say this in the interview:** "I write small Bash scripts for health checks and cleanup: variables, `$?` for exit codes, `if` with file tests, `for` and `while` loops, and I use `set -euo pipefail` so a script stops on errors. For example a disk-usage alert with `df` and `awk`, or a check that restarts a service if `systemctl is-active` fails."

---

### Q40. What are `strace` and `lsof`, and when would you use them?
**In simple words:** two detective tools for "the process is running but something is wrong and the logs don't say why."

- **`lsof`** = **l**i**s**t **o**pen **f**iles. Shows every file and network connection a process has open.
  - `lsof -p 1234` → what is process 1234 holding?
  - `lsof -i :8080` → who is using port 8080?
  - `lsof +L1` → deleted files still held open (the `df` vs `du` mystery).
  - `lsof /data` → who is using this folder (why `umount` says "busy").
- **`strace`** = trace system calls. Shows live what a process is asking the kernel to do: which files it opens, which hosts it connects to, where it hangs.
  - `strace -p 1234` → attach to a running process. If the output stops at `connect(... 10.0.0.5:5432)` the app is stuck connecting to the database. If you see `open("/etc/app/config.yml") = -1 ENOENT`, it can't find its config.
  - `strace -f ./start.sh` → trace a command and its children from the start.
  - `strace -c -p 1234` → a summary count of syscalls (useful when `sy` CPU is high).

**Say this in the interview:** "`lsof` lists open files and sockets per process, which I use to find who holds a port, a mount or a deleted file. `strace` shows a process's system calls, so when an application hangs or fails silently I attach with `strace -p` to see what file it can't open or which connection it's stuck on."

---

### Q41. What is a container in Linux terms? (Docker/Kubernetes connection)
**In simple words:** the recruiter listed Kubernetes, so you may get this bridge question. A container is **not** a small virtual machine. It is a **normal Linux process** that the kernel isolates using two features:
- **Namespaces** control what the process can **see**: its own process list (PID namespace), its own network interfaces and IP (network namespace), its own filesystem view (mount namespace), its own hostname.
- **cgroups** (control groups) control what the process can **use**: a maximum amount of CPU and memory.

Because of this, when a container exceeds its memory limit, it is **OOM-killed** exactly like Q15 and exits with code **137**. When you run `ps` on the host you see container processes as normal processes. All containers share the host's kernel, which is why they start in milliseconds, unlike VMs which boot a whole OS.

**Say this in the interview:** "A container is a regular Linux process isolated with namespaces, which limit what it can see, and cgroups, which limit CPU and memory. Containers share the host kernel, which is why they're lightweight. A container killed for exceeding its memory limit shows exit code 137, the same OOM mechanism as on a normal server."

---

### Q42. What is the difference between `restart` and `reload`, and between `stop` and `kill`?
**In simple words:** small words, big consequences in production.

- **`systemctl restart svc`**: stops the process and starts a new one. Existing connections are cut; there is a short outage.
- **`systemctl reload svc`**: tells the running process (usually with SIGHUP) to **re-read its configuration** without stopping. No outage. Only works if the service supports it (nginx, sshd, haproxy do). Use reload after a config change when possible.
- **`systemctl stop svc`**: sends SIGTERM, waits (default 90 s) for a clean shutdown, then SIGKILL if needed.
- **`kill -9`**: immediate, no cleanup (Q11). Avoid on databases and anything that writes files.

**Say this in the interview:** "Restart stops and starts the service, so connections drop. Reload re-reads configuration without stopping, so I prefer it after config changes when the service supports it. Stop sends SIGTERM for a graceful shutdown, and I only use kill -9 as a last resort."

---

### Q43. What is `/etc/hosts` vs `/etc/resolv.conf` vs `/etc/nsswitch.conf`?
**In simple words:** three files that decide how names become IPs (see Q28).
- **`/etc/hosts`**: manual `IP name` entries, checked first. Useful to test a new server before DNS is updated, or to temporarily point `db01` somewhere else.
- **`/etc/resolv.conf`**: which **DNS servers** to ask (`nameserver 10.0.0.2`) and which domains to append (`search example.com`, so `ping db01` tries `db01.example.com`).
- **`/etc/nsswitch.conf`**: the **order** of lookups: `hosts: files dns` means hosts file first, then DNS.

**Say this in the interview:** "`/etc/hosts` holds static name-to-IP entries checked first, `/etc/resolv.conf` lists the DNS servers and search domains, and `/etc/nsswitch.conf` defines the lookup order between them."

---

### Q44. What are environment variables and `$PATH`? Why do I get "command not found"?
**In simple words:** environment variables are named settings that programs read, like `HOME`, `USER`, `JAVA_HOME`. `env` lists them, `echo $HOME` prints one. `X=1` sets a variable only in the current shell; `export X=1` makes it available to programs you start from this shell. To make it permanent, put the `export` line in `~/.bashrc` or `~/.profile` (or `/etc/profile.d/` for everyone).

**`$PATH`** is the list of folders the shell searches when you type a command: `/usr/local/bin:/usr/bin:/bin`. If a program is installed in a folder not in `PATH`, you get **"command not found"** even though it exists. Fix: run it with the full path, or `export PATH=$PATH:/opt/tool/bin`. `which java` shows which file would run.

Application-support angle: apps often fail because `JAVA_HOME` or similar is set in your shell but **not** in the service's environment. For systemd services, set variables with `Environment=` in the unit file or `EnvironmentFile=`.

**Say this in the interview:** "Environment variables are key-value settings inherited by processes; `export` makes them available to child processes, and `$PATH` is the list of directories searched for commands. 'Command not found' usually means the binary's directory isn't in PATH. For services I set variables in the unit file because they don't read my shell profile."

---

### Q45. What is `vi`? How do you edit a config file with it?
**In simple words:** `vi`/`vim` is the editor that exists on every Linux server, including ones with nothing else installed. You must be able to edit a config file with it without panicking.

It has modes. You start in **command mode** (keys are commands, not text).
- Press **`i`** → **insert mode**: now you type text. Press **`Esc`** to go back to command mode.
- Press **`:`** → **last-line mode**: type a command and Enter: **`:w`** save, **`:q`** quit, **`:wq`** save and quit, **`:q!`** quit without saving.
- Useful command-mode keys: **`/text`** search (then `n` for next), **`dd`** delete the line, **`yy`** copy line, **`p`** paste, **`u`** undo, **`x`** delete a character, **`G`** go to end, **`gg`** go to top, **`:set nu`** show line numbers.

Typical task: `sudo vi /etc/ssh/sshd_config` → `/PermitRootLogin` → press `i`, change `yes` to `no`, `Esc`, `:wq` → `sshd -t` to validate → `systemctl reload sshd`.

**Say this in the interview:** "I'm comfortable with vi: `i` to insert, Esc to return to command mode, `/` to search, `dd` to delete a line, `u` to undo, and `:wq` or `:q!` to save or discard. After editing a config I validate it, for example `nginx -t` or `sshd -t`, and reload the service."

---

# Part B — Troubleshooting scenarios
Apple candidates reported a "pretend broken server" exercise. The interviewer describes a symptom and you talk through what you'd do. **Use this shape every time:**

> **Clarify** (what exactly, since when, one server or all, any recent change?) → **Check** (say the commands and what you look for) → **Cause** → **Mitigate now** (get users working again) → **Fix properly** → **Prevent** (monitoring, automation, runbook).

### The 60-second health check (memorise this order)
```
uptime                 → load average vs number of cores
dmesg -T | tail        → kernel errors: OOM kills, disk errors
top                    → which process; us / sy / wa / st; any D or Z states
free -h                → available memory, swap in use?
df -h ; df -i          → disk space AND inodes
iostat -x 1            → is the disk saturated? (%util near 100, high await)
ss -tulnp              → is the app listening, on which IP and port?
journalctl -u app -n 100 / tail -f /var/log/app.log  → errors, stack traces
curl -v localhost:PORT/health                         → does the app answer locally?
```

---

### B1. "Users say the application is slow."
**Clarify:** slow for everyone or some? Since when? Any deploy or config change today? Is it the web page, the API, or a specific action?
**Check:** run the 60-second check. Then branch on what you see:
- High **`us`** CPU on the app process → the app is working hard: traffic spike, a bad loop, heavy job. For Java, `top -H -p <pid>` shows the busy threads; a thread dump (`jstack`) shows what they're doing.
- High **`wa`** or D-state processes → disk is the bottleneck (`iostat -x`: `%util` ~100, high `await`). Logs flooding? Backup running? Slow storage?
- **Swap** in use, low `available` → memory pressure; look for a process with growing RSS.
- Normal CPU/memory but slow → it's probably **a dependency**: database (slow queries, locks, connection pool full), an external API, DNS slowness. Check app logs for timeouts, and test `curl -w "%{time_total}"` against the dependency.
**Mitigate:** restart the stuck process, roll back the latest deploy, scale out, or kill the runaway job.
**Prevent:** dashboards for latency and saturation, alerts, capacity planning, fix the query/code.

---

### B2. "The disk is full." / "`df` shows 100% but `du` doesn't."
See **Q22** and **Q23**. Key points to say: find big folders with `du`, **truncate** active logs instead of deleting, `lsof +L1` for deleted-but-open files, fix logrotate, alert at 80%, extend with LVM.

### B3. "There is free space but the app says 'No space left on device'."
**Cause:** inodes exhausted. **Check:** `df -i` → 100% IUse. **Find:** a folder with millions of tiny files (session files, cache, mail queue): `find /var -xdev -type f | cut -d/ -f1-3 | sort | uniq -c | sort -rn | head`. **Fix:** delete them, fix the application/cron that creates them, consider a filesystem with more inodes for that mount. (See Q7.)

### B4. "A process keeps dying randomly with nothing in its log."
**Cause:** usually the **OOM killer**. **Check:** `dmesg -T | grep -iE "oom|killed process"`, `journalctl -k`. Confirm which process and how much memory it had. **Fix:** memory leak (RSS growing day by day → restart as a stop-gap, fix the code), Java heap set too large for the machine, too many processes, add RAM or set limits. In Kubernetes the same shows as `OOMKilled` / exit 137. (See Q15.)

### B5. "The app is running but users can't reach it."
Work from inside out:
1. Is the process alive? `systemctl status app`.
2. Is it **listening**? `ss -tulnp | grep 8080`. On `0.0.0.0` or only on `127.0.0.1`? (127.0.0.1 = only local access.)
3. Does it answer locally? `curl -v localhost:8080/health`.
4. **Firewall:** `iptables -L -n` / `firewall-cmd --list-all` / cloud security group.
5. From another machine: `nc -zv server 8080` (refused = not listening or firewall REJECT, timeout = firewall DROP or routing).
6. **DNS:** does the name resolve to this server? `dig app.example.com`.
7. **Load balancer:** is this server marked healthy? Is the health-check path returning 200? Certificate expired?
(See Q30, Q31, Q33.)

### B6. "I can't SSH into the server."
1. `ssh -vvv user@host` → read where it stops.
2. `ping host`, `nc -zv host 22`: **timeout** = network/firewall/server down; **connection refused** = sshd not running or wrong port.
3. "Permission denied (publickey)" → wrong key/user, bad permissions (`~/.ssh` 700, `authorized_keys` 600), key missing from `authorized_keys`.
4. Through a console (cloud console / IPMI): `systemctl status sshd`, `/var/log/secure`, `df -h` (a full disk can break logins), `sshd -t` to validate config, fail2ban banned your IP?
(See Q34.)

### B7. "A service didn't come back after a reboot."
See **Q21**: `systemctl status` → `journalctl -xeu` → enabled? config valid? port in use? permissions? dependency/mount? disk full?

### B8. "The server won't boot. Someone edited `/etc/fstab` yesterday."
**Cause:** a wrong fstab line makes systemd wait for a filesystem that never mounts, so boot drops into **emergency mode**. **Fix:** at the emergency prompt (or rescue mode from GRUB), `mount -o remount,rw /`, open `/etc/fstab`, comment out or fix the bad line, `mount -a` to test, reboot. **Prevent:** always run `mount -a` after editing fstab, use UUIDs, add `nofail` for non-critical disks. (See Q24.)

### B9. "The application logs 'Too many open files'."
**Check:** `cat /proc/<pid>/limits | grep "open files"` vs `ls /proc/<pid>/fd | wc -l`. **Decide:** if usage is legitimately high for a busy server → raise `LimitNOFILE=65535` in the systemd unit, `daemon-reload`, restart. If the count **keeps growing** over time → a connection/file leak in the app (check `ss -tan | grep CLOSE_WAIT`); raising the limit only delays the crash; the code must close its resources. (See Q13.)

### B10. "The website returns 502 Bad Gateway."
**Meaning:** the proxy/load balancer (nginx, ALB) could not get a valid answer from the backend. **Check:** is the backend app running and listening on the port nginx expects? (`systemctl status`, `ss -tulnp`), nginx error log (`/var/log/nginx/error.log`: "connection refused" = backend down, "upstream timed out" = backend slow → that's 504 territory), did the backend just restart or crash (OOM?), is the upstream config pointing to the right host/port? **Mitigate:** restart/roll back the backend; **Prevent:** health checks, multiple backends, alerts on 5xx rate. (See Q32.)

### B11. "A config change was deployed and now the app is broken."
**Say this first:** "I'd roll back to restore service, then investigate." Restore the previous config (from Git, backup, or config management like Ansible), validate (`nginx -t`), reload/restart, confirm with `curl`. Then `diff` old vs new config to find the mistake, test in staging, and make sure future changes go through the change process with a tested rollback plan.

### B12. The course's final lab: "Bob's app works on his laptop but not on the dev server."
This is the KodeKloud module 10 story and it mirrors real support work. The flow:
1. Start the app → it crashes with a **stack trace**. **Read it**: "could not connect to database at localhost:5432."
2. On the DB server: `systemctl status postgresql` → `inactive (dead)` → start it, enable it.
3. Which port is it really on? `ss -tlnp | grep postgres`.
4. The app config still says `localhost` and the wrong port → find it (`grep -r "DATABASES" /opt/app`) and set the DB host and port correctly. Also allow the app host in `pg_hba.conf`.
5. Files were extracted as root → `chown -R mercury /opt/caleston-code` so the app user can read them.
6. Don't leave it running in a terminal: create a **systemd unit** with `Restart=on-failure` and `enable` it (Q19).
**Lessons to say out loud:** read the error first; check dependencies; "works on my machine" means a config difference; check permissions after copying as root; run apps as services.

---

# Part C — One-liners you may be asked to type
Say what each part does as you type it. Correct and simple beats clever.

| Task | Command | How to read it |
|---|---|---|
| Top 10 IPs in an access log | `awk '{print $1}' access.log \| sort \| uniq -c \| sort -rn \| head -10` | column 1 → group → count → biggest first → top 10 |
| Count HTTP status codes | `awk '{print $9}' access.log \| sort \| uniq -c \| sort -rn` | column 9 is the status in the common log format |
| How many ERROR lines | `grep -c ERROR app.log` | -c counts |
| Show exceptions with context | `grep -n -A5 "Exception" app.log` | 5 lines after each match |
| Errors in the last hour (systemd) | `journalctl -u app --since "1 hour ago" -p err` | |
| Files bigger than 1 GB | `find / -xdev -type f -size +1G -exec ls -lh {} + 2>/dev/null` | -xdev stays on this filesystem |
| Delete logs older than 7 days | `find /var/log/app -name "*.log" -mtime +7 -delete` | test first without `-delete` |
| Biggest folders | `du -xh / --max-depth=2 2>/dev/null \| sort -rh \| head -20` | |
| Who listens on 8080 | `ss -tulnp \| grep :8080` | |
| Follow a log, only ERROR lines | `tail -F app.log \| grep --line-buffered ERROR` | |
| Screen and file at once | `./run.sh 2>&1 \| tee -a run.log` | |
| Replace a port in all configs | `sed -i 's/8080/9090/g' /opt/app/conf/*.conf` | back up first |
| Top memory processes | `ps -eo pid,user,rss,cmd --sort=-rss \| head` | rss = real RAM in KB |
| Check many URLs | `while read u; do c=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$u"); [ "$c" != 200 ] && echo "FAIL $u $c"; done < urls.txt` | prints only failures |

---

# Part D — Last-day cheat sheet

| Need | Command |
|---|---|
| Load / cores | `uptime`, `nproc` |
| Kernel errors / OOM | `dmesg -T \| tail`, `dmesg -T \| grep -i oom` |
| Processes | `top` (P/M to sort), `ps -eo pid,ppid,user,%cpu,%mem,rss,stat,cmd --sort=-%cpu \| head` |
| Kill | `kill <pid>` (15, polite) → `kill -9 <pid>` (last resort), `pkill -f name` |
| Memory | `free -h` (look at *available*), `vmstat 1` (si/so should be 0) |
| Disk | `df -h`, `df -i`, `du -xh --max-depth=2 \| sort -rh \| head`, `lsof +L1`, `iostat -x 1` |
| Services | `systemctl status/start/stop/restart/reload/enable --now svc`, `daemon-reload`, `list-units --failed` |
| Logs | `journalctl -u svc -f`, `journalctl -xeu svc`, `tail -f /var/log/messages`, `grep -A5 Exception app.log` |
| Ports | `ss -tulnp`, `lsof -i :8080` |
| Network | `ip a`, `ip route`, `ping`, `traceroute`, `nc -zv host port`, `curl -v`, `dig`, `dig @8.8.8.8 name` |
| Firewall | `iptables -L -n --line-numbers`, `firewall-cmd --list-all` |
| Permissions | `ls -l`, `chmod 755`, `chmod 600 key`, `chown -R user:group dir` |
| Users | `id`, `whoami`, `who`, `last`, `usermod -aG group user`, `sudo` |
| Packages | `yum install/remove/provides`, `rpm -qa`, `apt update && apt upgrade`, `dpkg -l` |
| Storage | `lsblk`, `blkid`, `mount -a`, `pvs vgs lvs`, `lvextend -r -L +10G /dev/vg/lv` |
| SSH | `ssh -vvv user@host`, `ssh-copy-id`, perms 700/600, `ssh-keygen -R host` |
| Cron | `crontab -e`, `*/15 * * * * /path/job.sh >> /var/log/job.log 2>&1` |
| Deep dive | `strace -p <pid>`, `lsof -p <pid>`, `ls /proc/<pid>/fd \| wc -l` |

**Three sentences to say in every troubleshooting answer**
1. "First I'd confirm the impact and check what changed, then restore service, then find the root cause."
2. "I avoid `kill -9` and deleting live logs; I use SIGTERM and truncate."
3. "After fixing it I'd add an alert and a runbook or automation so it doesn't happen again."

**If you don't know something:** say "I haven't used that, but here's how I'd find out: man page, logs, and testing in a safe environment." Interviewers on the threads said they reward this.

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
