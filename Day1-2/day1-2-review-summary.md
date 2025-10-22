# Day 1-2: Topics to Study Deeper - Review Summary

Based on your questions and challenges during Day 1-2, here are the concepts you should review and practice more deeply.

---

## 1. Understanding `exec` vs `fork`

### What You Struggled With:
You initially thought the zombie creation command used sequential execs, but it actually uses fork + exec.

### Key Concept to Master:

**`fork()` creates a NEW process (child)**
```bash
sleep 30 &
# Shell forks → creates child process
# Shell continues, child runs sleep
```

**`exec` REPLACES the current process**
```bash
exec sleep 300
# Current process transforms into sleep 300
# Same PID, different program
# Process relationships PRESERVED
```

### Practice Exercise:

```bash
# Test 1: Fork behavior
echo "Shell PID: $$"
sleep 30 &
echo "Shell PID still: $$"  # Same PID!
# Shell created a CHILD, didn't transform

# Test 2: Exec behavior
echo "Shell PID: $$"
exec sleep 30
# Terminal closes after 30s because shell was REPLACED

# Test 3: Combined (in subshell to protect your terminal)
(echo "Subshell PID: $$"; sleep 10 & exec sleep 20)
# Subshell forks sleep 10
# Then subshell transforms into sleep 20
```

### Why This Matters:
- **Containers**: Understanding how processes spawn is crucial
- **Zombie creation**: You need exec to keep parent alive without reaping
- **Process management**: Knowing when processes are created vs replaced

---

## 2. Subshells `()` vs Background Jobs `&`

### What You Asked:
"Is the creation of a subshell with () the same as if i was doing sleep &?"

### Key Differences:

| Feature | `sleep 30 &` | `(sleep 30 &)` |
|---------|--------------|----------------|
| **New processes** | 1 (sleep only) | 2 (subshell + sleep) |
| **Sleep's parent** | Your shell | Subshell |
| **Returns immediately?** | Yes | Yes |
| **Subshell exits when?** | N/A | Immediately after starting sleep |

### Practice Exercise:

```bash
# Test 1: Background job (no subshell)
echo "My shell: $$"
sleep 30 &
CHILD_PID=$!
ps -o pid,ppid,comm -p $$ -p $CHILD_PID
# Parent of sleep = your shell

# Test 2: Subshell with background
echo "My shell: $$"
(sleep 30 &)
# Returns immediately
ps aux | grep sleep
# Parent of sleep = 1 (systemd) - orphaned!

# Test 3: Subshell with exec (zombie maker!)
(sleep 30 & exec sleep 300)
# Subshell becomes sleep 300
# sleep 30 still child of sleep 300
# After 30s → ZOMBIE!
```

### Why This Matters:
- **Process hierarchy**: Understanding parent-child relationships
- **Orphan processes**: When processes get adopted by systemd
- **Zombie creation**: Need the right structure to create zombies

### Deep Dive Activity:

Create this test script:
```bash
#!/bin/bash
# subshell_test.sh

echo "=== Test 1: No subshell ==="
sleep 100 &
PID1=$!
echo "Sleep PID: $PID1"
ps -o pid,ppid,comm -p $PID1
echo ""

echo "=== Test 2: With subshell ==="
(sleep 100 &)
sleep 1  # Let subshell exit
PID2=$(pgrep -n sleep)
echo "Sleep PID: $PID2"
ps -o pid,ppid,comm -p $PID2
echo ""

killall sleep
```

Run it and observe the PPID differences!

---

## 3. Namespace Isolation and PID Translation

### What You Understood Well:
You correctly grasped that the same process has different PIDs in different namespaces!

### But Practice More:

**The NSpid field** - this is KEY to understanding namespaces:

```bash
# Start a container
podman run -d --name test nginx

# Get host PID
HOST_PID=$(podman inspect test --format '{{.State.Pid}}')

# Check NSpid (THIS IS THE PROOF!)
sudo cat /proc/$HOST_PID/status | grep NSpid
# Output: NSpid:  17940   1
#                ^^^^^   ^
#                host    container

# This PROVES:
# - Process 17940 on host
# - Same process is PID 1 in container
```

### Practice Exercise:

```bash
# Test with multiple containers
podman run -d --name nginx1 nginx
podman run -d --name nginx2 nginx

# For each container
for name in nginx1 nginx2; do
  echo "=== Container: $name ==="
  HOST_PID=$(podman inspect $name --format '{{.State.Pid}}')
  echo "Host PID: $HOST_PID"
  sudo cat /proc/$HOST_PID/status | grep NSpid
  echo "Inside container:"
  podman exec $name cat /proc/1/status | grep "^Pid:"
  echo ""
done

# Cleanup
podman stop nginx1 nginx2
podman rm nginx1 nginx2
```

### Why This Matters:
- **Container isolation**: Core concept for EX188
- **Debugging**: Finding processes across namespaces
- **Security**: Understanding isolation boundaries

---

## 4. conmon's Role in Container Architecture

