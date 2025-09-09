# Linux SRE Troubleshooting Study Sheet

## 1. Linux Boot Process - From Power On to Login Prompt

**Boot Sequence:**
1. **BIOS/UEFI** - Hardware initialization, POST (Power-On Self-Test)
2. **Bootloader (GRUB)** - Loads kernel from disk
3. **Kernel Initialization** - Memory management, device drivers
4. **Init Process (PID 1)** - systemd/SysV init starts
5. **System Services** - Network, logging, etc.
6. **Display Manager** - Login prompt appears

**Key Files:**
- `/boot/grub/grub.cfg` - Bootloader configuration
- `/etc/systemd/system/` - Service definitions
- `/var/log/boot.log` - Boot messages

---

## 2. What Happens When You Type `ls`

**Process Flow:**
1. **Shell parsing** - `getline()` reads input, `strtok()` tokenizes
2. **Alias check** - Shell checks if `ls` is an alias/builtin
3. **PATH lookup** - Shell searches PATH directories for `ls` binary
4. **Process creation** - `fork()` creates child process (returns 0 to child, PID to parent)
5. **Program execution** - `execve()` gives child new address space
6. **Directory reading** - `ls` reads filesystem inodes
7. **Process termination** - `_exit(0)` called, kernel frees resources

**Debug command:** `strace ls` to see system calls

---

## 3. Linux Inodes

**Definition:** Data structure storing file/directory metadata

**Inode contains:**
- File permissions and ownership
- File size and timestamps
- Pointers to data blocks
- Link count

**Commands:**
- `ls -i` - Show inode numbers
- `stat filename` - Display inode information
- `df -i` - Show inode usage

---

## 4. Crash vs Panic

| Crash | Panic |
|-------|-------|
| Hardware/OS initiated | Application initiated |
| Memory access violations | Calls `abort()` function |
| SIGSEGV, SIGBUS, SIGILL | Controlled shutdown |

**Common crash signals:**
- **SIGSEGV** - Segmentation fault
- **SIGBUS** - Bus error  
- **SIGILL** - Illegal instruction

**Tools:** `gdb`, signal handlers, core dumps

---

## 5. /proc Filesystem

**Virtual filesystem** containing runtime system information

**Key directories/files:**
- `/proc/[PID]/` - Process-specific information
- `/proc/self/` - Current process
- `/proc/[PID]/maps` - Memory mappings
- `/proc/[PID]/cmdline` - Command line arguments
- `/proc/[PID]/environ` - Environment variables
- `/proc/[PID]/fd/` - File descriptors
- `/proc/locks` - File locks
- `/proc/sys/fs/file-nr` - Open file statistics
- `/proc/sys/vm/` - Virtual memory tuning

---

## 6. Filesystem Full but df Shows Space

**Troubleshooting steps:**
1. **Check inodes:** `df -i` (look for 0 IFree)
2. **Check deleted files in use:** `lsof | grep deleted`
3. **Restart processes** holding deleted files
4. **Check for large files:** `du -sh /*`

**Common causes:**
- Inode exhaustion
- Deleted files still open by processes
- Reserved blocks for root

---

## 7. Linux Performance Tools

**Essential monitoring tools:**
1. **uptime** - Load averages
2. **dmesg | tail** - Kernel messages
3. **vmstat 1** - Virtual memory statistics
4. **mpstat -P ALL 1** - CPU usage per core
5. **pidstat 1** - Process statistics
6. **iostat -xz 1** - I/O statistics
7. **free -m** - Memory usage
8. **sar -n DEV 1** - Network device statistics
9. **sar -n TCP,ETCP 1** - TCP statistics
10. **top/htop** - Process monitoring

---

## 8. Linux Filesystem Types

**Common filesystems:**
- **EXT4** - Default Linux filesystem
- **XFS** - High-performance journaling
- **BTRFS** - Copy-on-write with snapshots
- **ZFS** - Advanced features, checksumming
- **NTFS** - Windows compatibility

**Commands:**
- `mount` - Show mounted filesystems
- `lsblk` - List block devices
- `fsck` - Filesystem check

---

## 9. Kernel Space vs User Space

**User Space:**
- Applications run here
- Limited access to hardware
- Uses system calls to access kernel

**Kernel Space:**
- Operating system kernel
- Direct hardware access
- Memory protection enforced

**Communication:** System calls bridge user/kernel space
**Libraries:** libc provides system call wrappers

---

## 10. High I/O Troubleshooting

**Investigation steps:**
1. **Identify source:** `iotop`, `iostat -x 1`
2. **Check disk utilization:** `iostat -x 1` (look for %util)
3. **Find processes:** `iotop`, `pidstat -d 1`
4. **Check disk health:** `smartctl -a /dev/sda`
5. **Analyze I/O patterns:** `blktrace`

