# Tokens and Trust Boundaries

A ServiceAccount token is often described as a “credential” or a “secret”.
Both descriptions are misleading.

A token in Kubernetes is not a source of power.
It is **proof of identity** presented to the control plane exactly once per request.

Understanding this requires tracing where the token comes from, how it is validated, and—most importantly—where trust *ends*.

---

## What a ServiceAccount token actually is

A ServiceAccount token is a signed data structure issued by the Kubernetes control plane.
It contains claims that assert an identity, not permissions.

Those claims typically encode:
- the ServiceAccount name
- the namespace
- the intended audience
- an expiration time

The token does not describe *what is allowed*.
It only describes *who is speaking*.

Authorization is a separate decision.

---

## Why the token is a file

Tokens are delivered to Pods as files, not environment variables, for a reason.

The kubelet writes the token to disk and mounts it into the Pod filesystem.
From that moment on:
- Kubernetes is no longer involved
- the kernel enforces access
- the process decides whether to read or misuse it

This design deliberately places responsibility on the workload.
Kubernetes does not try to mediate runtime use of identity.

Once mounted, the token is just bytes.

---

## The kernel boundary

After the token is mounted, enforcement shifts completely.

The Linux kernel:
- governs file permissions
- handles reads
- routes network packets carrying the token

Kubernetes does not observe token usage.
It does not track reads.
It does not revoke tokens mid-flight.

This is the first trust boundary:
> **Once the token is mounted, Kubernetes assumes the process is trusted to handle it correctly.**

If that assumption is violated, Kubernetes cannot intervene.

---

## How the token is used

When a process inside the Pod wants to talk to the API server:
1. it reads the token file
2. it attaches the token to an HTTP request
3. the request leaves the node as TCP packets
4. the API server receives the request

Up to this point:
- the kernel has moved bytes
- networking has occurred
- Kubernetes has not trusted anything yet

Trust has not been granted.
It has only been *presented for consideration*.

---

## Where trust is actually granted

The API server is the sole authority that decides whether a token is trusted.

On receiving a request, the API server:
- validates the token signature
- checks expiration
- validates audience
- reconstructs the identity

If validation fails:
- the request is rejected
- RBAC is never evaluated
- nothing is written to etcd

If validation succeeds:
- identity is established
- RBAC may now apply constraints

This is the second trust boundary:
> **Tokens are trusted only at the API server. Nowhere else.**

---

## Why tokens rotate

Tokens are short-lived by design.

Rotation exists because:
- tokens cannot be revoked after issuance
- Kubernetes cannot reach into running processes
- trust must therefore decay automatically

Expiration is the only revocation mechanism Kubernetes has.

If a token leaks:
- Kubernetes cannot “pull it back”
- it must wait for expiration

This is not a weakness.
It is an explicit acknowledgment of the execution boundary.

---

## Why killing Pods does not revoke trust globally

Deleting or restarting a Pod:
- stops processes
- unmounts filesystems
- destroys containers

It does **not** invalidate tokens that were already issued.

If a token has leaked:
- restarting Pods does nothing
- redeploying workloads does nothing
- RBAC changes do nothing

The token remains valid until expiration.

This is the point where many security assumptions fail.

---

## The final enforcement boundary

Once a token is:
- validated
- accepted
- associated with an identity

Kubernetes considers the request legitimate.

From that moment onward:
- identity is fixed
- RBAC is applied
- intent is recorded
- reconciliation takes over

There is no second chance to intervene.

---

## The invariant

ServiceAccount tokens do not grant power.
They **prove identity**.

That proof:
- is materialized as a file
- carried by the kernel
- evaluated exactly once
- trusted only by the API server
- irrevocable until expiration

Every Kubernetes security decision downstream depends on this moment.

---

## Why this matters

This explains:
- why token leakage is catastrophic
- why `automountServiceAccountToken=false` is effective
- why RBAC is insufficient without identity hygiene
- why Kubernetes security failures are silent and durable

Trust, once granted, cannot be negotiated again.

That is not a flaw.
It is the cost of distributed systems.

