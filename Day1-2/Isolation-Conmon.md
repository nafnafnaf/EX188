###  PID namespace isolation in action!
## Conmon = Container Monitoring
The conmon process belongs to the host OS. When we run podman run ngnix the conmon process creates a Subprocess with a PID for the host OS and a PID for the container itself. One PID for the host's namespace and one PID for the container's Namespace.
From the host OS we can see both PIDs. From within the container we CANNOT see both PIDs, we can see only the PID of the container's NAMESPACE. The subprocess that the host's conmon creates insider the container's namespace has PID 1. It is the creating process of the container. If conmon's PID (host's namespace) dies then the container dies too.

We can check this:
running in the host OS:
pstree -p | grep conmon
           |               |-conmon(15183)---mariadbd(15185)-+-{mariadbd}(15427)
           |               |-conmon(17938)---nginx(17940)-+-nginx(17968)

we see that for example PID 17938 of conmon process in the host's namespace creates PID 17940 and PID 17968 also in the host's namespace. One for the nginx container and one for nginx's worker.
From within the container of nginx PID 17938 does NOT exist. Instead it has PID 1 as it is the child process of host's conmon process that created the container. 
So, from the host's OS namespace PIDs are visible for containers but from the container's perspective conmon PIDs are not visible. In the container's namespace we have only PID 1

## HOW DO YOU KNOW THAT ? 
How do we know that the host's PID 17938 coresponds to PID 1 in the container's namespace??
1) From host run: ### sudo cat /proc/17940/status | grep NSpid*
this will give something like : NSpid:	17940	1
meaning that host's PID 17940 has PID 1 inside the container.
2) or podman inspect 7a2c --format '{{.State.Pid}}' 
inspect the container. 
The container can only see it's own namespace:
podman top 7a2c -eo pid,comm
    PID COMMAND
      1 nginx
     24 nginx
     25 nginx
    226 ps
 This is what namespace / container isolation means.



### Docker VS Podman isolation
 Docker:
dockerd (daemon) ← always running
    └── containerd ← container runtime
        └── containerd-shim ← similar role to conmon
            └── runc ← creates container
                └── container process (PID 1 in container namespace)
Podman:
podman (CLI - exits after start)
    └── conmon ← monitor process
        └── crun/runc ← creates container
            └── container process (PID 1 in container namespace)
The Isolation is IDENTICAL:
Both use:

✅ PID namespaces - same isolation
✅ Network namespaces - same isolation
✅ Mount namespaces - same isolation
✅ User namespaces - same isolation
✅ IPC, UTS, Cgroup namespaces - same isolation

Key Architectural Differences:
FeatureDockerPodmanDaemonYes (dockerd)No (daemonless)Root requiredUsually yesCan run rootlessMonitor processcontainerd-shimconmonContainer runtimerunc (default)crun (default) or runcSystemd integrationVia daemonDirect (no daemon)
Proof - Compare them:
Docker:
bash# Start container
docker run -d nginx

# Check process tree
pstree -p | grep containerd-shim
# Shows: containerd-shim---nginx(PID)

# Inside container
docker exec <container> ps aux
# Shows: nginx as PID 1 (same as Podman!)
Podman:
bash# Start container  
podman run -d nginx

# Check process tree
pstree -p | grep conmon
# Shows: conmon---nginx(PID)

# Inside container
podman exec <container> ps aux
# Shows: nginx as PID 1
Same isolation, different "babysitter" process!
The Bottom Line:

Isolation mechanism: Identical (Linux kernel namespaces)
Architecture: Different (daemon vs daemonless)
containerd-shim ≈ conmon: Both monitor containers from the host

The namespace isolation you learned applies to ALL container runtimes that use Linux namespaces (Docker, Podman, containerd, CRI-O, etc.)!
Does this clarify things? Want to explore the differences more? 🐳🦭RetryClaude can make mistakes. Please double-check responses. Sonnet 4.5