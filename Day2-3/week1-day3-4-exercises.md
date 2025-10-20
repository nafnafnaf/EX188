# Week 1, Day 3-4: Process Scheduling & Priorities - Hands-On Exercises

## Exercise 1: Nice Value Impact on CPU Distribution

### Objective
Understand how nice values affect CPU time allocation in CFS scheduler.

### Instructions

**Step 1: Create CPU-intensive processes with different nice values**

```bash
#!/bin/bash
# Create a script: cpu_nice_test.sh

echo "Starting CPU-intensive processes..."

# Process with nice 0 (default)
nice -n 0 sha256sum /dev/zero &
PID_NICE_0=$!
echo "Nice 0 PID: $PID_NICE_0"

# Process with nice 5
nice -n 5 sha256sum /dev/zero &
PID_NICE_5=$!
echo "Nice 5 PID: $PID_NICE_5"

# Process with nice 10
nice -n 10 sha256sum /dev/zero &
PID_NICE_10=$!
echo "Nice 10 PID: $PID_NICE_10"

# Process with nice 19 (lowest)
nice -n 19 sha256sum /dev/zero &
PID_NICE_19=$!
echo "Nice 19 PID: $PID_NICE_19"

echo "All processes started. Monitoring for 30 seconds..."
sleep 30

echo "Collecting CPU statistics..."
ps -o pid,ni,%cpu,comm -p $PID_NICE_0,$PID_NICE_5,$PID_NICE_10,$PID_NICE_19

echo "Killing all processes..."
kill $PID_NICE_0 $PID_NICE_5 $PID_NICE_10 $PID_NICE_19

echo "Done!"
```

**Step 2: Run the script**

```bash
chmod +x cpu_nice_test.sh
./cpu_nice_test.sh
```

**Step 3: While running, monitor in another terminal**

```bash
# Real-time monitoring
top
# Press 'P' to sort by CPU usage
# Press 'f' and enable 'NI' column

# Or use htop (if available)
htop
```

### Tasks

1. **Record CPU percentages** for each process after 30 seconds
2. **Create a table** showing nice value vs CPU percentage:
   ```
   Nice Value | CPU %
   -----------|-------
   0          | ?
   5          | ?
   10         | ?
   19         | ?
   ```
3. **Calculate ratios**: How much more CPU does nice 0 get compared to nice 10?

### Questions to Answer

1. Is the CPU distribution proportional to nice values?
2. What happens if you have only one CPU-intensive process? (Test with `nice -n 19`)
3. Change nice 19 to nice -20 (requires sudo). How does this change the distribution?
4. What happens if you run 10 processes all at nice 0?

### Expected Learning

- Nice values provide proportional CPU sharing
- Lower nice = more CPU time
- Fair distribution under contention
- Nice values only matter when CPU is contended

---

## Exercise 2: CPU Affinity and Performance

### Objective
Learn how CPU affinity affects process performance and cache locality.

### Instructions