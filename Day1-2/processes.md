# Week 1, Day 1-2: Process Fundamentals

## Theory Section

### What are Processes?

A **process** is an instance of a running program in Linux. When you execute a program, the kernel creates a process to run that program's code. Each process has its own memory space, execution context, and system resources.

**Key Characteristics:**
- Processes are isolated from each other
- Each process has a unique Process ID (PID)
- Processes can create child processes
- Processes have owners (UIDs) and permissions
- Processes can be in different states at any given time

### Processes vs Threads

**Process:**
- Independent execution unit with its own memory space
- Has its own virtual address space
- Cannot directly access another process's memory
- Creating processes is more expensive (fork operation)
- Inter-process communication (IPC) is needed for communication

**Thread:**
- Lightweight execution unit within a process
- Shares memory space with other threads in the same process
- Can directly access shared memory
- Creating threads is cheaper than creating processes
- Threads within a process can easily share data

**Why This Matters for Containers:**
Containers isolate processes using namespaces. Understanding process isolation is fundamental to understanding how container isolation works. Each container typically has a PID 1 process, just like a system has init/systemd as PID 1.

---

### Process Lifecycle

The life of a process follows these key stages:

#### 1. **Creation (fork)**
- A parent process calls `fork()` system call
- Kernel creates an exact copy of the parent process
- Child process gets a new PID
- Both processes continue execution from the same point
- `fork()` returns 0 to the child, and the child's PID to the parent

#### 2. **Program Loading (exec)**
- After fork, child often calls `exec()` family of system calls
- `exec()` replaces the process's memory with a new program
- Process PID remains the same, but the program changes
- Common pattern: fork + exec to run new programs

#### 3. **Execution**
- Process runs its code
- Can be scheduled on/off CPU by the kernel scheduler
- Can create more child processes
- Can interact with files, networks, and other processes

#### 4. **Termination (exit)**
- Process calls `exit()` or returns from main()
- Kernel frees most process resources
- Process becomes a "zombie" until parent collects exit status
- Exit status is stored for parent to retrieve

#### 5. **Reaping (wait)**
- Parent calls `wait()` or `waitpid()` to collect child's exit status
- Once reaped, zombie process is fully removed
- If parent dies first, init/systemd adopts orphaned children

**Container Relevance:**
In containers, the main container process (PID 1 in the container's PID namespace) must properly reap zombie processes, otherwise they accumulate. This is why many containers use init systems or process managers.

---

### Process States

At any moment, a process is in one of several states:

#### **R - Running/Runnable**
- Currently executing on a CPU, OR
- Ready to run and waiting for CPU time
- In the run queue

#### **S - Interruptible Sleep**
- Waiting for an event (I/O, signal, timer)
- Can be woken up by signals
- Most common state for idle processes
- Example: waiting for keyboard input

#### **D - Uninterruptible Sleep**
- Waiting for I/O that cannot be interrupted
- Cannot be killed with signals (even SIGKILL)
- Usually very brief state
- If stuck in D state = usually indicates kernel/driver problem

#### **T - Stopped**
- Execution stopped by signal (SIGSTOP, SIGTSTP)
- Can be resumed with SIGCONT
- Used by debuggers and job control

#### **Z - Zombie/Defunct**
- Process has terminated but not yet been reaped by parent
- No resources except PID and exit status
- Shows as `<defunct>` in process listing
- Accumulating zombies indicate parent not calling wait()

#### **I - Idle** (kernel thread)
- Idle kernel threads
- Not relevant for user processes

**Viewing States:**
```bash
ps aux          # See STAT column
top             # See S column
cat /proc/[PID]/status | grep State
```

---

### Parent-Child Process Relationships

Linux processes form a tree structure:

```
systemd (PID 1)
├── sshd
│   └── bash (your login shell)
│       ├── vim
│       └── sleep
├── podman
│   └── conmon
│       └── container process (PID 1 in container)
└── httpd
    ├── httpd worker
    └── httpd worker
```

**Key Concepts:**

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

### Process Identifiers

Every process has multiple IDs that identify it in different contexts:

#### **PID (Process ID)**
- Unique identifier for the process
- Assigned sequentially by kernel
- Visible in the process's own PID namespace
- In containers: PID in container namespace differs from host namespace

#### **PPID (Parent Process ID)**
- PID of the process that created this process
- If parent dies, PPID becomes 1 (init/systemd)
- Shows the process tree hierarchy

#### **PGID (Process Group ID)**
- Group of related processes
- Used for job control (foreground/background jobs)
- All processes in a pipeline share a PGID
- Example: `cat file | grep text | wc -l` = one process group

#### **SID (Session ID)**
- Collection of process groups
- Created when user logs in
- Associated with a controlling terminal
- When you log out, all processes in your session typically terminate

#### **TID (Thread ID)**
- Thread identifier within a process
- For single-threaded processes, TID = PID
- Multi-threaded processes have one PID, multiple TIDs

#### **TGID (Thread Group ID)**
- The PID of the main thread
- All threads in a process share the same TGID

**Viewing Process IDs:**
```bash
ps -eo pid,ppid,pgid,sid,comm
ps -T -p [PID]                    # Show threads
cat /proc/[PID]/status            # Complete info
```

**Container Perspective:**
```bash
# On host
ps aux | grep container_process   # Shows host PID

# In container
ps aux                            # Shows container PID (often PID 1)

# The same process has different PIDs in different namespaces!
```

---

## Self-Assessment Questions

Answer these questions to test your understanding. Research and experiment on your Linux system to find the answers!

### Multiple Choice Questions

**Q1.** What happens immediately after a process calls `fork()`?
- a) The process terminates
- b) A new program is loaded into memory
- c) An identical copy of the process is created
- d) The process enters sleep state

