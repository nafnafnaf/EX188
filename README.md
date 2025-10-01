# Linux Containers Deep Dive - Learning Path Schedule

## Course Overview
**Duration:** 4 Weeks (October 2025)  
**Goal:** Master Linux namespaces, cgroups, and processes for EX188 exam preparation  
**Target Exam Date:** Before Christmas 2025  
**Study Approach:** Theory + Self-Assessment Questions + Hands-on Labs

---

## Week 1: Linux Processes & Process Management
**Focus:** Understanding the foundation of process management in Linux

### Day 1-2: Process Fundamentals
- What are processes and threads?
- Process lifecycle (fork, exec, wait, exit)
- Process states (Running, Sleeping, Stopped, Zombie)
- Parent-child process relationships
- Process IDs (PID, PPID, PGID, SID)

**Materials:**
- [Theory](./Day1-2/theory.md)
- [Questions](./Day1-2/questions.md)
- [Hands-On Exercises](./Day1-2/exercises.md)

---

### Day 3-4: Process Scheduling & Priorities
- Linux scheduler types (CFS, Real-time)
- Nice values and priorities
- CPU affinity and NUMA awareness
- Process scheduling policies (SCHED_NORMAL, SCHED_FIFO, SCHED_RR)

**Materials:**
- [Theory](./Day3-4/theory.md)
- [Questions](./Day3-4/questions.md)
- [Hands-On Exercises](./Day3-4/exercises.md)

---

### Day 5-6: The /proc Filesystem
- /proc structure and organization
- Reading process information from /proc/[pid]/
- Key files: status, cmdline, environ, fd/, maps, limits
- System-wide information in /proc

**Materials:**
- [Theory](./Day5-6/theory.md)
- [Questions](./Day5-6/questions.md)
- [Hands-On Exercises](./Day5-6/exercises.md)

---

### Day 7: Signals & IPC
- Signal types and handling
- Inter-process communication mechanisms
- Pipes, message queues, shared memory
- Process synchronization

**Materials:**
- [Theory](./Day7/theory.md)
- [Questions](./Day7/questions.md)
- [Hands-On Exercises](./Day7/exercises.md)

**Week 1 Assessment:** 20 questions + 5 hands-on exercises

---

## Week 2: Linux Namespaces
**Focus:** Deep dive into namespace isolation mechanisms

### Day 8-9: Namespace Fundamentals
- What are namespaces and why they exist
- The 7 namespace types overview
- Namespace lifecycle and management
- Tools: unshare, nsenter, lsns
- Introduction to podman unshare vs system unshare

**Materials:**
- [Theory](./Day8-9/theory.md)
- [Questions](./Day8-9/questions.md)
- [Hands-On Exercises](./Day8-9/exercises.md)

---

### Day 10-11: PID, Mount & UTS Namespaces
- PID namespace: Process isolation and PID 1
- Mount namespace: Filesystem isolation
- UTS namespace: Hostname and domain isolation
- Practical examples and use cases

**Materials:**
- [Theory](./Day10-11/theory.md)
- [Questions](./Day10-11/questions.md)
- [Hands-On Exercises](./Day10-11/exercises.md)

---

### Day 12-13: Network, IPC & User Namespaces
- Network namespace: Network stack isolation
- IPC namespace: Inter-process communication isolation
- User namespace: UID/GID mapping and security
- **podman unshare basics: Entering rootless user namespace**
- Rootless containers explained

**Materials:**
- [Theory](./Day12-13/theory.md)
- [Questions](./Day12-13/questions.md)
- [Hands-On Exercises](./Day12-13/exercises.md)

---

### Day 14: Cgroup & Time Namespaces
- Cgroup namespace: Control group isolation
- Time namespace: System time virtualization
- Namespace nesting and hierarchies
- How Podman creates and manages namespaces

**Materials:**
- [Theory](./Day14/theory.md)
- [Questions](./Day14/questions.md)
- [Hands-On Exercises](./Day14/exercises.md)

**Week 2 Assessment:** 25 questions + 7 hands-on exercises

---

## Week 3: Control Groups (cgroups)
**Focus:** Resource management and limitation

### Day 15-16: Cgroups Architecture
- Cgroups v1 vs v2 differences
- Cgroup filesystem structure
- Controllers and hierarchies
- Systemd's role in cgroup management

**Materials:**
- [Theory](./Day15-16/theory.md)
- [Questions](./Day15-16/questions.md)
- [Hands-On Exercises](./Day15-16/exercises.md)

---

### Day 17-18: Resource Controllers - Part 1
- CPU controller: Shares, quotas, and periods
- Memory controller: Limits, reservations, OOM handling
- Understanding container resource constraints
- Practical resource limiting scenarios

