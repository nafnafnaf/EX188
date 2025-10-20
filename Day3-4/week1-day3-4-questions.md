# Week 1, Day 3-4: Process Scheduling & Priorities - Self-Assessment Questions

## Multiple Choice Questions

### Q1. What is the default scheduler for normal processes in Linux?
- a) Round-Robin (RR)
- b) First-In-First-Out (FIFO)
- c) Completely Fair Scheduler (CFS)
- d) Deadline scheduler

---

### Q2. Which nice value gives a process the HIGHEST priority in CFS?
- a) -20
- b) 0
- c) 19
- d) 99

---

### Q3. What happens when a real-time SCHED_FIFO process becomes runnable?
- a) It waits for the current process to finish its time slice
- b) It preempts lower-priority processes immediately
- c) It joins the CFS runqueue
- d) It runs after all SCHED_RR processes

---

### Q4. What is the main difference between SCHED_FIFO and SCHED_RR?
- a) FIFO has higher priority than RR
- b) RR uses time slicing, FIFO does not
- c) FIFO is for normal processes, RR is for real-time
- d) There is no difference

---

### Q5. Which command sets CPU affinity for a running process?
- a) `nice -n 5 -p [PID]`
- b) `chrt -f 50 [PID]`
- c) `taskset -cp 0-3 [PID]`
- d) `renice -n 10 [PID]`

---

## Practical Questions

### Q6. Nice Value Experimentation
Run two CPU-intensive processes with different nice values and observe CPU allocation:

```bash
# Terminal 1
nice -n 0 yes > /dev/null &
PID1=$!
echo "Process 1 PID: $PID1"

# Terminal 2
nice -n 10 yes > /dev/null &
PID2=$!
echo "Process 2 PID: $PID2"

# Terminal 3
top -p $PID1,$PID2
# Or: ps -eo pid,ni,%cpu,comm | grep yes
```

**Questions:**
- What percentage of CPU does each process get?
- How does this relate to their nice values?
- What happens if you change nice 10 to nice 19?

**Cleanup:**
```bash
kill $PID1 $PID2
```

---

### Q7. Viewing Scheduling Information
Examine scheduling information for your shell:

```bash
# Get your shell's PID
echo $$

# View scheduling policy
chrt -p $$
ps -o pid,class,rtprio,ni,comm -p $$

# View detailed scheduling stats
cat /proc/$$/sched | head -20
```

**Questions:**
- What scheduling policy does your shell use?
- What is its nice value?
- What is the `nr_voluntary_switches` value? What does this mean?

---

### Q8. CPU Affinity Testing
Test CPU affinity by binding a process to specific cores:

```bash
# Start a CPU-intensive process on all CPUs
yes > /dev/null &
PID=$!

# Check which CPU it's running on (watch it migrate)
ps -o pid,psr,comm -p $PID
# Or: taskset -cp $PID

# Bind to CPU 0 only
taskset -cp 0 $PID

# Verify it stays on CPU 0
watch -n 1 "ps -o pid,psr,comm -p $PID"

# Cleanup
kill $PID
```

**Questions:**
- Before binding, does the process migrate between CPUs?
- After binding to CPU 0, does it stay on CPU 0?
- What is the PSR field showing?

---

### Q9. Container CPU Limits
Experiment with container CPU limits:

```bash
# Run container with 50% CPU limit
podman run -d --name cpu-limited --cpu-quota=50000 --cpu-period=100000 alpine sh -c "while true; do :; done"

# Run container without limits
podman run -d --name cpu-unlimited alpine sh -c "while true; do :; done"

# Monitor CPU usage
podman stats

# Check from host
ps aux | grep "while true"
top
```

**Questions:**
- Does the limited container stay at ~50% CPU?
- What happens if you run multiple unlimited containers?
- How does `--cpu-shares` differ from `--cpu-quota`?

**Cleanup:**
```bash
podman stop cpu-limited cpu-unlimited
podman rm cpu-limited cpu-unlimited
```

---

### Q10. Real-Time Priority (Requires root)
Experiment with real-time priorities:

```bash
# Start normal process
sleep 100 &
NORMAL_PID=$!

# Start RT process (requires root)
sudo chrt -f 50 sleep 100 &
RT_PID=$!

# Compare scheduling
ps -o pid,class,rtprio,ni,comm -p $NORMAL_PID,$RT_PID
chrt -p $NORMAL_PID
chrt -p $RT_PID
```

**Questions:**
- What is the class (CLS) for each process?
- What happens if you try to set RT priority without sudo?
- Why is RT priority potentially dangerous?

**Cleanup:**
```bash
kill $NORMAL_PID
sudo kill $RT_PID
```

---

## Advanced Questions

