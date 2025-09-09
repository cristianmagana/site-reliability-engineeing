# Linux Commands Reference Guide

This guide provides essential Linux commands for system administration, monitoring, and troubleshooting with practical examples and output explanations.

## Process and System Information

### Process Information Commands

#### `ps aux`

Shows all running processes with detailed information.

```bash
ps aux
```

**Output columns:**

- USER: Process owner
- PID: Process ID
- %CPU: CPU usage percentage
- %MEM: Memory usage percentage
- VSZ: Virtual memory size (KB)
- RSS: Resident memory size (KB)
- TTY: Terminal type
- STAT: Process state
- START: Start time
- TIME: CPU time used
- COMMAND: Command that started the process

#### `pstree`

Displays processes in a tree format showing parent-child relationships.

```bash
pstree
```

Shows hierarchical process relationships, useful for understanding process spawning.

#### `top`

Real-time display of running processes and system resource usage.

```bash
top
```

**Key metrics displayed:**

- Load average (1, 5, 15 minutes)
- CPU usage breakdown
- Memory usage (total, used, free, buffers)
- Process list sorted by CPU usage

#### `htop`

Enhanced interactive version of top with color coding and mouse support.

```bash
htop
```

More user-friendly interface with color-coded information and additional features.

### Process Identification and Management

#### Finding Processes

**Find processes by name:**

```bash
ps aux | grep processname
pgrep processname          # Returns PID only
pgrep -f processname       # Match full command line
```

**Find processes by user:**

```bash
ps aux | grep username
ps -u username
```

**Find processes using a port:**

```bash
netstat -tuln | grep :8080
ss -tuln | grep :8080
lsof -i :8080
```

**Find processes by PID:**

```bash
ps -p 1234
ps aux | grep 1234
```

#### Process State Codes (STAT column in ps aux)

- **R**: Running or runnable
- **S**: Interruptible sleep (waiting for event)
- **D**: Uninterruptible sleep (usually I/O)
- **Z**: Zombie process
- **T**: Stopped (by job control signal)
- **t**: Stopped by debugger
- **W**: Paging (not valid since kernel 2.6)
- **X**: Dead (should never be seen)

**Additional modifiers:**

- **<**: High priority (not nice to other users)
- **N**: Low priority (nice to other users)
- **L**: Has pages locked into memory
- **s**: Session leader
- **l**: Multi-threaded
- **+**: In foreground process group

#### Killing Processes

**Kill by PID:**

```bash
kill 1234              # Send SIGTERM (graceful shutdown)
kill -9 1234          # Send SIGKILL (force kill)
kill -15 1234         # Send SIGTERM explicitly
```

**Kill by process name:**

```bash
pkill processname      # Kill all processes matching name
killall processname   # Kill all processes by name
pkill -f pattern      # Kill processes matching pattern in command line
```

**Kill processes by user:**

```bash
pkill -u username     # Kill all processes owned by user
killall -u username   # Alternative syntax
```

**Signal types:**

- **SIGTERM (15)**: Graceful termination request
- **SIGKILL (9)**: Immediate termination (cannot be caught/ignored)
- **SIGHUP (1)**: Hangup signal (often reloads config)
- **SIGSTOP (19)**: Stop process (cannot be caught/ignored)
- **SIGCONT (18)**: Continue stopped process

#### Advanced Process Commands

**Process tree with PIDs:**

```bash
pstree -p              # Show PIDs in tree
pstree -p username     # Show tree for specific user
```

**Detailed process information:**

```bash
ps -ef                 # Full format listing
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu    # Custom format sorted by CPU
```

**Process resource usage:**

```bash
pidstat               # Per-process statistics
pidstat -p 1234       # Statistics for specific PID
iotop                 # Real-time I/O usage by process
```

**Background process management:**

```bash
nohup command &       # Run command immune to hangups
jobs                  # List active jobs
bg %1                 # Send job to background
fg %1                 # Bring job to foreground
disown %1             # Remove job from shell's job table
```

---

### System Information Commands

#### `uname -a`

Display system information.

```bash
uname -a
```

**Example output:**

