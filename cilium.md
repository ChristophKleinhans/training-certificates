# Cilium Certified Associate (CCA)

## Understand the Role of Cilium in Kubernetes Environments

Cilium is a eBPF based CNI plugin for kubernetes. It provides Networking, security and Observability. It is the layer that connects the pods and controls/observes the connection.

**Why do we need it**: Traditional kubernetes is based on kube-proxy + ip-tables, it is IP based and does not scale (iptables rules grow linearly with services/pods, and pod IPs are ephemeral so IP-based firewalling is fragile. Cilium uses eBPF to program the kernel datapath directly).

**What does it provide**:

- Networking (connectivity) — Provides the pod network: pod-to-pod connectivity, IPAM, and service load balancing (can replace kube-proxy).
- Security — Enforces network policies at L3/L4 (allow/deny policies enforced in eBPF datapath) and L7 (HTTP, gRPC, Kafka, DNS), using **identity-based security rather than IP-based!**. For external endpoints outside the cluster where cilium has no identity, Cilium does fall back to CIDR/IP-based rules.
- Observability — Via Hubble, gives visibility into network flows, service dependencies, and L7 traffic.

## Cilium Architecture

<img width="1044" height="782" alt="image" src="https://github.com/user-attachments/assets/0685d751-c6af-497e-a50a-b8d4c550cda3" />

The two planes:
- **Control Plane**: *Decides* what should happen, computes identities, compiles policies, managed IPAM. It runs in userspace
- **Data plane (datapath)**: *Does* it: eBPF programs attached to kernel hook points forward, load-balance, enforce policy
