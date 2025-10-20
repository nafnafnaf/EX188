# Week 1, Day 1-2: Process Fundamentals - Self-Assessment Questions

## Multiple Choice Questions

### Q1. What happens immediately after a process calls `fork()`?
- a) The process terminates
- b) A new program is loaded into memory
- c) An identical copy of the process is created
- d) The process enters sleep state

---

### Q2. Which process state CANNOT be interrupted by signals, including SIGKILL?
- a) S (Interruptible Sleep)
- b) D (Uninterruptible Sleep)
- c) T (Stopped)
- d) Z (Zombie)

---

### Q3. What is the PPID of an orphaned process?
- a) 0
- b) 1
- c) The PID of the process that killed the parent
- d) It becomes undefined

---

### Q4. In a container, what is special about the process running as PID 1?
- a) It always runs as root user
- b) It must reap zombie processes
- c) It cannot be killed
- d) It uses the most CPU resources

---

### Q5. What system call is typically used after `fork()` to load a new program?
- a) `load()`
- b) `exec()`
- c) `run()`
- d) `start()`

---

## Practical Questions

### Q6. Process Identification
On your Linux system, run `ps -eo pid,ppid,pgid,sid,stat,comm` and find:
- What is the PID of systemd/init?
- What is the PPID of your current shell?
- Find a process in state 'S' - what is it waiting for?

---

### Q7. Creating a Zombie Process
Create a zombie process:
```bash
# Run this in bash
(sleep 30 & exec sleep 300)
```
While this is running, use `ps aux | grep defunct` to find the zombie. 
- What is its state? 
- Why is it a zombie?

---

### Q8. Process Relationships
Examine process relationships:
```bash
pstree -p
```
- Find the process tree for your shell
- How many parent processes are between your shell and PID 1?
- Run a container with `podman run -d nginx` and find it in the pstree output

---

### Q9. Thread Information
Check thread information:
```bash
# Start a multi-threaded process
firefox &  # or any multi-threaded application
ps -T -p [PID_of_firefox]
```
- How many threads does it have?
- What is the TID of the main thread?
- How does TID compare to PID?

---

### Q10. Process State Transitions
Experiment with process states:
```bash
# Terminal 1
cat > /dev/null

# Terminal 2 - find the cat process
ps aux | grep cat
```
- What state is the cat process in?
- Now press Ctrl+Z in Terminal 1. What state is it in now?
- Run `fg` to resume it. What state does it return to?

---

## Advanced Questions

### Q11. /proc Filesystem Investigation
Using the /proc filesystem, find detailed information about your shell process:
```bash
echo $$                              # Get your shell's PID
cat /proc/$$/status | grep -E "Pid|PPid|Tgid|Ngid"
```
- What is the relationship between Pid and Tgid?
- Why might Ngid be 0?

---

### Q12. Container PID Namespaces
In a container context, explain why this command shows different PIDs:
```bash
# On host
podman run -d nginx
podman top [container_id]

# Inside container
podman exec [container_id] ps aux
```
What causes this difference?

---

### Q13. Container PID 1 Dangers
Research and explain: Why is it dangerous for a container's PID 1 process to be a shell script that doesn't properly handle signals? What problems can occur?

---

### Q14. Process Groups
Create a process group and observe it:
```bash
sleep 100 | sleep 100 | sleep 100 &
ps -eo pid,pgid,comm | grep sleep
```
- Do all three sleep processes have the same PGID?
- What is the significance of them sharing a PGID?
- Kill the entire process group with: `kill -TERM -[PGID]`

---

### Q15. Pipeline Execution Challenge
Advanced challenge: Write a simple explanation of what happens step-by-step when you run this command in bash:
```bash
ls -l | grep txt | wc -l
```
Include in your explanation:
- fork operations
- exec operations
- pipes creation
- process groups
- how the shell manages these processes

---

## Answer Guidelines

**For Multiple Choice Questions (Q1-Q5):**
- Choose the single best answer
- Verify your answer by testing on your system when possible

**For Practical Questions (Q6-Q10):**
- Run the commands on your Linux system
- Document your observations
- Take screenshots if helpful
- Explain what you discovered

**For Advanced Questions (Q11-Q15):**
- Research using man pages, documentation, and experimentation
- Provide detailed explanations
- Include command outputs to support your answers
- Think about the "why" not just the "what"

---

## Scoring Guide

- **Q1-Q5 (Multiple Choice):** 1 point each = 5 points
- **Q6-Q10 (Practical):** 2 points each = 10 points
- **Q11-Q15 (Advanced):** 3 points each = 15 points

**Total:** 30 points

**Passing Score:** 21/30 (70%)

---

## Next Steps

After completing these questions:
1. Review any questions you got wrong
2. Re-read the relevant theory sections
3. Practice the commands until you're comfortable
4. Move on to the hands-on exercises
5. When ready, proceed to Day 3-4 material