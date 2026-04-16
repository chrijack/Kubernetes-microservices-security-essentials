# Lesson 3.1 — Containing Containers

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Understand the fundamental containerization mechanisms: Linux namespaces for process isolation and control groups (cgroups) for resource allocation. Learn how these technologies work together to create isolated container environments.

## Prerequisites

- Docker installed and running
- Linux system with cgroups v2 support (for advanced cgroup exercises)
- Basic familiarity with Docker commands
- `unshare`, `ip`, and `stress` utilities available

## Steps

### Step 1: Understand Linux Namespaces

Linux namespaces provide isolation for different system resources, enabling containers to run in isolated environments. The main namespace types are:

- **PID**: Process ID isolation
- **NET**: Network interface isolation
- **MNT**: Mount point isolation
- **UTS**: Hostname and domain name isolation
- **IPC**: Inter-process communication isolation
- **User**: User and group ID isolation
- **Cgroup**: Control group isolation
- **Time**: System time isolation

#### Create a PID Namespace

```bash
docker run -d --name pid_ns_demo --pid=container: --rm alpine sleep 1000
docker exec -it pid_ns_demo sh
ps aux  # Verify isolated process list
```

#### Explore Other Namespaces

Network namespace demonstration:

```bash
docker run -d --name net_ns_demo --net=none --rm alpine sleep 1000
docker exec -it net_ns_demo sh
ifconfig  # Verify isolated network interfaces
```

Mount namespace demonstration:

```bash
docker run -d --name mnt_ns_demo --rm alpine sleep 1000
docker exec -it mnt_ns_demo sh
mount  # Check mount points
```

#### Hands-on Namespace Exploration

Create a new PID namespace using `unshare`:

```bash
sudo unshare --fork --pid --mount-proc bash
ps aux  # Observe isolated process list
```

Network namespace creation and management:

```bash
sudo ip netns add netns1
sudo ip netns exec netns1 ip link list
sudo ip netns del netns1
```

Mount namespace with temporary filesystem:

```bash
sudo unshare --mount bash
mount -t tmpfs tmpfs /mnt
mount | grep tmpfs
exit
mount | grep tmpfs  # Note the difference outside the namespace
```

### Step 2: Control Groups (cgroups)

Control groups limit and prioritize resource allocation for processes. They provide fine-grained control over CPU, memory, storage, network, and device access.

#### Create a cgroup for CPU Limitation

```bash
docker run -d --name cpu_cgroup_demo --cpus="0.5" --rm alpine sleep 1000
docker stats cpu_cgroup_demo  # Verify CPU usage
```

#### Create a cgroup for Memory Limitation

```bash
docker run -d --name mem_cgroup_demo --memory="256m" --rm alpine sleep 1000
docker stats mem_cgroup_demo  # Verify memory usage
```

#### Advanced cgroups v2 Usage

Create a cgroup with CPU limits using the filesystem directly:

```bash
sudo mkdir -p /sys/fs/cgroup/demo
echo "+cpu" | sudo tee /sys/fs/cgroup/demo/cgroup.subtree_control
echo "50000" | sudo tee /sys/fs/cgroup/demo/cpu.max
echo $$ | sudo tee /sys/fs/cgroup/demo/cgroup.procs
stress --cpu 4  # Observe CPU usage
```

## Verification

Verify namespace isolation by observing:

- Different process lists in PID-isolated containers vs. the host
- Absence of network interfaces in network-isolated containers
- Isolated mount points in mount-isolated namespaces

Verify resource limiting by checking:

- CPU and memory usage reported by `docker stats` matches configured limits
- Processes within cgroups respect CPU and memory constraints

## Cleanup

Stop and remove demo containers:

```bash
docker stop pid_ns_demo net_ns_demo mnt_ns_demo cpu_cgroup_demo mem_cgroup_demo
docker rm pid_ns_demo net_ns_demo mnt_ns_demo cpu_cgroup_demo mem_cgroup_demo
```

Remove advanced cgroup resources:

```bash
sudo rmdir /sys/fs/cgroup/demo
```
