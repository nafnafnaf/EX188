# Week 1, Day 1-2: Process Fundamentals - Theory

## What are Processes?

A **process** is an instance of a running program in Linux. When you execute a program, the kernel creates a process to run that program's code. Each process has its own memory space, execution context, and system resources.

**Key Characteristics:**
- Processes are isolated from each other
- Each process has a unique Process ID (PID)
- Processes can create child processes
- Processes have owners (UIDs) and permissions
- Processes can be in different states at any given time

## Processes vs Threads

### Process:
- Independent execution unit with its own memory space
- Has its own virtual address space
- Cannot directly access another process's memory
- Creating processes is more expensive (fork operation)
- Inter-process communication (IPC) is needed for communication

### Thread:
- Lightweight execution unit within a process
- Shares memory space with other threads in the same process
- Can directly access shared memory
- Creating threads is cheaper than creating processes
- Threads within a process can easily share data

### Why This Matters for Containers:
Containers isolate processes using namespaces. Understanding process isolation is fundamental to understanding how container isolation works. Each container typically has a PID 1 process, just like a system has init/systemd as PID 1.

---

## Process Lifecycle

The life of a process follows these key stages:

### 1. Creation (fork)
- A parent process calls `fork()` system call
- Kernel creates an exact copy of the parent process
- Child process gets a new PID
- Both processes continue execution from the same point
- `fork()` returns 0 to the child, and the child's PID to the parent

### 2. Program Loading (exec)
- After fork, child often calls `exec()` family of system calls
- `exec()` replaces the process's memory with a new program
- Process PID remains the same, but the program changes
- Common pattern: fork + exec to run new programs

### 3. Execution
- Process runs its code
- Can be scheduled on/off CPU by the kernel scheduler
- Can create more child processes
- Can interact with files, networks, and other processes

### 4. Termination (exit)
- Process calls `exit()` or returns from main()
- Kernel frees most process resources
- Process becomes a "zombie" until parent collects exit status
- Exit status is stored for parent to retrieve

### 5. Reaping (wait)
- Parent calls `wait()` or `waitpid()` to collect child's exit status
- Once reaped, zombie process is fully removed
- If parent dies first, init/systemd adopts orphaned children

