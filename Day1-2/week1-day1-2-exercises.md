# Week 1, Day 1-2: Process Fundamentals - Hands-On Exercises

## Exercise 1: Process Creation Chain

### Objective
Understand how fork() creates child processes and how they relate to parents.

### Instructions
1. Create a script named `process_chain.sh`:

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
```

2. Make it executable: `chmod +x process_chain.sh`
3. Run it: `./process_chain.sh`

### Tasks
- Record the PIDs of parent and child
- While the script is running, in another terminal run: `ps -eo pid,ppid,comm | grep -E "bash|sleep"`
- What is the PPID of the child process?
- What happens when the parent calls `wait`?

### Questions to Answer
1. Why does the child have a different PID than the parent?
2. What would happen if the parent didn't call `wait` and exited immediately?
3. How does this relate to container processes?

---

## Exercise 2: Zombie Hunter

### Objective
Learn to identify, create, and clean up zombie processes.

### Instructions

**Step 1: Create Zombies**
```bash
# Run this in terminal 1
(sleep 30 & exec sleep 300)
```

**Step 2: Identify Zombies**
```bash
# In terminal 2
ps aux | grep defunct
ps -eo pid,ppid,stat,comm | grep Z
```

**Step 3: Investigate the Zombie**
```bash
# Find the zombie's PID from the output above
cat /proc/[ZOMBIE_PID]/status
```

**Step 4: Clean Up**
```bash
# Find the parent PID (PPID) of the zombie
# Kill the parent to let systemd reap the zombie
kill [PARENT_PID]

# Verify the zombie is gone
ps aux | grep defunct
```

### Tasks
- Take screenshots of the zombie in ps output
- Note the zombie's state character (Z)
- Document what happens when you kill the parent
- Try to kill the zombie directly with `kill -9` - what happens?

### Questions to Answer
1. Why can't you kill a zombie process directly?
2. What information does a zombie process still contain?
3. In a container, what process is responsible for reaping zombies?
4. What happens if zombies accumulate in a container?

---

## Exercise 3: Container Process Investigation

### Objective
Understand how PID namespaces make the same process have different PIDs.

### Instructions

**Step 1: Start a Container**
```bash
podman run -d --name test nginx
```

**Step 2: View from Host**
```bash
# Get container ID
podman ps

# View processes from host perspective
podman top test

# View process tree on host
ps aux | grep nginx
pstree -p | grep nginx
```

**Step 3: View from Container**
```bash
# Execute commands inside container
podman exec test ps aux
podman exec test ps -eo pid,ppid,comm
```

**Step 4: Compare PIDs**
```bash
# Get the main nginx process PID on host
HOST_PID=$(podman inspect test --format '{{.State.Pid}}')
echo "Host PID: $HOST_PID"

# Get PID inside container
CONTAINER_PID=$(podman exec test pidof nginx | awk '{print $NF}')
echo "Container PID: $CONTAINER_PID"
```

**Step 5: Inspect Namespaces**
```bash
# View namespace IDs
sudo ls -l /proc/$HOST_PID/ns/

# Compare with host namespace
sudo ls -l /proc/1/ns/
```

### Tasks
- Document the PID differences in a table
- Draw a diagram showing host vs container PID view
- Note which namespaces are different

### Questions to Answer
1. Why does the same process have different PIDs?
2. What is the PID of nginx inside the container?
3. What happens if you kill the container's PID 1 from inside the container?
4. How does Podman implement this PID isolation?

---

## Exercise 4: State Transitions

### Objective
Observe a process moving through different states and understand state transitions.

### Instructions

**Step 1: Start a Process**
```bash
# Terminal 1
sleep 1000
```

**Step 2: Observe Running State**
```bash
# Terminal 2
ps aux | grep sleep
# Note: Should be in R or S state
```

**Step 3: Stop the Process**
```bash
# Terminal 1 - Press Ctrl+Z
# This sends SIGTSTP
```

**Step 4: Check Stopped State**
```bash
# Terminal 2
ps aux | grep sleep
# Note: Should be in T state
```

**Step 5: Resume in Background**
```bash
# Terminal 1
bg
```

**Step 6: Check State Again**
```bash
# Terminal 2
ps aux | grep sleep
# Note: Should be in S state (background)
```

**Step 7: Bring to Foreground**
```bash
# Terminal 1
fg
```

**Step 8: Kill the Process**
```bash
# Terminal 1 - Press Ctrl+C
# Or from Terminal 2: kill [PID]
```

### Tasks
Create a state transition diagram showing:
- Initial state
- State after Ctrl+Z
- State after bg
- State after fg
- Final state after kill

### Questions to Answer
1. What signals cause state transitions?
2. Why does `sleep` go to S state instead of R state?
3. What's the difference between `kill` and `kill -9`?
4. Can a process in D state be stopped with Ctrl+Z?

---

## Exercise 5: Process Tree Analysis

### Objective
Map and understand your session's process hierarchy.

### Instructions

**Step 1: View Your Process Tree**
```bash
pstree -p $$
```

**Step 2: Detailed Tree**
```bash
pstree -p -s $$
```

**Step 3: Full System Tree**
```bash
pstree -p | less
```

**Step 4: Create Child Processes**
```bash
# Open a new terminal or create background jobs
sleep 1000 &
sleep 2000 &
cat > /dev/null &
```

**Step 5: View Your Updated Tree**
```bash
pstree -p $$
```

**Step 6: Start Container Processes**
```bash
podman run -d nginx
podman run -d redis
pstree -p | grep -A5 podman
```

### Tasks
1. Draw your complete process tree on paper from PID 1 to your shell
2. Label each process with:
   - PID
   - PPID
   - Process name
   - State
3. Identify all container-related processes
4. Show where conmon fits in the tree

### Questions to Answer
1. How many levels are between your shell and PID 1?
2. What is the role of `conmon` in container process trees?
3. What happens to child processes if you kill your shell?
4. How do systemd user sessions appear in the tree?

---

## Submission Checklist

For each exercise, document:
- [ ] Command outputs (copy-paste or screenshots)
- [ ] Answers to all questions
- [ ] Any errors encountered and how you solved them
- [ ] Additional observations or insights
- [ ] Time spent on exercise

## Evaluation Criteria

Each exercise is worth 20 points:
- **Completion:** 10 points
- **Understanding:** 5 points
- **Documentation:** 5 points

**Total:** 100 points
**Passing Score:** 70/100

---

## Tips for Success

1. **Take your time** - Don't rush through exercises
2. **Document everything** - You'll reference this later
3. **Experiment** - Try variations of the commands
4. **Ask questions** - If something doesn't make sense, investigate
5. **Relate to containers** - Always think about how this applies to Podman/containers

---

## Next Steps

After completing all exercises:
1. Review your answers to the questions
2. Compare your findings with the theory material
3. Identify any concepts that are still unclear
4. Practice the commands until they feel natural
5. When confident, move on to Day 3-4: Process Scheduling & Priorities