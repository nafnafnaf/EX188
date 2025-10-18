# Week 1, Day 1-2: Process Fundamentals

## Theory Section
## Key Takeaways

✅ Processes are isolated instances of running programs with unique PIDs

✅ fork() creates processes, exec() loads new programs

✅ Process states (R, S, D, T, Z) indicate what a process is doing

✅ Parent-child relationships form a tree rooted at PID 1

✅ Zombie processes occur when parents don't call wait()

✅ Container PID 1 must properly handle child processes

✅ PIDs are namespace-specific - same process has different PIDs in different namespaces
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

# 🧬 Every Process Has a Parent — But Not Always Bash

In Linux, every process (except the first one) is a child of another process. But that parent could be:

* a terminal shell (like bash, zsh, fish),
* a system daemon,
* a graphical session process, or
* systemd/init directly.

## 🌱 The Root of It All: `init` / `systemd` (PID 1)

When Linux boots, the very first user-space process is:

```
PID 1 → /sbin/init   (or systemd on most modern distros)
```

Everything else in the system eventually descends from PID 1.

Example of a simple hierarchy:

```
systemd (PID 1)
 ├─ gdm
 │   ├─ gnome-session
 │   │   ├─ gnome-terminal
 │   │   │   └─ bash
 │   │   │       └─ python3
 │   │   └─ firefox
 └─ cron
```

Notice how Firefox and cron jobs didn't come from bash, but they still trace back to PID 1.

## ✅ When ARE Processes Children of Bash?

Only when you start them from inside a shell session, e.g.:

```bash
$ bash
$ nano myfile.txt     ← child of bash
$ python3 app.py      ← child of bash
```

Or if you do something like:

```
bash -> ssh -> bash -> podman -> conmon -> your_container_process
```

Then yes, those are in your shell's lineage.

## ❌ When Processes Are NOT Children of Bash

Lots of situations:

### 1. Services and Background Daemons

```
systemd → NetworkManager → dhclient
systemd → sshd → user session
```

### 2. GUI Apps Launched from Menus

Clicking Firefox from GNOME:

```
systemd → gnome-shell → firefox
```

No bash involved.

### 3. Cron/Systemd Timers

```
systemd → systemd-timer → backup.sh
```

Not tied to any terminal.

### 4. Login Shells for Non-Interactive Sessions

e.g. SSH keys, scripts, containers.

## ✅ So the Corrected Statement:

* All processes have a parent, but
* Only some processes are children of a Bash terminal — mainly the ones you launch manually there.

If you run this:

```bash
ps -ef --forest
```

You'll see the full "family tree" and be able to spot which ones descend from bash vs systemd vs GNOME, etc.

---

## 🔧 About Shebangs in Startup Scripts

Ah yes, the shebang connection! Here's the key insight:

**The shebang (`#!/bin/bash`) tells the kernel *which interpreter to use*, but that interpreter becomes a child of *whatever called the script*, not some pre-existing bash shell.**

### Example 1: Systemd Service Script

```bash
#!/bin/bash
# /usr/local/bin/myapp-start.sh
echo "Starting myapp..."
exec /opt/myapp/server
```

When systemd runs this:

```
systemd → bash (running myapp-start.sh) → server
```

The bash process is a *child of systemd*, not a child of any interactive bash shell.

### Example 2: Script Run from Your Terminal

```bash
$ ./myapp-start.sh
```

Now the hierarchy is:

```
gnome-terminal → bash (your interactive shell) → bash (running myapp-start.sh) → server
```

Same script, different parent!

### The Shebang's Job

The shebang is just instructions for the kernel:

> "Hey kernel, when someone tries to execute this file, please run `/bin/bash` and pass this file as an argument."

It's equivalent to:

```bash
/bin/bash /path/to/script.sh
```

---

## 🔍 Tracing Parent-Child Relationships

Want to see the actual process tree for a running service? Try these commands:

### View the Full Process Tree

```bash
ps -ef --forest
```

or for a more compact view:

```bash
pstree -p
```

### Find a Specific Process's Parent

```bash
ps -o pid,ppid,cmd -p <PID>
```

### Trace the Full Ancestry of a Process

```bash
pstree -s -p <PID>
```

This shows all ancestors up to PID 1.

---

## 💡 The `exec` Trick in Startup Scripts

Many startup scripts use `exec` to replace the shell process entirely:

