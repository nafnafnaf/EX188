## Commands ##:
# Get the htop from the container:
podman exec -it task2-db top
podman exec -it task2-db sh
podman exec -it task2-db ps aux

# check the conmon status:
podman inspect task2-db | grep -A5 -B5 conmon

# 1. Check conmon is running for your container
ps aux | grep $(podman inspect task2-db --format '{{.ConmonPid}}')

# 2. Monitor MariaDB logs via conmon
podman logs -f task2-db

# 3. Check resource usage
podman stats task2-db

# 4. Run this: 
> # Start a container
podman run -d --name test nginx

# View from host
ps aux | grep conmon
# You'll see conmon with a HOST PID (e.g., 12345)

# View process tree on host
pstree -p | grep conmon
# Shows: your_user─┬─conmon(12345)─┬─nginx(12346)

# View INSIDE the container
podman exec test ps aux
# You WON'T see conmon here!
# You'll only see nginx and its workers

# Check namespaces
sudo ls -l /proc/$(pgrep conmon)/ns/pid
sudo ls -l /proc/$(podman inspect test --format '{{.State.Pid}}')/ns/pid
# Different namespace IDs - conmon is in host namespace!