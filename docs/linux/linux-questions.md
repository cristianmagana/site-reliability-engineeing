# MongoDB SRE Interview - Complete Study Guide

## Table of Contents

1. [Interview Overview](#interview-overview)
2. [Linux Fundamentals](#linux-fundamentals)
3. [Networking Fundamentals](#networking-fundamentals)
4. [Troubleshooting Scenarios](#troubleshooting-scenarios)
5. [Essential Commands Reference](#essential-commands-reference)
6. [Common Interview Traps](#common-interview-traps)
7. [Communication Tips](#communication-tips)

---

## Interview Overview

### Format (60 minutes)

- **Coding (30 min)**: Language-agnostic practical problems
- **Linux/Networking (30 min)**: Q&A focused on conceptual understanding

### Linux/Networking Focus

- File systems (inodes, links, permissions)
- Process signals and states
- Shell behavior (globbing, parsing)
- TCP/UDP protocols
- Load balancing
- TLS basics

---

## Linux Fundamentals

### File System Operations

#### What Happens When You Run `rm filename`

**Process Flow:**

1. Shell locates `rm` binary via $PATH
2. Shell forks, child process exec's `rm`
3. `rm` calls `unlink()` system call
4. Directory entry (linking filename to inode) is removed
5. Inode's link count is decremented
6. If link count = 0 AND no processes have file open:
   - Inode and data blocks marked as free in filesystem bitmaps
7. If processes still have file open:
   - Deletion deferred until last file descriptor closes
8. Data isn't immediately overwritten (why file recovery works)

**Key Points:**

- No signals involved (SIGTERM is for killing processes, not files)
- Filename lives in directory entry, not inode
- Inode contains: metadata, permissions, timestamps, pointers to data blocks

---

#### Hard Links vs Symbolic Links

| Feature                   | Hard Link                         | Symbolic Link (Symlink)             |
| ------------------------- | --------------------------------- | ----------------------------------- |
| **What it is**            | Directory entry pointing to inode | Special file containing path string |
| **Inode**                 | Shares same inode as target       | Has its own inode                   |
| **After deleting target** | Still works (shares inode)        | Becomes "dangling" (broken link)    |
| **Cross filesystem**      | ❌ No (different inode tables)    | ✅ Yes                              |
| **Link to directories**   | ❌ No (prevents cycles)           | ✅ Yes                              |
| **Visibility**            | Looks like regular file           | Shows as lrwxrwxrwx with -> target  |

**Commands:**

```bash
# Hard link
ln file1.txt file2.txt
ls -i file1.txt file2.txt  # Same inode number

# Symbolic link
ln -s file1.txt file2_link
ls -i file1.txt file2_link  # Different inode numbers
```

**When target is deleted:**

- **Hard link**: Continues to work (both are equal directory entries)
- **Symlink**: Breaks (path no longer exists)

---

#### Recovering from `chmod 000 /bin/chmod`

**Problem:** Even root cannot execute a file with no execute permissions.

**Solutions (in order of preference):**

```bash
# Method 1: Use Python (quickest)
python3 -c "import os; os.chmod('/bin/chmod', 0o755)"

# Method 2: Use Perl
perl -e 'chmod 0755, "/bin/chmod"'

# Method 3: Use busybox
busybox chmod 755 /bin/chmod

# Method 4: Use install command
install -m 755 /bin/chown /tmp/chmod_backup
install -m 755 /tmp/chmod_backup /bin/chmod

# Method 5: Copy from another location
cp /usr/bin/chmod /bin/chmod  # If available elsewhere
```

**Why this works:** Interpreted languages call `chmod()` system call directly without needing to execute the `/bin/chmod` binary.

---

### Process Management

#### Process States

| State                      | Symbol | Description                            | Can Kill?            |
| -------------------------- | ------ | -------------------------------------- | -------------------- |
| Running                    | R      | Executing or ready to run              | ✅ Yes               |
| Sleeping (Interruptible)   | S      | Waiting for event, can be interrupted  | ✅ Yes               |
| Sleeping (Uninterruptible) | D      | Waiting for I/O, cannot be interrupted | ❌ No                |
| Zombie                     | Z      | Terminated but not reaped by parent    | ❌ No (already dead) |
| Stopped                    | T      | Paused by signal (SIGSTOP/SIGTSTP)     | ✅ Yes               |

#### The D State Problem

**What it means:**

- Process stuck in kernel space waiting for I/O
- Cannot receive signals (including SIGKILL)
- Usually lasts milliseconds; hours = serious problem

**Common Causes:**

1. **Hung NFS mount** (most common)
2. Failing disk hardware
3. Overloaded storage backend
4. Kernel bug or driver issue

**Investigation Steps:**

```bash
# 1. Find processes in D state
ps aux | awk '$8 ~ /D/ {print}'
ps -eo pid,stat,comm,wchan | grep "^[0-9]* D"

# 2. Check what it's waiting for (WCHAN column)
# Common values:
# - nfs_sync_inode_wait: NFS issue
# - wait_on_page_writeback: Disk write
# - io_schedule: Generic I/O wait

# 3. See what files are open
sudo lsof -p <PID>

# 4. Check kernel stack trace
sudo cat /proc/<PID>/stack

# 5. I/O investigation
iostat -x 1 5    # Look for %util at 100%, high await
sudo iotop -o    # Per-process I/O
cat /proc/<PID>/io

# 6. Check mount points
mount | grep nfs
timeout 5 ls -l /nfs/mount/point  # Test if responsive
dmesg | grep -i "stale\|nfs"

# 7. Hardware issues
dmesg | grep -i "error\|fail\|ata\|scsi"
sudo smartctl -a /dev/sda
```

**Can you kill it?**

- **Direct answer: NO**
- Signals only delivered when process returns to user space
- Process is stuck in kernel space

**Solutions:**

```bash
# Fix the underlying I/O issue
sudo umount -f /nfs/mount   # Force unmount NFS
sudo umount -l /nfs/mount   # Lazy unmount

# For NFS, use soft mounts in future
mount -t nfs -o soft,timeo=10,intr server:/export /mnt

# Last resort: Reboot (but fix root cause!)
```

---

#### Signals

| Signal  | Number | Default Action | Can Catch? | Use Case                       |
| ------- | ------ | -------------- | ---------- | ------------------------------ |
| SIGTERM | 15     | Terminate      | ✅ Yes     | Graceful shutdown (default)    |
| SIGKILL | 9      | Terminate      | ❌ No      | Force kill                     |
| SIGSTOP | 19     | Stop           | ❌ No      | Pause process                  |
| SIGCONT | 18     | Continue       | ✅ Yes     | Resume stopped process         |
| SIGTSTP | 20     | Stop           | ✅ Yes     | Ctrl+Z (can catch for cleanup) |
| SIGINT  | 2      | Terminate      | ✅ Yes     | Ctrl+C (interrupt)             |
| SIGHUP  | 1      | Terminate      | ✅ Yes     | Hangup / reload config         |

**Important Corrections:**

- **SIGINT does NOT pause** - it requests termination
- **SIGSTOP/SIGTSTP pause** processes
- **SIGTERM is graceful**, SIGKILL is forceful

**Commands:**

```bash
kill -l                    # List all signals
kill PID                   # SIGTERM (default, graceful)
kill -TERM PID            # Same as above
kill -KILL PID            # Force kill (cannot be caught)
kill -9 PID               # Same as SIGKILL
kill -STOP PID            # Pause
kill -CONT PID            # Resume
kill -HUP PID             # Reload config (many daemons support this)

# Process groups (negative PID)
kill -TERM -1234          # Kill entire process group 1234
```

---

#### Process Group Signals

**Difference between `kill PID` and `kill -PID`:**

```bash
# kill PID: Send signal to single process
kill -TERM 1234

# kill -PID: Send signal to entire process group
kill -TERM -1234

# This kills parent AND all children with same PGID
```

**When to use process groups:**

- Killing parent with all children (shell scripts with subprocesses)
- Stopping entire job (terminal job control)
- Related to terminal sessions and `nohup`

**Terminal sessions:**

```bash
# When you logout, terminal sends SIGHUP to session leader
# Session leader forwards to process group
# All processes in group receive SIGHUP

# nohup protects from this:
nohup ./long_running_script.sh &
# Process ignores SIGHUP, survives logout

# Alternative: disown
./script.sh &
disown

# Or use screen/tmux
screen -S mysession
# Detach with Ctrl+A, D
# Reattach with: screen -r mysession
```

---

#### Investigating High CPU Usage

**Systematic Approach:**

```bash
# Step 1: Identify the process
top -o %CPU
htop
ps aux --sort=-%cpu | head -n 10

# Note: %CPU can exceed 100% on multi-core
# 400% = using 4 cores fully

# Step 2: Get process details
cat /proc/PID/status
cat /proc/PID/cmdline | tr '\0' ' '
ps -p PID -o args

# Step 3: Understand what it's doing
strace -c -p PID  # System call summary
# High CPU with few syscalls = pure computation/infinite loop

# Step 4: Stack trace
pstack PID           # C/C++
jstack PID           # Java
py-spy dump -p PID   # Python

# Step 5: Check threads
ps -T -p PID
top -H -p PID
cat /proc/PID/task/*/status | grep -E "^(Name|State):"

# Step 6: Profile the application
perf top -p PID
perf record -p PID -g
perf report

# Step 7: Check system resources
vmstat 1
# us = user CPU, sy = system CPU
# id = idle, wa = I/O wait

mpstat -P ALL 1      # Per-core CPU
pidstat -w -p PID 1  # Context switches
iotop -p PID         # I/O activity
```

**Common Causes:**

- Infinite loop or logic bug (high CPU, minimal syscalls)
- Inefficient algorithm (O(n²) processing large dataset)
- Busy-wait polling loop
- Lock contention in multi-threaded app

---

### Disk Space Issues

#### Disk Full but `df -h` Shows Space

**Common Scenarios:**

**1. Deleted Files Still Open (Most Common)**

```bash
# Problem: File deleted but process still has it open
# df counts it, du doesn't see it

# Find them:
lsof +L1
lsof | grep deleted

# Example output:
# java  1234  app  10w  REG  253,1  5.0G  0  /var/log/app.log (deleted)

# Solutions:
# Option 1: Restart the process
systemctl restart application

# Option 2: Truncate the file descriptor
truncate -s 0 /proc/PID/fd/FD_NUMBER

# Option 3: Send SIGHUP (many daemons reopen logs)
kill -HUP PID
```

**2. Inode Exhaustion**

```bash
# Check inode usage
df -i

# Output showing problem:
# Filesystem      Inodes   IUsed   IFree IUse% Mounted on
# /dev/sda1      3276800 3276800       0  100% /

# Find directories with most files:
for dir in /*; do
  echo -n "$dir: ";
  find "$dir" -type f | wc -l;
done | sort -t: -k2 -n

# Or:
find / -xdev -printf '%h\n' | sort | uniq -c | sort -k 1 -n
```

**3. Reserved Blocks**

```bash
# Ext filesystems reserve ~5% for root
tune2fs -l /dev/sda1 | grep -i "reserved block"

# Non-root users see "disk full"
# Root sees available space
```

**4. Hidden Mount Points**

```bash
# Data written before mount is hidden but uses space
echo "hidden" > /mnt/test.txt
mount /dev/sdb1 /mnt    # Hides test.txt

# To find:
umount /mnt
ls /mnt                 # Now you see hidden files
```

**Debugging Script:**

```bash
#!/bin/bash
# Quick disk space diagnostic

echo "=== Disk Space ==="
df -h /

echo -e "\n=== Inode Usage ==="
df -i /

echo -e "\n=== Deleted but Open Files ==="
lsof +L1 2>/dev/null | grep -v "0t0" | grep "REG"

echo -e "\n=== Largest Directories ==="
du -sh /* 2>/dev/null | sort -h | tail -10

echo -e "\n=== Compare df vs du ==="
echo "df reports: $(df -h / | tail -1 | awk '{print $3}') used"
echo "du reports: $(du -sh / 2>/dev/null | awk '{print $1}') used"
```

---

## Networking Fundamentals

### TCP Three-Way Handshake

**The Process:**

```
Client                          Server
  |                               |
  |  [1] SYN (seq=X)             |
  |----------------------------->|  State: LISTEN → SYN-RECEIVED
  |  State: SYN-SENT             |
  |                              |
  |  [2] SYN-ACK (seq=Y, ack=X+1)|
  |<-----------------------------|
  |                              |
  |  [3] ACK (seq=X+1, ack=Y+1)  |
  |----------------------------->|
  |                              |
  (State: ESTABLISHED)      (State: ESTABLISHED)
```

**What Each Step Does:**

1. **Client sends SYN:**

   - Includes initial sequence number (ISN) = X
   - Client enters SYN-SENT state
   - "I want to connect, here's my starting sequence"

2. **Server sends SYN-ACK:**

   - Server's sequence number = Y
   - Acknowledgment = X+1
   - Server enters SYN-RECEIVED state
   - "Got your SYN, here's my starting sequence"

3. **Client sends ACK:**
   - Sequence = X+1, Acknowledgment = Y+1
   - Both enter ESTABLISHED state
   - "Got your SYN-ACK, let's send data"

---

#### If SYN-ACK is Lost

**What Happens:**

```
Client                          Server
  |  [1] SYN (seq=X)             |
  |----------------------------->|
  |  [2] SYN-ACK ✗✗✗ LOST ✗✗✗    |
  |                              |
  | ... timeout (1-3 seconds)... | (Server waiting in SYN-RECEIVED)
  |                              |
  |  [1] SYN RETRANSMIT          |
  |----------------------------->|
  |  [2] SYN-ACK                 |
  |<-----------------------------|
  |  [3] ACK                     |
  |----------------------------->|
```

**Key Points:**

- Client's retransmission timer expires
- Client retransmits **the same SYN** (same seq=X)
- Exponential backoff: 1s, 2s, 4s, 8s, 16s, 32s...
- After 5-6 attempts (configurable), gives up
- Server also has timer for missing ACK
- No data flows until handshake completes

**Tuning:**

```bash
# Number of SYN retries before giving up
cat /proc/sys/net/ipv4/tcp_syn_retries  # Default: 6
```

---

### TCP vs UDP

#### TCP Retransmission

**How it Works:**

- TCP is reliable and connection-oriented
- Every segment has a sequence number
- Receiver sends ACKs for received segments
- Sender maintains retransmission timer for unacked segments

**When Retransmission Happens:**

1. **Timeout:** ACK not received before timer expires
2. **Duplicate ACKs:** Receiver sends duplicate ACKs when gap detected
   - 3 duplicate ACKs trigger **fast retransmit**

**What Gets Retransmitted:**

- Exact same segment with same sequence number
- Receiver uses sequence numbers to:
  - Reassemble data in order
  - Detect and discard duplicates

**Performance Impact:**

- Reduces throughput
- TCP adjusts congestion window
- RTT (Round Trip Time) affects retransmission delay

**Causes:**

- Network congestion
- Packet corruption
- Packet loss
- Out-of-order delivery

---

#### UDP Packet Loss

**How it Works:**

- UDP is unreliable and connectionless
- No acknowledgments, no sequence numbers
- No retransmission at transport layer
- Sender has no idea packet was lost

**When Packets Are Lost:**

- Sender doesn't know
- Receiver simply never gets data
- No automatic recovery

**Application Responsibility:**

- If reliability needed, app must implement it
- Examples:
  - **QUIC** (HTTP/3): Reliability over UDP
  - **RTP** (VoIP): Uses sequence numbers but may not retransmit
  - **DNS**: Application-level retry after timeout

**Why Use UDP Despite Loss:**

1. Lower latency (no connection setup, no ACK waiting)
2. Less overhead (8-byte header vs 20+ for TCP)
3. Better for real-time (old data is useless in live video)
4. Multicast/broadcast capable

**Use Cases:**

- DNS queries (retry is fine)
- VoIP/video streaming (some loss acceptable)
- Gaming (speed > perfect accuracy)
- DHCP (simple request/response)

---

### TCP Connection States

| State        | Description                                         |
| ------------ | --------------------------------------------------- |
| LISTEN       | Server waiting for connections                      |
| SYN-SENT     | Client sent SYN, waiting for SYN-ACK                |
| SYN-RECEIVED | Server sent SYN-ACK, waiting for ACK                |
| ESTABLISHED  | Connection active, data flowing                     |
| FIN-WAIT-1   | Sent FIN, waiting for ACK                           |
| FIN-WAIT-2   | Received ACK of FIN, waiting for peer's FIN         |
| TIME-WAIT    | Connection closed, waiting for stray packets (2MSL) |
| CLOSE-WAIT   | Peer closed connection, local not yet closed        |
| LAST-ACK     | Sent FIN after CLOSE-WAIT, waiting for ACK          |
| CLOSING      | Both sides closing simultaneously                   |

**Important States:**

- **CLOSE-WAIT**: Application not closing sockets properly
- **TIME-WAIT**: Normal after connection close (prevents packet confusion)

---

### netstat vs ss

| Feature     | netstat                     | ss                            |
| ----------- | --------------------------- | ----------------------------- |
| Speed       | Slow (reads /proc)          | Fast (netlink sockets)        |
| Data Source | /proc/net/\* files          | Direct kernel query           |
| Performance | Heavy with many connections | Minimal overhead              |
| Status      | Legacy/deprecated           | Actively maintained           |
| Filtering   | Limited (need grep)         | Built-in powerful filters     |
| TCP Details | Basic                       | Detailed (RTT, cwnd, retrans) |

**Basic Usage:**

```bash
# List all TCP/UDP listening ports
netstat -tuln
ss -tuln

# With process information (requires root)
sudo netstat -tulnp
sudo ss -tulnp

# All connections (listening + established)
netstat -tuan
ss -tuan
```

**ss Advanced Features:**

```bash
# Extended TCP information
ss -ti

# Output includes:
# - rto: Retransmission timeout
# - rtt: Round-trip time
# - cwnd: Congestion window
# - bytes_retrans: Retransmitted bytes
# - send rate, delivery rate

# Powerful filtering
ss -tn dst :443                    # Connections to port 443
ss -tn src 192.168.1.100          # From specific IP
ss -tn state established          # Only established
ss -tn state time-wait            # Only TIME-WAIT
ss -tn '( dport = :80 or dport = :443 )' state established

# Summary statistics
ss -s

# Memory usage per socket
ss -tm

# Timers
ss -to
```

**Performance Comparison:**

```bash
# On system with 50,000 connections:
time netstat -tan > /dev/null
# real    0m2.341s

time ss -tan > /dev/null
# real    0m0.123s
```

**When to Use:**

- **netstat:** Legacy systems, quick routing table checks
- **ss:** Modern systems (faster, more powerful), production troubleshooting, detailed TCP metrics

---

## Troubleshooting Scenarios

### Connection Reset by Peer

**What RST Means:**

- TCP RST packet abruptly closes connection
- No graceful FIN handshake
- Connection immediately destroyed

**Common Causes:**

1. Application explicitly closed connection
2. Firewall actively rejecting
3. Connection tracking table full
4. Kernel limits (file descriptors, socket buffers)
5. TCP backlog full

---

#### Diagnostic Flow

**Step 1: Capture the RST**

```bash
# Capture RST packets
sudo tcpdump -i any -nn 'tcp[tcpflags] & tcp-rst != 0' -w rst.pcap

# Specific port
sudo tcpdump -i any -nn port 8080 and 'tcp[tcpflags] & tcp-rst != 0'

# Analyze
tcpdump -r rst.pcap -vvv -nn

# Look for:
# - Which side sent RST (client or server)
# - When in connection lifecycle
# - Sequence number issues
```

**Step 2: Check Application**

```bash
# Application running?
systemctl status myapp
ps aux | grep myapp

# Application logs
journalctl -u myapp -f --since "5 minutes ago"

# Look for:
# - "Connection closed", "Timeout"
# - "Too many connections"
# - Exception traces

# Metrics (if available)
curl http://localhost:8080/metrics
```

**Step 3: Check Connection Tracking**

```bash
# Current count vs maximum
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Usage percentage
echo "scale=2; $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max) * 100" | bc

# If close to 100%, this is likely the problem!

# Statistics
cat /proc/net/stat/nf_conntrack
# Look for 'drop' or 'early_drop' increasing

# View table
conntrack -L | head -20

# Count by state
conntrack -L | awk '{print $4}' | sort | uniq -c | sort -rn

# Temporary fix:
sudo sysctl -w net.netfilter.nf_conntrack_max=262144

# Permanent in /etc/sysctl.conf:
net.netfilter.nf_conntrack_max = 262144
```

**Step 4: Check Firewall**

```bash
# Check for REJECT/DROP rules
sudo iptables -L -n -v | grep -E "REJECT|DROP"

# Specific chains
sudo iptables -L INPUT -n -v
sudo iptables -L FORWARD -n -v

# Count dropped packets
sudo iptables -L -n -v -Z  # Zero counters
# Wait, then check again
sudo iptables -L -n -v

# Rate limiting
sudo iptables -L -n -v | grep limit
```

**Step 5: Check with ss**

```bash
# Retransmissions
ss -ti | grep bytes_retrans | grep -v "bytes_retrans:0"

# Connection states
ss -tan state established | wc -l
ss -tan state time-wait | wc -l
ss -tan state close-wait | wc -l   # Many = app not closing sockets

# Summary statistics
ss -s
# Look for:
# - TCPBacklogDrop
# - ListenDrops
# - TCPTimeouts
```

**Step 6: Check Kernel Limits**

```bash
# File descriptors
ulimit -n
cat /proc/sys/fs/file-nr

# Socket buffers
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem

# TCP backlog
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.core.somaxconn

# Statistics
netstat -s | grep -i drop
netstat -s | grep -i listen
```

---

#### Distinguishing Between Causes

**Application Issue:**

- RST in middle of established connection
- Application logs show errors
- Happens under load or after timeout
- Many CLOSE-WAIT states

**Firewall Issue:**

- RST immediately after SYN or early
- iptables counters show REJECT/DROP increasing
- Consistent from certain IPs
- RST from server IP (not app process)

**Connection Tracking Full:**

- nf_conntrack_count ≈ nf_conntrack_max
- Intermittent during high traffic
- dmesg shows "table full" messages
- Sudden RST on established connections

**Kernel Limits:**

- netstat -s shows backlog/listen drops
- File descriptor limits reached
- Happens under high connection rate

---

### Quick Diagnostic Script

```bash
#!/bin/bash
echo "=== Connection Tracking ==="
echo "Count: $(cat /proc/sys/net/netfilter/nf_conntrack_count)"
echo "Max: $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
echo "Usage: $(echo "scale=2; $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max) * 100" | bc)%"

echo -e "\n=== TCP Socket States ==="
ss -tan | awk '{print $1}' | sort | uniq -c

echo -e "\n=== Netstat Drops ==="
netstat -s | grep -i "drop\|overflow"

echo -e "\n=== Recent Kernel Messages ==="
dmesg | tail -20

echo -e "\n=== Firewall Drops ==="
iptables -L -n -v | grep -E "DROP|REJECT" | head -10
```

---

## Essential Commands Reference

### Process Investigation

```bash
ps aux --sort=-%cpu              # CPU hogs
ps aux --sort=-%mem              # Memory hogs
ps -eLf                          # All threads
ps -o ppid= -p PID               # Get parent PID
pgrep -a process_name            # Find by name
pidof process_name               # Get PID
/proc/PID/status                 # Detailed info
/proc/PID/cmdline                # Command line
/proc/PID/fd/                    # Open file descriptors
lsof -p PID                      # Open files
strace -c -p PID                 # System call summary
pstack PID                       # Stack trace
```

### Signals

```bash
kill -l                          # List all signals
kill -TERM PID                   # Graceful (15)
kill -KILL PID                   # Force (9)
kill -STOP PID                   # Pause (19)
kill -CONT PID                   # Resume (18)
kill -HUP PID                    # Reload config (1)
killall process_name             # Kill by name
pkill -f "pattern"               # Kill by pattern
kill -TERM -PGID                 # Kill process group
```

### File System

```bash
df -h                            # Disk space
df -i                            # Inode usage
du -sh /*                        # Directory sizes
lsof +L1                         # Deleted but open
lsof | grep deleted              # Alternative
stat filename                    # File metadata
ls -i                            # Show inode numbers
find / -inum INODE               # Find by inode
tune2fs -l /dev/sda1             # Filesystem info
```

### Network - Basic

```bash
ss -tulnp                        # All listening
ss -tanp                         # All connections
ss -s                            # Summary stats
ss -ti                           # TCP details
ss 'sport = :80'                 # Filter by port
ss state established             # By state
ping -c 4 host                   # Connectivity
traceroute host                  # Path
mtr host                         # Combined ping/trace
```

### Network - Advanced

```bash
ip addr show                     # Interfaces
ip -s link                       # Interface stats
ip route show                    # Routing table
netstat -s                       # Protocol stats
tcpdump -i any host X            # Packet capture
tcpdump -nn port 443             # Specific port
lsof -i :PORT                    # What's using port
nc -zv host port                 # Test connection
dig domain                       # DNS lookup
conntrack -L                     # Connection tracking table
```

### Performance Monitoring

```bash
top / htop                       # Real-time overview
vmstat 1                         # Virtual memory
iostat -x 1                      # Disk I/O
mpstat -P ALL 1                  # Per-CPU stats
sar -u 1 10                      # Historical CPU
iotop                            # I/O by process
iftop                            # Network by connection
dstat                            # Combined view
```

### System Information

```bash
uname -a                         # Kernel version
cat /etc/os-release              # OS version
lscpu                            # CPU info
free -h                          # Memory
uptime                           # Load average
dmesg | tail                     # Kernel messages
journalctl -xe                   # System logs
systemctl status service         # Service status
```

---

## Common Interview Traps

### ❌ Avoid These Mistakes

**Trap 1: "I'd restart the server"**

- ✅ Better: Show systematic troubleshooting first

**Trap 2: Wrong signals**

- ❌ "SIGINT pauses processes"
- ✅ SIGSTOP/SIGTSTP pause; SIGINT terminates

**Trap 3: Vague answers**

- ❌ "I'd check the logs"
- ✅ "I'd check application logs with `journalctl -u service -f` and kernel messages with `dmesg`"

**Trap 4: Forgetting permissions**

- Many commands need sudo: ss -p, lsof, strace, tcpdump
- Mention this in answers

**Trap 5: Tool confusion**

- ❌ "netstat is better than ss"
- ✅ ss is faster and more powerful

---

## Communication Tips

### Structure Your Answers

1. **Acknowledge:** "That's a good question about..."
2. **State approach:** "I'd begin by checking..."
3. **Show systematic thinking:** "First... then... finally..."
4. **Mention trade-offs:** "This approach is fast but less detailed"
5. **Admit uncertainty:** "I'm not certain, but I believe..." is better than guessing

### Think Out Loud

- "I'm considering whether this is a network or application issue..."
- "Let me work through the TCP state machine..."
- "That's interesting, I haven't seen this exact scenario, but I'd approach it by..."

### Ask Clarifying Questions

- "Is this a production system where I need to avoid service disruption?"
- "Do I have root access?"
- "Are there any monitoring tools already in place?"
- "What's the typical load on this system?"

---

## Key Concepts Summary

### Inodes

- Store metadata (permissions, timestamps, size, pointers to data)
- Do NOT store filename or file content
- Filename → Directory Entry → Inode → Data Blocks
- Running out of inodes = can't create files despite free space
- Check with: `df -i`
- View with: `ls -i`

### File Deletion

- `rm` calls `unlink()` system call
- Removes directory entry, decrements inode link count
- Data deleted when link count = 0 AND no processes have file open
- Data not immediately overwritten (file recovery possible)

### Hard Links vs Symlinks

- **Hard:** Same inode, survives target deletion, same filesystem only
- **Symlink:** Own inode with path string, breaks if target deleted, cross-filesystem OK

### Process States

- **R:** Running/runnable
- **S:** Sleeping (interruptible)
- **D:** Sleeping (uninterruptible) - CANNOT BE KILLED
- **Z:** Zombie (already dead, waiting for parent to reap)
- **T:** Stopped (SIGSTOP/SIGTSTP)

### Signals

- **SIGTERM (15):** Graceful shutdown (catchable, default)
- **SIGKILL (9):** Force kill (uncatchable)
- **SIGSTOP (19):** Pause (uncatchable)
- **SIGCONT (18):** Resume
- **SIGHUP (1):** Hangup/reload config
- **SIGINT (2):** Interrupt (Ctrl+C)
- **SIGTSTP (20):** Stop (Ctrl+Z, catchable)

### TCP States

- **LISTEN:** Waiting for connections
- **SYN-SENT/SYN-RECEIVED:** Handshake in progress
- **ESTABLISHED:** Active connection
- **TIME-WAIT:** Connection closed, waiting 2MSL for stray packets
- **CLOSE-WAIT:** Remote closed, local hasn't closed yet (app issue if many)

### TCP vs UDP

- **TCP:** Reliable, connection-oriented, retransmits lost packets, higher overhead
- **UDP:** Unreliable, connectionless, no retransmission, lower latency, less overhead
- **TCP use:** HTTP, SSH, file transfer (need reliability)
- **UDP use:** DNS, VoIP, streaming, gaming (need speed)

### netstat vs ss

- **ss:** Faster (netlink), more powerful filtering, detailed TCP info, modern
- **netstat:** Slower (/proc), basic info, legacy, deprecated

### Disk Space Issues

1. **Deleted files open:** `lsof +L1`
2. **Inode exhaustion:** `df -i`
3. **Reserved blocks:** `tune2fs -l`
4. **Hidden mounts:** `umount` and check

### Connection Reset

- **Application:** RST mid-connection, logs show errors, many CLOSE-WAIT
- **Firewall:** RST after SYN, iptables counters increasing
- **Connection tracking:** nf_conntrack_count near max, dmesg warnings
- **Kernel limits:** netstat -s shows drops, file descriptor exhaustion

---

## Practice Scenarios

### Scenario 1: Mysterious Disk Usage

```
Problem: Server reports 95% disk usage, but du -sh / shows only 60% used

Your approach:
1. Check for deleted files still open: lsof +L1
2. Check inode usage: df -i
3. Compare df vs du output
4. Check for hidden mounts
5. Look for large logs being rotated but held open

Expected answer: Identify root cause and explain recovery steps
```

### Scenario 2: Unkillable Process

```
Problem: Process consuming 100% CPU, won't respond to kill -9

Your approach:
1. Check process state: ps aux | grep PID
2. If state is D: investigate I/O (iostat, lsof -p PID, /proc/PID/stack)
3. If state is not D: verify you're killing correct PID, check if zombie
4. Explain why D state can't be killed
5. Propose solutions based on root cause

Expected answer: Systematic diagnosis, understand kernel vs userspace
```

### Scenario 3: Intermittent Connection Failures

```
Problem: Web app occasionally returns "Connection reset by peer"

Your approach:
1. Capture RST packets: tcpdump
2. Check connection tracking: nf_conntrack_count vs max
3. Check application logs
4. Use ss to check TCP health: bytes_retrans, socket states
5. Check firewall rules: iptables -L -n -v
6. Check kernel limits: netstat -s, ulimit

Expected answer: Distinguish between app, firewall, and kernel causes
```

### Scenario 4: Slow File Transfer

```
Problem: scp transfer between servers is very slow despite high bandwidth

Your approach:
1. Check network basics: ping, traceroute
2. Check interface stats: ip -s link, ethtool
3. Check TCP performance: ss -ti for RTT, cwnd, retransmissions
4. Test throughput: iperf3
5. Check for packet loss: netstat -s | grep retrans
6. Consider TCP tuning, MTU issues, SSH cipher overhead

Expected answer: Layer-by-layer network troubleshooting
```

---

## MongoDB-Specific Context

### What MongoDB Values

Based on their engineering principles:

**1. Resilience**

- Think about failure modes
- How would you monitor this?
- What happens when it breaks?
- "Design for failure" mentality

**2. Operational Excellence**

- Automation over manual intervention
- Observability and debugging
- Learning from incidents

**3. Systematic Thinking**

- Structured troubleshooting approach
- Root cause analysis
- Reproducible investigations

### During Interview

**Show operational thinking:**

- "I'd want to set up monitoring for this metric..."
- "To prevent this in future, I'd implement..."
- "This would let me debug issues without impacting users..."

**Consider distributed systems:**

- "In a distributed environment, I'd also check..."
- "Network partitions could cause..."
- "Eventual consistency means..."

**Focus on reliability:**

- "The safest approach would be..."
- "To avoid downtime, I'd first..."
- "This has lower risk because..."

---

## Day-Before Checklist

### Mental Preparation

**Remember:**

- Partial answers with reasoning > wrong confident answers
- It's OK to say "I'm not sure, but here's how I'd investigate..."
- Think out loud - interviewers want to see your process
- Ask clarifying questions - shows thoroughness
- Communication matters as much as technical knowledge

**Get Good Sleep:**

- This interview rewards clear thinking over memorization
- Being well-rested helps with systematic reasoning
- Don't cram the night before

---

## Quick Reference - Most Important Commands

### The "Top 20" You Must Know

```bash
# Process management
ps aux --sort=-%cpu
top / htop
kill -TERM PID
kill -STOP PID
/proc/PID/status

# File system
df -h / df -i
du -sh /*
lsof +L1
ls -i
stat filename

# Networking
ss -tulnp
ss -ti
ping / traceroute
tcpdump -i any port 80
ip addr show

# Troubleshooting
dmesg | tail
journalctl -xe
strace -c -p PID
iostat -x 1
vmstat 1
```

### Quick Answers to Common Questions

**"Why can't I create files despite free space?"**
→ Check `df -i` for inode exhaustion

**"Why can't I kill this process?"**
→ Check state with `ps aux | grep PID`. If D state, it's in uninterruptible I/O

**"What's using all my disk space?"**
→ Check `lsof +L1` for deleted but open files

**"Connections keep resetting"**
→ Check connection tracking: `cat /proc/sys/net/netfilter/nf_conntrack_count`

**"How do I see what a process is doing?"**
→ `strace -c -p PID` for syscalls, `lsof -p PID` for files, `/proc/PID/status` for state

**"Process won't respond to SIGTERM"**
→ Escalate to `kill -KILL PID`, unless D state (then fix I/O issue)

**"Network is slow"**
→ Check with `ss -ti` for retransmissions and RTT, `iostat` for I/O wait

**"Too many connections in TIME-WAIT"**
→ Normal for busy servers (2MSL timeout), tune `tcp_tw_reuse` if needed

---

## Final Tips

### What Interviewers Look For

**1. Structured Thinking**

- Not jumping to conclusions
- Systematic elimination of causes
- Layer-by-layer approach

**2. Practical Knowledge**

- Real commands, not just theory
- Understanding when to use which tool
- Edge cases and failure modes

**3. Communication**

- Explaining thought process
- Acknowledging uncertainty
- Asking good questions

**4. Operational Mindset**

- Impact on production
- Monitoring and prevention
- Automation opportunities

### Red Flags to Avoid

- ❌ Guessing without admitting uncertainty
- ❌ Memorized answers without understanding
- ❌ Ignoring edge cases
- ❌ Forgetting to mention need for sudo/root
- ❌ Vague answers ("check the logs" without specifics)
- ❌ Immediate restart without investigation

### Green Flags to Show

- ✅ Structured troubleshooting approach
- ✅ Multiple solution approaches
- ✅ Awareness of trade-offs
- ✅ Thinking about prevention
- ✅ Clear communication
- ✅ Honest about gaps in knowledge

---

## Good Luck!

Remember: This interview is about **how you think**, not just what you know.

- Stay calm and systematic
- Think out loud
- Ask clarifying questions
- Acknowledge when you're uncertain
- Show your reasoning process

You've got this! 🚀