```bash
#!/bin/bash
# setup.sh
export MY_VAR=value
cd /opt/myapp
exec ./myapp-binary
```

**Without `exec`:**
```
systemd → bash → myapp-binary
```

**With `exec`:**
```
systemd → myapp-binary
```

The bash process is replaced, so there's no extra shell layer taking up resources. This is especially common in Docker entrypoint scripts and systemd service wrappers!

# ✅ What the Shebang Really Does

When you execute a script directly (e.g., `./start_app.sh`), the kernel looks at the first line:

```bash
#!/bin/bash
```

and uses that interpreter to execute the file's contents. But the parent process is still whatever launched the script in the first place — not necessarily a Bash terminal.

## Example 1 — Launched from Your Shell

If you run:

```bash
$ ./start_app.sh
```

Process lineage looks like:

```
bash (your shell)
 └─ start_app.sh (run by /bin/bash because of the shebang)
    └─ child processes…
```

Here, yes — your login bash shell is the parent.

## Example 2 — Launched by systemd

A service file like:

```ini
ExecStart=/usr/local/bin/start_app.sh
```

Even if `start_app.sh` begins with `#!/bin/bash`, the parent is systemd, not a bash terminal:

```
systemd
 └─ start_app.sh  (interpreter: /bin/bash)
    └─ app binary
```

No interactive shell is involved.

## Example 3 — Cron Job

```bash
# /etc/crontab
* * * * * /path/to/start_app.sh
```

Tree might look like:

```
systemd
 └─ cron
     └─ start_app.sh (run via shebang /bin/bash)
```

Again, parent is cron, not bash.

## Example 4 — Container Environments

If a script is the container's entrypoint and has a shebang, e.g.:

```dockerfile
ENTRYPOINT ["/start_app.sh"]
```

Then you'd get:

```
container runtime
 └─ start_app.sh (executed by interpreter set in shebang)
```

Still not tied to any login shell.

---

## ✅ Summary — Shebang ≠ Bash Parent

* **Shebang** = tells the kernel which interpreter to use
* **Parent process** = whatever actually `execve()`'d the script
* **Bash is only the parent if you ran it from a bash session**

So your intuition makes sense, but the shebang doesn't make the script a "bash child" — it just means "use bash to run this file's content."

---

## 🔬 Live Demonstration

Here's a quick mini-demo you can try on your own system that clearly shows how a script with a shebang behaves when launched in different ways.

### 1. Create a Simple Script

Create `demo.sh`:

```bash
#!/bin/bash
echo "PID: $$"
echo "Parent PID: $PPID"
sleep 10
```

Make it executable:

```bash
chmod +x demo.sh
```

### 2. Case A — Run it from Your Shell

```bash
./demo.sh
```

While it's sleeping, in another terminal:

```bash
ps -ef | grep demo.sh
```

You'll see something like:

```
user     13250  12980  0  ... ./demo.sh
```

Here:
* `13250` → script PID
* `12980` → your bash shell PID (its parent)

**So in this case, bash is the parent, because you started it from bash.**

### 3. Case B — Run it via Cron

Add a quick cron entry:

```bash
* * * * * /full/path/to/demo.sh
```

Wait for it to run, then do:

```bash
ps -ef | grep demo.sh
```

You'll see something like:

```
user     14521   1100  0  ... /bin/bash /full/path/to/demo.sh
```

Here, the parent PID (`1100`) is **cron**, not your bash terminal, even though the script uses `#!/bin/bash`.

### 4. Case C — Run it via systemd

Create a service file `/etc/systemd/system/demo.service`:

```ini
[Unit]
Description=Demo script

[Service]
ExecStart=/full/path/to/demo.sh

[Install]
WantedBy=multi-user.target
```

Start and check:

```bash
systemctl start demo.service
ps -ef | grep demo.sh
```

You'll see something like:

```
root     15710     1  0  ... /bin/bash /full/path/to/demo.sh
```

Parent PID is `1` (**systemd**), not a bash shell.

---

## ✅ What This Proves:

* The **shebang only sets the interpreter** (e.g. `/bin/bash`)
* The **parent process is whoever launched the script**
  * If you launched it from a terminal → bash is parent
  * If systemd launched it → systemd is parent
  * If cron launched it → cron is parent

**So a script using bash ≠ a child of a bash terminal.**


















































































































































































































































































































































































