**Materials:**
- [Theory](./Day17-18/theory.md)
- [Questions](./Day17-18/questions.md)
- [Hands-On Exercises](./Day17-18/exercises.md)

---

### Day 19-20: Resource Controllers - Part 2
- I/O controller (blkio): Bandwidth and IOPS limits
- PIDs controller: Process number limits
- Device controller: Device access control
- CPUSET controller: CPU and memory node binding

**Materials:**
- [Theory](./Day19-20/theory.md)
- [Questions](./Day19-20/questions.md)
- [Hands-On Exercises](./Day19-20/exercises.md)

---

### Day 21: Cgroups in Practice
- How Podman uses cgroups
- Systemd slice, scope, and service units
- Monitoring and debugging cgroup usage
- Performance implications and tuning

**Materials:**
- [Theory](./Day21/theory.md)
- [Questions](./Day21/questions.md)
- [Hands-On Exercises](./Day21/exercises.md)

**Week 3 Assessment:** 25 questions + 8 hands-on exercises

---

## Week 4: Integration & Advanced Topics
**Focus:** Putting it all together for real-world scenarios

### Day 22-23: Security Contexts
- SELinux fundamentals for containers
- SELinux policies and labels
- AppArmor profiles
- Capabilities and seccomp filters
- Security in rootless vs rootful containers

**Materials:**
- [Theory](./Day22-23/theory.md)
- [Questions](./Day22-23/questions.md)
- [Hands-On Exercises](./Day22-23/exercises.md)

---

### Day 24-25: Rootless Containers Deep Dive
- User namespaces in depth
- UID/GID mapping strategies
- **podman unshare deep dive:**
  - Accessing rootless container storage
  - Working with UID/GID mappings in practice
  - Troubleshooting storage permissions
  - Modifying container filesystems as root
  - Common exam scenarios with podman unshare
- Networking in rootless mode (slirp4netns, pasta)
- Storage considerations (fuse-overlayfs)
- Limitations and workarounds

**Materials:**
- [Theory](./Day24-25/theory.md)
- [Questions](./Day24-25/questions.md)
- [Hands-On Exercises](./Day24-25/exercises.md)

---

### Day 26-27: Performance & Troubleshooting
- Container performance analysis
- Resource contention debugging
- Common issues and solutions
- Logging and monitoring best practices
- Network troubleshooting in containers

**Materials:**
- [Theory](./Day26-27/theory.md)
- [Questions](./Day26-27/questions.md)
- [Hands-On Exercises](./Day26-27/exercises.md)

---

### Day 28: Exam Preparation & Practice Scenarios
- EX188 exam format and objectives review
- Complex multi-container scenarios
- Troubleshooting exercises
- Time-based practice challenges
- Review of weak areas

**Materials:**
- [Theory](./Day28/theory.md)
- [Questions](./Day28/questions.md)
- [Hands-On Exercises](./Day28/exercises.md)

**Week 4 Assessment:** 30 questions + 10 comprehensive hands-on scenarios

---

## Study Recommendations

### Daily Commitment
- **Weekdays:** 2-3 hours (theory + practice)
- **Weekends:** 3-4 hours (hands-on labs + review)

### Practice Environment
- RHEL 8/9 or Rocky Linux system
- Podman installed
- Root and non-root user access
- Minimum 2 CPU cores, 4GB RAM

### Study Methods
1. Read theory section thoroughly
2. Take notes on key concepts
3. Answer self-assessment questions
4. Perform hands-on exercises
5. Research topics you find challenging
6. Review weekly summaries

### Additional Resources
- Red Hat EX188 exam objectives
- Podman official documentation
- Linux man pages (namespaces(7), cgroups(7), etc.)
- Kernel documentation on namespaces and cgroups

---

## File Structure

```
linux-containers-course/
├── README.md (this file)
├── Day1-2/
│   ├── theory.md
│   ├── questions.md
│   └── exercises.md
├── Day3-4/
│   ├── theory.md
│   ├── questions.md
│   └── exercises.md
├── Day5-6/
│   ├── theory.md
│   ├── questions.md
│   └── exercises.md
├── Day7/
│   ├── theory.md
│   ├── questions.md
│   └── exercises.md
├── Day8-9/
│   ├── theory.md
│   ├── questions.md
│   └── exercises.md
[... continues for all days through Day28 ...]
```

---

## Weekly Milestones

- **End of Week 1:** Understand process management and /proc filesystem
- **End of Week 2:** Can manually create and manage namespaces
- **End of Week 3:** Can configure and monitor cgroup resources
- **End of Week 4:** Ready for exam-level scenarios

---

## Final Preparation (Week 5 - Early December)
- Review all weekly assessments
- Focus on weak areas
- Take full practice exams
- Hands-on troubleshooting marathon
- Final confidence check before scheduling exam

**Good luck with your EX188 preparation! 🚀**