**Q2.** Which process state CANNOT be interrupted by signals, including SIGKILL?
- a) S (Interruptible Sleep)
- b) D (Uninterruptible Sleep)
- c) T (Stopped)
- d) Z (Zombie)

**Q3.** What is the PPID of an orphaned process?
- a) 0
- b) 1
- c) The PID of the process that killed the parent
- d) It becomes undefined

**Q4.** In a container, what is special about the process running as PID 1?
- a) It always runs as root user
- b) It must reap zombie processes
- c) It cannot be killed
- d) It uses the most CPU resources

**Q5.** What system call is typically used after `fork()` to load a new program?
- a) `load()`
- b) `exec()`
- c) `run()`
- d) `start()`

### Practical Questions

**Q6.** On your Linux system, run `ps -eo pid,ppid,pgid,sid,stat,comm` and find:
- What is the PID of systemd/init?
- What is the PPID of your current shell?
- Find a process in state 'S' - what is it waiting for?
--->ps -p 5565 -o pid,stat,wchan,comm
--->cat /proc/5565/wchan
--->sudo strace -p 5565

**Q7.** Create a zombie process:
```bash
# Run this in bash
(sleep 30 & exec sleep 300)
```
While this is running, use `ps aux | grep defunct` to find the zombie. What is its state? Why is it a zombie?
--->
sleep 30 & exec sleep 300
Here's the trick:

Before exec:

bash (PID 12345)
  └── sleep 30 (PID 12346, background child)

After exec sleep 300:

sleep 300 (PID 12345) ← SAME PID as bash had!
  └── sleep 30 (PID 12346) ← Still a child!
Key Point: exec DOESN'T create a new process!

## exec replaces the program code in the existing process
## The PID stays the same
The parent-child relationships are preserved
sleep 30 still has the same PPID (parent PID)


**Q8.** Examine process relationships:
```bash
pstree -p
```
- Find the process tree for your shell
- How many parent processes are between your shell and PID 1?
- Run a container with `podman run -d nginx` and find it in the pstree output
---> pstree -p | wc -l 
---> pstree -p | grep -E "conmon|nginx"
---> pstree -p -s 19833
systemd(1)───systemd(5122)───conmon(19831)───nginx(19833)─┬─nginx(19861)
                                                          └─nginx(19862)


**Q9.** Check thread information:
```bash
# Start a multi-threaded process
firefox &  # or any multi-threaded application
ps -T -p [PID_of_firefox]
```
- How many threads does it have?
- What is the TID of the main thread?
- How does TID compare to PID?
ps -T -o pid,tid,comm -p 20280 | head -10
```bash
PID     TID COMMAND
  20280   20280 firefox          ← Main thread: TID = PID
  20280   20281 GMUnitThread     ← Other threads: TID ≠ PID
  20280   20282 Compositor       ← Different TID, same PID
  20280   20283 ImageIO          ← Different TID, same PID
```

**Q10.** Experiment with process states:
```bash
# Terminal 1
cat > /dev/null

# Terminal 2 - find the cat process
ps aux | grep cat
```
- What state is the cat process in?
- Now press Ctrl+Z in Terminal 1. What state is it in now?
- Run `fg` to resume it. What state does it return to?

### Advanced Questions

**Q11.** Using the /proc filesystem, find detailed information about your shell process:
```bash
echo $$                              # Get your shell's PID
cat /proc/$$/status | grep -E "Pid|PPid|Tgid|Ngid"
```
- What is the relationship between Pid and Tgid?
- Why might Ngid be 0?

**Q12.** In a container context, explain why this command shows different PIDs:
```bash
# On host
podman run -d nginx
podman top [container_id]

# Inside container
podman exec [container_id] ps aux
```
What causes this difference?

**Q13.** Research and explain: Why is it dangerous for a container's PID 1 process to be a shell script that doesn't properly handle signals? What problems can occur?

**Q14.** Create a process group and observe it:
```bash
sleep 100 | sleep 100 | sleep 100 &
ps -eo pid,pgid,comm | grep sleep
```
- Do all three sleep processes have the same PGID?
- What is the significance of them sharing a PGID?
- Kill the entire process group with: `kill -TERM -[PGID]`

**Q15.** Advanced challenge: Write a simple explanation of what happens step-by-step when you run this command in bash:
```bash
ls -l | grep txt | wc -l
```
Include: fork, exec, pipes, process groups, and how the shell manages these processes.

---

## Hands-On Exercises

### Exercise 1: Process Creation Chain
Create a script that demonstrates fork-exec behavior:
```bash
#!/bin/bash
echo "Parent PID: $$"
echo "Starting child process..."
bash -c 'echo "Child PID: $$"; sleep 10' &
echo "Parent continuing..."
wait
```
- What are the PIDs of parent and child?
- What is the PPID of the child?

### Exercise 2: Zombie Hunter
Create and identify zombie processes, then clean them up:
1. Create zombies using the earlier technique
2. Use `ps aux | grep defunct` to find them
3. Kill the parent process to make systemd reap them
4. Verify they're gone

### Exercise 3: Container Process Investigation
Run a container and investigate its process structure:
```bash
podman run -d --name test nginx
podman top test
podman exec test ps aux
ps aux | grep nginx
```
Document the PID differences between host and container views.

### Exercise 4: State Transitions
Document a process moving through different states:
1. Start a process
2. Stop it (Ctrl+Z or SIGSTOP)
3. Resume it (fg or SIGCONT)
4. Put it in background
5. Kill it
Use `ps` to observe state changes at each step.

### Exercise 5: Process Tree Analysis
Use `pstree -p` to map out your entire session:
- Identify your login process
- Find all child processes you've created
- Find any container-related processes
- Draw the tree structure on paper

---

## Key Takeaways

✅ Processes are isolated instances of running programs with unique PIDs
✅ fork() creates processes, exec() loads new programs
✅ Process states (R, S, D, T, Z) indicate what a process is doing
✅ Parent-child relationships form a tree rooted at PID 1
✅ Zombie processes occur when parents don't call wait()
✅ Container PID 1 must properly handle child processes
✅ PIDs are namespace-specific - same process has different PIDs in different namespaces

---

## Next Steps

After completing these questions and exercises, you should be comfortable with:
- How processes are created and managed in Linux
- The significance of process states
- Why PID 1 is special in containers
- How to investigate process relationships

**Tomorrow we'll cover:** Process Scheduling & Priorities (Day 3-4)

Continue experimenting with the commands and concepts. The more hands-on practice you get, the better prepared you'll be for the EX188 exam!