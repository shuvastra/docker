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
