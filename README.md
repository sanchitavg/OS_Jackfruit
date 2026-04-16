# Multi-Container Runtime — OS Jackfruit

## 1. Team Information

| Sanchita V G | PES2UG24CS439 |
| Samyuktha Ramesh Babu | PES2UG24CS437 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites
- Ubuntu 22.04 or 24.04 VM (tested on 24.04 aarch64)
- Secure Boot OFF
- Dependencies: `sudo apt install -y build-essential linux-headers-$(uname -r) git wget`

### Build

```bash
cd boilerplate
make clean
make
```

### Prepare Root Filesystems

```bash
cd ..   # go to repo root
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-minirootfs-3.20.3-aarch64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-aarch64.tar.gz -C rootfs-base
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta

# Copy workload binaries into rootfs
cp boilerplate/cpu_hog rootfs-alpha/
cp boilerplate/memory_hog rootfs-alpha/
cp boilerplate/cpu_hog rootfs-beta/
cp boilerplate/memory_hog rootfs-beta/
```

### Load Kernel Module

```bash
cd boilerplate
sudo insmod monitor.ko
ls -l /dev/container_monitor   # verify device exists
sudo dmesg | tail -3           # verify module loaded
```

### Start the Supervisor (Terminal 1)

```bash
cd boilerplate
sudo ./engine supervisor ./rootfs-base
```

### Launch Containers (Terminal 2)

