# ServiceAccounts Are Identities

A Pod does not possess permissions.  
A Pod does not authenticate itself.  
A Pod is not a security principal.

Yet Pods routinely interact with the Kubernetes API and perform privileged actions.  
This contradiction disappears only when identity is traced to its actual origin and enforcement boundary.

---

## Identity does not originate at runtime

Identity in Kubernetes is not established when a container starts running.  
It is established **before execution**, during Pod admission.

When the API server accepts a Pod specification, exactly one identity decision is made:

spec.serviceAccountName


If this field is absent, Kubernetes injects one implicitly.

From that point forward, the Pod will **only ever act as that ServiceAccount**.  
This decision is immutable for the lifetime of the Pod.

The Pod itself never becomes an identity.

---

## Pods borrow identity, they do not own it

ServiceAccounts are Kubernetes identities.  
Pods are execution artifacts.

This separation mirrors earlier boundaries:
- Controllers own intent, Pods execute it
- ServiceAccounts own identity, Pods borrow it

Restarting a Pod does not change identity.  
Rescheduling a Pod does not change identity.  
Changing container images does not change identity.

Identity is orthogonal to execution.

---

## How identity is materialized inside a Pod

Once a Pod is admitted and assigned to a node, kubelet becomes responsible for realizing the identity decision.

The sequence is concrete and mechanical:

1. kubelet resolves the ServiceAccount
2. kubelet requests a token for that ServiceAccount
3. kubelet writes the token to disk
4. kubelet mounts that file into the Pod

No networking is involved.  
No RBAC is involved.  
No API authorization occurs here.

This is a filesystem operation.

If the token is not mounted, the Pod has no identity.

---

## The kernel boundary

Inside the container, the token is just a file.

At this point, Kubernetes is no longer participating.

The Linux kernel:
- tracks the inode
- enforces file permissions
- serves `read()` syscalls

Kubernetes does not protect the token after it is mounted.  
There is no runtime guardrail.

Once mounted, identity becomes the responsibility of the process.

This is a hard boundary.

---

## How identity leaves the Pod

A Pod does not announce who it is.  
It presents proof.

When a container makes an API call:

1. the process reads the token file
2. the token is attached to an HTTP request
3. the request exits the node as TCP packets
4. the kernel routes the packets
5. the API server receives the bytes

Up to this point:
- identity has not been evaluated
- RBAC has not run
- no trust decision has been made

It is just traffic.

---

## Where identity is actually evaluated

The **first Kubernetes component that evaluates identity** is the API server.

Not kubelet.  
Not etcd.  
Not the kernel.

The API server:
- parses the request
- extracts the token
- validates it
- reconstructs the identity

At this moment, the Pod ceases to matter.

The request is no longer associated with a Pod.  
It is associated with an identity string:

system:serviceaccount:<namespace>:<name>


That string is the security subject.

---

## The enforcement boundary

Once identity is resolved, two things may happen:

1. RBAC evaluates the request
2. The request is either rejected or allowed to mutate cluster state

If RBAC denies the request:
- the API server rejects it
- nothing is written to etcd
- no controller reacts
- no kernel action occurs

If RBAC allows the request:
- etcd may be mutated
- controllers may act
- kubelet may eventually create or destroy processes

RBAC’s enforcement boundary is the API server.

Everything beyond that point is consequence, not control.

---

## Where Kubernetes can no longer intervene

Once the API server accepts a request:

- identity has been trusted
- authorization has passed
- intent has been recorded

From this point onward:
- RBAC is done
- identity cannot be re-evaluated
- Kubernetes cannot revoke the action

Control moves to reconciliation.

This is why identity mistakes are catastrophic.  
They fail **before** safety mechanisms can engage.

---

## The invariant

Pods do not have permissions.  
Pods borrow identity.

That identity is:
- decided at admission
- materialized as a file
- carried by the kernel
- trusted exactly once
- enforced at the API server

After that, Kubernetes stops negotiating.

---

## Why this boundary matters

This explains, without abstraction:

- why disabling `automountServiceAccountToken` is powerful
- why token leakage is worse than config leakage
- why restarting Pods does not fix identity compromise
- why RBAC misconfigurations are silent and dangerous

Identity is established before execution,  
enforced at the control plane boundary,  
and irreversible once accepted.