**Key metrics:**
- **IOPS** - I/O operations per second
- **Throughput** - MB/s
- **Latency** - Response time
- **Queue depth** - Pending I/O requests

---

## 11. Processes vs Threads

| Processes | Threads |
|-----------|---------|
| Independent memory space | Shared memory space |
| Higher creation overhead | Lower creation overhead |
| Inter-process communication needed | Direct memory sharing |
| Isolated execution | Can interfere with each other |

**Process Control Block (PCB)** contains:
- Process ID (PID)
- Memory pointers
- CPU registers
- I/O status

---

## 12. Kernel Memory Management

**Key concepts:**
- **Virtual Memory** - Abstraction layer over physical memory
- **Page Tables** - Virtual to physical address mapping
- **Memory Zones** - DMA, Normal, HighMem
- **Slab Allocator** - Kernel object caching
- **Page Cache** - File system caching

**Commands:**
- `/proc/meminfo` - Memory statistics
- `free -m` - Memory usage
- `/proc/slabinfo` - Slab cache info

---

## 13. Process States

**Task states:**
- **TASK_RUNNING (R)** - Executing or runnable
- **TASK_INTERRUPTIBLE (S)** - Sleeping, waiting for signal
- **TASK_UNINTERRUPTIBLE (D)** - Deep sleep, usually I/O wait
- **TASK_STOPPED (T)** - Stopped by signal
- **TASK_ZOMBIE (Z)** - Terminated, awaiting parent cleanup

**Commands:**
- `ps aux` - Show process states
- `top` - Real-time process monitoring

---

## 14. Linux Concurrency and Race Conditions

**Concurrency mechanisms:**
- **Mutexes** - Mutual exclusion locks
- **Semaphores** - Counting locks
- **Spinlocks** - Busy-wait locks
- **RCU** - Read-Copy-Update

**Race conditions** occur when multiple threads access shared data simultaneously

**Prevention:** Proper locking, atomic operations

---

## 15. Stack vs Heap Memory

| Stack | Heap |
|-------|------|
| Automatic allocation | Manual allocation |
| LIFO structure | Random access |
| Fast allocation/deallocation | Slower allocation |
| Limited size | Large size |
| Local variables | Dynamic allocation |

**Stack:** Function calls, local variables
**Heap:** Dynamic memory allocation (`malloc`, `new`)

---

## 16. Memory Leaks

**Types:**
1. **Classic leak** - Unreachable memory not freed
2. **Semantic leak** - Reachable but unnecessary memory

**Detection tools:**
- `valgrind` - Memory error detector
- `mtrace` - GNU malloc debugging
- `AddressSanitizer` - Google's memory error detector

**Prevention:** RAII, smart pointers, garbage collection

---

## 17. Linux Interrupt Handling

**Interrupt processing:**
1. **Hardware interrupt** - Device signals CPU
2. **Context save** - Current process state saved
3. **Interrupt handler** - Kernel executes ISR
4. **Context restore** - Return to interrupted process

**Types:**
- **Hardware interrupts** - External devices
- **Software interrupts** - System calls, exceptions

**Commands:**
- `/proc/interrupts` - Show interrupt statistics
- `cat /proc/stat` - System statistics

---

## 18. Load Average

**Definition:** Average number of processes that are either:
- Running on CPU
- Waiting for CPU
- Waiting for I/O to complete

**Time periods:** 1, 5, and 15 minutes

**Interpretation:**
- **< 1.0** - No wait time
- **= 1.0** - System fully utilized
- **> 1.0** - Processes waiting

**Commands:** `uptime`, `w`, `top`

---

## 19. What Happens When You curl a Website

**Network flow:**
1. **DNS lookup** - Resolve domain to IP
2. **TCP connection** - 3-way handshake
3. **TLS handshake** - SSL/TLS negotiation (HTTPS)
4. **HTTP request** - Send GET/POST request
5. **Server processing** - Web server handles request
6. **HTTP response** - Server sends response
7. **Connection close** - TCP teardown

**Tools for debugging:**
- `nslookup/dig` - DNS lookup
- `tcpdump` - Packet capture
- `netstat` - Network connections
- `curl -v` - Verbose output

---

## 20. Production Debugging

### Core Dumps Analysis

**What is a Core Dump:**
A core dump is a snapshot of a process's memory and CPU state when it crashes, used for post-mortem debugging.