### What You Learned:
conmon is on the HOST, not in the container!

### Deepen Your Understanding:

**The complete hierarchy:**
```
systemd(1) [root namespace]
  └── systemd(5122) [user namespace - your user]
      └── conmon(17938) [HOST namespace]
          └── nginx(17940) [CONTAINER namespace]
```

### Practice Exercise:

```bash
# Start container
podman run -d --name test nginx

# Find the COMPLETE chain
CONTAINER_PID=$(podman inspect test --format '{{.State.Pid}}')
echo "Container main process: $CONTAINER_PID"

# Find conmon (parent of container process)
CONMON_PID=$(ps -o ppid= -p $CONTAINER_PID | tr -d ' ')
echo "Conmon PID: $CONMON_PID"

# Verify conmon is on host
sudo readlink /proc/$CONMON_PID/ns/pid
sudo readlink /proc/$CONTAINER_PID/ns/pid
# Different namespace IDs!

# See full tree
pstree -p -s $CONTAINER_PID

# Cleanup
podman stop test
podman rm test
```

### Why This Matters:
- **Rootless containers**: Understanding user vs system processes
- **Troubleshooting**: Finding container processes
- **Architecture**: How Podman differs from Docker

### Advanced Challenge:

Compare Docker vs Podman architecture:
```bash
# If you have Docker
docker run -d --name test nginx
docker ps
pstree -p | grep -E "dockerd|containerd|nginx"

# With Podman
podman run -d --name test nginx
podman ps
pstree -p | grep -E "conmon|nginx"

# Notice: Docker has dockerd (root), Podman has no daemon
```

---

## 5. Zombie Process Creation Mechanics

### What You Initially Misunderstood:
The exact structure needed to create zombies.

### Master This Pattern:

**For a zombie, you need ALL of these:**
1. ✅ Parent process that stays alive
2. ✅ Child process that dies BEFORE parent
3. ✅ Parent that DOESN'T call `wait()`

### The Working Pattern:

```bash
(sleep 30 & exec sleep 300)
```

**Why this works:**
```
Step 1: Subshell created
Step 2: sleep 30 starts in background (child of subshell)
Step 3: Subshell transforms into sleep 300 (exec)
        - PID stays same
        - Program changes
        - Child relationship preserved
Step 4: After 30s, sleep 30 dies
Step 5: Parent (sleep 300) never calls wait()
Step 6: ZOMBIE! 🧟
```

### Practice Exercise - Zombie Lab:

```bash
# Create zombie_lab.sh
#!/bin/bash

echo "=== Creating Zombie ==="
(sleep 5 & exec sleep 60) &
PARENT_PID=$!
echo "Parent (sleep 60) PID: $PARENT_PID"

sleep 6  # Wait for child to become zombie

echo "=== Finding Zombie ==="
ps aux | grep defunct
ps -eo pid,ppid,stat,comm | grep Z

echo "=== Zombie Details ==="
ZOMBIE_PID=$(ps -eo pid,stat,comm | grep "Z" | grep sleep | awk '{print $1}')
if [ -n "$ZOMBIE_PID" ]; then
  echo "Zombie PID: $ZOMBIE_PID"
  echo "Zombie's parent: $PARENT_PID"
  ps -o pid,ppid,stat,comm -p $ZOMBIE_PID
  
  echo "=== Zombie has NO resources ==="
  cat /proc/$ZOMBIE_PID/status | grep -E "VmSize|VmRSS"
fi

echo "=== Killing Parent to Clean Zombie ==="
kill $PARENT_PID
sleep 1

echo "=== Verify Zombie Gone ==="
ps aux | grep defunct
echo "Done!"
```

### Why This Matters:
- **Container PID 1**: Must reap zombies or they accumulate
- **Process management**: Understanding parent responsibilities
- **Troubleshooting**: Identifying zombie accumulation issues

---

## 6. Process States and Transitions

### What You Got Right:
S → T → S+ state transitions with Ctrl+Z and fg

### Deepen Understanding:

**Complete state transition map:**

```
Process starts
    ↓
  [R] Running/Runnable
    ↓ (waits for I/O)
  [S] Interruptible Sleep
    ↓ (Ctrl+Z / SIGSTOP)
  [T] Stopped
    ↓ (fg / SIGCONT)
  [S] Interruptible Sleep
    ↓ (process exits)
  [Z] Zombie (if parent doesn't wait)
    ↓ (parent calls wait)
  [Gone] Fully reaped
```

### Practice Exercise:

```bash
# Create state_transitions.sh
#!/bin/bash

echo "Starting process in background..."
cat > /dev/null &
PID=$!
echo "PID: $PID"

for i in {1..10}; do
  STATE=$(ps -o stat= -p $PID 2>/dev/null)
  echo "State: $STATE"
  sleep 1
  
  if [ $i -eq 3 ]; then
    echo "Stopping process..."
    kill -STOP $PID
  fi
  
  if [ $i -eq 6 ]; then
    echo "Continuing process..."
    kill -CONT $PID
  fi
done

echo "Terminating process..."
kill $PID
sleep 1
echo "Final state (should be gone):"
ps -p $PID 2>&1

echo "Done!"
```

