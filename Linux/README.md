# [based off the xv6/x86 architecture](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)

> ### Interfaces
An operating system provides services to user programs through an interface 


![Interface](https://github.com/cristianmagana/site-reliability-engineeing/blob/master/Linux/img/interface.png)

<br>

> ### Kernel
<br> 
The core interface between a computer’s hardware and its processes. The kernel uses the CPU’s hardware protection mechanisms to ensure that each process executing in user space can access only its own memory. The kernel executes with the hardware privileges required to implement these protections; user programs execute without those privileges. When a user program invokes a system call, the hardware raises the privilege level and starts executing a pre-arranged function in the kernel. 

<br>The kernel has 4 jobs: <br><br>

1. Memory management: Keep track of how much memory is used to store what, and where

2. Process management: Determine which processes can use the central processing unit (CPU), when, and for how long
3. Device drivers: Act as mediator/interpreter between the hardware and processes
4. System calls and security: Receive requests for service from the processes

<br>

> ### Shell
The shell is an ordinary program that reads commands from the user and executes them.The fact that the shell is a user program, not part of the kernel, illustrates the power of the system call interface.

<br>


> ### Process
<br> 

Each running program, called a process, has memory containing instructions, data, and a stack. The instructions implement the program’s computation. The data are the variables on which the computation acts. The stack organizes the program’s procedure calls. A process alternates between executing in user space and kernel space.

<br>


> ### System Calls 
<br>
When a process needs to invoke a kernel service, it invokes a procedure call in the operating system interface. The system call enters the kernel; the kernel performs the service and returns.

<br>

<br>

<details>

<br>

  <summary>List of System Calls</summary>
  
| System call | Description |
| -------- | ------ |
| fork()     | Create a process |
| exit()     | Terminate the current process  |
| wait()     | Wait for a child process to exit |
| kill(pid)  | Terminate process pid | 
| getpid()   | Return the current process’s pid |
| sleep(n) |  Sleep for n clock ticks | 
| exec(filename, *argv)    |  Load a file and execute it  |
| sbrk(n) | Grow process’s memory by n bytes | 
| open(filename, flags)    |  Open a file; the flags indicate read/write  |
| read(fd, buf, n) |  Read n bytes from an open file into buf | 
| write(fd, buf, n)   | Write n bytes to an open file  |
| close(fd) | Release open file fd | 
| dup(fd)    |   Duplicate fd |
| pipe(p) |  Create a pipe and return fd’s in p | 
| chdir(dirname)    |  Change the current directory  |
| mkdir(dirname) |  Create a new directory | 
| mknod(name, major, minor) |  Create a device file | 
| fstat(fd) |  Return info about an open file | 
| link(f1, f2) |  Create another name (f2) for the file f1 | 
| unlink(filename) | Remove a file  | 


</details>