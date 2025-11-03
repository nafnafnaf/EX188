# Podman, Conmon, and Toolbox 

## 🔧 First — What Is `conmon`?

`conmon` is a monitoring utility used by container runtimes like Podman and CRI-O. Its job is to:

* Launch and monitor a container process
* Handle its input/output (stdio)
* Reap zombie processes
* Manage logging
* Notify the runtime when the container exits

Every container started via Podman or CRI-O uses a `conmon` instance per container.

## 🧰 So What About Toolbox?

Yes, when you run a Toolbox container, it's really a Podman container under the hood. Each Toolbox container is:

* A regular OCI container launched via Podman
* Running interactively with system-level integration
* Usually with its own `conmon` process managing it

So yes — for every running Toolbox container, you'll have:

* A Podman container process
* A conmon child process managing it

You can check it like this:

```bash
ps aux | grep conmon
```

Or list running containers:

```bash
podman ps
```

## 🔍 Example: Multiple Dev Environments

Let's say you have three development environments:

* `django-dev` → Python + PostgreSQL client
* `rust-dev` → Rust toolchain
* `go-dev` → Go CLI tools

Each of these would be a separate Toolbox container, and if they're running concurrently, each will have its own:

* Podman container
* `conmon` process managing that container

So yes — effectively you're running multiple isolated environments, each with its own child `conmon` process just like you described.

## 🎯 TL;DR

**Question:** "Is it like having multiple conmon child instances running? One for each dev container?"

**Answer:** ✅ Exactly. Each Toolbox container is a Podman container, and each Podman container is managed by its own `conmon` process.

It's a clean and efficient way to isolate environments without polluting the base OS — especially ideal for development workflows.

---

## Using VS Code with Vim Extension and Containerized Django Apps

**Question:** "Do I do it as usual: go to the dir where my django files are and open a shell and modify them with vscode and then run it with the 'central' podman instance?"

Yes — you've got the right idea, and Fedora Silverblue's design still lets you work like you're used to, with just a few small adjustments for how containers and tools are managed.

### 🧭 Typical Workflow (with Silverblue)

Let's assume you have your Django project in `~/dev/myproject`.

#### ✅ 1. Install VS Code (Flatpak)

You can install VS Code from Flatpak (GUI or terminal):

```bash
flatpak install flathub com.visualstudio.code
```

✅ The Flatpak version supports extensions like Vim just fine.

#### ✅ 2. Create a Toolbox container for development

Toolbox containers are your mutable dev environments. Let's create one:

```bash
toolbox create --container django-dev
toolbox enter django-dev
```

Now you're inside your dev container. You can `dnf install` packages freely here:

```bash
sudo dnf install python3 python3-pip python3-virtualenv podman
```

This container will share your home directory, so `~/dev/myproject` is accessible both inside and outside the container.

#### ✅ 3. Set up your Django environment

From inside the toolbox:

```bash
cd ~/dev/myproject
python3 -m venv .venv
source .venv/bin/activate
pip install django psycopg2
```

You can develop normally here.

#### ✅ 4. Open the same project in VS Code

From the host system, just open the folder:

```bash
cd ~/dev/myproject
flatpak run com.visualstudio.code .
```

VS Code can read and write to the same files because the project is in your home directory, which is shared between host, toolbox, and containers.

#### ✅ 5. Install the Vim extension

Inside VS Code (Flatpak), go to Extensions → search for "Vim" → Install. It works the same as in native VS Code.

#### ✅ 6. Run your Django app in a Podman container

You can build a container for your Django app, just like usual:

```dockerfile
# Dockerfile
FROM python:3.11

WORKDIR /app
COPY . /app

RUN pip install -r requirements.txt

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

Then build + run it using Podman (either from the host or inside toolbox):

```bash
podman build -t django-app .
podman run -p 8000:8000 django-app
```

### 🔗 How It All Ties Together

| Tool | Where It Runs | Purpose |
|------|---------------|---------|
| VS Code (Flatpak) | Host | Editor with Vim extension |
| Toolbox (`django-dev`) | Podman container | CLI + package manager + pip + Python |
| Podman container (`django-app`) | Host or Toolbox | Runs your Django app, DB, etc. |

* You can edit files with VS Code on host
* You can run the dev server from Toolbox (or container)
* All layers share your files (via your home directory)

### 🧠 Tips

* If your Django app is containerized, you can build/test it inside Toolbox or directly on the host (since Podman is rootless and integrated well).
* If your Django app needs a database (PostgreSQL), you can spin up a DB container with Podman and link the two together (via pod or network aliasing).
* You can use VS Code Dev Containers extension (Remote Containers) with Podman — though this works better with VS Code non-Flatpak. For Flatpak, integration is limited.

### ✅ Summary

**Question:** "Can I use VS Code + Vim to edit my Django files and run them with Podman, like I normally would?"

**Answer:** ✅ Yes. Just keep your project in your home directory, use Toolbox for dev CLI, use Flatpak VS Code for editing, and run your containers with Podman. Everything will work together cleanly.

---

## Understanding Conmon Child Processes

**Question:** "Will the django app be run in a container that is a conmon child process of the 'central' podman instance or of the 'dev' podman instance?"

### 🔍 Answer

The Django app container will be a `conmon` child of the Podman process in the environment you run it from — Toolbox or Host. **There is no central or shared Podman daemon, because Podman is daemonless.**

### 🧠 Deeper Explanation

#### 🔸 Podman Architecture

* Podman is daemonless — unlike Docker, it doesn't run a background service.
* Every time you run a container, a new `conmon` process is spawned to manage it.
* These `conmon` processes are child processes of the user shell (or process) that launched them.

### Two Scenarios

#### ✅ 1. Running Podman from the Host (Silverblue)

You run:

```bash
podman run -d -p 8000:8000 django-app
```

* A new `conmon` process is launched by your host shell session.
* The container is managed independently from any Toolbox environment.

#### ✅ 2. Running Podman Inside Toolbox

You enter your toolbox:

```bash
toolbox enter django-dev
podman run -d -p 8000:8000 django-app
```

* Here, the Podman CLI is running inside a container, but it talks to the user's Podman storage and socket by default (thanks to how Toolbox is integrated).
* The `conmon` process that gets launched will be:
  * A child of the `podman` process running inside the toolbox container
  * But still using your user's Podman environment, because Toolbox shares your user namespace, cgroup, and filesystem.

So even though you're inside Toolbox, the containers are not nested — they are regular rootless containers.

### 🧪 What About the "Dev" Podman Instance?

There's no truly separate "central" vs "dev" Podman instance:

* Podman is per-user and daemonless.
* Toolbox doesn't run its own separate Podman backend — it uses the same per-user rootless Podman setup.
* The only difference is where you launched the command (host vs toolbox shell).

### 🧵 Process Tree

If you run `podman` from the host shell:

```
bash (host)
└── podman
    └── conmon
        └── container (Django)
```

If you run `podman` from inside Toolbox:

```
toolbox (podman container)
└── bash (in toolbox)
    └── podman
        └── conmon
            └── container (Django)
```

### ✅ Conclusion

🧠 The Django container will be a child of the `podman` process that launched it — whether you're in Toolbox or on the host.

* The `conmon` instance is per-container.
* There's no "central" Podman instance like Docker's daemon model.
* Podman containers are isolated but not nested, even when launched from within a toolbox.

# Toolbox Networking and Purpose Discussion

## 🔥 Scenario: Accessing Django Dev Server from Outside Toolbox

You're running your Django dev server inside a Toolbox container and want to access it from outside — i.e., from:

* The host OS (Silverblue)
* Your browser
* Or even another device (if allowed)

### ⚙️ How Networking Works in Toolbox

Toolbox containers are designed to behave like part of the host system:

* They share the network namespace with the host.
* This means:
  * IP address is the same as the host
  * Ports exposed inside the toolbox are also open on the host

✅ So you don't need to port forward anything if you're just running Django inside Toolbox.

### 🧪 Example: Run Django Dev Server in Toolbox

1. Enter your toolbox:

```bash
toolbox enter django-dev
```

2. Start Django like this:

```bash
python manage.py runserver 0.0.0.0:8000
```

3. From the host browser (or curl), access:

```bash
http://localhost:8000
```

✅ It just works.

### 🌐 What About Access from Other Devices?

If you want to access the Django server from another device on your LAN, you need to:

1. Bind to all interfaces (`0.0.0.0`) — you already did
2. Know your host's LAN IP (e.g., `192.168.x.x`)
3. Allow the port through the firewall (Fedora has `firewalld`)

🔥 Allow Django's Port Through Firewall:

```bash
sudo firewall-cmd --add-port=8000/tcp --permanent
sudo firewall-cmd --reload
```

Now you can access:

```
http://<your-laptop-ip>:8000
```

From phones, tablets, or other laptops.

### 🔁 What If You Run Django in a Podman Container Instead?

Then yes — you need to explicitly expose or forward ports, just like Docker.

```bash
podman run -p 8000:8000 django-app
```

### ✅ Summary

| Where Django Runs | Port Forwarding Needed? | Accessed From |
|-------------------|-------------------------|---------------|
| Toolbox | ❌ No | Directly via `localhost:8000` |
| Podman container | ✅ Yes, use `-p` | Whatever you map |
| Other devices on LAN | ✅ Yes, plus firewall rules | `http://<host-ip>:8000` |

---

## Understanding the Purpose of Toolbox

**Question:** "If Toolbox shares all these host resources... what's the point of using it at all?"

The answer is subtle but powerful. It's not about isolating the container from the host — it's about isolating the effects of your development environment from your system, without restricting your ability to develop like a normal user.

### 🧰 So What Is the Point of Toolbox?

Toolbox is designed to provide:

✅ **A clean, mutable, package-managed development space**