### Q11. CFS Behavior Analysis
Research and explain:

Using the `/proc/[PID]/sched` file, explain what these fields mean:
```bash
cat /proc/$$/sched
```

**Questions:**
- What is `se.vruntime` and why is it important?
- What do `nr_voluntary_switches` and `nr_involuntary_switches` indicate?
- How does `prio` relate to nice value?

---

### Q12. NUMA Architecture
If your system has NUMA (check with `numactl --hardware`):

```bash
# Show NUMA topology
numactl --hardware

# Run process on node 0
numactl --cpunodebind=0 --membind=0 sleep 100 &
PID=$!

# Check NUMA allocation
numastat -p $PID

# Cleanup
kill $PID
```

**Questions:**
- How many NUMA nodes does your system have?
- Why would you bind a process to a specific NUMA node?
- What's the performance impact of remote memory access?

**If no NUMA:**
- Research and explain when NUMA becomes important
- Describe the difference between local and remote memory access

---

### Q13. RT Throttling
Examine and explain RT throttling:

```bash
# View current RT throttling settings
cat /proc/sys/kernel/sched_rt_runtime_us
cat /proc/sys/kernel/sched_rt_period_us

# Calculate percentage
# (sched_rt_runtime_us / sched_rt_period_us) * 100
```

**Questions:**
- What percentage of CPU time is reserved for RT tasks?
- Why does Linux reserve some CPU time for non-RT tasks?
- What happens if RT tasks try to use more than their allocation?
- How would you disable RT throttling (and why might this be dangerous)?

---

### Q14. Scheduler Class Hierarchy
Create a scenario with processes at different scheduler levels:

```bash
# Normal process (CFS)
nice -n 0 sleep 100 &
CFS_PID=$!

# Batch process (CFS, lower priority)
nice -n 19 sleep 100 &
BATCH_PID=$!

# Real-time RR (requires root)
sudo chrt -r 50 sleep 100 &
RR_PID=$!

# Real-time FIFO (requires root)
sudo chrt -f 50 sleep 100 &
FIFO_PID=$!

# Compare all
ps -o pid,class,rtprio,ni,pri,comm -p $CFS_PID,$BATCH_PID,$RR_PID,$FIFO_PID
```

**Questions:**
- Rank these processes by priority (highest to lowest)
- What does the PRI column represent?
- If all were CPU-bound, which would get CPU time first?
- How does RT priority 1 compare to nice -20?

**Cleanup:**
```bash
kill $CFS_PID $BATCH_PID
sudo kill $RR_PID $FIFO_PID
```

---

### Q15. Container Scheduling Deep Dive
Research and explain:

```bash
# Start a container
podman run -d --name test --cpus=2 --cpu-shares=512 nginx

# Find the container's cgroup
podman inspect test --format '{{.HostConfig.CgroupParent}}'

# Examine CPU settings
cat /sys/fs/cgroup/cpu/[cgroup_path]/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/[cgroup_path]/cpu.cfs_period_us
cat /sys/fs/cgroup/cpu/[cgroup_path]/cpu.shares
```

**Questions:**
- How does `--cpus=2` translate to cgroup settings?
- What is the relationship between `cpu.shares` and nice values?
- How does CFS bandwidth control enforce CPU limits?
- What happens when multiple containers compete for CPU?

**Cleanup:**
```bash
podman stop test
podman rm test
```

---

## Answer Guidelines

**For Multiple Choice Questions (Q1-Q5):**
- Choose the single best answer
- Verify by testing when possible
- Research documentation if unsure

**For Practical Questions (Q6-Q10):**
- Run all commands on your system
- Document outputs and observations
- Note any unexpected behavior
- Explain what you discovered

**For Advanced Questions (Q11-Q15):**
- Research using man pages and documentation
- Provide detailed technical explanations
- Include command outputs to support answers
- Consider real-world applications

---

## Scoring Guide

- **Q1-Q5 (Multiple Choice):** 1 point each = 5 points
- **Q6-Q10 (Practical):** 3 points each = 15 points
- **Q11-Q15 (Advanced):** 4 points each = 20 points

**Total:** 40 points
**Passing Score:** 28/40 (70%)

---

## Additional Research Topics

If you finish early, explore these topics:
- Difference between `nice` and `ionice` (I/O scheduling)
- CPU frequency scaling and governors
- Scheduler statistics in `/proc/schedstat`
- cgroup v2 CPU controller improvements
- Real-time kernel patches (PREEMPT_RT)

---

## Next Steps

After completing these questions:
1. Review any incorrect answers
2. Re-read relevant theory sections
3. Complete the hands-on exercises
4. Experiment beyond the required questions
5. When ready, proceed to Day 5-6: The /proc Filesystem