```
Linux ubuntu-pod 5.4.0-74-generic #83-Ubuntu SMP Sat May 8 02:35:39 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

Shows: kernel name, hostname, kernel version, build info, architecture.

#### `/proc/version`

Kernel version and build information.

```bash
cat /proc/version
```

**Example output:**

```
Linux version 5.4.0-74-generic (buildd@lcy01-amd64-030) (gcc version 9.4.0)
```

#### `/proc/cpuinfo`

Detailed CPU information.

```bash
cat /proc/cpuinfo
```

**Key information:**

- processor: CPU core number
- model name: CPU model
- cpu MHz: Current frequency
- cache size: CPU cache information
- flags: Supported CPU features

#### `/proc/meminfo`

Detailed memory information.

```bash
cat /proc/meminfo
```

**Key fields:**

- MemTotal: Total physical memory
- MemFree: Free memory
- MemAvailable: Available memory for applications
- Buffers: Memory used for buffers
- Cached: Memory used for caching

#### `free -h`

Memory usage in human-readable format.

```bash
free -h
```

**Example output:**

```
              total        used        free      shared  buff/cache   available
Mem:           7.8G        2.1G        3.2G        156M        2.5G        5.3G
Swap:          2.0G          0B        2.0G
```

### File System Commands

#### `df -h`

Display filesystem disk space usage in human-readable format.

```bash
df -h
```

**Example output:**

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        20G  8.5G   10G  46% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
```

#### `mount`

Display mounted filesystems.

```bash
mount
```

Shows all currently mounted filesystems with mount options.

#### `/proc/mounts`

Kernel's view of mounted filesystems.

```bash
cat /proc/mounts
```

More detailed than `mount` command, shows kernel perspective.

#### `lsblk`

List block devices in tree format.

```bash
lsblk
```

**Example output:**

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk
└─sda1   8:1    0   20G  0 part /
```

## Memory and Virtual Memory

### Memory Statistics

#### `/proc/vmstat`

Virtual memory statistics.

```bash
cat /proc/vmstat
```

**Key metrics:**

- nr_free_pages: Free memory pages
- nr_dirty: Dirty pages waiting to be written
- pgpgin/pgpgout: Pages paged in/out
- pswpin/pswpout: Pages swapped in/out

#### `vmstat 1 5`

Display virtual memory statistics every 1 second for 5 iterations.

```bash
vmstat 1 5
```

**Example output:**

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 3298516 268844 2598808    0    0     8    12   42   78  2  1 97  0  0
```

**Key columns explained:**

**System columns:**
- `in`: Interrupts per second (42/sec) - includes clock and device interrupts
- `cs`: Context switches per second (78/sec) - process/thread switching

**Relationship between `in` and `cs`:**
- Context switches typically exceed interrupts (78 > 42 in this example)
- One interrupt can trigger multiple context switches
- Context switches also occur due to scheduling, I/O waits, and time slicing
- Ratio here: ~1.86 context switches per interrupt

**CPU and Core Detection:**
- vmstat shows combined CPU statistics across all cores
- Cannot directly determine CPU/core count from vmstat output
- Use these commands for CPU/core information:
  ```bash
  nproc                    # Total processing units
  lscpu                    # Detailed CPU architecture
  cat /proc/cpuinfo        # Per-core details
  ```

**Analysis of example output:**
- Light system load (r=1, low context switches)
- Minimal swapping (si=so=0)
- Low I/O activity (bi=8, bo=12)
- CPU mostly idle (id=97%) indicating good availability

### Process Memory Information

#### `/proc/self/maps`

Memory mapping of current process.

```bash
cat /proc/self/maps
```

Shows virtual memory areas, permissions, and backing files.

#### `pmap $$`

Display memory map of current shell process.

```bash
pmap $$
```

Shows detailed memory layout of the specified process.

## Network Internals

### Network Interface Commands

#### `ip addr show`

Display network interface information.

```bash
ip addr show
```

**Example output:**

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
```

#### `ip route show`

Display routing table.

```bash
ip route show
```

Shows how network traffic is routed.

#### `netstat -tuln`

Show listening ports and network connections.

```bash
netstat -tuln
```

**Flags:**

- -t: TCP connections
- -u: UDP connections
- -l: Listening ports only
- -n: Numerical addresses instead of hostnames

#### `ss -tuln`

Modern replacement for netstat.

```bash
ss -tuln
```

Faster and more feature-rich than netstat.

### Network Statistics

#### `/proc/net/dev`

Network interface statistics.

```bash
cat /proc/net/dev
```

**Example output:**

```
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
    lo: 1728      24    0    0    0     0          0         0     1728      24    0    0    0     0       0          0
  eth0: 2847234   1842    0    0    0     0          0         0   156742     876    0    0    0     0       0          0
