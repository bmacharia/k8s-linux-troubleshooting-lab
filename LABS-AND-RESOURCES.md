# 12-Week Lab Guide & Resources

> **How to use this file:** For each week, work through the sections in order — Learn → Break → Debug → Postmortem. Every failure session must answer the four questions at the bottom of each week. Keep your notes in the corresponding directory.

---

## Table of Contents

- [Week 0 – Lab Baseline](#week-0--lab-baseline)
- [Week 1 – Linux Process & CPU Failures](#week-1--linux-process--cpu-failures)
- [Week 2 – Linux Memory & OOM Killers](#week-2--linux-memory--oom-killers)
- [Week 3 – Disk, Filesystems & Inodes](#week-3--disk-filesystems--inodes)
- [Week 4 – Linux Networking Fundamentals](#week-4--linux-networking-fundamentals)
- [Week 5 – Containers & Runtimes](#week-5--containers--runtimes)
- [Week 6 – Kubernetes Control Plane](#week-6--kubernetes-control-plane)
- [Week 7 – Scheduler & Resource Failures](#week-7--scheduler--resource-failures)
- [Week 8 – Pod Lifecycle & Controllers](#week-8--pod-lifecycle--controllers)
- [Week 9 – Kubernetes Networking (CNI)](#week-9--kubernetes-networking-cni)
- [Week 10 – Storage (CSI & PVCs)](#week-10--storage-csi--pvcs)
- [Week 11 – Security Failures (CKS Core)](#week-11--security-failures-cks-core)
- [Week 12 – Chaos & Exam Simulation](#week-12--chaos--exam-simulation)
- [Postmortem Template](#postmortem-template)
- [Master Resource List](#master-resource-list)

---

## Week 0 – Lab Baseline

**Goal:** A known-good cluster you will break repeatedly. Snapshot everything before touching it.

### Required Components

| Component | Role | Notes |
|-----------|------|-------|
| 1 control-plane Pi | etcd + API server + scheduler | Label: `node-role.kubernetes.io/master` |
| 2–3 worker Pis | Run workloads | Arm64 compatible images only |
| K3s | Lightweight Kubernetes | Ships with containerd + Flannel |
| CoreDNS | In-cluster DNS | Verify with `nslookup kubernetes.default` |
| Traefik / nginx ingress | Ingress controller | K3s ships Traefik by default |
| Prometheus + Grafana | Observability | kube-prometheus-stack via Helm |
| local-path / Longhorn | Persistent storage | local-path is built into K3s |

### Setup Steps

```bash
# 1. Install K3s on control plane
curl -sfL https://get.k3s.io | sh -

# 2. Get node token
sudo cat /var/lib/rancher/k3s/server/node-token

# 3. Join workers (run on each worker Pi)
curl -sfL https://get.k3s.io | K3S_URL=https://<CONTROL_PLANE_IP>:6443 \
  K3S_TOKEN=<NODE_TOKEN> sh -

# 4. Verify cluster
kubectl get nodes -o wide

# 5. Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 6. Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# 7. Snapshot etcd (K3s uses SQLite by default, backup the DB file)
sudo cp /var/lib/rancher/k3s/server/db/state.db ~/k3s-backup-$(date +%Y%m%d).db

# 8. SD card snapshot (run from another machine)
# sudo dd if=/dev/sdX bs=4M | gzip > pi-snapshot-$(date +%Y%m%d).img.gz
```

### Verify Baseline Health

```bash
kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by='.lastTimestamp'
systemctl status k3s                          # control plane
systemctl status k3s-agent                    # workers
journalctl -u k3s -n 50 --no-pager
```

### Resources

- [K3s Quick Start](https://docs.k3s.io/quick-start)
- [kube-prometheus-stack Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Longhorn install guide](https://longhorn.io/docs/latest/deploy/install/)
- [etcd backup & restore](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)

---

## Week 1 – Linux Process & CPU Failures

**CKA:** Node troubleshooting | **CKS:** Process behavior, least privilege

### Learn

**Key concepts:**

| Concept | What it means |
|---------|--------------|
| Load average | 1/5/15-min moving average of runnable + uninterruptible tasks |
| Run queue | Threads waiting for a CPU core — `r` column in `vmstat` |
| Zombie process | Finished process whose parent hasn't called `wait()` — PID exists, no CPU |
| CPU throttling | cgroup CPU quota enforced — task sleeps even on an idle node |
| CPU starvation | No quota, but too many competing tasks — task waits in the run queue |

**Mental model:** `top` load > # cores = pressure. `%wa` high = I/O wait. `%st` high = steal (VM host taking your CPU).

### Lab Exercises

#### Exercise 1 – Observe baseline CPU

```bash
# On a worker node
vmstat 1 10                     # columns: r (runqueue), us, sy, id, wa
mpstat -P ALL 1 5               # per-core breakdown
top -b -n 3 -d 1                # batch mode, 3 snapshots
```

#### Exercise 2 – Fork bomb in a container

```bash
# Deploy a stress pod (safe: PID limit enforced)
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: stress-cpu
spec:
  containers:
  - name: stress
    image: containerstack/stress-ng:latest
    command: ["stress-ng", "--cpu", "4", "--timeout", "120s"]
    resources:
      requests:
        cpu: "100m"
      limits:
        cpu: "200m"
EOF

# Watch throttling in real time
watch -n1 kubectl top pod stress-cpu

# On the node hosting the pod — check cgroup throttle counters
CGPATH=$(find /sys/fs/cgroup -name "cpu.stat" | xargs grep -l "nr_throttled" 2>/dev/null | head -5)
cat $CGPATH
```

#### Exercise 3 – Trace pod → cgroup → throttle

```bash
# Get the container ID
CONTAINER_ID=$(kubectl get pod stress-cpu -o jsonpath='{.status.containerStatuses[0].containerID}' | sed 's/containerd:\/\///')

# Find the cgroup path for this container
find /sys/fs/cgroup -name "*.scope" | xargs grep -l "$CONTAINER_ID" 2>/dev/null

# Read throttle stats
cat /sys/fs/cgroup/cpu,cpuacct/<path-from-above>/cpu.stat
# nr_throttled = number of throttled periods
# throttled_time = nanoseconds spent throttled
```

#### Exercise 4 – Absurd CPU limit (watch scheduler fight)

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: cpu-starved
spec:
  containers:
  - name: worker
    image: busybox
    command: ["sh", "-c", "while true; do :; done"]
    resources:
      limits:
        cpu: "1m"   # 1 millicpu = 0.001 core — almost nothing
EOF

kubectl exec -it cpu-starved -- time echo "hello"
```

### Debug Commands

```bash
# Is it throttling or starvation?
# Throttling: nr_throttled > 0 in cgroup cpu.stat
# Starvation: load average >> num_cpus, r column in vmstat > num_cpus

# Process-level view
ps aux --sort=-%cpu | head -20
pidstat -u 1 5                  # per-PID CPU over time

# Trace the full path: pod → container → PID → cgroup
kubectl describe pod <pod>      # find containerID and nodeName
crictl inspect <containerID>    # see cgroup path from CRI side
cat /proc/<PID>/cgroup          # the cgroup the process is in
```

### Resources

- [Linux Load Averages – Brendan Gregg](https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html)
- [Linux CPU Scheduling – kernel.org](https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html)
- [Kubernetes CPU requests/limits – docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [cgroup v2 CPU controller](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#cpu)
- **Tools:** `sysstat` (vmstat, mpstat, pidstat), `stress-ng`, `bcc-tools` (cpudist, runqlat)

---

## Week 2 – Linux Memory & OOM Killers

**CKA:** Pod lifecycle | **CKS:** Resource isolation

### Learn

| Term | Meaning |
|------|---------|
| RSS | Resident Set Size — physical RAM in use right now |
| Page cache | Disk data cached in RAM — Linux reclaims this first |
| OOM score | Kernel ranks processes; highest score is killed first |
| cgroup memory limit | Hard ceiling — crossing it triggers OOMKill immediately |
| Eviction | Kubelet evicts pods before kernel OOM hits (softer) |

**Mental model:** Linux kills the highest `oom_score_adj` process. Kubernetes sets containers to `1000` (max killable). Node system daemons are set to `-999` (protected).

### Lab Exercises

#### Exercise 1 – Read memory accounting

```bash
# On a worker node
free -h                          # total / used / free / cache+buffers
cat /proc/meminfo                # detailed breakdown
vmstat -s                        # memory event counters

# For a running pod's PID
PID=$(pgrep -f "my-process")
cat /proc/$PID/status | grep -E "VmRSS|VmSize|OomScore"
```

#### Exercise 2 – Memory leak pod

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: memory-leak
spec:
  containers:
  - name: leak
    image: polinux/stress
    command: ["stress", "--vm", "1", "--vm-bytes", "300M", "--vm-keep"]
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "128Mi"   # Will OOMKill when it crosses 128Mi
EOF

# Watch it die and restart
kubectl get pod memory-leak -w

# After OOMKill, check the exit reason
kubectl describe pod memory-leak | grep -A5 "Last State"
```

#### Exercise 3 – Read cgroup memory limits

```bash
# Find cgroup path for the container
CONTAINER_ID=$(kubectl get pod memory-leak -o jsonpath='{.status.containerStatuses[0].containerID}' | sed 's/containerd:\/\///')
CGPATH=$(find /sys/fs/cgroup/memory -name "*.scope" | xargs grep -rl "$CONTAINER_ID" 2>/dev/null | head -1 | xargs dirname)

cat $CGPATH/memory.limit_in_bytes    # cgroup v1
cat $CGPATH/memory.current           # cgroup v2 — current usage
cat $CGPATH/memory.events            # cgroup v2 — oom count
```

#### Exercise 4 – Node memory pressure

```bash
# Create many pods that collectively push toward the eviction threshold
# Default eviction threshold is 100Mi available
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-hog
spec:
  replicas: 5
  selector:
    matchLabels:
      app: hog
  template:
    metadata:
      labels:
        app: hog
    spec:
      containers:
      - name: hog
        image: polinux/stress
        command: ["stress", "--vm", "1", "--vm-bytes", "200M", "--vm-keep"]
        resources:
          requests:
            memory: "50Mi"
          limits:
            memory: "250Mi"
EOF

# Watch kubelet eviction in action
kubectl get events -w --field-selector reason=Evicted
journalctl -u k3s -f | grep -i evict
```

### Debug Commands

```bash
# Was it OOMKilled?
kubectl describe pod <pod> | grep -i oom
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'

# Node-level OOM events
dmesg | grep -i "killed process"
dmesg | grep -i oom

# Which process was killed and why
dmesg | grep -A3 "Out of memory"

# Distinguish eviction vs OOMKill
# OOMKill: kernel kills the process — exit code 137, reason: OOMKilled
# Eviction: kubelet kills the pod — reason: Evicted in events
```

### Resources

- [Linux OOM Killer – lwn.net](https://lwn.net/Articles/391222/)
- [Kubernetes Out-of-Resource Handling](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [cgroup v2 memory controller](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory)
- [Container memory explained – Ivan Velichko](https://iximiuz.com/en/posts/container-learning-path/)
- **Tools:** `free`, `vmstat`, `smem`, `pmap`, `dmesg`, `/proc/meminfo`

---

## Week 3 – Disk, Filesystems & Inodes

**CKA:** Node health | **CKS:** Storage isolation

### Learn

| Concept | What it means |
|---------|--------------|
| Inode | Metadata entry for a file — name, permissions, pointer to blocks |
| Block | Fixed-size chunk of data on disk (usually 4096 bytes) |
| Disk pressure | Kubelet condition: `DiskPressure=True` → pods get evicted |
| Container layers | overlay2 mounts: lowerdir (image), upperdir (writes), workdir (staging) |
| `/var/lib/kubelet` | Where kubelet stores pod data — filling this breaks everything |

### Lab Exercises

#### Exercise 1 – Baseline disk health

```bash
df -h                            # human-readable disk usage
df -i                            # inode usage (often the silent killer)
du -sh /var/lib/kubelet          # kubelet data footprint
du -sh /var/lib/containerd       # container image storage
lsblk                            # block devices and mount points
```

#### Exercise 2 – Fill `/var/lib/kubelet` (simulate DiskPressure)

```bash
# WARNING: do this on a worker node, not control plane
# Create a pod that writes aggressively to its local storage
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: disk-filler
spec:
  containers:
  - name: filler
    image: busybox
    command: ["sh", "-c", "dd if=/dev/zero of=/data/fill bs=1M count=2000; sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF

# Watch for DiskPressure condition
watch -n5 kubectl describe node <worker-node> | grep -A5 Conditions
kubectl get events --field-selector reason=EvictionThresholdMet -w
```

#### Exercise 3 – Exhaust inodes

```bash
# SSH to a worker node
# Create millions of tiny files in /tmp
mkdir /tmp/inode-test
for i in $(seq 1 200000); do touch /tmp/inode-test/f$i; done

# Watch inodes disappear
watch -n2 'df -i /tmp'

# Try to create a new file once inodes are exhausted
touch /tmp/new-file   # ENOSPC even though blocks are available
```

#### Exercise 4 – Inspect container filesystem layers

```bash
# Find the overlay mount for a running container
CONTAINER_ID=$(kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].containerID}' | sed 's/containerd:\/\///')

# crictl inspect shows the rootfs
crictl inspect $CONTAINER_ID | jq '.info.runtimeSpec.mounts'

# Or find the overlay mount directly
mount | grep overlay | grep $CONTAINER_ID
```

### Debug Commands

```bash
# Detect DiskPressure early
kubectl describe node <node> | grep -A10 "Conditions:"
kubectl get node <node> -o jsonpath='{.status.conditions[?(@.type=="DiskPressure")].status}'

# Find what's eating space
du -sh /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/*/ 2>/dev/null | sort -rh | head -20
journalctl --disk-usage          # how much journal is consuming

# Clean up without reboot
crictl rmi --prune               # remove unused images
journalctl --vacuum-size=100M    # trim logs
kubectl delete pod --field-selector=status.phase=Succeeded -A  # clean completed pods
```

### Resources

- [Linux filesystem internals – The Linux Documentation Project](https://tldp.org/LDP/tlk/fs/filesystem.html)
- [Kubernetes DiskPressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#eviction-signals)
- [containerd snapshotter docs](https://github.com/containerd/containerd/blob/main/docs/snapshotters/overlayfs.md)
- [Brendan Gregg – Linux Disk IO tools](https://www.brendangregg.com/linuxperf.html)
- **Tools:** `df`, `du`, `lsblk`, `iostat`, `iotop`, `ncdu`, `crictl`

---

## Week 4 – Linux Networking Fundamentals

**CKA:** Cluster networking | **CKS:** Network boundaries

### Learn

| Concept | What it means |
|---------|--------------|
| TCP 3-way handshake | SYN → SYN-ACK → ACK — failure here means no connection |
| conntrack | Kernel table tracking all NAT/stateful connections — has a size limit |
| NAT | Network address translation — source IPs rewritten for pod traffic |
| Ephemeral ports | Source ports 32768–60999 — can be exhausted under high connection rate |
| `ss` | Socket statistics — replacement for `netstat` |

### Lab Exercises

#### Exercise 1 – Baseline network state

```bash
# On a worker node
ss -tulpn                        # listening sockets
ss -s                            # summary of socket states
ip route show                    # routing table
ip addr show                     # interface addresses
cat /proc/net/nf_conntrack | wc -l   # current conntrack entries
cat /proc/sys/net/netfilter/nf_conntrack_max   # the limit
```

#### Exercise 2 – Exhaust conntrack table

```bash
# Reduce the conntrack table limit temporarily (run as root on a worker node)
# WARNING: this will cause connection drops
sysctl -w net.netfilter.nf_conntrack_max=100

# Generate connections to fill the table
# Deploy a test pod
kubectl run nettest --image=nicolaka/netshoot -- sleep 3600
kubectl exec -it nettest -- bash

# Inside the pod, hammer connections
for i in $(seq 1 200); do curl -s --max-time 1 http://kubernetes.default/ & done

# Watch conntrack drops
watch -n1 'cat /proc/net/stat/nf_conntrack | awk "NR==2{print \"drops: \" $8}"'
dmesg | grep "nf_conntrack: table full"

# Restore the limit
sysctl -w net.netfilter.nf_conntrack_max=131072
```

#### Exercise 3 – DNS resolution failure

```bash
# Break CoreDNS — scale it to 0
kubectl scale deployment coredns -n kube-system --replicas=0

# From a pod, watch DNS fail
kubectl exec -it nettest -- nslookup kubernetes.default
kubectl exec -it nettest -- curl -v http://kubernetes.default/

# Diagnose
kubectl get endpoints -n kube-system kube-dns
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Fix
kubectl scale deployment coredns -n kube-system --replicas=2
```

#### Exercise 4 – Packet tracing with tcpdump

```bash
# On a worker node — capture all traffic to/from a pod IP
POD_IP=$(kubectl get pod nettest -o jsonpath='{.status.podIP}')
sudo tcpdump -i any host $POD_IP -nn -c 50

# Capture DNS queries specifically
sudo tcpdump -i any port 53 -nn -c 20

# Inside a pod using netshoot
kubectl exec -it nettest -- tcpdump -i eth0 -nn -c 20
```

### Debug Commands

```bash
# Prove where packets die
ip route get <destination-ip>           # which interface/gateway
traceroute <destination-ip>
kubectl exec -it <pod> -- traceroute <service-clusterip>

# Check iptables rules (kube-proxy mode)
iptables -t nat -L KUBE-SERVICES -n --line-numbers | head -30

# Check IPVS rules (if using ipvs mode)
ipvsadm -ln | grep <service-ip>

# DNS debug
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl exec -it <pod> -- dig @10.96.0.10 kubernetes.default SRV
```

### Resources

- [The TCP/IP Guide – online reference](http://www.tcpipguide.com/free/index.htm)
- [Linux conntrack explained – Cloudflare Blog](https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-flows/)
- [Kubernetes networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Julia Evans – How DNS Works](https://jvns.ca/blog/2022/01/11/how-to-find-a-dns-leak/)
- **Tools:** `ss`, `ip`, `tcpdump`, `traceroute`, `dig`, `nslookup`, `netshoot` image

---

## Week 5 – Containers & Runtimes

**CKA:** Runtime troubleshooting | **CKS:** Image trust & isolation

### Learn

| Concept | What it means |
|---------|--------------|
| containerd | The CRI runtime K3s uses — manages image pull, unpack, container lifecycle |
| PID 1 | The first process in a container — if it exits, the container stops |
| shim | containerd-shim-runc-v2 — process that owns the container lifecycle |
| OCI spec | JSON that defines container config — see via `crictl inspect` |
| Image manifest | Index of layers and their digests — corruption breaks pull |

### Lab Exercises

#### Exercise 1 – Inspect running containers with crictl

```bash
# On any node (not through kubectl)
sudo crictl ps                           # list running containers
sudo crictl pods                         # list pod sandboxes
sudo crictl images                       # cached images
sudo crictl inspect <container-id>       # full OCI spec + state
sudo crictl logs <container-id>          # container logs directly
sudo crictl exec -it <container-id> sh   # exec without kubectl
```

#### Exercise 2 – Crash PID 1

```bash
# A process that causes PID 1 to exit immediately
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pid1-crash
spec:
  containers:
  - name: crasher
    image: busybox
    command: ["sh", "-c", "exit 1"]    # PID 1 exits with error
  restartPolicy: Always
EOF

kubectl get pod pid1-crash -w           # watch it CrashLoop
kubectl describe pod pid1-crash         # see exit codes and restart count
```

#### Exercise 3 – Image pull auth failure

```bash
# Reference a private image without credentials
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pull-failure
spec:
  containers:
  - name: app
    image: registry.example.com/private/myapp:latest
EOF

# Debug the pull failure
kubectl describe pod pull-failure        # ErrImagePull or ImagePullBackOff
crictl pull registry.example.com/private/myapp:latest  # direct pull attempt on node

# Check containerd logs
journalctl -u containerd -n 50 --no-pager | grep -i "pull\|error"
```

#### Exercise 4 – Distinguish runtime vs kubelet boundary

```bash
# kubelet talks to containerd via CRI gRPC
# View the gRPC socket
ls -la /run/containerd/containerd.sock

# Check containerd is healthy
sudo ctr version
sudo ctr namespaces list                 # k8s.io namespace is where kubelet's containers live
sudo ctr --namespace k8s.io containers list

# Check kubelet can reach containerd
journalctl -u k3s | grep -i "cri\|runtime\|containerd" | tail -20
```

### Debug Commands

```bash
# Runtime boundary triage
# Step 1: Is containerd running?
systemctl status containerd

# Step 2: Can kubelet talk to it?
journalctl -u kubelet | grep -i "failed\|error" | tail -20

# Step 3: What did the CRI log say about the container?
crictl logs --tail 50 <container-id>

# Step 4: Image issue or runtime issue?
crictl pull <image>                      # test pull directly
crictl inspecti <image-id>               # inspect the image layers
```

### Resources

- [containerd architecture](https://github.com/containerd/containerd/blob/main/docs/architecture.md)
- [OCI Runtime Spec](https://github.com/opencontainers/runtime-spec/blob/main/spec.md)
- [crictl user guide](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [PID 1 problem in containers – Phusion blog](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/)
- **Tools:** `crictl`, `ctr`, `runc`, `nerdctl`, `skopeo`

---

## Week 6 – Kubernetes Control Plane

**CKA:** Core architecture | **CKS:** Admission & auth

### Learn

| Component | Role | Failure symptom |
|-----------|------|----------------|
| etcd | Stores all cluster state | API server returns 500s, no writes work |
| API server | All clients talk here | `kubectl` hangs or returns errors |
| controller-manager | Reconciles desired→actual | Deployments don't scale, pods not created |
| scheduler | Assigns pods to nodes | Pods stay Pending forever |

**Request path:** `kubectl` → API server → etcd (for reads/writes) → API server notifies controllers → scheduler → kubelet

### Lab Exercises

#### Exercise 1 – Map the control plane components (K3s)

```bash
# K3s bundles everything in one process
ps aux | grep k3s
journalctl -u k3s --no-pager | tail -30

# But you can still query individual component health via the API
kubectl get componentstatuses        # may be deprecated but still useful
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /livez
```

#### Exercise 2 – Stop the K3s server (simulate etcd/API outage)

```bash
# On control plane node — this stops etcd + API server
sudo systemctl stop k3s

# On a worker node, watch what happens
kubectl get pods                     # hangs until timeout
kubectl get nodes                    # same
# Workers continue running existing pods (disconnected mode)

# Watch kubelet logs on worker — it keeps trying to reconnect
journalctl -u k3s-agent -f

# Restore
sudo systemctl start k3s
kubectl get nodes                    # nodes come back after re-registration
```

#### Exercise 3 – Simulate API server slowness

```bash
# Add artificial latency using tc (traffic control) on the control plane network interface
# WARNING: this affects all traffic on the interface — restore immediately after
IFACE=$(ip route get 1.1.1.1 | awk '{print $5; exit}')
sudo tc qdisc add dev $IFACE root netem delay 500ms 100ms

# Watch kubectl slow down
time kubectl get pods -A

# Check controller-manager behavior — reconciliation slows
kubectl get events --sort-by='.lastTimestamp' | tail -20

# Remove the latency
sudo tc qdisc del dev $IFACE root
```

#### Exercise 4 – Break controller-manager (K3s embedded)

```bash
# K3s runs everything in a single binary, but you can observe controller behavior
# Create a deployment and watch the controller reconcile
kubectl create deployment test-deploy --image=nginx --replicas=3
kubectl get pods -w

# Now delete a pod and watch reconciliation
kubectl delete pod <one-of-the-pods>
kubectl get pods -w     # controller creates a replacement within seconds

# The log for reconciliation
journalctl -u k3s | grep -i "reconcil\|deployment\|replicaset" | tail -20
```

### Debug Commands

```bash
# API server is the first thing to check
kubectl cluster-info
kubectl get --raw /healthz/etcd      # etcd health from API server's view

# etcd health (K3s uses embedded etcd or SQLite)
# For SQLite (default K3s):
sudo sqlite3 /var/lib/rancher/k3s/server/db/state.db ".tables"
sudo sqlite3 /var/lib/rancher/k3s/server/db/state.db "SELECT count(*) FROM kine;"

# Controller-manager logs
journalctl -u k3s | grep "controller" | tail -30

# Scheduler logs
journalctl -u k3s | grep "scheduler\|Binding\|Scheduling" | tail -30
```

### Resources

- [Kubernetes Components – docs](https://kubernetes.io/docs/concepts/overview/components/)
- [etcd operations guide](https://etcd.io/docs/v3.5/op-guide/)
- [API server request flow – Tim Hockin](https://github.com/thockin/kubernetes-talk-2019-api-machinery)
- [K3s architecture](https://docs.k3s.io/architecture)
- **Tools:** `kubectl`, `etcdctl`, `k9s`, journalctl

---

## Week 7 – Scheduler & Resource Failures

**CKA:** Scheduling | **CKS:** Policy enforcement

### Learn

| Term | Meaning |
|------|---------|
| Request | The guaranteed amount — scheduler uses this to decide placement |
| Limit | The maximum allowed — enforced by cgroup at runtime |
| Pending | Pod hasn't been assigned to a node yet |
| Unschedulable | Scheduler found no suitable node — check events for reason |
| Taint | A node repels pods that don't tolerate it |

**Pending pod reasons (in events):**
- `Insufficient cpu` — no node has enough schedulable CPU
- `Insufficient memory` — same for memory
- `node(s) had taint {node.kubernetes.io/not-ready}` — nodes not ready
- `0/3 nodes are available` — look at the rest of the message

### Lab Exercises

#### Exercise 1 – Create an unschedulable pod

```bash
# Request more CPU than any node has
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: too-big
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "64"       # 64 cores — Pi has 4
        memory: "128Gi"
EOF

kubectl get pod too-big
kubectl describe pod too-big | grep -A5 Events
```

#### Exercise 2 – Taint a node and fix with tolerations

```bash
# Taint a worker node
kubectl taint nodes <worker-node> env=prod:NoSchedule

# Try to schedule a pod — it will be Pending
kubectl run no-toleration --image=nginx

# Fix with toleration
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: "env"
    operator: "Equal"
    value: "prod"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx
EOF

# Remove the taint when done
kubectl taint nodes <worker-node> env=prod:NoSchedule-
```

#### Exercise 3 – Node pressure taints (observe automatic tainting)

```bash
# Kubelet automatically adds taints when node is under pressure
# Trigger DiskPressure (from Week 3 exercise)
# Then watch for the automatic taint
kubectl describe node <node> | grep Taints
# Should see: node.kubernetes.io/disk-pressure:NoSchedule

# Verify pods stop being scheduled there
kubectl get pods -o wide    # new pods avoid the tainted node
```

#### Exercise 4 – Overcommit CPU and observe scheduling vs runtime

```bash
# Overcommit is allowed at scheduling time (requests, not limits)
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overcommit
spec:
  replicas: 10
  selector:
    matchLabels:
      app: overcommit
  template:
    metadata:
      labels:
        app: overcommit
    spec:
      containers:
      - name: cpu-burn
        image: containerstack/stress-ng:latest
        command: ["stress-ng", "--cpu", "1", "--timeout", "0"]
        resources:
          requests:
            cpu: "100m"   # scheduler sees 100m × 10 = 1 core needed
          limits:
            cpu: "2"      # each pod allowed up to 2 cores — but throttled
EOF

# Observe: all pods scheduled, but throttled at runtime
kubectl top pods -l app=overcommit
```

### Debug Commands

```bash
# Read scheduler events — the most important debug signal
kubectl describe pod <pending-pod> | grep -A20 Events
kubectl get events --field-selector involvedObject.name=<pod>

# View scheduler logs
journalctl -u k3s | grep -i "scheduler\|predicat\|filter\|score" | tail -30

# Node allocatable vs requested
kubectl describe node <node> | grep -A15 "Allocated resources"

# See all node taints
kubectl get nodes -o=custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### Resources

- [Kubernetes scheduler docs](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Resource requests and limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Scheduler debugging](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)
- **Tools:** `kubectl describe node`, `kubectl top`, `k9s`

---

## Week 8 – Pod Lifecycle & Controllers

**CKA:** Workloads | **CKS:** Secure defaults

### Learn

| Phase | Meaning |
|-------|---------|
| Pending | Scheduled but containers not yet started |
| Init | Init containers running (sequentially) |
| Running | At least one container is running |
| CrashLoopBackOff | Container keeps crashing — kubelet applies exponential backoff |
| Completed | All containers exited 0 |
| OOMKilled | Container killed by kernel OOM (exit 137) |

**Probe types:**
- `livenessProbe` — container killed and restarted if it fails
- `readinessProbe` — pod removed from Service endpoints if it fails
- `startupProbe` — disables liveness during startup; allows slow-start apps

### Lab Exercises

#### Exercise 1 – Bad readiness probe (silent traffic drop)

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: bad-readiness
  labels:
    app: bad-readiness
spec:
  containers:
  - name: web
    image: nginx
    readinessProbe:
      httpGet:
        path: /does-not-exist   # 404 → probe fails
        port: 80
      initialDelaySeconds: 2
      periodSeconds: 5
EOF

# Pod runs but is never Ready
kubectl get pod bad-readiness
kubectl describe pod bad-readiness | grep -A10 "Conditions\|Readiness"

# Service still created — but no endpoints
kubectl expose pod bad-readiness --port=80 --target-port=80
kubectl get endpoints bad-readiness
```

#### Exercise 2 – CrashLoop via bad config

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: crashloop
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: WRONG_VAR
      value: "this causes the app to exit"
    command: ["sh", "-c", "echo Starting; exit 1"]
  restartPolicy: Always
EOF

kubectl get pod crashloop -w

# Debug without redeploy
kubectl logs crashloop                   # current logs
kubectl logs crashloop --previous        # last crash's logs
kubectl describe pod crashloop           # see exit codes and restart count
```

#### Exercise 3 – Broken init container

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: bad-init
spec:
  initContainers:
  - name: setup
    image: busybox
    command: ["sh", "-c", "echo Initializing; exit 1"]   # init fails
  containers:
  - name: app
    image: nginx
EOF

# Main container never starts
kubectl get pod bad-init
kubectl describe pod bad-init | grep -A5 "Init Containers"
kubectl logs bad-init -c setup   # logs from the init container
```

#### Exercise 4 – Deployment rollout failure

```bash
kubectl create deployment rollout-test --image=nginx:1.25
kubectl set image deployment/rollout-test nginx=nginx:nonexistent-tag

# Watch the rollout fail
kubectl rollout status deployment/rollout-test
kubectl get pods -l app=rollout-test

# Check events
kubectl describe deployment rollout-test
kubectl get events --sort-by='.lastTimestamp' | tail -10

# Roll back
kubectl rollout undo deployment/rollout-test
kubectl rollout status deployment/rollout-test
```

### Debug Commands

```bash
# Pod lifecycle reasoning
kubectl get pod <pod> -o yaml | grep -A10 "status:"
kubectl describe pod <pod>              # the most information-dense view

# Controller reconciliation
kubectl describe replicaset <rs>        # why pods are being created/deleted
kubectl describe deployment <dep>       # revision history and conditions

# Check if the problem is the probe
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].livenessProbe}'
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].readinessProbe}'
```

### Resources

- [Pod lifecycle – docs](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Configure liveness, readiness, and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Deployments – docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- **Tools:** `kubectl describe`, `kubectl logs --previous`, `kubectl rollout`

---

## Week 9 – Kubernetes Networking (CNI)

**CKA:** Networking | **CKS:** Network policies

### Learn

| Component | What it does |
|-----------|-------------|
| CNI plugin | Assigns pod IPs and sets up networking on each node |
| Flannel (K3s default) | VXLAN overlay — wraps pod traffic in UDP |
| kube-proxy | Programs iptables/IPVS rules for Service → Pod routing |
| CoreDNS | Resolves `service.namespace.svc.cluster.local` |
| ClusterIP | Virtual IP backed by iptables/IPVS rules — not a real interface |

**Service routing chain:** `client pod` → iptables DNAT rule → real `pod IP` on the correct node.

### Lab Exercises

#### Exercise 1 – Trace pod-to-pod connectivity

```bash
# Deploy two pods
kubectl run pod-a --image=nicolaka/netshoot -- sleep 3600
kubectl run pod-b --image=nginx

POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')

# From pod-a, trace the path to pod-b
kubectl exec -it pod-a -- ping -c3 $POD_B_IP
kubectl exec -it pod-a -- traceroute $POD_B_IP
kubectl exec -it pod-a -- curl http://$POD_B_IP
```

#### Exercise 2 – Mismatched Service selector (silent routing failure)

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: backend
  labels:
    app: backend
    version: v2      # note: version label
spec:
  containers:
  - name: server
    image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
    version: v1      # mismatched! v1 ≠ v2 — no endpoints
  ports:
  - port: 80
    targetPort: 80
EOF

# Service exists but traffic goes nowhere
kubectl get endpoints backend-svc       # will show: <none>
kubectl exec -it pod-a -- curl http://backend-svc    # Connection refused

# Fix by correcting the selector
kubectl patch service backend-svc -p '{"spec":{"selector":{"app":"backend","version":"v2"}}}'
kubectl get endpoints backend-svc       # now shows the pod IP
```

#### Exercise 3 – Kill the CNI daemon (Flannel)

```bash
# K3s uses Flannel — find the flannel pod
kubectl get pods -n kube-system | grep flannel

# Scale it down on one node (delete the pod)
kubectl delete pod -n kube-system <flannel-pod-on-worker-1>

# Immediately try pod-to-pod communication from that worker's pods
# New pods on that node may fail to get IPs

# Watch the DaemonSet recreate the Flannel pod
kubectl get pods -n kube-system -w | grep flannel
```

#### Exercise 4 – Break and restore CoreDNS

```bash
# Scale CoreDNS to 0
kubectl scale deployment coredns -n kube-system --replicas=0

# All DNS-based communication fails
kubectl exec -it pod-a -- nslookup backend-svc
kubectl exec -it pod-a -- curl http://backend-svc

# But IP-based still works
kubectl exec -it pod-a -- curl http://$(kubectl get svc backend-svc -o jsonpath='{.spec.clusterIP}')

# Restore
kubectl scale deployment coredns -n kube-system --replicas=2
kubectl exec -it pod-a -- nslookup backend-svc    # resolves again
```

### Debug Commands

```bash
# Full trace: pod → service → endpoint → node
kubectl get svc <svc>
kubectl get endpoints <svc>
kubectl describe endpoints <svc>        # lists individual pod IPs + ports

# iptables rules for a service
SVC_IP=$(kubectl get svc <svc> -o jsonpath='{.spec.clusterIP}')
iptables -t nat -L -n | grep $SVC_IP

# CNI debug
ls /etc/cni/net.d/                      # CNI config files
ls /opt/cni/bin/                        # CNI plugin binaries
journalctl -u k3s | grep -i "cni\|flannel\|vxlan" | tail -20
```

### Resources

- [CNI specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)
- [Kubernetes Services – docs](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes networking with Flannel](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md)
- [A visual guide to Kubernetes networking – Ahmet Alp Balkan](https://speakerdeck.com/thockin/illustrated-guide-to-kubernetes-networking)
- **Tools:** `netshoot` image, `iptables`, `ipvsadm`, `tcpdump`, `wireshark`

---

## Week 10 – Storage (CSI & PVCs)

**CKA:** Storage | **CKS:** Secure mounts

### Learn

| Term | Meaning |
|------|---------|
| PV | PersistentVolume — actual storage resource |
| PVC | PersistentVolumeClaim — pod's request for storage |
| StorageClass | Template for dynamic provisioning |
| CSI | Container Storage Interface — plugin that provisions/mounts volumes |
| Binding modes | Immediate vs WaitForFirstConsumer |

**Lifecycle:** PVC created → StorageClass triggers CSI provisioner → PV created → bound to PVC → pod mounts the PV.

### Lab Exercises

#### Exercise 1 – Trace a PVC from creation to mount

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path   # K3s default
EOF

# Watch the binding
kubectl get pvc test-pvc -w
kubectl describe pvc test-pvc

# Now mount it in a pod
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pvc-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo hello > /data/test.txt; sleep 3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF
```

#### Exercise 2 – PVC Pending forever (no matching StorageClass)

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stuck-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: nonexistent-class   # doesn't exist
EOF

kubectl get pvc stuck-pvc    # shows: Pending
kubectl describe pvc stuck-pvc | grep -A5 Events
# "no persistent volumes available... provisioner: nonexistent-class/..."
```

#### Exercise 3 – Permission denied on mount

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: perm-denied
spec:
  securityContext:
    runAsUser: 1000           # non-root user
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls -la /data; touch /data/file; sleep 3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF

kubectl logs perm-denied     # observe permission denied vs success
kubectl exec -it perm-denied -- ls -la /data   # fsGroup sets group ownership
```

#### Exercise 4 – Inspect CSI logs

```bash
# K3s uses local-path-provisioner
kubectl get pods -n kube-system | grep local-path
kubectl logs -n kube-system <local-path-provisioner-pod> | tail -30

# Watch provisioner work when PVC is created
kubectl logs -n kube-system <local-path-provisioner-pod> -f &
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: watch-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Mi
  storageClassName: local-path
EOF
```

### Debug Commands

```bash
# PVC not binding
kubectl describe pvc <pvc>              # Events section tells you why
kubectl get pv                          # is a PV available?
kubectl get storageclass                # is the class there?

# Pod can't mount
kubectl describe pod <pod> | grep -A10 Events
# "Unable to mount volumes" or "timeout expired waiting for volumes"

# Node-level mount issues
journalctl -u k3s | grep -i "mount\|csi\|volume" | tail -30
mount | grep <pvc-name>                 # is it mounted?
```

### Resources

- [Kubernetes storage concepts](https://kubernetes.io/docs/concepts/storage/)
- [CSI driver development](https://kubernetes-csi.github.io/docs/)
- [Persistent volumes – docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner)
- [Longhorn documentation](https://longhorn.io/docs/latest/)
- **Tools:** `kubectl describe pvc/pv`, `mount`, `lsblk`, CSI driver logs

---

## Week 11 – Security Failures (CKS Core)

**CKA:** Light overlap | **CKS:** Main focus

### Learn

| Concept | What it means |
|---------|--------------|
| RBAC | Role-Based Access Control — who can do what to which resources |
| AuthN | Authentication — who are you? (certificates, tokens, OIDC) |
| AuthZ | Authorization — are you allowed? (RBAC) |
| Admission | After AuthZ — webhooks/policies that can mutate or reject |
| Pod Security Standards | restricted / baseline / privileged — enforced via labels |

**RBAC evaluation:** Request → API server verifies identity (AuthN) → checks RBAC rules (AuthZ) → if allowed, admission webhooks run → object stored in etcd.

### Lab Exercises

#### Exercise 1 – RBAC: remove a permission and watch it break

```bash
# Create a service account with limited access
kubectl create serviceaccount limited-sa
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=default:limited-sa

# Test what the SA can do
kubectl auth can-i list pods --as=system:serviceaccount:default:limited-sa
kubectl auth can-i create pods --as=system:serviceaccount:default:limited-sa  # should say: no
kubectl auth can-i list secrets --as=system:serviceaccount:default:limited-sa # should say: no

# Remove a permission and verify
kubectl delete rolebinding pod-reader-binding
kubectl auth can-i list pods --as=system:serviceaccount:default:limited-sa   # now: no
```

#### Exercise 2 – Pod Security Standards enforcement

```bash
# Label a namespace with restricted standards
kubectl create namespace restricted-ns
kubectl label namespace restricted-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# Try to run a privileged pod in it
kubectl apply -n restricted-ns -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true    # violates restricted policy
EOF
# Should be rejected with admission error
```

#### Exercise 3 – Capability drops

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: no-caps
spec:
  containers:
  - name: app
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        drop: ["ALL"]           # drop every Linux capability
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
EOF

kubectl exec -it no-caps -- ping 8.8.8.8     # EPERM — no NET_RAW capability
kubectl exec -it no-caps -- tcpdump           # EPERM — no NET_ADMIN
kubectl exec -it no-caps -- id                # non-root
```

#### Exercise 4 – Break admission: malformed webhook

```bash
# List existing admission webhooks
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Observe what happens with a misconfigured webhook
# A webhook with failurePolicy=Fail + unreachable service blocks ALL creates
# (To safely simulate, use failurePolicy=Ignore)
kubectl apply -f - <<'EOF'
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: test-webhook
webhooks:
- name: test.example.com
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
  clientConfig:
    url: "https://127.0.0.1:9999/validate"   # unreachable
  failurePolicy: Ignore    # safe: allows creation even if webhook fails
  admissionReviewVersions: ["v1"]
  sideEffects: None
EOF

# Clean up
kubectl delete validatingwebhookconfiguration test-webhook
```

### Debug Commands

```bash
# AuthN vs AuthZ failures
# AuthN failure: "Unauthorized" (401) — bad certificate or token
# AuthZ failure: "Forbidden" (403) — authenticated but not permitted

# Check what a user/SA can do
kubectl auth can-i <verb> <resource> --as=<user>
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>

# View API server audit logs
# In K3s, enable audit logging in /etc/rancher/k3s/config.yaml
# kube-apiserver-arg: ["audit-log-path=/var/log/k3s-audit.log"]

# Admission chain
kubectl api-resources | grep admission
kubectl get mutatingwebhookconfigurations -o yaml
kubectl get validatingwebhookconfigurations -o yaml
```

### Resources

- [RBAC authorization – docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Admission controllers reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Linux capabilities man page](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [CKS curriculum – CNCF](https://github.com/cncf/curriculum)
- **Tools:** `kubectl auth can-i`, `kubectl auth reconcile`, audit logs, `trivy`, `kubeaudit`

---

## Week 12 – Chaos & Exam Simulation

**CKA:** Full coverage | **CKS:** Advanced reasoning

### Chaos Scenarios

Run these in combination. Do NOT look up answers mid-session.

#### Scenario A – Kill kubelet on a worker

```bash
# On a worker node
sudo systemctl stop k3s-agent

# Observe on control plane
kubectl get nodes    # node goes NotReady after ~40s
kubectl get pods -o wide    # pods on that node show Unknown

# Restore
sudo systemctl start k3s-agent
kubectl get nodes    # returns to Ready
```

#### Scenario B – Network partition (block pod CIDR traffic)

```bash
# Get pod CIDR for your cluster
kubectl get node -o jsonpath='{.items[*].spec.podCIDR}'

# On a worker node, block inter-node pod traffic
# Replace with your actual pod CIDR
iptables -I FORWARD -d 10.42.0.0/16 -j DROP

# Pods on other nodes can't reach pods on this node
# Restore
iptables -D FORWARD -d 10.42.0.0/16 -j DROP
```

#### Scenario C – Break DNS + storage simultaneously

```bash
# Step 1: Scale CoreDNS to 0
kubectl scale deployment coredns -n kube-system --replicas=0

# Step 2: Delete all PVCs in a test namespace
kubectl delete pvc -n default --all

# Now diagnose systematically:
# - Are services reachable by IP? (tests networking)
# - Are services reachable by name? (tests DNS)
# - Do pods with PVCs start? (tests storage)
# Fix in order: storage → DNS → validate end-to-end
```

### 3-Hour CKA Simulation Rules

```
Time limit: 3 hours
No notes, no bookmarks (kubernetes.io/docs is allowed during real exam)
Tasks to fix (set these up before starting the timer):
  [ ] 1. A node is NotReady — find why and fix it
  [ ] 2. A deployment has 0/3 pods Running — fix it
  [ ] 3. A Service has no endpoints — fix it
  [ ] 4. A PVC is Pending — fix it
  [ ] 5. A pod CrashLoops — fix it without redeploying
  [ ] 6. DNS resolution fails — fix CoreDNS
  [ ] 7. Create a NetworkPolicy that allows only frontend→backend
  [ ] 8. Give a ServiceAccount read-only access to pods
  [ ] 9. A node has DiskPressure — clear it without rebooting
  [ ] 10. etcd backup — snapshot and verify
```

### Scoring Yourself

| Result | Meaning |
|--------|---------|
| Fixed in <5 min | You know this domain |
| Fixed in 5-15 min | Solid, but practice speed |
| Fixed in >15 min | Review this week's material |
| Couldn't fix | Re-do that week's break/fix exercises |

### Resources

- [CKA Curriculum – CNCF](https://github.com/cncf/curriculum)
- [CKS Curriculum – CNCF](https://github.com/cncf/curriculum)
- [killer.sh CKA simulator](https://killer.sh) — closest to the real exam
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes the Hard Way – Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way)

---

## Postmortem Template

Copy this into `postmortem.md` in every week's lab directory after each failure session.

```markdown
# Postmortem – [Week N: Topic]

**Date:**
**Lab scenario:**

## 1. What symptom appeared first?
<!-- The first observable sign — what you saw in kubectl/logs/metrics -->

## 2. What signal confirmed root cause?
<!-- The specific command/file/metric that proved you were right -->

## 3. What fix restored state?
<!-- Exact commands or change made -->

## 4. What guardrail prevents recurrence?
<!-- Resource limits / alerts / network policies / RBAC / monitoring rule -->

## Investigation path
<!-- The actual commands you ran, in order, including dead ends -->

## Time to diagnose:
## Time to fix:
## Exam-readiness score (1–5):
```

---

## Master Resource List

### Official Documentation

| Topic | Link |
|-------|------|
| Kubernetes docs | https://kubernetes.io/docs |
| kubectl reference | https://kubernetes.io/docs/reference/kubectl/ |
| CKA/CKS curriculum | https://github.com/cncf/curriculum |
| K3s documentation | https://docs.k3s.io |
| Linux kernel docs | https://www.kernel.org/doc/html/latest/ |
| cgroup v2 reference | https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html |
| man7.org (Linux manpages) | https://man7.org/linux/man-pages/ |

### Key Tools (install on Pi nodes)

```bash
# System performance
sudo apt install -y sysstat htop iotop ncdu tree

# Network debugging
sudo apt install -y tcpdump traceroute dnsutils netcat-openbsd

# Container debugging
sudo apt install -y jq

# stress testing
sudo apt install -y stress stress-ng

# BCC/eBPF tools (optional, powerful)
sudo apt install -y bpfcc-tools linux-headers-$(uname -r)
```

### Curated Reading List

| Author/Source | Why Read It |
|--------------|-------------|
| [Brendan Gregg – Linux Performance](https://www.brendangregg.com/linuxperf.html) | The definitive Linux performance reference |
| [Julia Evans – Zines & Blog](https://jvns.ca) | Linux/networking concepts explained clearly |
| [Ivan Velichko – iximiuz.com](https://iximiuz.com) | Container internals at the right depth |
| [Kubernetes Blog](https://kubernetes.io/blog/) | Official deep-dives on features |
| [lwn.net](https://lwn.net) | Kernel-level articles (OOM, scheduling, cgroups) |
| [The Linux Command Line – W. Shotts](https://linuxcommand.org/tlcl.php) | Free, thorough Linux foundations book |

### Debugging Cheat Sheet

```
Pod not starting?
  → describe pod → check Events section
  → check node has capacity: describe node → Allocated resources
  → check image exists: crictl pull <image>

Pod CrashLooping?
  → kubectl logs <pod> --previous
  → kubectl describe pod → check exit codes
  → exec into the image with a sleep override

Service not routing?
  → kubectl get endpoints <svc>   # if empty: selector mismatch
  → kubectl describe endpoints <svc>
  → test by IP to isolate DNS vs routing

Node NotReady?
  → kubectl describe node → check Conditions
  → ssh to node → systemctl status k3s-agent
  → journalctl -u k3s-agent -n 100

PVC Pending?
  → kubectl describe pvc → check Events
  → kubectl get storageclass
  → check provisioner pod logs

High CPU but pod metrics look fine?
  → SSH to node → cat /sys/fs/cgroup/.../cpu.stat (nr_throttled)
  → It's throttling, not contention
```

---

*Track progress: after each week, mark the outcome in the postmortem. After 12 weeks you will have debugged every major Kubernetes failure domain under real conditions.*
