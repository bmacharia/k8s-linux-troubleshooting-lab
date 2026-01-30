Below is a **12-week execution plan** that does **three things in parallel** every single week:

1. Builds **real Linux + Kubernetes troubleshooting skill**
2. Maps **directly to CKA / CKS objectives**
3. Uses a **Pi + K3s home-lab failure curriculum** (hands-on, break/fix)

---

# 🧱 WEEK 0 – LAB BASELINE (DO THIS FIRST)

**Goal:** A known-good cluster you _will_ break repeatedly

**Your Pi/K3s Setup**

- 1 control-plane Pi
- 2–3 worker Pis
- containerd
- CoreDNS
- Traefik or nginx ingress
- Prometheus + Grafana
- Local-path or Longhorn storage

📌 Snapshot SD cards or etcd before starting
This is your **restore point**.

---

# 🗓️ 12-WEEK EXECUTION PLAN

---

## **Week 1 – Linux Process & CPU Failures**

**CKA:** Node troubleshooting
**CKS:** Process behavior, least privilege

### Learn

- Load average vs CPU %
- Run queues
- Zombie/orphan processes
- CPU throttling vs starvation

### Break the Lab

- Run a fork bomb inside a container
- Pin CPU with `stress-ng`
- Set absurd CPU limits

### Debug

- Identify throttling vs contention
- Trace pod → container → process → cgroup

📌 **Outcome:**
You can explain _why_ CPU spikes don’t match pod metrics.

---

## **Week 2 – Linux Memory & OOM Killers**

**CKA:** Pod lifecycle
**CKS:** Resource isolation

### Learn

- RSS vs cache
- Kernel OOM logic
- cgroup memory enforcement
- Evictions vs OOMKills

### Break the Lab

- Memory leak in a pod
- Set memory limit lower than heap
- Disable swap

### Debug

- Read `/sys/fs/cgroup`
- Interpret `OOMKilled=true`
- Distinguish node pressure vs container kill

📌 **Outcome:**
You stop blaming Kubernetes for Linux behavior.

---

## **Week 3 – Disk, Filesystems & Inodes**

**CKA:** Node health
**CKS:** Storage isolation

### Learn

- Inodes vs blocks
- Disk pressure conditions
- Container filesystem layers

### Break the Lab

- Fill `/var/lib/kubelet`
- Exhaust inodes
- Corrupt a writable layer

### Debug

- Detect DiskPressure early
- Recover without reboot

📌 **Outcome:**
You fix “disk full” _before_ pods start dying.

---

## **Week 4 – Linux Networking Fundamentals**

**CKA:** Cluster networking
**CKS:** Network boundaries

### Learn

- TCP handshake
- conntrack tables
- NAT + ephemeral ports

### Break the Lab

- Exhaust conntrack
- Drop SYN packets
- DNS resolution failure

### Debug

- `ss`, `ip route`, `tcpdump`
- Identify kernel vs pod networking

📌 **Outcome:**
You can prove _where_ packets die.

---

## **Week 5 – Containers & Runtimes**

**CKA:** Runtime troubleshooting
**CKS:** Image trust & isolation

### Learn

- containerd internals
- PID 1 issues
- Exec vs attach

### Break the Lab

- Crash PID 1
- Broken image manifest
- Image pull auth failures

### Debug

- CRI logs
- Runtime vs kubelet boundary

📌 **Outcome:**
You debug containers without kubectl crutches.

---

## **Week 6 – Kubernetes Control Plane**

**CKA:** Core architecture
**CKS:** Admission & auth

### Learn

- etcd read/write path
- API server request flow
- Controllers vs scheduler

### Break the Lab

- Stop etcd
- Add latency to API server
- Break controller-manager

### Debug

- Identify reconciliation failures
- Explain “kubectl hangs”

📌 **Outcome:**
You know _why_ deleting pods doesn’t fix things.

---

## **Week 7 – Scheduler & Resource Failures**

**CKA:** Scheduling
**CKS:** Policy enforcement

### Learn

- Requests vs limits
- Pending pod reasons
- Taints & tolerations

### Break the Lab

- Unschedulable pods
- Overcommit CPU
- Node pressure taints

### Debug

- Scheduler logs
- Events vs reality

📌 **Outcome:**
You can fix Pending pods in minutes.

---

## **Week 8 – Pod Lifecycle & Controllers**

**CKA:** Workloads
**CKS:** Secure defaults

### Learn

- Init containers
- Probes (and probe lies)
- Restart policies

### Break the Lab

- Bad readiness probe
- CrashLoop via config
- Broken init container

### Debug

- Pod lifecycle reasoning
- Controller reconciliation

📌 **Outcome:**
You debug without redeploying blindly.

---

## **Week 9 – Kubernetes Networking (CNI)**

**CKA:** Networking
**CKS:** Network policies

### Learn

- CNI responsibilities
- kube-proxy vs IPVS
- Services vs endpoints

### Break the Lab

- Kill CNI daemon
- Mismatch Service selectors
- Break CoreDNS

### Debug

- Trace pod → service → endpoint → node

📌 **Outcome:**
You stop guessing with “network issues.”

---

## **Week 10 – Storage (CSI & PVCs)**

**CKA:** Storage
**CKS:** Secure mounts

### Learn

- CSI lifecycle
- PVC binding modes
- Permission models

### Break the Lab

- PVC Pending forever
- CSI crash
- Permission denied mounts

### Debug

- Node vs controller CSI logs

📌 **Outcome:**
You fix storage without nuking data.

---

## **Week 11 – Security Failures (CKS Core)**

**CKA:** Light overlap
**CKS:** Main focus

### Learn

- RBAC evaluation order
- Pod Security Standards
- Capability drops

### Break the Lab

- Remove permissions
- Break admission
- Deny network traffic

### Debug

- AuthN vs AuthZ
- Admission vs runtime

📌 **Outcome:**
You can explain _why access fails_.

---

## **Week 12 – Chaos & Exam Simulation**

**CKA:** Full coverage
**CKS:** Advanced reasoning

### Do

- Kill kubelet
- Partition nodes
- Break DNS + storage simultaneously

### Simulate

- 3-hour CKA-style run
- No notes
- Fix only what matters

📌 **Outcome:**
You are calm under fire.

---

# 🧪 HOME-LAB FAILURE CURRICULUM (REPEATABLE)

Every failure must answer:

1. **What symptom appears first?**
2. **What signal confirms root cause?**
3. **What fix restores state?**
4. **What guardrail prevents recurrence?**

Write this every time.

---

# 🎯 REAL-WORLD GAPS THIS FILLS (EXAMS DON’T)

| Skill                  | Exam | Reality |
| ---------------------- | ---- | ------- |
| Debug without redeploy | ❌   | ✅      |
| Kernel-level reasoning | ❌   | ✅      |
| Silent failures        | ❌   | ✅      |
| Partial outages        | ❌   | ✅      |
| Root cause analysis    | ❌   | ✅      |

---
