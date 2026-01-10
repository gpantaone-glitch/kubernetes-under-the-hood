# Identity Comes Before Authorization

RBAC is often treated as the place where Kubernetes decides whether an action is allowed.  
That framing is convenient — and incorrect.

Kubernetes does not begin by asking *what* a request is allowed to do.  
It begins by asking *who* is making the request.

Until that question is answered, authorization is undefined.

---

## Where requests actually enter Kubernetes

Every interaction with Kubernetes starts as an HTTP request sent to the API server.

That request arrives as:
- TCP packets handled by the kernel
- reassembled into an HTTPS connection
- parsed into an HTTP request

Up to this point, Kubernetes has not made any trust decision.  
The kernel has moved bytes. Nothing more.

---

## Authentication is the first real boundary

The API server processes every request through a fixed pipeline.

The first meaningful step is **authentication**.

At this stage, the API server attempts to answer exactly one question:

> *Can I trust who this request claims to be?*

This may involve:
- validating a ServiceAccount token
- verifying a client certificate
- calling an external authentication webhook

If authentication fails:
- the request is rejected immediately
- RBAC is never evaluated
- no state is read or written

A request without identity does not “fail RBAC”.
It never reaches RBAC.

---

## Why authorization cannot run first

Authorization systems require a subject.

RBAC operates on:
- identities
- groups
- verbs
- resources
- namespaces

Without identity:
- there is no subject
- no policy can be applied
- no rule can match

This is why authentication is non-negotiable.
Authorization without identity is meaningless.

---

## Where RBAC actually sits

RBAC is evaluated **only after** authentication succeeds.

At that point, the API server has constructed a request context that includes:
- a resolved identity
- associated groups
- the requested action
- the target resource and scope

RBAC is then applied as a filter:
- if any rule allows the action → proceed
- if no rule allows the action → deny

RBAC does not infer intent.
It does not consult runtime state.
It does not examine workloads.

It performs pure matching against declared rules.

---

## The enforcement boundary

If RBAC denies a request:
- the API server returns an error
- etcd is untouched
- no controller reacts
- no workload changes occur

If RBAC allows a request:
- the API server may mutate etcd
- controllers may reconcile
- kubelet may eventually act
- the kernel executes the result

Once RBAC allows a request, its job is finished.

There is no re-evaluation.
There is no runtime enforcement.
There is no rollback.

---

## Where Kubernetes stops caring

After a request is accepted:
- identity is trusted
- authorization is complete
- intent is recorded

From this point onward:
- RBAC is irrelevant
- identity is fixed for this request
- Kubernetes assumes the action was intentional

Any consequences are handled by reconciliation, not security mechanisms.

---

## The invariant

Authentication establishes identity.  
Authorization constrains identity.

RBAC:
- does not create power
- does not supervise execution
- does not protect running workloads

It guards a single boundary:
> **the moment intent enters the control plane**

Everything beyond that is execution.

---

## Why this matters

This explains:
- why RBAC misconfigurations fail silently
- why leaked tokens bypass expectations
- why security issues often look like logic bugs
- why fixing RBAC cannot undo past damage

Kubernetes will not decide what you are allowed to do  
until it is convinced of who you are.

That decision happens once, at the API server boundary.

