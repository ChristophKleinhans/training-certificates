# Cilium Certified Associate (CCA)

## Understand the Role of Cilium in Kubernetes Environments

Cilium is a eBPF based CNI plugin for kubernetes. It provides Networking, security and Observability. It is the layer that connects the pods and controls/observes the connection.

**Why do we need it**: Traditional kubernetes is based on kube-proxy + ip-tables, it is IP based and does not scale (iptables rules grow linearly with services/pods, and pod IPs are ephemeral so IP-based firewalling is fragile. Cilium uses eBPF to program the kernel datapath directly).

**What does it provide**:

- Networking (connectivity) — Provides the pod network: pod-to-pod connectivity, IPAM, and service load balancing (can replace kube-proxy).
- Security — Enforces network policies at L3/L4 (allow/deny policies enforced in eBPF datapath) and L7 (HTTP, gRPC, Kafka, DNS), using **identity-based security rather than IP-based!**. For external endpoints outside the cluster where cilium has no identity, Cilium does fall back to CIDR/IP-based rules.
- Observability — Via Hubble, gives visibility into network flows, service dependencies, and L7 traffic.

## Cilium Architecture

<img width="674" height="505" alt="image" src="https://github.com/user-attachments/assets/4870d7e8-c431-4359-8033-9ba1ae42f08d" />


The two planes:
- **Control Plane**: *Decides* what should happen, computes identities, compiles policies, managed IPAM. It runs in userspace
- **Data plane (datapath)**: *Does* it: eBPF programs attached to kernel hook points forward, load-balance, enforce policy

The main components:
- *Clilium Agent*, it's a DaemonSet, one per node. The workhorse, it watches the K8s API, manages endpoints and indentities on its node. It serves also the hubble data.
- *Cilium Operator*, A Deployment, once per cluster. Handles cluster wide tasks that should not run per-node. IPAM allocation, garbage collection of stale identities/CRDs, kvstore sync. (Traffic keeps flowing even if operator is briefly down)
- *CNI plugin*, the small binary that kubelet invokes when a pod is created, it calls the agent to wire up networking
- *eBPF datapath*, the programs the agent loads, the actual forwarding/enforcement layer
- *Hubble*, observability, build on top (hubble server in the agent)

The *KFStore* stores the state of identities, endpoints and policies. They live in either Kubernetes CRDs/API-Server (default it lives in k8s etcd) or an external etcd for very large clusters

**The flow**:
1. Pod is scheduled
2. Cilium Agent sees it via K8s API. Allocates an IP and computes endpoints identity from it's label
3. Cilium Agent compiles resulting policy + service load-balancing rules into eBPF and loads them into the kernel on that node
4. eBPF datapath enforces and forwardes in-kernel

