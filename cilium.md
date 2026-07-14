# Cilium Certified Associate (CCA)

## Understand the Role of Cilium in Kubernetes Environments

Cilium is a eBPF based CNI plugin for kubernetes. It provides Networking, security and Observability. It is the layer that connects the pods and controls/observes the connection.

**Why do we need it**: Traditional kubernetes is based on kube-proxy + ip-tables, it is IP based and does not scale (iptables rules grow linearly with services/pods, and pod IPs are ephemeral so IP-based firewalling is fragile. Cilium uses eBPF to program the kernel datapath directly).

**What does it provide**:

- Networking (connectivity) — Provides the pod network: pod-to-pod connectivity, IPAM, and service load balancing (can replace kube-proxy).
- Security — Enforces network policies at L3/L4 and L7 (HTTP, gRPC, Kafka, DNS), using **identity-based security rather than IP-based!**.
- Observability — Via Hubble, gives visibility into network flows, service dependencies, and L7 traffic.

## Cilium Architecture

