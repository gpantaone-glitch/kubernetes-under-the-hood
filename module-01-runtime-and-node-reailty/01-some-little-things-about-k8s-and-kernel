# Module 01 — Runtime and Node Reality

> **Format**: Failure-driven learning
>
> This module is not a tutorial. It is a record of invariants discovered by deliberately breaking a real Kubernetes cluster. Each section follows the same structure: an assumption, a concrete failure, an observation that contradicted expectation, and the invariant that survived repeated experiments.

---

## Experiment 1 — Stopping containerd should kill running Pods

### Minimal theory context

In Kubernetes documentation, containerd is described as the *container runtime*. This wording easily leads to the mental model that containerd is analogous to a process supervisor. In reality, containerd’s responsibility ends at container **creation**, not execution. The experiment below exists to validate (or falsify) that distinction.

### Assumption

The natural assumption was simple: *containerd is the container runtime; therefore it must be the component that keeps containers running*. If that assumption were true, stopping containerd should terminate all running containers.

### Failure injected

On a node actively running production-like workloads, containerd was stopped:

```
systemctl stop containerd
```

No drain. No Pod deletion. No kubelet restart.

### Observation

Nothing meaningful happened.

* All containers continued running
* Applications continued serving traffic
* The node remained `Ready`
* No eviction occurred

The system behaved as if containerd were irrelevant to runtime execution.

### Invariant

> **Once a container is started, no userspace runtime is required to keep it alive.**

containerd participates only in container *creation*. After `runc` hands the process to the kernel, containerd is no longer part of the execution path. This immediately disproves the idea that containerd is an execution boundary.

---

## Experiment 2 — Removing the kernel kills everything instantly

### Minimal theory context

Linux containers are not virtual machines. They rely entirely on the host kernel for scheduling, memory management, and process lifecycle. If containers truly depend on the kernel in the same way normal processes do, then kernel failure should represent a hard execution boundary.

### Hypothesis

If stopping containerd does not kill containers, then something lower must be responsible for keeping them alive. The only remaining candidate is the Linux kernel.

If that hypothesis is correct, removing the kernel should terminate everything immediately.

### Failure injected

The same node was rebooted:

```
reboot
```

### Observation

* All container processes terminated instantly
* All Pods on the node disappeared
* kubelet heartbeats stopped
* The control plane eventually marked the node `NotReady`

There was no grace period at the process level. Execution stopped the moment the kernel disappeared.

### Invariant

> **The Linux kernel is the execution boundary.**

Runtime daemons can fail without killing workloads. Kernel failure kills everything without exception.

---

## Experiment 3 — Dead Pods did not immediately move

### Minimal theory context

Kubernetes markets itself as a self-healing system. A common interpretation of this claim is that Kubernetes reacts immediately to failures. This experiment tests whether Kubernetes optimizes for *speed* or *stability* when nodes fail.

### Assumption

If all containers on a node die, Kubernetes should immediately reschedule those Pods onto healthy nodes.

This assumption treats Kubernetes as a reactive system.

### Observation

After the reboot:

* Pods were gone from the node
* Replacement Pods were *not* immediately scheduled
* Several minutes passed with no rescheduling activity

The system appeared to stall.

### Invariant

> **Kubernetes treats node failure as potentially temporary.**

Immediate rescheduling is intentionally avoided to prevent cascading instability during transient failures.

---

## Experiment 4 — The five‑minute silence

### Minimal theory context

Kubernetes models node unavailability using taints and tolerations rather than instantaneous eviction. This mechanism exists to prevent unnecessary rescheduling during transient failures such as network partitions or short reboots.

### Observation refined

The delay before rescheduling was not random. Across repeated failures, it consistently aligned with a fixed window.

### Explanation uncovered

Most Pods implicitly tolerate the `node.kubernetes.io/not-ready:NoExecute` taint for approximately five minutes.

During this window:

* Pods are not evicted
* The scheduler is not involved
* Kubernetes waits for node recovery

### Invariant

> **Eviction is policy-driven, not reactive.**

Kubernetes prefers stability over speed by default.

---

## Experiment 5 — Node recovers before eviction

### Minimal theory context

Pod scheduling and Pod execution are decoupled in Kubernetes. As long as a Pod object remains bound to a node, the scheduler has no authority to intervene. This experiment explores that boundary.

### Failure injected

The node was rebooted and allowed to recover *before* the toleration window expired.

### Observation

* kubelet reconnected
* The node returned to `Ready`
* Pods restarted locally
* No rescheduling occurred

### Invariant

> **As long as eviction does not occur, Pod ownership does not change.**

In this scenario, the scheduler is never involved. kubelet simply reconciles local state.

---

## Experiment 6 — Eviction is irreversible

### Minimal theory context

Eviction is implemented by deleting the Pod object. Deletion is a semantic boundary in Kubernetes: once crossed, ownership and scheduling history are discarded. This experiment validates eviction as a point of no return.

### Failure injected

The node was kept offline beyond the toleration window.

### Observation

* The Node Controller evicted the Pods
* Pod objects were deleted
* New Pods were created
* The scheduler selected new nodes

### Invariant

> **Eviction is a one‑way door.**

Once a Pod is evicted, its original node is permanently out of the decision path.

---

## Experiment 7 — Pod Disruption Budgets override operators

### Minimal theory context

Pod Disruption Budgets are not safety nets; they are contracts. They encode explicit availability guarantees that Kubernetes will not violate, even when commanded to do so by an operator.

### Failure injected

A Pod Disruption Budget was applied with:

* `replicas = 1`
* `minAvailable = 1`

A node drain was attempted.

### Observation

* Drain stalled
* No eviction occurred
* Kubernetes refused to proceed

### Invariant

> **PDBs protect intent, not progress.**

They apply only to voluntary disruptions and explicitly prioritize availability over operational convenience.

---

## Experiment 8 — StatefulSets refuse to move

### Minimal theory context

StatefulSets exist to preserve identity and state across failures. Unlike Deployments, they intentionally restrict scheduling freedom to prevent data corruption. This experiment examines the cost of that guarantee.

### Failure injected

A StatefulSet-backed Pod with persistent storage was disrupted.

### Observation

* Replacement Pod retained the same identity
* Scheduling was constrained
* Pod remained Pending when storage could not move

### Invariant

> **StatefulSets sacrifice availability to preserve correctness.**

They are deliberately conservative. Speed is dangerous when state is involved.

---

## Consolidated invariants

Across all failures, the following truths held without exception:

* Kubernetes coordinates intent; Linux executes reality
* containerd creates containers; it does not keep them alive
* the kernel is the execution boundary
* node health matters more than daemon health
* eviction is delayed by policy, not accident
* eviction permanently changes ownership
* availability guarantees can block operators

---

## Closing

Nothing in this module describes a bug. Every outcome is the result of an explicit trade‑off.

Kubernetes becomes predictable only when it is understood as a system of invariants, not features.

*End of Module 01 — Runtime and Node Reality*