```

#### `/proc/net/tcp`

TCP connection information.

```bash
cat /proc/net/tcp
```

Shows active TCP connections with socket states.

## File System and I/O

### I/O Statistics

#### `iostat`

Display I/O statistics for devices and partitions.

```bash
iostat
```

**Key metrics:**

- %user: CPU time in user mode
- %system: CPU time in system mode
- %iowait: CPU time waiting for I/O
- tps: Transfers per second
- kB_read/s, kB_wrtn/s: Data read/written per second

#### `/proc/diskstats`

Disk I/O statistics.

```bash
cat /proc/diskstats
```

Raw disk statistics used by tools like iostat.

### File System Information

#### `/proc/filesystems`

Supported filesystem types.

```bash
cat /proc/filesystems
```

Lists filesystem types supported by the kernel.

#### `findmnt`

Display mounted filesystems in tree format.

```bash
findmnt
```

Modern tool for displaying mount information.

#### `stat /etc/passwd`

Display detailed file information.

```bash
stat /etc/passwd
```

**Example output:**

```
  File: /etc/passwd
  Size: 2956      	Blocks: 8          IO Block: 4096   regular file
Device: 801h/2049d	Inode: 131588      Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-06-15 10:30:25.000000000 +0000
Modify: 2021-06-15 10:30:25.000000000 +0000
Change: 2021-06-15 10:30:25.000000000 +0000
```

#### `ls -li /etc/`

List files with inode numbers.

```bash
ls -li /etc/
```

Shows inode numbers along with file details.

## Process Tracing

### System Call Tracing

#### `strace ls /etc/`

Trace system calls made by a command.

```bash
strace ls /etc/
```

Shows all system calls made during command execution.

#### `strace -e trace=file ls /etc/`

Trace only file-related system calls.

```bash
strace -e trace=file ls /etc/
```

Filters to show only file operations (open, read, write, etc.).

#### `ltrace ls /etc/`

Trace library function calls.

```bash
ltrace ls /etc/
```

Shows library function calls instead of system calls.

### Process Monitoring

#### `watch -n 1 'ps aux | head -20'`

Monitor top 20 processes, refreshing every second.

```bash
watch -n 1 'ps aux | head -20'
```

Continuously monitors process list with automatic refresh.

## Kernel and System Calls

### Kernel Modules

#### `lsmod`

List loaded kernel modules.

```bash
lsmod
```

**Example output:**

```
Module                  Size  Used by
fuse                  131072  3
xt_CHECKSUM            16384  1
xt_MASQUERADE          20480  3
```

#### `/proc/modules`

Kernel modules information.

```bash
cat /proc/modules
```

Detailed information about loaded kernel modules.

### System Information

#### `/proc/sys/kernel/osrelease`

Kernel release information.

```bash
cat /proc/sys/kernel/osrelease
```

Shows kernel version string.

#### `sysctl -a | grep vm`

Display virtual memory related kernel parameters.

```bash
sysctl -a | grep vm
```

Shows tunable kernel parameters related to virtual memory.

## Container and Namespace Exploration

### Namespace Information

#### `ls -la /proc/self/ns/`

List namespaces for current process.

```bash
ls -la /proc/self/ns/
```

**Example output:**

```
lrwxrwxrwx 1 root root 0 Jun 15 10:30 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 15 10:30 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Jun 15 10:30 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Jun 15 10:30 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Jun 15 10:30 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jun 15 10:30 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 15 10:30 uts -> 'uts:[4026531838]'
```

#### `readlink /proc/self/ns/*`

Read namespace identifiers.

```bash
readlink /proc/self/ns/*
```

Shows the actual namespace identifiers.

### Cgroup Information

#### `/proc/self/cgroup`

Control group information for current process.

```bash
cat /proc/self/cgroup
```

**Example output:**

```
12:cpuset:/
11:cpu,cpuacct:/
10:memory:/
9:devices:/docker/abc123def456
```

#### `ls -la /sys/fs/cgroup/`

List cgroup filesystem structure.

```bash
ls -la /sys/fs/cgroup/
```

Shows available cgroup controllers and hierarchies.

## Usage Tips

1. **Combine commands**: Use pipes to combine commands for more specific information

   ```bash
   ps aux | grep nginx
   netstat -tuln | grep :80
   ```

2. **Use watch for monitoring**: Add `watch` to continuously monitor changing values

   ```bash
   watch -n 2 'free -h'
   watch -n 1 'ss -tuln'
   ```

3. **Filter with grep**: Most /proc files can be filtered with grep

   ```bash
   cat /proc/cpuinfo | grep "model name"
   cat /proc/meminfo | grep "Mem"
   ```

4. **Save outputs**: Redirect output to files for later analysis
   ```bash
   ps aux > process_snapshot.txt
   dmesg > kernel_messages.log
   ```

This reference provides a foundation for Linux system administration and troubleshooting. Each command offers different perspectives on system state and behavior, making them valuable tools for SRE work.
