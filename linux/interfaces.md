# 🐧 Linux Operating System Interfaces

> **Based on** [xv6/x86 architecture](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)  
> **Course Reference** [UCI ICS 143A](https://www.ics.uci.edu/~aburtsev/143A/)

## 🔌 Interfaces

An operating system provides services to user programs through an interface that acts as a bridge between applications and system resources.

![Interface](./img/interface.png)

## ⚙️ Kernel

The **kernel** is the core interface between a computer's hardware and its processes. It uses the CPU's hardware protection mechanisms to ensure that each process executing in user space can access only its own memory. The kernel executes with the hardware privileges required to implement these protections; user programs execute without those privileges. When a user program invokes a system call, the hardware raises the privilege level and starts executing a pre-arranged function in the kernel.

### 🎯 The Kernel's Four Key Responsibilities

1. **🧠 Memory Management**  
   Keep track of how much memory is used to store what, and where

2. **⚡ Process Management**  
   Determine which processes can use the central processing unit (CPU), when, and for how long

3. **🔧 Device Drivers**  
   Act as mediator/interpreter between the hardware and processes

4. **🔒 System Calls and Security**  
   Receive requests for service from the processes

## 🐚 Shell

The **shell** is an ordinary program that reads commands from the user and executes them. The fact that the shell is a user program, not part of the kernel, illustrates the power of the system call interface.

## 🔄 Process

Each running program, called a **process**, has memory containing instructions, data, and a stack:
- **Instructions** implement the program's computation
- **Data** are the variables on which the computation acts  
- **Stack** organizes the program's procedure calls

A process alternates between executing in **user space** and **kernel space**.

## 🧵 Threads

A **thread** is the smallest unit of execution within a process. While a process is an independent program with its own memory space, threads are lightweight execution units that share the same memory space within a process.

### Key Characteristics

- **Shared Memory**: All threads within a process share the same address space, including code, data, and heap
- **Private Stack**: Each thread has its own stack and registers (including program counter)
- **Lightweight**: Thread creation and context switching is faster than process creation
- **Concurrency**: Multiple threads can execute simultaneously on multi-core systems

### Thread vs Process

| Aspect | 🔄 Process | 🧵 Thread |
|--------|-----------|----------|
| **Memory Space** | Independent | Shared within process |
| **Creation Cost** | High (fork overhead) | Low |
| **Context Switch** | Expensive | Cheaper |
| **Communication** | IPC (pipes, sockets) | Shared memory (direct) |
| **Isolation** | Strong (separate memory) | Weak (shared memory) |

### Threading Models

**1. User-Level Threads (Many-to-One)**
- Managed entirely in user space
- Fast creation and switching
- Cannot take advantage of multi-core CPUs
- Blocking syscall blocks entire process

**2. Kernel-Level Threads (One-to-One)**
- Each user thread maps to a kernel thread
- True parallelism on multi-core systems
- Higher overhead for creation/switching
- Linux uses this model (NPTL - Native POSIX Thread Library)

**3. Hybrid Model (Many-to-Many)**
- Multiplexes many user threads onto smaller number of kernel threads
- Combines benefits of both approaches

### Thread System Calls

| System Call | 🔍 Description |
|-------------|----------------|
| `clone()` | 🧬 Create a new thread (low-level, used by pthread) |
| `pthread_create()` | ✨ Create a new thread (POSIX API) |
| `pthread_join()` | 🤝 Wait for a thread to terminate |
| `pthread_exit()` | 🚪 Terminate the calling thread |
| `pthread_self()` | 🆔 Return the thread ID of calling thread |
| `pthread_detach()` | 🔓 Mark thread as detached (resources freed on exit) |
| `pthread_mutex_lock()` | 🔒 Acquire a mutex lock |
| `pthread_mutex_unlock()` | 🔓 Release a mutex lock |
| `pthread_cond_wait()` | ⏸️ Wait on a condition variable |
| `pthread_cond_signal()` | 📢 Signal a condition variable |

### Synchronization Primitives

Since threads share memory, synchronization is critical to prevent race conditions:

- **Mutexes** 🔒: Mutual exclusion locks to protect critical sections
- **Semaphores** 🚦: Counter-based synchronization mechanism
- **Condition Variables** 📢: Allow threads to wait for specific conditions
- **Read-Write Locks** 📖✍️: Allow multiple readers or single writer
- **Barriers** 🚧: Synchronization point for multiple threads

### Thread-Local Storage (TLS)

Each thread can have its own private data using thread-local storage, which appears global within the thread but is actually unique per thread.

## 📞 System Calls

When a process needs to invoke a kernel service, it invokes a procedure call in the operating system interface. The system call enters the kernel; the kernel performs the service and returns.

### 📋 System Call Reference

| System Call | 🔍 Description |
|-------------|----------------|
| `fork()` | 🍴 Create a process |
| `exit()` | ❌ Terminate the current process |
| `wait()` | ⏳ Wait for a child process to exit |
| `kill(pid)` | 💀 Terminate process pid |
| `getpid()` | 🆔 Return the current process's pid |
| `sleep(n)` | 💤 Sleep for n clock ticks |
| `exec(filename, *argv)` | 🚀 Load a file and execute it |
| `sbrk(n)` | 📈 Grow process's memory by n bytes |
| `open(filename, flags)` | 📂 Open a file; the flags indicate read/write |
| `read(fd, buf, n)` | 📖 Read n bytes from an open file into buf |
| `write(fd, buf, n)` | ✍️ Write n bytes to an open file |
| `close(fd)` | 🔒 Release open file fd |
| `dup(fd)` | 📋 Duplicate fd |
| `pipe(p)` | 🔗 Create a pipe and return fd's in p |
| `chdir(dirname)` | 📁 Change the current directory |
| `mkdir(dirname)` | 📁➕ Create a new directory |
| `mknod(name, major, minor)` | 🛠️ Create a device file |
| `fstat(fd)` | ℹ️ Return info about an open file |
| `link(f1, f2)` | 🔗 Create another name (f2) for the file f1 |
| `unlink(filename)` | 🗑️ Remove a file |

## 📁 I/O and File Descriptors

### 🔢 File Descriptor

A **file descriptor** is a small integer representing a kernel-managed object that a process may read from or write to. A process may obtain a file descriptor by:
- Opening a file, directory, or device
- Creating a pipe
- Duplicating an existing descriptor

The file descriptor interface abstracts away the differences between files, pipes, and devices, making them all look like streams of bytes.

#### Standard File Descriptors
1. **fd0** = `stdin` 📥 (Standard Input)
2. **fd1** = `stdout` 📤 (Standard Output)  
3. **fd2** = `stderr` 🚨 (Standard Error)

## 🔗 Pipes

A **pipe** is a small kernel buffer exposed to processes as a pair of file descriptors:
- One for **reading** 📖
- One for **writing** ✍️

Writing data to one end of the pipe makes that data available for reading from the other end. Pipes provide a way for processes to communicate.

## 🗂️ File System

The file system provides:

- **Data files** - uninterpreted byte arrays
- **Directories** - contain named references to data files and other directories

The directories form a tree structure, starting at a special directory called the **root** (`/`).

### 📇 Inode

An **inode** (index node) represents the underlying file structure. Key concepts:

- A file's **name** is distinct from the file itself
- The same underlying file (inode) can have **multiple names**, called **links**
- The `link()` system call creates another filename referring to the same inode
- Inodes **do not store actual data** - they store metadata indicating where to find the storage blocks of each file's data