* On Fedora Silverblue, the host system is immutable — you can't `dnf install`, `pip install --system`, etc.
* Toolbox gives you a mutable container where you can use DNF, GCC, Python, pip, etc., just like you would on a normal Fedora system.
* This keeps your base OS clean, minimal, and stable.

### 🔄 Toolbox = Mutability Without Risk

| Feature | Host (Silverblue) | Toolbox |
|---------|-------------------|---------|
| `dnf install` | ❌ Not allowed | ✅ Yes |
| `pip install --system` | ❌ No | ✅ Yes |
| Custom build tools | ❌ Risky | ✅ Safe |
| Reproducibility | ✅ Stable base | ✅ Dev container you can rebuild or version |
| Access to host files | ✅ | ✅ (home dir is mounted) |

### 🧠 Think of Toolbox like:

* A development layer on top of your stable system
* A "safe zone" to experiment with packages, compilers, Python versions, etc.
* A way to simulate a full Fedora workstation, without actually installing tons of stuff onto the base OS

### 🧪 Toolbox vs. Other Containers

| Behavior | Toolbox | Docker/Podman Container |
|----------|---------|-------------------------|
| Access to host network | ✅ | ❌ (unless `--network=host`) |
| Access to host cgroups/devices | ✅ | ❌ by default |
| Access to user home dir | ✅ | ❌ (usually not mounted) |
| Intended for GUI use | ✅ | ❌ |
| Uses your user UID/GID | ✅ | ❌ (runs as root or mapped UID) |
| Easy to `dnf install`, use systemd-ish tools | ✅ | ❌ (often stripped down images) |

So Toolbox is a developer's environment — not a hard-isolation sandbox like Docker or Podman.

### 🔒 But… Is It Really Isolated?

Only lightly:

* No strong network isolation (shares host net namespace)
* No filesystem isolation beyond what you choose (your home is mounted)
* No real cgroup or namespace fencing (it's friendly, not secure)

Toolbox isn't meant for security isolation. It's meant to protect your system from your development activity — not to protect the system from you, or you from the container.

### 🛠️ Use Case Summary

| Use Case | Use Toolbox? | Use Podman/Docker? |
|----------|--------------|-------------------|
| Writing and testing Python/Django apps | ✅ Yes | ✅ Yes |
| Compiling things with GCC, testing dev tools | ✅ Yes | ❌ (Podman is too restrictive) |
| Running services for testing (Postgres, Redis) | ✅ or Podman | ✅ Better in Podman |
| Simulating production envs / CI | ❌ | ✅ Yes |
| Security sandboxing | ❌ | ✅ Yes |

### ✅ TL;DR

Toolbox gives you a mutable, full-featured Fedora dev environment — without dirtying your host system. It's not about security isolation, it's about clean separation of development and system layers, especially on immutable systems like Silverblue.

---

## Toolbox as a "Second OS" for Development

**Concept:** Toolbox is like a "second OS" for development while your host OS remains clean and intact.

### Comparison Table

| Host OS (Fedora Silverblue) | Toolbox Container |
|-----------------------------|-------------------|
| Immutable — read-only root filesystem (`/usr`, etc.) | Fully mutable — install, uninstall, modify |
| No `dnf install` | Full `dnf`, `pip`, build tools, compilers |
| No global changes allowed | Local, disposable, per-toolbox environments |
| Ideal for stability, consistency | Ideal for development, experimentation |
| Shares your `/home/$USER` | Has access to all your project files |
| Runs GNOME, system services | Acts like a full Fedora shell — no GUI by default, but can run GUI apps too |

### 🔁 Toolbox = Dev Environment Layer

Think of it like this:

```
[ Fedora Silverblue Host ]
         │
         ▼
[ Toolbox container: django-dev ]
         │
         ├─ dnf install python3
         ├─ pip install psycopg2
         └─ python manage.py runserver
```

You can have multiple of these:

* `toolbox create django-dev`
* `toolbox create rust-dev`
* `toolbox create c-dev`

Each one:

* Has its own installed packages
* Can be configured differently
* Can be deleted and recreated easily
* Does not pollute or modify the base system

### 🧪 Example

Let's say you're on Silverblue:

1. You can't install PostgreSQL globally:

```bash
sudo dnf install postgresql  # ❌ Won't work — host is immutable
```

2. But inside toolbox:

```bash
toolbox enter django-dev
sudo dnf install postgresql  # ✅ Works — inside mutable dev container
```

### 🧼 Why It's Great

* Keeps your host clean
* Easy to reproduce dev setups
* Good for trying new versions of compilers, interpreters, SDKs
* Supports multiple, isolated environments
* If something breaks? Just delete and recreate the toolbox

### ✅ TL;DR

Yes — Toolbox gives you a separate, full-featured Fedora environment per container, while keeping your main OS immutable, clean, and stable.

### Next Steps

You can:

* Save/share toolboxes across machines
* Set up persistent tooling inside them
* Use GUI apps from toolbox