# Module 02 — Control Loops and Ownership

> **Format**: Reasoned failure experiments
>
> This module is written as a sequence of reasoned experiments. The failures here are logical, not physical. No nodes are rebooted. No daemons are killed. Instead, incorrect mental models are deliberately pushed until they contradict observable Kubernetes behavior.
>
> The goal is to identify *ownership boundaries* — who is responsible for correctness, and who is not.

---

## Experiment 01 — Pods are not the source of truth

### Initial assumption

My original mental model was simple and wrong: Pods felt like Kubernetes-native containers. I assumed they were the ultimate entity Kubernetes lived and died for — the source of truth. If a Pod existed, Kubernetes should preserve it. If it died, Kubernetes should bring it back.

In this model:

* Pods were equivalent to containers
* Kubernetes revolved around keeping Pods alive
* Higher-level objects felt like conveniences layered on top

If this model were correct, then deleting a Pod should trigger Kubernetes to recreate it automatically.

---

### The thought experiment

Consider the simplest possible workload: a single Pod created directly, without any controller.

If Pods are the source of truth, then deleting this Pod should be treated as a failure. Kubernetes should react by restoring the Pod to satisfy correctness.

This is the same expectation one would have from a process supervisor or an init system.

---

### What actually happens

When a standalone Pod is deleted:

* The Pod disappears
* Nothing replaces it
* No reconciliation occurs
* No error is raised

Kubernetes remains perfectly satisfied.

There is no restart. There is no recovery. There is no attempt to preserve existence.

---

### The contradiction

This behavior directly contradicts the assumption that Pods are the source of truth.

If Pods mattered intrinsically, their disappearance would violate desired state. But Kubernetes shows no such concern.

The system behaves as if nothing important was lost.

---

### Invariant discovered

> **Pods do not own their existence.**

A Pod is not a goal. It is a consequence.

Without a controller declaring intent, a Pod has no semantic importance. Kubernetes does not try to preserve it because nothing asked for it to exist in the first place.

---

### Minimal theory context

*********
What are control loops?

Control loops are systems that automatically regulate a process by continuously comparing what is happening to what should be happening, then making adjustments to reduce the difference.
They’re fundamental in engineering, biology, economics, and everyday technology.

A control loop measures output → compares it to a desired value → adjusts inputs → repeats continuously.

Why control loops matter
They allow systems to:
Adapt to disturbances
Maintain stability
Reduce human intervention
Improve efficiency and safety
Without control loops, modern automation, robotics, aviation, and even biology wouldn’t function reliably.

<pre>

[ Measure ] → [ Decide ] → [ Act ]
      ↑                       ↓
      └──────── Feedback ─────┘

</pre>
********

Kubernetes is built around control loops. Desired state lives in controllers, not in the objects they create.

A Pod created directly expresses no ongoing intent. It is a one-time request, not a contract. Once the request is satisfied, Kubernetes has nothing left to enforce.

This explains why Pods feel disposable: they are.

---

### Boundary clarified

From this point forward, a hard boundary becomes visible:

* Pods describe *how* something runs
* Controllers describe *whether* it should exist

Confusing these roles leads directly to incorrect expectations about restarts, recovery, and availability.

---

*End of Experiment 01 — Pods are not the source of truth*

