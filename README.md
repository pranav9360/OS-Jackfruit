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

<img width="677" height="617" alt="image" src="https://github.com/user-attachments/assets/0915b05e-38ff-49f5-a86a-d826101572b2" />


---

**Screenshot 2 — ps output showing container metadata**

<img width="1280" height="800" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/333.jpeg" />

---

**Screenshot 3 — Logging pipeline proof**

<img width="1280" height="800" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/333.jpeg" />

---

**Screenshot 4 — CLI command and supervisor response**

<img width="1063" height="207" alt="Screenshot 2026-04-15 192257" src="https://github.com/user-attachments/assets/194c4dd2-394e-4c9b-b853-4f99582feb0a" />


---

**Screenshot 5 — Soft limit warning in dmesg**

<img width="1280" height="202" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/666.jpeg" />

---

**Screenshot 6 — Hard limit kill in dmesg**

<img width="1280" height="202" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/666.jpeg" />

---

**Screenshot 7 — Scheduler experiment results**

<img width="1280" height="589" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/777.jpeg" />

<img width="1280" height="401" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/888.jpeg" />

---

**Screenshot 8 — Clean teardown, no zombies**

<img width="1280" height="202" src="https://github.com/pranav9360/OS-Jackfruit/blob/main/Screenshot/999.jpeg" />

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

The runtime uses Linux namespaces (`CLONE_NEWPID`, `CLONE_NEWUTS`, `CLONE_NEWNS`) and `chroot` for isolation. Containers get separate PID space, hostname, and mount space, while sharing the same kernel.

---

### 4.2 Supervisor and Lifecycle

A supervisor process manages containers, prevents zombies using `waitpid`, and handles communication via UNIX sockets.

---

### 4.3 IPC and Logging

* Pipes capture container stdout/stderr
* Producer-consumer model stores logs
* UNIX domain sockets handle CLI commands

---

### 4.4 Memory Monitoring

* RSS (Resident Set Size) tracks actual RAM usage
* Soft limit → warning
* Hard limit → enforced kill via SIGKILL

---

### 4.5 Scheduling Behavior

Linux CFS scheduler distributes CPU based on `nice` values. CPU-bound and I/O-bound processes behave differently due to blocking.

---

## 5. Design Decisions and Tradeoffs

* No network namespace (simplifies implementation)
* Single-threaded supervisor (simpler, safe)
* Kernel module ensures accurate enforcement

---

## 6. Scheduler Experiment Results

* Higher priority → more CPU share
* I/O-bound processes remain responsive
