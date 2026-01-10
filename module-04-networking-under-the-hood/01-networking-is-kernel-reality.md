# Networking Is Kernel Reality

Kubernetes does not move packets.

This statement is not philosophical.  
It is a mechanical boundary.

Every byte that enters or leaves a Pod is ultimately handled by the Linux kernel.  
Kubernetes influences *how* the kernel behaves, but it never replaces it.

Understanding Kubernetes networking begins by identifying where Kubernetes stops
and where the kernel becomes the final authority.

---

## Where packets actually originate

A process inside a container makes a network call.

From the system’s point of view, this is not a “Pod sending traffic”.
It is a userspace process issuing a syscall.

The kernel receives a request to:
- open a socket
- write bytes
- route packets

At this point:
- Kubernetes is not involved
- CNI is not involved
- kube-proxy is not involved

It is pure Linux networking.

---

## Network namespaces: the first illusion

Containers appear isolated because they run inside **network namespaces**.

A network namespace provides:
- its own interfaces
- its own routing table
- its own ARP cache
- its own conntrack state

From inside the container:
- `eth0` looks real
- routes look local
- the world appears private

This isolation is an illusion created by the kernel.

Kubernetes did not invent it.
Kubernetes merely asks the kernel to create it.

---

## The veth boundary

A Pod network namespace is not connected to the world directly.

It is connected through a **veth pair**.

A veth pair is:
- two virtual interfaces
- connected back-to-back
- existing in different namespaces

One end lives inside the Pod namespace as `eth0`.  
The other end lives in the node namespace.

Packets crossing this boundary are no longer “Pod traffic”.
They are node traffic.

This is the first hard boundary in Kubernetes networking.

---

## What Kubernetes actually controls

Kubernetes never forwards packets.

Instead, it:
- asks the CNI plugin to configure interfaces
- programs routes
- installs filtering and NAT rules
- sets policy constraints

Once those are installed, Kubernetes steps away.

From that moment onward:
> **The kernel decides what happens to every packet.**

---

## Why this boundary matters

This explains several behaviors that confuse people:

- Stopping kubelet does not stop traffic
- Restarting kube-proxy does not drop connections
- Killing control-plane components does not affect data-plane flow

Once the kernel is configured, it continues to forward packets
until something explicitly changes kernel state.

Kubernetes does not sit in the data path.

---

## Where networking enforcement actually happens

Every networking decision ultimately resolves to one of these kernel mechanisms:

- routing tables
- neighbor tables
- iptables / nftables
- eBPF programs
- conntrack state

If you cannot point to one of these, the networking behavior is imaginary.

This rule will apply to:
- Pod-to-Pod traffic
- Services
- NodePort
- LoadBalancer
- NetworkPolicy

Every abstraction must collapse into kernel behavior.

---

## The invariant

Kubernetes networking is not a network stack.

It is a **control system that configures the Linux network stack**.

Packets obey the kernel.
Kubernetes only sets the conditions.

---

## Why this framing is necessary

Without this boundary, it becomes impossible to reason about:

- why traffic still flows when components crash
- why some policies apply immediately and others do not
- why debugging requires `ip`, `route`, `iptables`, and `tcpdump`
- why networking bugs feel nondeterministic to beginners

Once the boundary is clear, the system becomes predictable.

---

## What comes next

With the kernel boundary established, the next step is inevitable:

> **How a Pod is actually connected to the node**

That requires tracing:
- `eth0`
- veth pairs
- routing tables
- and the node network namespace

Next file:


