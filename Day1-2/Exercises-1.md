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


### Complete State Breakdown:

| State | Meaning | When You See It |
| --- | --- | --- |
| **S** | Interruptible Sleep | Waiting for I/O, input, events |
| **S+** | Interruptible Sleep (foreground) | Same, but in foreground process group |
| **T** | Stopped | After Ctrl+Z or SIGSTOP |
| **R** | Running/Runnable | Actively using CPU or ready to run |
| **D** | Uninterruptible Sleep | Waiting for critical I/O (can't interrupt) |
| **Z** | Zombie | Terminated, waiting to be reaped |

```bash
# Terminal 1
cat > /dev/null

# Terminal 2
ps aux | grep cat        # S+

# Terminal 1 - Ctrl+Z
# Terminal 2
ps aux | grep cat        # T

# Terminal 1
bg                       # Resume in BACKGROUND
# Terminal 2
ps aux | grep cat        # S (no +, because now background!)

# Terminal 1
fg                       # Bring back to foreground
# Terminal 2
ps aux | grep cat        # S+ (back to foreground)
```


### Advanced Questions

**Q11.** Using the /proc filesystem, find detailed information about your shell process:
```bash
echo $$                              # Get your shell's PID
cat /proc/$$/status | grep -E "Pid|PPid|Tgid|Ngid"
```
- What is the relationship between Pid and Tgid?
- Why might Ngid be 0?

# ANSWER:
Pid is equal to Tgid (proccess id = Thread id because shell is not multi-threaded)
Ngid = 0 means your shell is in the root PID namespace (the host namespace), NOT in a nested PID namespace.

```bash
# Your shell (host namespace only)
cat /proc/$$/status | grep NSpid
NSpid:  18933           ← Only one PID (in host namespace)

# Container process (multiple namespaces)
sudo cat /proc/19833/status | grep NSpid
NSpid:  19833   1       ← Two PIDs: 19833 in host, 1 in container
        ^^^^^   ^
        host    container
```



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
### ANSWER:
# 1) PID 1 has a special kernel responsibility: it must reap zombie processes (call wait() on terminated child processes).
```bash
#!/bin/bash
# Bad PID 1 script
/app/start-services.sh &
nginx &
wait
```
Shell scripts typically don't reap zombie children properly
When child processes die, they become zombies
Zombies accumulate, consuming PIDs
Eventually run out of available PIDs
New processes can't start → container becomes unusable

# 2) Signal Handling - Won't Terminate Properly
Containers need to respond to signals like:

SIGTERM (graceful shutdown)
SIGINT (Ctrl+C)
SIGKILL (force kill)
```bash
#!/bin/bash
nginx
```
Shell scripts don't forward signals to child processes by default
When you send SIGTERM to the container:

Signal goes to the shell (PID 1)
Shell ignores or mishandles it
Child processes (nginx) don't receive the signal
Container doesn't shutdown gracefully


Docker/Podman must use SIGKILL (force kill) after timeout
Data loss or corrupted state because processes couldn't cleanup
```bash
# Try to stop container
podman stop mycontainer

# What happens internally:
# 1. Podman sends SIGTERM to PID 1 (bash script)
# 2. Bash doesn't forward it to nginx
# 3. Nginx keeps running
# 4. After 10 seconds, Podman sends SIGKILL
# 5. Nginx killed abruptly - no graceful shutdown!
```
### 3) Child Process Becomes Orphaned
```bash
#!/bin/bash
/app/main-app &   # Starts in background
exit 0            # Shell exits
```
Main app is now orphaned
In a container, there's no init system to adopt it
Container exits (PID 1 died)
App still running but container is "stopped"
Inconsistent state

### Real-World Example - The Problem:
### Bad Dockerfile:
```dockerfile
FROM nginx
COPY start.sh /start.sh
CMD ["/start.sh"]
```
### Bad start.sh:
```bash
bash#!/bin/bash
echo "Starting nginx..."
nginx -g "daemon off;" &
echo "Started!"
wait
```
### Problems:
❌ Doesn't handle SIGTERM
❌ Doesn't reap zombies
❌ wait only waits for direct children
❌ If nginx spawns workers, zombies accumulate

### The Solution: Use a Proper Init System ✅
### Option 1: Use exec (simplest)
```bash
bash#!/bin/bash
# Good script - replaces itself
exec nginx -g "daemon off;"
```
Shell is replaced by nginx
nginx becomes PID 1
nginx handles signals properly

### Option 2: Use tini (tiny init)
```dockerfile
FROM nginx
RUN apt-get update && apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["nginx", "-g", "daemon off;"]
```
tini becomes PID 1
Forwards signals
Reaps zombies
Only 10KB!

### Option 3: Use dumb-init
```dockerfile
FROM nginx
RUN wget -O /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.5/dumb-init_1.2.5_x86_64
RUN chmod +x /usr/local/bin/dumb-init
ENTRYPOINT ["/usr/local/bin/dumb-init", "--"]
CMD ["nginx", "-g", "daemon off;"]
```
### Option 4: Run process directly (best if possible)
```dockerfile
FROM nginx
CMD ["nginx", "-g", "daemon off;"]
```
No shell wrapper
### nginx is PID 1 directly


**Q14.** Create a process group and observe it:
```bash
sleep 100 | sleep 100 | sleep 100 &
ps -eo pid,pgid,comm | grep sleep
```
- Do all three sleep processes have the same PGID?
- What is the significance of them sharing a PGID?
- Kill the entire process group with: `kill -TERM -[PGID]`
### ANSWER:
Yes 
```bash
sleep 10 | sleep 10 | sleep 10 & ps -eo pid,pgid,comm | grep sleep
[4] 21816
  21814   21814 sleep
  21815   21814 sleep
  21816   21814 sleep
```
```bash
# Terminal 2 - Kill the entire group (note the NEGATIVE sign!)
kill -TERM -22345
#           ↑ NEGATIVE PGID kills entire group!
```
### But:
```bash
kill -TERM 22345     # Kills ONLY PID 22345
```
```bash
kill -TERM -22345    # Kills ALL processes in PGID 22345
#          ↑
#   NEGATIVE = Process Group!
```
They are three different processes that share a Process Group ID:
```bash
Process 1: PID 21814, PGID 21814  ← Group Leader
Process 2: PID 21815, PGID 21814  ← Group Member
Process 3: PID 21816, PGID 21814  ← Group Member
```
### Process Group Leader:
The first process (PID 21814) is the group leader
### Its PID = PGID (21814 = 21814)
Other processes join this group


**Q15.** Advanced challenge: Write a simple explanation of what happens step-by-step when you run this command in bash:
```bash
ls -l | grep txt | wc -l
```
Include: fork, exec, pipes, process groups, and how the shell manages these processes.
### ANSWER:
  ```bash
  When bash executes `ls -l | grep txt | wc -l`:

1. **Parse**: Shell identifies 3 commands and 2 pipes
2. **Create Pipes**: Shell creates 2 pipes (4 file descriptors total)
3. **Fork**: Shell forks 3 child processes simultaneously
4. **Process Group**: First child becomes group leader, others join (shared PGID)
5. **FD Setup**: Each child redirects stdin/stdout to appropriate pipes
6. **Exec**: Each child calls exec() to become ls, grep, or wc
7. **Parent Cleanup**: Shell closes all pipe FDs
8. **Execute**: All 3 processes run concurrently, data flows through pipes
9. **Wait**: Shell waits for all 3 children to finish
10. **Reap**: Shell collects exit statuses, cleans up

Key Points:
- THREE forks (not one)
- THREE execs (not two)
- All processes run CONCURRENTLY (not sequentially)
- Pipes created BEFORE forking
- All share same PGID for job control
- Data flows: ls → pipe1 → grep → pipe2 → wc → terminal
```

### The Data Flow:
```bash
ls -l produces:
-rw-r--r-- 1 user group  100 Jan 1 file.txt
-rw-r--r-- 1 user group  200 Jan 1 doc.pdf
-rw-r--r-- 1 user group  150 Jan 1 notes.txt

↓ pipe1 ↓

grep txt filters:
-rw-r--r-- 1 user group  100 Jan 1 file.txt
-rw-r--r-- 1 user group  150 Jan 1 notes.txt

↓ pipe2 ↓

wc -l counts:
2
```
3 forks, not 1
3 execs, not 2
Concurrent execution, not sequential
Pipes created before forking

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
## SOLUTION:
```bash
#!/bin/bash
echo "Parent PID: $$"
echo "Starting child process..."
bash -c 'echo "Child PID: $$"; sleep 10' &
CHILD_PID=$!
echo "Child PID from parent: $CHILD_PID"
echo "Parent continuing..."
wait
echo "Child has finished"

ps -o pid,ppid,comm
    PID    PPID COMMAND
  21352   17399 bash -->1st shell pid and its parent's pid
  33285   21352 sleep -->command pid and its parent's pid
  33290   21352 ps -->command pid and its parent's pid
  ```



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