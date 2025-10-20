# Week 1, Day 3-4: Process Scheduling & Priorities - Theory

## Introduction to Process Scheduling

The **Linux scheduler** is the kernel component responsible for deciding which process runs on which CPU core and for how long. This is critical for system performance, responsiveness, and fairness.

### Why Scheduling Matters for Containers

Understanding scheduling is essential for:
- **Resource allocation**: Ensuring containers get their fair share of CPU
- **Performance tuning**: Optimizing container workloads
- **Priority management**: Critical containers get more CPU time
- **Real-time requirements**: Meeting timing guarantees for time-sensitive applications
- **Multi-tenant systems**: Fair resource distribution among containers

---

## Linux Scheduler Types

Linux uses different schedulers for different types of workloads:

### 1. CFS (Completely Fair Scheduler)

**CFS** is the default scheduler for normal processes in Linux.

#### Key Concepts:

**Virtual Runtime (vruntime):**
- Each process tracks how much CPU time it has consumed
- Normalized by priority (nice value)
- Processes with lower vruntime run first
- Goal: Keep all processes' vruntime roughly equal

**Red-Black Tree:**
- CFS organizes processes in a red-black tree
- Sorted by vruntime
- Leftmost node = process that has run least → runs next
- O(log n) complexity for scheduling decisions

**Time Slices:**
- CFS doesn't use fixed time slices
- Instead uses a "targeted latency" (typically 6-20ms)
- Time slice = targeted latency / number of runnable processes
- Minimum granularity ensures processes don't get too little time

#### How CFS Works:

```
1. Process becomes runnable → added to red-black tree
2. Scheduler picks leftmost node (lowest vruntime)
3. Process runs for its calculated time slice
4. vruntime increased based on execution time and priority
5. Process reinserted into tree
6. Repeat
```

#### CFS Characteristics:

✅ **Fair**: All processes get proportional CPU time
✅ **Scalable**: Works well with many processes
✅ **Interactive**: Good desktop responsiveness
✅ **Default**: Used for most workloads
❌ **Not deterministic**: No guaranteed timing
❌ **Not suitable for real-time**: Latency not bounded

---

### 2. Real-Time Schedulers

Linux provides two real-time scheduling classes for time-critical applications:

#### RT Scheduler Overview:

- **Higher priority than CFS**: RT processes preempt normal processes
- **Deterministic**: Predictable latency
- **Priority-based**: Higher priority always runs first
- **Used for**: Audio/video processing, industrial control, robotics

---

#### SCHED_FIFO (First-In, First-Out)

**Characteristics:**
- Process runs until it blocks, yields, or is preempted by higher priority
- **No time slicing** within same priority
- If multiple processes at same priority, runs first one that became ready
- Continues until completion or blocked

**Use cases:**
- Long-running time-critical tasks
- Tasks that need exclusive CPU until completion
- Predictable execution time

**Danger:**
- Can monopolize CPU
- Must be carefully designed to yield/block
- Can lock up system if buggy

```bash
# Example behavior
Process A (RT priority 50, SCHED_FIFO): Runs indefinitely
Process B (RT priority 50, SCHED_FIFO): Waits until A blocks
Process C (RT priority 49): Never runs while A or B are ready
```

---

#### SCHED_RR (Round-Robin)

**Characteristics:**
- Like SCHED_FIFO but with **time slicing**
- Each process gets a time quantum (default 100ms)
- After time quantum expires, moves to end of queue
- Processes at same priority share CPU fairly

**Use cases:**
- Multiple time-critical tasks of equal importance
- When fairness among RT tasks is needed
- Preventing RT task monopolization

**Advantages over FIFO:**
- Prevents CPU hogging
- Fairer among equal-priority tasks
- Safer for multi-tenant environments

```bash
# Example behavior
Process A (RT priority 50, SCHED_RR): Runs 100ms
Process B (RT priority 50, SCHED_RR): Runs 100ms
Process C (RT priority 50, SCHED_RR): Runs 100ms
# Cycle repeats
```

---

### 3. SCHED_DEADLINE (Deadline Scheduler)

**Most advanced real-time scheduler** (added in kernel 3.14):

**Characteristics:**
- Based on Earliest Deadline First (EDF) algorithm
- Tasks specify: runtime, deadline, period
- Kernel guarantees tasks meet deadlines if schedulable
- Highest priority class (preempts FIFO/RR)

**Use cases:**
- Hard real-time systems
- Cyclic tasks with strict timing requirements
- Multimedia streaming
- Industrial automation

**Not commonly used in containers** but important to know about.

---

### Scheduler Class Hierarchy

```
Priority Order (highest to lowest):
1. SCHED_DEADLINE  ← Deadline tasks
2. SCHED_FIFO      ← RT FIFO tasks (priority 1-99)
3. SCHED_RR        ← RT Round-robin (priority 1-99)
4. SCHED_NORMAL    ← CFS (nice -20 to +19)
5. SCHED_BATCH     ← CFS batch processing
6. SCHED_IDLE      ← CFS lowest priority
```

