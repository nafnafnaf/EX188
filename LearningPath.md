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

### Day 3-4: Process Scheduling & Priorities
- Linux scheduler types (CFS, Real-time)
- Nice values and priorities
- CPU affinity and NUMA awareness
- Process scheduling policies (SCHED_NORMAL, SCHED_FIFO, SCHED_RR)

### Day 5-6: The /proc Filesystem
- /proc structure and organization
- Reading process information from /proc/[pid]/
- Key files: status, cmdline, environ, fd/, maps, limits
- System-wide information in /proc

### Day 7: Signals & IPC
- Signal types and handling
- Inter-process communication mechanisms
- Pipes, message queues, shared memory
- Process synchronization

**Week 1 Assessment:** 20 questions + 5 hands-on exercises

---

## Week 2: Linux Namespaces
**Focus:** Deep dive into namespace isolation mechanisms

### Day 8-9: Namespace Fundamentals
- What are namespaces and why they exist
- The 7 namespace types overview
- Namespace lifecycle and management
- Tools: unshare, nsenter, lsns

### Day 10-11: PID, Mount & UTS Namespaces
- PID namespace: Process isolation and PID 1
- Mount namespace: Filesystem isolation
- UTS namespace: Hostname and domain isolation
- Practical examples and use cases

### Day 12-13: Network, IPC & User Namespaces
- Network namespace: Network stack isolation
- IPC namespace: Inter-process communication isolation
- User namespace: UID/GID mapping and security
- Rootless containers explained

### Day 14: Cgroup & Time Namespaces
- Cgroup namespace: Control group isolation
- Time namespace: System time virtualization
- Namespace nesting and hierarchies
- How Podman creates and manages namespaces

**Week 2 Assessment:** 25 questions + 7 hands-on exercises

---

## Week 3: Control Groups (cgroups)
**Focus:** Resource management and limitation

### Day 15-16: Cgroups Architecture
- Cgroups v1 vs v2 differences
- Cgroup filesystem structure
- Controllers and hierarchies
- Systemd's role in cgroup management

### Day 17-18: Resource Controllers - Part 1
- CPU controller: Shares, quotas, and periods
- Memory controller: Limits, reservations, OOM handling
- Understanding container resource constraints
- Practical resource limiting scenarios

### Day 19-20: Resource Controllers - Part 2
- I/O controller (blkio): Bandwidth and IOPS limits
- PIDs controller: Process number limits
- Device controller: Device access control
- CPUSET controller: CPU and memory node binding

### Day 21: Cgroups in Practice
- How Podman uses cgroups
- Systemd slice, scope, and service units
- Monitoring and debugging cgroup usage
- Performance implications and tuning

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

### Day 24-25: Rootless Containers Deep Dive
- User namespaces in depth
- UID/GID mapping strategies
- Networking in rootless mode (slirp4netns, pasta)
- Storage considerations (fuse-overlayfs)
- Limitations and workarounds

### Day 26-27: Performance & Troubleshooting
- Container performance analysis
- Resource contention debugging
- Common issues and solutions
- Logging and monitoring best practices
- Network troubleshooting in containers

### Day 28: Exam Preparation & Practice Scenarios
- EX188 exam format and objectives review
- Complex multi-container scenarios
- Troubleshooting exercises
- Time-based practice challenges
- Review of weak areas

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