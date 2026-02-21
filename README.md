# 📘 Short Notes — PID Namespace (Linux Foundation for Containers)

## 1️⃣ Container Fundamental Truth

A container is:

> A normal Linux process
>
> * isolation (namespaces)
> * resource limits (cgroups)

It is **not** a VM.
It does **not** have its own kernel.

---

# 2️⃣ PID Namespace — Definition

A PID namespace provides:

> An isolated process ID number space.

Inside a PID namespace:

* Processes see only processes within that namespace.
* PID numbering starts from 1.
* The first process becomes PID 1.

---

# 3️⃣ What We Observed

### On Host (Root Namespace)

```bash
ps -e | wc -l → 90
echo $$ → 777
```

Host sees all system processes.

---

### Inside New PID Namespace

Command used:

```bash
sudo unshare --fork --pid --mount-proc bash
```

Observed:

```bash
ps
PID 1 → bash
```

Only few processes visible.

---

# 4️⃣ Key Technical Points

## A. Same Kernel

PID namespace does NOT:

* Create new kernel
* Create new OS
* Create new VM

It only changes process visibility.

---

## B. One Process, Multiple PIDs

A single process has:

* Host PID (global PID)
* Namespace PID (local PID)

Example:

| View      | PID  |
| --------- | ---- |
| Host      | 4210 |
| Namespace | 1    |

Kernel maintains PID mapping internally.

---

## C. `/proc` Behavior

`/proc` is a virtual filesystem.

* Each numeric directory = one process
* With `--mount-proc`, a fresh `/proc` is mounted
* It reflects only processes in that namespace

So:

Host `/proc` → 147 entries
Namespace `/proc` → 57 entries

Isolation is visibility-based.

---

## D. Visibility Rule (Important)

Parent namespace (host):
✔ Can see child namespace processes

Child namespace:
❌ Cannot see parent processes

This is a hierarchical isolation model.

---

# 5️⃣ Critical Container Insight

Inside container:

* Main process = PID 1
* If PID 1 exits → container stops
* If PID 1 does not handle signals → shutdown issues occur

This is why PID 1 behavior matters in Docker & Kubernetes.

---

# 6️⃣ Final Mental Model

PID namespace:

* Does not change hardware
* Does not duplicate processes
* Only changes what a process can see

It creates a “process illusion boundary.”

---

# Mount Namespace — Short Summary Notes

## 1️⃣ Definition

Mount namespace isolates:

> The mount table (filesystem view), not the physical storage.

Each mount namespace has its own independent list of mount points.

---

## 2️⃣ What Actually Changes

When you ran:

```bash
sudo unshare --mount bash
```

Kernel created:

* A new mount namespace
* A separate mount table for that shell

Nothing else changed.

---

## 3️⃣ What Is Isolated

Inside mount namespace:

* New mounts are visible only inside that namespace
* Unmounts affect only that namespace
* Bind mounts are private to that namespace

---

## 4️⃣ What Is NOT Isolated

| Component     | Shared or Isolated? |
| ------------- | ------------------- |
| Disk blocks   | Shared              |
| Files on disk | Shared              |
| Kernel        | Shared              |
| Mount table   | Isolated            |

Important:

If you modify a file inside a shared filesystem,
it affects host too.

Mount namespace does NOT duplicate files.
It only isolates mount structure.

---

## 5️⃣ Your Practical Proof

Inside namespace:

```bash
mount -t tmpfs tmpfs /mnt
touch /mnt/testfile
```

Host did NOT see:

* tmpfs mount
* testfile

Because mount table was isolated.

---

## 6️⃣ Key Concept

Mount namespace isolates:

> How filesystems are attached

It does NOT isolate:

> What data exists on disk

---

## 7️⃣ Why Containers Need This

Docker uses mount namespace to:

* Mount container root filesystem
* Mount overlay layers
* Hide host filesystem
* Provide container-specific `/proc`, `/sys`

Without mount namespace:

Containers would see full host filesystem.

---

## 8️⃣ Important Distinction

PID namespace → isolates processes
Mount namespace → isolates filesystem view
Network namespace → isolates network stack
cgroups → enforce resource limits

Together → container
---

# Network Namespace — Core Definition

> Network namespace isolates the entire network stack for a process.

It provides an independent instance of:

* Network interfaces
* IP addresses
* Routing table
* ARP table
* Netfilter (iptables)
* Port space

Same kernel.
Different TCP/IP world.

---

# What Happens When You Create One?

Command:

```bash
sudo unshare --net bash
```

Kernel does:

* Creates a new network namespace
* Initializes empty network stack
* Only `lo` (loopback) interface exists
* `lo` is DOWN by default
* No routes configured

Result:

```text
ip addr → only lo
ip route → empty
```

No internet.
No eth0.
No docker0.

---

# Key Properties

## 1️⃣ Port Space Is Namespaced

Port binding is per namespace.

So:

Namespace A:

* Process listening on :8080

Namespace B:

* Process listening on :8080

No conflict.

Conflict happens only inside same namespace.

---

## 2️⃣ Isolation Is Bidirectional for Networking

Unlike PID:

* Namespaces cannot see each other’s interfaces
* Cannot share routes
* Cannot access each other unless explicitly connected

---

## 3️⃣ Fresh Network Stack

A new network namespace has:

| Component     | State     |
| ------------- | --------- |
| Interfaces    | Only `lo` |
| Routing table | Empty     |
| Internet      | No        |
| Gateway       | No        |

Connectivity must be manually configured.

---

# How Containers Use It

When Docker starts container:

1. Creates veth pair
2. One end inside container namespace
3. Other end attached to bridge (`docker0`)
4. Assigns IP
5. Sets default route
6. Configures NAT

Then container gets internet.

---

# Kubernetes Model

* Each Pod = one network namespace
* All containers inside Pod share it
* Different Pods → different namespaces
* Same internal ports allowed across Pods

---

# Important Mental Model

Network namespace = isolated TCP/IP stack

It is not:

* New kernel
* New NIC hardware
* New VM

It is:

* Separate logical networking instance within same kernel.

---

# Minimal Diagram

Host Namespace:

```
eth0 → 172.x.x.x
docker0 → 172.17.0.1
routes present
```

New Network Namespace:

```
lo (DOWN)
no routes
no connectivity
```

---

# Critical Invariants

* Isolation is default
* Connectivity must be explicitly created
* Port conflicts are namespace-scoped
* Host port mapping crosses namespace boundary

---


When we run `docker run`, the Docker CLI communicates with the Docker daemon. The daemon prepares an OCI runtime specification and passes it to containerd, which invokes runc. runc then calls the Linux `clone()` system call with namespace flags like CLONE_NEWPID and CLONE_NEWNET. The Linux kernel creates new namespace objects in kernel memory and creates a new process attached to them. Inside a new PID namespace, the first process automatically becomes PID 1. These namespaces exist as long as at least one process references them. When the container’s init process exits, the namespaces are garbage collected by the kernel.