**Rule:** Higher scheduler class always preempts lower class.

---

## Process Priorities

### Nice Values (CFS Scheduling)

**Nice values** control process priority in CFS scheduler:

**Range:** -20 (highest priority) to +19 (lowest priority)
**Default:** 0 (normal priority)

#### How Nice Values Work:

- **Lower nice value** = higher priority = more CPU time
- **Higher nice value** = lower priority = less CPU time
- Each nice level = ~10% difference in CPU allocation

**Weight calculation:**
```
Weight = 1024 / (1.25 ^ nice_value)

nice -20: Weight ≈ 88761  (highest)
nice   0: Weight = 1024   (default)
nice +19: Weight ≈ 15     (lowest)
```

**CPU time distribution:**
```
Process A (nice 0):  Gets 50% CPU  (1024 / 2048)
Process B (nice 0):  Gets 50% CPU  (1024 / 2048)

Process A (nice 0):  Gets 91% CPU  (1024 / 1124)
Process B (nice 10): Gets 9% CPU   (100 / 1124)
```

#### Setting Nice Values:

```bash
# Start process with specific nice value
nice -n 10 ./myapp

# Change nice value of running process
renice -n 5 -p [PID]

# View nice values
ps -eo pid,ni,comm
top  # Press 'r' to renice
```

#### Permissions:

- **Regular users**: Can only increase nice value (lower priority)
- **Root**: Can set any nice value
- **Containers**: Nice values relative to host scheduler

---

### Real-Time Priorities

**RT priority range:** 1 (lowest) to 99 (highest)

**Important:** RT priority 1 still preempts all CFS processes!

```
RT Priority 99 > RT Priority 50 > RT Priority 1 > CFS nice -20
```

#### Setting RT Priorities:

```bash
# Set SCHED_FIFO with priority 50
chrt -f 50 ./myapp

# Set SCHED_RR with priority 30
chrt -r 30 ./myapp

# View scheduling policy and priority
chrt -p [PID]
ps -eo pid,class,rtprio,comm
```

#### RT Throttling:

Linux implements **RT throttling** to prevent RT tasks from starving the system:

- By default: RT tasks can use 95% of CPU
- Remaining 5% reserved for non-RT tasks
- Configurable via `/proc/sys/kernel/sched_rt_runtime_us`

```bash
# View RT throttling settings
cat /proc/sys/kernel/sched_rt_runtime_us    # 950000 (95%)
cat /proc/sys/kernel/sched_rt_period_us     # 1000000 (1 second)
```

---

## CPU Affinity

**CPU affinity** binds processes to specific CPU cores.

### Why Use CPU Affinity?

**Benefits:**
- **Cache locality**: Process data stays in CPU cache
- **NUMA optimization**: Process runs on CPU near its memory
- **Predictable performance**: Avoid migration overhead
- **Isolation**: Dedicate cores to specific workloads

**Use cases:**
- High-performance computing
- Real-time applications
- Container resource management
- Database servers

### Setting CPU Affinity:

```bash
# Run process on CPU 0 only
taskset -c 0 ./myapp

# Run on CPUs 0,1,2
taskset -c 0-2 ./myapp

# Run on CPUs 0 and 3
taskset -c 0,3 ./myapp

# Set affinity of running process
taskset -cp 0-3 [PID]

# View current affinity
taskset -cp [PID]
```

### Affinity Mask:

Affinity represented as bitmask:
```
CPU 0 = bit 0 = 0x1
CPU 1 = bit 1 = 0x2
CPU 2 = bit 2 = 0x4
CPU 3 = bit 3 = 0x8

CPUs 0,2 = 0x1 | 0x4 = 0x5 = binary 0101
```

### In Containers:

Podman supports CPU affinity:
```bash
# Run container on CPUs 0-3
podman run --cpuset-cpus="0-3" myimage

# View container's CPU affinity
podman inspect [container] --format '{{.HostConfig.CpusetCpus}}'
```

---

## NUMA (Non-Uniform Memory Access)

### What is NUMA?

Modern multi-socket systems have NUMA architecture:

```
Socket 0 (CPUs 0-15)    Socket 1 (CPUs 16-31)
    |                           |
Local Memory (Node 0)   Local Memory (Node 1)
    |                           |
    +--------Interconnect-------+
```

**Key concept:** Memory access speed depends on locality:
- **Local memory**: Fast (same socket as CPU)
- **Remote memory**: Slower (different socket, via interconnect)

### NUMA Impact:

- **2-3x slower** remote memory access
- Critical for performance on large systems
- Linux tries to allocate memory on local node
- Processes should run on CPUs near their memory

### NUMA Tools:

```bash
# Show NUMA topology
numactl --hardware

# View NUMA statistics
numastat

# Run process on specific NUMA node
numactl --cpunodebind=0 --membind=0 ./myapp

# Show process NUMA allocation
numastat -p [PID]
```

### NUMA in Containers:

```bash
# Bind container to NUMA node 0
podman run --cpuset-cpus="0-15" --cpuset-mems="0" myimage

# Check NUMA policy in container
podman exec mycontainer numactl --show
```