```bash
cd boilerplate

# Start two containers
sudo ./engine start alpha ../rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ../rootfs-beta  /cpu_hog --soft-mib 64 --hard-mib 96

# List containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha

# Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Memory Limit Test

```bash
sudo ./engine start memtest ../rootfs-alpha /memory_hog --soft-mib 10 --hard-mib 20
sudo dmesg | tail -10   # watch for SOFT LIMIT and HARD LIMIT events
```

### Scheduling Experiment

```bash
sudo ./engine start highpri ../rootfs-alpha /cpu_hog --nice -10
sudo ./engine start lowpri  ../rootfs-beta  /cpu_hog --nice 10
sleep 12
sudo ./engine logs highpri
sudo ./engine logs lowpri
```

### Unload Module and Clean Up

```bash
# Stop supervisor with Ctrl+C in Terminal 1, then:
sudo rmmod monitor
make clean
```

---

## 3. Demo Screenshots

### Screenshot 1 — Multi-Container Supervision
Two containers (alpha, beta) running concurrently under one supervisor process.


*Two containers started with `engine start`, both showing `running` state under a single supervisor.*

---

### Screenshot 2 — Metadata Tracking
Output of `engine ps` showing tracked container metadata.


*`engine ps` shows ID, PID, STATE, STARTED time, and LOG path for each tracked container.*

---

### Screenshot 3 — Bounded-Buffer Logging
Log file contents captured through the logging pipeline.


*`engine logs alpha` streams log data captured by the producer/consumer pipeline. `cat logs/alpha.log` confirms persistence to disk.*

---

### Screenshot 4 — CLI and IPC
CLI command issued to supervisor over UNIX domain socket, supervisor responding.


*`engine logs alpha` sends a request over `/tmp/mini_runtime.sock`; supervisor responds with `log: logs/alpha.log` and streams the log content.*

---

### Screenshot 5 — Soft-Limit Warning
`dmesg` showing soft-limit warning event for container `memtest`.


*Kernel module detects RSS exceeding soft limit (10 MiB) and logs: `SOFT LIMIT container=memtest pid=3731 rss=17170432 limit=10485760`*

---

### Screenshot 6 — Hard-Limit Enforcement
`dmesg` showing container killed after exceeding hard limit.


*Kernel module sends SIGKILL when RSS exceeds hard limit (20 MiB): `HARD LIMIT container=memtest pid=3731 rss=25559040 limit=20971520`. Entry is then removed from the monitored list.*

---

### Screenshot 7 — Scheduling Experiment
Two CPU-bound containers running with different `nice` values (-10 vs +10).

*`highpri` (nice=-10) and `lowpri` (nice=10) both complete 10-second runs. Both accumulate similar iteration counts because the VM has 4 CPUs — scheduling differences are more visible under CPU contention on a single-core system.*

---

### Screenshot 8 — Clean Teardown
Supervisor exits cleanly after Ctrl+C; no zombie processes remain.


*`[supervisor] Shutting down...` then `[supervisor] Exited cleanly.` confirms all containers were stopped, logging thread joined, and all resources freed.*

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

The runtime achieves isolation by calling `clone()` with three namespace flags: `CLONE_NEWPID` gives the container its own PID namespace so processes inside see themselves as PID 1; `CLONE_NEWUTS` gives each container its own hostname (set to the container ID via `sethostname`); `CLONE_NEWNS` gives each container its own mount namespace. After cloning, `child_fn` calls `chroot()` into the container's dedicated rootfs directory, then mounts `/proc` so tools like `ps` work inside. The host kernel is still shared — the container's PID 1 is visible as a normal process on the host, memory management and scheduling are host-managed, and networking is not isolated in this implementation.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is useful because containers need a parent that outlives them to reap exit status and prevent zombie processes. When a container process exits, the kernel sends `SIGCHLD` to its parent (the supervisor). The supervisor installs a `SA_RESTART` signal handler that calls `waitpid(-1, &status, WNOHANG)` in a loop to reap all exited children without blocking. Metadata (state, exit code, exit signal) is updated under `metadata_lock`. The `stop_requested` flag distinguishes voluntary stop (SIGTERM sent by `engine stop`) from a forced kill (SIGKILL by the kernel module), which maps to `CONTAINER_STOPPED` vs `CONTAINER_KILLED` in metadata.

### 4.3 IPC, Threads, and Synchronization

The project uses two distinct IPC mechanisms. **Path A (logging)**: each container's stdout and stderr are connected to the supervisor via a `pipe()`. A dedicated producer thread per container reads from the pipe and inserts chunks into a shared bounded buffer. A single consumer thread (the logger) drains the buffer and writes to per-container log files. The bounded buffer is protected by a `pthread_mutex_t`; `pthread_cond_t not_full` blocks producers when the buffer is full, and `pthread_cond_t not_empty` blocks the consumer when it is empty. Without these, producers and consumer would race on `head`, `tail`, and `count`, causing lost writes or corruption. **Path B (control)**: the CLI client connects to the supervisor over a UNIX domain socket (`/tmp/mini_runtime.sock`), sends a `control_request_t` struct, and reads back a `control_response_t`. A separate `metadata_lock` mutex protects the container linked list from concurrent access by the SIGCHLD handler and the main event loop.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the pages of a process currently loaded in physical RAM. It does not count pages that have been swapped out, memory-mapped files not yet read, or shared library pages counted multiple times. Soft and hard limits are different policies: the soft limit triggers a warning log when first crossed but does not stop the process, giving the runtime a chance to react gracefully; the hard limit enforces a strict ceiling by sending SIGKILL. Enforcement belongs in kernel space because a user-space poller could be delayed by scheduling, a misbehaving process could ignore signals from user space, and the kernel has direct access to `mm_struct` RSS counters via `get_mm_rss()` without needing to parse `/proc`.

### 4.5 Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS) which allocates CPU time proportional to weight, where weight is derived from `nice` value. A process with nice=-10 gets roughly 9× the weight of a process with nice=+10. In our experiment, `highpri` (nice=-10) and `lowpri` (nice=10) both completed their 10-second runs in approximately the same wall-clock time because the VM has 4 CPUs and both containers could run in parallel. On a single-core system or under heavier load, `highpri` would finish sooner while `lowpri` would be starved. The experiment confirms the scheduler is tracking both workloads and applying the priority configuration passed through our `nice()` call inside `child_fn`.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** Used `clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` and `chroot()`.
**Tradeoff:** `chroot` is simpler than `pivot_root` but can be escaped via `..` traversal by a root process inside the container.
**Justification:** Sufficient for a research/demo runtime; `pivot_root` would be the right choice for a production container engine.

### Supervisor Architecture
**Choice:** Single-process supervisor with `select()`-based event loop and one logger consumer thread.
**Tradeoff:** A single event loop serializes command handling — a slow `CMD_RUN` blocks other commands.
**Justification:** Simple and correct for the assignment scope; a production system would use a thread-per-client or async I/O model.

### IPC / Logging
**Choice:** Pipes for logging (Path A) and UNIX domain sockets for control (Path B).
**Tradeoff:** Pipes are unidirectional and simple but require one pipe pair per container. Sockets add complexity but enable bidirectional request/response.
**Justification:** The two-path design matches the project spec and cleanly separates log streaming from control signalling.

### Kernel Monitor
**Choice:** `mutex` (sleeping lock) rather than `spinlock` for the monitored list.
**Tradeoff:** Mutexes cannot be held in hard interrupt context. Since our timer callback and ioctl handler are both process/softirq context (not hard IRQ), a mutex is safe and avoids busy-waiting.
**Justification:** The list iterations can be long (many containers), making a spinlock wasteful. Mutex allows the CPU to sleep during contention.

### Scheduling Experiments
**Choice:** Used `nice()` in `child_fn` to set priority, and compared cpu_hog logs between high/low priority containers.
**Tradeoff:** Nice values affect CFS weight but don't give real-time guarantees; results vary with number of CPUs.
**Justification:** Nice values are the simplest Linux scheduling knob available without requiring root-level `SCHED_FIFO` or `SCHED_RR` policies.

---

## 6. Scheduler Experiment Results

### Experiment: CPU-bound workloads at different priorities

| Container | Nice Value | Duration | Final Accumulator |
|-----------|-----------|----------|-------------------|
| highpri   | -10       | 10s      | 3178973564847289  |
| lowpri    | +10       | 10s      | 1267999038734506  |

Both containers completed in 10 seconds (wall clock). The VM has 4 CPUs so both could run fully in parallel. The difference in accumulator values reflects CPU instruction throughput — `highpri` ran more iterations per second due to higher scheduler weight reducing context switch overhead and getting larger time slices.

**Conclusion:** On a multi-core VM, nice value differences are less dramatic than on a single CPU because containers don't compete for the same core. The Linux CFS scheduler correctly applies the weight difference, as evidenced by the higher iteration count for the high-priority container. On a single-core system or under heavier CPU load, the low-priority container would show significantly longer completion times.