**Enabling Core Dumps:**
```bash
# Check current limits
ulimit -c

# Enable unlimited core dumps
ulimit -c unlimited

# Set system-wide core dump pattern
echo '/var/core/core.%e.%p.%t' > /proc/sys/kernel/core_pattern

# Enable core dumps for systemd services
echo 'DefaultLimitCORE=infinity' >> /etc/systemd/system.conf
```

**Core Dump Analysis with GDB:**
```bash
# Generate core dump from running process
gcore <PID>

# Analyze core dump
gdb /path/to/binary /path/to/core.file

# Key GDB commands for debugging:
(gdb) bt              # Backtrace - show call stack
(gdb) bt full         # Full backtrace with local variables
(gdb) info registers  # Show CPU register values
(gdb) info threads    # Show all threads
(gdb) thread 2        # Switch to thread 2
(gdb) print variable  # Print variable value
(gdb) x/10i $pc      # Examine 10 instructions at program counter
```

**Core Dump Analysis without GDB:**
```bash
# Use file command to verify core dump
file core.1234

# Extract stack trace with addr2line
addr2line -e /path/to/binary -f -C < addresses.txt

# Use objdump for disassembly
objdump -d /path/to/binary | grep -A10 -B10 <address>
```

### Memory Leak Detection

**Using Valgrind:**
```bash
# Detect memory leaks
valgrind --tool=memcheck --leak-check=full --show-leak-kinds=all ./program

# Detailed leak detection with call stack
valgrind --tool=memcheck --leak-check=full --track-origins=yes ./program

# Generate suppression file for known false positives
valgrind --gen-suppressions=all ./program 2>&1 | grep -A20 "{"
```

**Using AddressSanitizer (ASan):**
```bash
# Compile with AddressSanitizer
gcc -fsanitize=address -g -o program program.c

# Run with additional options
ASAN_OPTIONS=detect_leaks=1:abort_on_error=1 ./program
```

**Production Memory Analysis:**
```bash
# Monitor process memory usage over time
while true; do
    ps -o pid,vsz,rss,pmem,comm -p <PID>
    sleep 60
done

# Use pmap to analyze memory layout
pmap -x <PID>

# Check for memory leaks in running process
cat /proc/<PID>/smaps | grep -E "(Size|Rss|Pss)"

# Monitor memory allocation patterns
echo 1 > /proc/<PID>/clear_refs
# Wait some time
cat /proc/<PID>/smaps | awk '/^[0-9]/{print $1}' | while read addr; do
    grep -A15 "^$addr" /proc/<PID>/smaps | grep Referenced
done
```

**Heap Analysis Tools:**
```bash
# Use heaptrack for heap profiling
heaptrack ./program
heaptrack_gui heaptrack.program.<PID>.gz

# Use massif (Valgrind tool) for heap profiling
valgrind --tool=massif ./program
ms_print massif.out.<PID>

# Monitor malloc/free patterns with ltrace
ltrace -e malloc,free,realloc ./program
```

**Memory Debugging in Production:**
```bash
# Check for OOM kills
dmesg | grep -i "killed process"
journalctl -u <service> | grep -i "out of memory"

# Monitor memory pressure
cat /proc/pressure/memory

# Check memory cgroups limits
cat /sys/fs/cgroup/memory/<cgroup>/memory.usage_in_bytes
cat /sys/fs/cgroup/memory/<cgroup>/memory.limit_in_bytes

# Use jemalloc for better memory tracking (if available)
LD_PRELOAD=libjemalloc.so.2 MALLOC_CONF=prof:true ./program
```

**Quick Memory Leak Detection:**
```bash
# Simple memory growth detection
function check_memory_growth() {
    local pid=$1
    local baseline=$(ps -o rss= -p $pid)
    sleep 300  # Wait 5 minutes
    local current=$(ps -o rss= -p $pid)
    local growth=$((current - baseline))
    echo "Memory growth: ${growth}KB in 5 minutes"
    [ $growth -gt 10240 ] && echo "WARNING: Potential memory leak detected"
}
```

---

## Quick Reference Commands

**System Info:**
```bash
uname -a                 # System information
cat /etc/os-release     # OS version
lscpu                   # CPU information
lsmem                   # Memory information
```

**Process Management:**
```bash
ps aux                  # All processes
pgrep -f pattern        # Find processes by name
kill -9 PID            # Force kill process
nohup command &        # Run command in background
```

**File Operations:**
```bash
find / -name filename   # Find files
lsof filename          # Show processes using file
fuser -v filename      # Show processes using file
```

**Network:**
```bash
ss -tulpn              # Show listening ports
netstat -i             # Network interfaces
iftop                  # Network traffic monitor
```

**Disk/Filesystem:**
```bash
lsblk                  # List block devices
mount                  # Show mounted filesystems
du -sh /*              # Directory sizes
```