### NUMA Policies:

1. **Default**: Allocate memory on local node
2. **Bind**: Restrict to specific nodes
3. **Preferred**: Prefer specific node but allow others
4. **Interleave**: Spread memory across nodes

---

## Scheduling Policies Summary

### SCHED_NORMAL (CFS)
- **Default policy** for most processes
- Fair CPU time distribution
- Nice values (-20 to +19) control priority
- Good for general-purpose workloads
- **Used by**: Most containers, user applications

### SCHED_BATCH (CFS)
- For CPU-intensive batch jobs
- Slight optimization for throughput over latency
- Still uses nice values
- **Used by**: Background processing, batch jobs

### SCHED_IDLE (CFS)
- Lowest priority possible
- Only runs when no other process is runnable
- Even nice +19 has higher priority
- **Used by**: Screen savers, non-critical background tasks

### SCHED_FIFO (Real-Time)
- Fixed priority (1-99)
- No time slicing
- Runs until blocks or preempted
- **Used by**: Long-running RT tasks

### SCHED_RR (Real-Time)
- Fixed priority (1-99)
- Time slicing (round-robin)
- Fairer than FIFO at same priority
- **Used by**: Multiple RT tasks of equal importance

### SCHED_DEADLINE (Real-Time)
- Earliest Deadline First (EDF)
- Highest priority class
- Guaranteed deadline meeting
- **Used by**: Hard real-time systems

---

## Container Scheduling Considerations

### Podman/Docker CPU Limits:

```bash
# Limit to 2 CPUs
podman run --cpus=2 myimage

# CPU shares (relative weight, like nice)
podman run --cpu-shares=512 myimage  # Half default weight

# CPU quota (% of CPU time)
podman run --cpu-quota=50000 --cpu-period=100000 myimage  # 50% CPU

# Pin to specific CPUs
podman run --cpuset-cpus="0-3" myimage
```

### How Container CPU Limits Work:

1. **cgroups CPU controller**: Enforces limits
2. **CFS bandwidth control**: Throttles CPU usage
3. **CPU shares**: Proportional allocation when contention
4. **CPU affinity**: Physical core binding

**Note:** Containers use **CFS (SCHED_NORMAL)** by default. Real-time scheduling requires special configuration and privileges.

---

## Scheduler Tuning Parameters

### CFS Tuning:

```bash
# Minimum time slice (granularity)
cat /proc/sys/kernel/sched_min_granularity_ns
# Default: 3000000 (3ms)

# Target latency (time for all tasks to run once)
cat /proc/sys/kernel/sched_latency_ns
# Default: 24000000 (24ms)

# Wakeup granularity (preemption threshold)
cat /proc/sys/kernel/sched_wakeup_granularity_ns
# Default: 4000000 (4ms)
```

### RT Tuning:

```bash
# RT runtime throttling
cat /proc/sys/kernel/sched_rt_runtime_us
# Default: 950000 (95% of CPU for RT tasks)

cat /proc/sys/kernel/sched_rt_period_us
# Default: 1000000 (1 second period)
```

---

## Viewing Scheduling Information

### Commands to Check Scheduling:

```bash
# View scheduling policy and priority
ps -eo pid,class,rtprio,ni,comm
chrt -p [PID]

# CPU affinity
taskset -cp [PID]

# Detailed scheduling stats
cat /proc/[PID]/sched
cat /proc/[PID]/status | grep -E "voluntary|nonvoluntary"

# Real-time view
top
htop  # Press F2 → Columns → Add PRIORITY, NICE

# System-wide scheduler stats
cat /proc/schedstat
```

### Interpreting Output:

```bash
$ ps -eo pid,class,rtprio,ni,comm | head
  PID CLS RTPRIO  NI COMMAND
    1 TS       -   0 systemd
  100 FF      99   - migration/0
  500 TS       -   0 bash
 1000 TS       -  10 low_priority_task
 2000 RR      50   - realtime_task
```

- **CLS**: Scheduling class (TS=SCHED_NORMAL, FF=FIFO, RR=RR)
- **RTPRIO**: Real-time priority (1-99, or - for non-RT)
- **NI**: Nice value (-20 to +19, or - for RT)

---

## Key Takeaways

✅ **CFS (Completely Fair Scheduler)** is default for normal processes
✅ **Nice values** (-20 to +19) control CFS priority
✅ **Real-time schedulers** (FIFO, RR) provide deterministic latency
✅ **RT priorities** (1-99) preempt all normal processes
✅ **CPU affinity** binds processes to specific cores for performance
✅ **NUMA awareness** is critical on multi-socket systems
✅ **Containers use CFS** with cgroups for resource management
✅ **Understanding scheduling** is essential for container performance tuning

---

## Next Steps

After understanding the theory:
1. Answer the self-assessment questions
2. Experiment with nice values and priorities
3. Practice setting CPU affinity
4. Explore container CPU limits
5. Monitor scheduling behavior in real scenarios

**Tomorrow:** The /proc Filesystem (Day 5-6)