### Advanced Activity:

Monitor a process through ALL states:
```bash
# Terminal 1
sleep 100

# Terminal 2 - monitor
watch -n 0.5 'ps -o pid,stat,wchan,comm -p [PID]'

# Terminal 1 - Create state changes:
# 1. Ctrl+Z → see T state
# 2. fg → see S state
# 3. Ctrl+C → process exits
```

---

## 7. Pipeline and Process Groups

### What You Understood:
Processes in a pipeline share a PGID

### Master the Details:

**Pipeline = Process Group:**
```bash
ls -l | grep txt | wc -l
```

Creates:
- Process 1: ls (PID 1000, PGID 1000) ← Group leader
- Process 2: grep (PID 1001, PGID 1000)
- Process 3: wc (PID 1002, PGID 1000)

### Practice Exercise:

```bash
# Create pgid_test.sh
#!/bin/bash

echo "=== Creating Pipeline ==="
sleep 100 | sleep 100 | sleep 100 &

sleep 1

echo "=== Process Group Info ==="
ps -eo pid,pgid,stat,comm | grep sleep

PGID=$(pgrep -n sleep | xargs ps -o pgid= -p)
echo "Process Group ID: $PGID"

echo "=== All processes in group ==="
ps -eo pid,pgid,comm | awk -v pgid=$PGID '$2==pgid'

echo "=== Killing entire group ==="
kill -TERM -$PGID  # Note the NEGATIVE sign!

sleep 1
echo "=== Verify all gone ==="
ps -eo pid,pgid,comm | grep sleep || echo "All killed!"
```

### Critical Detail to Remember:

```bash
kill 12345      # Kills process with PID 12345
kill -12345     # Kills ALL processes in PGID 12345
     ^
     NEGATIVE sign = process group!
```

---

## 8. Parent Process Reaping Responsibilities

### Key Concept:

**Who reaps whom:**
- Bash (shell) → calls `wait()`, reaps children ✅
- `sleep` program → never calls `wait()`, doesn't reap ❌
- systemd → always calls `wait()`, reaps orphans ✅

### Practice Exercise:

```bash
# Test 1: Bash reaps properly
bash -c 'sleep 5 &'
# After 5s, no zombie (bash reaped it)

# Test 2: sleep doesn't reap
(sleep 5 & exec sleep 10)
# After 5s, zombie! (sleep doesn't reap)

# Test 3: systemd reaps orphans
sleep 5 &
CHILD=$!
exit  # Exit shell, orphan the child
# Systemd adopts and reaps when done
```

### Why This Matters for Containers:

```dockerfile
# BAD - Shell script as PID 1
FROM alpine
CMD ["sh", "-c", "app1 & app2 & wait"]
# Shell might not reap properly!

# GOOD - Direct execution
FROM alpine
CMD ["app"]
# app is PID 1, no intermediary

# BETTER - Use init
FROM alpine
RUN apk add tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["app"]
# tini is PID 1, reaps zombies properly
```

---

## Summary: Your Study Checklist

### Master These Commands:

```bash
# Process creation
fork()          # Creates new process
exec()          # Replaces current process
&              # Background job
()             # Subshell

# Process inspection
ps -eo pid,ppid,pgid,stat,comm
pstree -p -s [PID]
cat /proc/[PID]/status | grep NSpid

# Namespace investigation
sudo readlink /proc/[PID]/ns/pid
podman inspect [container] --format '{{.State.Pid}}'

# Process group management
kill -TERM -[PGID]    # Kill entire group (note negative!)
ps -eo pid,pgid,comm  # View process groups
```

### Practice Scenarios:

1. **Create zombies** 3 different ways and explain why each works
2. **Compare namespace IDs** for host vs container processes
3. **Trace a container's lineage** from systemd to container PID 1
4. **Create a pipeline** and kill it by process group
5. **Document state transitions** of a single process through all states

---

## Next Steps:

1. ✅ Complete all practice exercises in this document
2. ✅ Re-do any Day 1-2 exercises you struggled with
3. ✅ Create your own test scenarios for each concept
4. ✅ When confident, move to Day 3-4

**Remember:** These concepts are FUNDAMENTAL for containers. Master them now and everything else will be easier!

---

## Quick Reference Card (Print This!)

```
PROCESS CREATION:
  fork() → creates child (new PID)
  exec() → replaces current (same PID)
  
ZOMBIE RECIPE:
  (child & exec parent_that_doesnt_reap)
  
NAMESPACE PROOF:
  sudo cat /proc/[PID]/status | grep NSpid
  
PROCESS GROUP KILL:
  kill -TERM -[PGID]  ← NEGATIVE!
  
CONMON LOCATION:
  HOST namespace (not container)
  
SUBSHELL:
  () → creates new bash process
  & → backgrounds job (no subshell)
```

Good luck with your deeper study! 🎯