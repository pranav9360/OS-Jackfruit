# OS-Jackfruit — Supervised Multi-Container Runtime

## 1. Team Information

| Name         | SRN           |
| ------------ | ------------- |
| D PRANAV SAI | PES1UG24CS139 |
| DARSHAN GS   | PES1UG24CS138 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

* Ubuntu 22.04 or 24.04 in a VM (Secure Boot OFF, no WSL)
* Install dependencies:

```bash
sudo apt install -y build-essential linux-headers-$(uname -r)
```

---

### Download Alpine rootfs

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

---

### Build everything

```bash
make
```

This builds:

* `engine`
* `memory_hog`
* `cpu_hog`
* `io_pulse`
* `monitor.ko`

---

### Load kernel module

```bash
sudo insmod monitor.ko
sudo sysctl kernel.dmesg_restrict=0
ls -la /dev/container_monitor
```

---

### Start supervisor (Terminal 1)

```bash
sudo ./engine supervisor ./rootfs-base
```

---

### Container operations (Terminal 2)

```bash
sudo ./engine start alpha ./rootfs-base /bin/sh
sudo ./engine run test1 ./rootfs-base /bin/hostname
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
```

---

### Memory limit test (IMPORTANT)

```bash
sudo cp memory_hog ./rootfs-base/
sudo ./engine start memtest ./rootfs-base /memory_hog --soft-mib 5 --hard-mib 10
dmesg | grep container_monitor
```

👉 When the hard limit is exceeded, the kernel module sends **SIGKILL** to terminate the container.

---

### Scheduler experiments

```bash
sudo cp cpu_hog ./rootfs-base/
sudo cp io_pulse ./rootfs-base/

# Experiment 1: Different priorities
sudo ./engine start cpu_hi ./rootfs-base /cpu_hog --nice 0
sudo ./engine start cpu_lo ./rootfs-base /cpu_hog --nice 19
sleep 12
cat logs/cpu_hi.log
cat logs/cpu_lo.log

# Experiment 2: CPU-bound vs I/O-bound
sudo ./engine start cpu_w ./rootfs-base /cpu_hog --nice 0
sudo ./engine start io_w  ./rootfs-base /io_pulse --nice 0
sleep 12
cat logs/cpu_w.log
cat logs/io_w.log
```

---

### Clean shutdown

```bash
sudo pkill engine
sudo rmmod monitor
dmesg | grep container_monitor | tail -5
```

---

## 3. Demo Screenshots

**Screenshot 1 — Two containers running under one supervisor**

<img width="818" height="88" alt="111" src="https://github.com/user-attachments/assets/caf2e22a-df88-4ae1-8d9f-2094a2d65a56" />



---

**Screenshot 2 — ps output showing container metadata**

<img width="1280" height="800" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/333.jpeg" />

---

**Screenshot 3 — Logging pipeline proof**

<img width="677" height="617" alt="image" src="https://github.com/user-attachments/assets/0915b05e-38ff-49f5-a86a-d826101572b2" />

---

**Screenshot 4 — CLI command and supervisor response**

<img width="1063" height="207" alt="Screenshot 2026-04-15 192257" src="https://github.com/user-attachments/assets/194c4dd2-394e-4c9b-b853-4f99582feb0a" />


---

**Screenshot 5 — Soft limit screenshot**

<img width="1280" height="202" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/666.jpeg" />

---

**Screenshot 6 — Hard limit screenshot**

<img width="1280" height="202" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/666.jpeg" />

---

**Screenshot 7 — Scheduler experiment results**

<img width="712" height="590" alt="image" src="https://github.com/user-attachments/assets/6c46d7a3-e061-4851-83fb-9fe4bf669458" />

<img width="770" height="616" alt="image" src="https://github.com/user-attachments/assets/9c208a5e-646f-4d8b-8fe3-66f170104179" />


---

**Screenshot 8 — Clean teardown, no zombies**

<img width="1280" height="202" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/999.jpeg" />

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

In our project, we achieved container isolation mainly using Linux namespaces along with `chroot()`.

We used three namespaces:

* **PID namespace (`CLONE_NEWPID`)** so that each container has its own process ID space. Inside the container, processes start from PID 1, even though the host still tracks their real PIDs.
* **UTS namespace (`CLONE_NEWUTS`)** to give each container its own hostname. This helps in distinguishing containers without affecting the host system.
* **Mount namespace (`CLONE_NEWNS`)** to isolate the filesystem view. For example, when we mount `/proc` inside a container, it is only visible inside that container.

We also used `chroot()` to change the root directory of the container process to the container’s filesystem. This ensures the container cannot access files outside its root.

However, all containers still share the same Linux kernel. So while processes are isolated, system calls are still handled by the host kernel.

---

### 4.2 Supervisor and Process Lifecycle

We implemented a long-running supervisor process to manage all containers.

Containers are created using `clone()` instead of `fork()` because `clone()` allows us to specify namespace flags. Each container becomes a child process of the supervisor.

The supervisor keeps track of:

* container ID
* PID
* current state

When a container exits, the kernel sends a `SIGCHLD` signal. The supervisor handles this using `waitpid()` with `WNOHANG` to clean up the process.

This is important because without this, exited processes would become zombies and remain in the process table. The supervisor ensures proper lifecycle management and cleanup.

---

### 4.3 IPC, Threads, and Synchronization

We used two different IPC mechanisms in our design:

* **Pipes** to capture container output (`stdout` and `stderr`)
* **UNIX domain sockets** for communication between CLI commands and the supervisor

For logging, we implemented a **producer-consumer model**:

* Producer threads read data from pipes and push it into a buffer
* A consumer thread reads from the buffer and writes to log files

Since multiple threads access shared data, race conditions can occur. For example:

* Two producers writing at the same time can overwrite data
* The consumer might read incomplete data

To prevent this, we used:

* `pthread_mutex_t` for locking
* `pthread_cond_t` for coordination between threads

This ensures that logging is consistent and no data is lost.

---

### 4.4 Memory Management and Enforcement

We monitored memory usage using **RSS (Resident Set Size)**, which represents the actual physical memory used by a process.

RSS only includes memory currently in RAM. It does not include swapped-out memory or unused virtual memory.

We implemented two types of limits:

* **Soft limit** → gives a warning when exceeded
* **Hard limit** → forces the container to terminate

When the hard limit is crossed, the kernel module sends a **SIGKILL** signal to immediately stop the container.

We chose to implement this in kernel space instead of user space because:

* The kernel has direct access to process memory structures
* There is no delay between detection and enforcement
* It avoids race conditions that can happen in user-space monitoring

---

### 4.5 Scheduling Behavior

We observed scheduling behavior using CPU-bound workloads.

Linux uses the **Completely Fair Scheduler (CFS)**, which distributes CPU time based on priority (`nice` values).

From our experiments:

* CPU-bound processes continuously used CPU and produced increasing output
* Processes that perform I/O or sleep give up CPU time, allowing other processes to run

This shows that CFS tries to maintain:

* **Fairness** → CPU is shared between processes
* **Responsiveness** → I/O-bound processes are not starved
* **Efficiency** → CPU is utilized effectively

Overall, the behavior we observed matched the expected working of the Linux scheduler.

## 5. Design Decisions and Tradeoffs

* No network namespace (simplifies implementation)
* Single-threaded supervisor (simpler, safe)
* Kernel module ensures accurate enforcement

---

## 6. Scheduler Experiment Results

* Higher priority → more CPU share
* I/O-bound processes remain responsive
