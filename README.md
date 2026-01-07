# Kubernetes from First Principles

This repository documents my journey of learning Kubernetes by starting
from Linux kernel behavior and building up to higher-level Kubernetes
abstractions.

The goal of this repository is **not** to list tools or provide YAML
snippets, but to understand:

- how containers actually run
- how Kubernetes controls (but does not execute) workloads
- how networking really works at the packet level
- why failures behave the way they do

All notes are written from a first-principles perspective, as if
explaining the system to another engineer.

## What this repository focuses on

- kubelet, containerd, and the Linux kernel
- reconciliation loops and controllers
- eviction, tolerations, and disruption budgets
- Linux networking (`ip`, routing, veth pairs)
- Kubernetes Services and kube-proxy behavior

## What this repository intentionally avoids

- shallow tutorials
- copy-pasted documentation
- tool-centric explanations without fundamentals

## Modules

- **Module 01** — Runtime & Node Reality  
- **Module 02** — Controllers & Reconciliation  
- **Module 03** — Configuration, Secrets & Identity  
- **Module 04** — Networking from the Kernel Up  

This is a living repository and will grow as my understanding deepens.
