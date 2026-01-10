# RBAC as a Constraint System

RBAC in Kubernetes is frequently described as a permission system.  
That description is convenient — and misleading.

RBAC does not create capabilities.  
RBAC does not grant power.  

RBAC only **constrains identities that already exist**.

Understanding this requires tracing where authority comes from and where RBAC is allowed to intervene.

---

## Power exists before RBAC

Before RBAC is evaluated, several things have already happened:

- an identity has been authenticated
- a token has been trusted
- the API server has accepted the subject as real

RBAC never participates in any of these steps.

By the time RBAC is consulted, Kubernetes has already answered the hardest question:

> *Do I trust who is speaking?*

RBAC only answers a narrower one.

---

## What RBAC actually evaluates

When a request reaches the authorization phase, the API server constructs a request context:

- identity (user or ServiceAccount)
- groups
- verb
- resource
- namespace
- subresource (if any)

RBAC performs **pure matching** against this context.

It asks only:

> *Is there any rule that explicitly allows this identity to perform this action in this scope?*

There is no inference.  
There is no learning.  
There is no inspection of runtime state.

If no rule matches, the request is denied.

---

## Why RBAC feels passive

RBAC does not react to events.
It does not observe failures.
It does not watch Pods, nodes, or workloads.

It is invoked only when a request is evaluated by the API server.

This explains a common confusion:

- RBAC does not stop bad things from happening
- RBAC stops bad **requests** from being accepted

If an action does not go through the API server, RBAC is irrelevant.

---

## The enforcement boundary

RBAC enforcement ends at the API server.

If RBAC denies a request:
- the request is rejected
- etcd is untouched
- no controller reacts
- no workload changes occur

If RBAC allows a request:
- etcd may be mutated
- controllers reconcile
- kubelet eventually acts
- the kernel executes

Once RBAC allows a request, it is done.

There is no second evaluation.
There is no runtime enforcement.
There is no rollback.

---

## Why RBAC cannot protect running workloads

RBAC does not:
- intercept syscalls
- block file access
- inspect packets
- supervise processes

Once a Pod is running, RBAC has no visibility into what it does.

This is not a limitation of RBAC.
It is a consequence of Kubernetes’ architecture.

RBAC protects **control-plane intent**, not execution.

---

## Why misconfigured RBAC fails silently

RBAC has one default behavior:

> *Deny if no rule allows.*

This is safe.
It is also quiet.

When RBAC is misconfigured:
- requests fail
- controllers stall
- workloads stop progressing
- no explicit error explains why

The system behaves correctly.
The operator experiences confusion.

RBAC is unforgiving by design.

---

## The relationship to earlier invariants

RBAC mirrors patterns seen elsewhere in Kubernetes:

- ReplicaSets enforce numeric constraints
- OwnerReferences enforce deletion boundaries
- RBAC enforces action constraints

None of these systems react emotionally.
None of them infer intent.
None of them compensate for misunderstanding.

They apply rules until convergence or refusal.

---

## The invariant

RBAC does not grant authority.  
RBAC limits authenticated identities.

It operates:
- after identity is trusted
- before state is mutated
- entirely inside the API server
- with no awareness of runtime behavior

If identity is wrong, RBAC cannot save you.  
If RBAC is wrong, Kubernetes will enforce it faithfully.

---

## Why this matters

This explains:
- why RBAC fixes cannot undo past damage
- why leaked tokens bypass RBAC protections
- why runtime exploits ignore RBAC entirely
- why security failures often look like logic bugs

RBAC is not a shield around workloads.

It is a gate in front of the control plane.

Once you pass through, Kubernetes assumes you meant it.