### Container Relevance:
In containers, the main container process (PID 1 in the container's PID namespace) must properly reap zombie processes, otherwise they accumulate. This is why many containers use init systems or process managers.

---

## Process States

At any moment, a process is in one of several states:

### R - Running/Runnable
- Currently executing on a CPU, OR
- Ready to run and waiting for CPU time
- In the run queue

### S - Interruptible Sleep
- Waiting for an event (I/O, signal, timer)
- Can be woken up by signals
- Most common state for idle processes
- Example: waiting for keyboard input

### D - Uninterruptible Sleep
- Waiting for I/O that cannot be interrupted
- Cannot be killed with signals (even SIGKILL)
- Usually very brief state
- If stuck in D state = usually indicates kernel/driver problem

### T - Stopped
- Execution stopped by signal (SIGSTOP, SIGTSTP)
- Can be resumed with SIGCONT
- Used by debuggers and job control

### Z - Zombie/Defunct
- Process has terminated but not yet been reaped by parent
- No resources except PID and exit status
- Shows as `<defunct>` in process listing
- Accumulating zombies indicate parent not calling wait()

### I - Idle (kernel thread)
- Idle kernel threads
- Not relevant for user processes

### Viewing States:
```bash
ps aux          # See STAT column
top             # See S column
cat /proc/[PID]/status | grep State
```

---

## Parent-Child Process Relationships

Linux processes form a tree structure:

```
systemd (PID 1) - the "grandparent" of all processes
├── sshd - handles remote connections
│   └── bash - your terminal shell
│       ├── vim - text editor you opened
│       └── sleep - a command you ran
├── podman - container tool
│   └── conmon - watches over containers
│       └── container process - the app running in your container
└── httpd - web server
    ├── httpd worker - handles web requests
    └── httpd worker - handles more web requests
```
Important Process Types

**Orphan Process**: When a parent process dies before its child, the child gets "adopted" by systemd (PID 1). This is actually helpful because it prevents abandoned processes from piling up.

**Zombie Process**: When a child process finishes but the parent hasn't acknowledged it yet. It's like a ghost - it shows up in the process list as <defunct> but doesn't use any real resources except its process ID number.

**Daemon Process**: A background service that keeps running even after you log out. It has no terminal attached to it and usually gets adopted by systemd.

**Container PID 1**: The main process inside a container. Its job is to clean up any zombie processes that appear. If this process dies, the whole container shuts down.

**What is Conmon?**
Conmon is a small helper program that babysits containers in Podman. Think of it as a nanny that watches over a single container.

**What Conmon Does**
When you start a container with podman run, here's what happens:

**Starts up**: Podman creates a Conmon process to watch your container

**Becomes independent**: Conmon disconnects from Podman so it can run on its own

**Launches the container**: Conmon starts the actual container runtime (like runc) which runs your container
**Stays on duty**: Conmon keeps running the whole time your container is alive

**Handles communication**: It forwards anything you type into the container and shows you the container's output

**Reports back**: When the container stops, Conmon records when it stopped and why

Why Conmon Matters****

**Tiny and efficient**: It's a very small program that barely uses any memory
**No daemon needed**: Unlike Docker, Podman doesn't need a big background service running all the time - Conmon handles the monitoring instead
**One per container**: Each container gets its own personal Conmon to watch over it
**Simple communication**: It uses a socket (like a private phone line) to talk to Podman

### Key Concepts:


**Orphan Process:**
- Parent dies before child
- Child is "adopted" by init (PID 1) or systemd
- Prevents zombies from accumulating

**Zombie Process:**
- Child terminated but parent hasn't called wait()
- Takes up a PID but no other resources
- Shows as `<defunct>` in ps output

**Daemon Process:**
- Background process with no controlling terminal
- Usually has PPID = 1 (adopted by init/systemd)
- Runs independently of user sessions

**Container PID 1:**
- First process in a container's PID namespace
- Responsible for reaping zombie processes
- If it dies, entire container stops

---

## Process Identifiers

Every process has multiple IDs that identify it in different contexts:

### PID (Process ID)
- Unique identifier for the process
- Assigned sequentially by kernel
- Visible in the process's own PID namespace
- In containers: PID in container namespace differs from host namespace

### PPID (Parent Process ID)
- PID of the process that created this process
- If parent dies, PPID becomes 1 (init/systemd)
- Shows the process tree hierarchy

### PGID (Process Group ID)
- Group of related processes
- Used for job control (foreground/background jobs)
- All processes in a pipeline share a PGID
- Example: `cat file | grep text | wc -l` = one process group

### SID (Session ID)
- Collection of process groups
- Created when user logs in
- Associated with a controlling terminal
- When you log out, all processes in your session typically terminate

### TID (Thread ID)
- Thread identifier within a process
- For single-threaded processes, TID = PID
- Multi-threaded processes have one PID, multiple TIDs

### TGID (Thread Group ID)
- The PID of the main thread
- All threads in a process share the same TGID

### Viewing Process IDs:
```bash
ps -eo pid,ppid,pgid,sid,comm
ps -T -p [PID]                    # Show threads
cat /proc/[PID]/status            # Complete info
```

### Container Perspective:
```bash
# On host
ps aux | grep container_process   # Shows host PID

# In container
ps aux                            # Shows container PID (often PID 1)

# The same process has different PIDs in different namespaces!
```

---

## Key Takeaways

✅ Processes are isolated instances of running programs with unique PIDs

✅ fork() creates processes, exec() loads new programs

✅ Process states (R, S, D, T, Z) indicate what a process is doing

✅ Parent-child relationships form a tree rooted at PID 1

✅ Zombie processes occur when parents don't call wait()

✅ Container PID 1 must properly handle child processes

✅ PIDs are namespace-specific - same process has different PIDs in different namespaces