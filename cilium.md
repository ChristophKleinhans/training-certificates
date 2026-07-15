# Cilium Certified Associate (CCA)

## Understand the Role of Cilium in Kubernetes Environments

Cilium is a eBPF based CNI plugin for kubernetes. It provides Networking, security and Observability. It is the layer that connects the pods and controls/observes the connection.

**Why do we need it**: Traditional kubernetes is based on kube-proxy + ip-tables, it is IP based and does not scale (iptables rules grow linearly with services/pods, and pod IPs are ephemeral so IP-based firewalling is fragile. Cilium uses eBPF to program the kernel datapath directly).

**What does it provide**:

- Networking (connectivity) — Provides the pod network: pod-to-pod connectivity, IPAM, and service load balancing (can replace kube-proxy).
- Security — Enforces network policies at L3/L4 (allow/deny policies enforced in eBPF datapath) and L7 (HTTP, gRPC, Kafka, DNS), using **identity-based security rather than IP-based!**. For external endpoints outside the cluster where cilium has no identity, Cilium does fall back to CIDR/IP-based rules.
- Observability — Via Hubble, gives visibility into network flows, service dependencies, and L7 traffic.

## Cilium Architecture and the Cilium component roles

<img width="674" height="505" alt="image" src="https://github.com/user-attachments/assets/4870d7e8-c431-4359-8033-9ba1ae42f08d" />


The two planes:
- **Control Plane**: *Decides* what should happen, computes identities, compiles policies, managed IPAM. It runs in userspace
- **Data plane (datapath)**: *Does* it: eBPF programs attached to kernel hook points forward, load-balance, enforce policy

The main components:
- *Clilium Agent*, it's a DaemonSet, one per node. The workhorse, it watches the K8s API, manages endpoints and indentities on its node. Compiles policy+service rules into eBPF and loads them into the kernel. It runs the hubble server.
- *Cilium Operator*, A Deployment, once per cluster. Handles cluster wide tasks that should not run per-node. IPAM allocation, garbage collection of stale identities/CRDs, kvstore sync. (Traffic keeps flowing even if operator is briefly down)
- *CNI plugin*, the small binary that kubelet invokes when a pod is created, it calls the agent to wire up networking
- *eBPF datapath*, the programs the agent loads, the actual forwarding, load balancing policy enforcement in-kernel. Here the packets get moved.
- *Hubble*, the Hubble server embedded in each agent, the Hubble relay (aggregates all nodes for clusterwide view) and Hubble UI/CLI
- *Envoy proxy (L7)* An embedded proxy per node (not per pod sidecars) that the datapath transparently redirects traffic to when L7 policy/mesh features are needed
- KVStore (etcd), optional state backend, by default the state is in the K8s API server, a dedicated etcd is only for very large clusters or Cluster Mesh

The *KFStore* stores the state of identities, endpoints and policies. They live in either Kubernetes CRDs/API-Server (default it lives in k8s etcd) or an external etcd for very large clusters

**The flow**:
1. Pod is scheduled
2. Cilium Agent sees it via K8s API. Allocates an IP and computes endpoints identity from it's label
3. Cilium Agent compiles resulting policy + service load-balancing rules into eBPF and loads them into the kernel on that node
4. eBPF datapath enforces and forwardes in-kernel

## IP address management (IPAM)

IPAM is how Cilium decides which IP address each pod gets and where those IPs come from.
The *IPAM mode* determines how pod CIDRs are carved up and allocated.

**The core job**: Every pod needs a routeable IP. IPAM mages the pool of available IPs, hands one to each new pod via the CNI plugin, and reclaims it when the pod dies.

**The 4 IPAM modes**:

1. *Cluster Scope (default)*: The Cilium operator assigns a PodCIDR block to each node. Uses `CiliumNode` CRDs to track allocations
2. *Kubernetes*: Cilium defers to Kubernetes and reads the podCIDR that Kubernetes already assigns to each node. Kubernetes is the source of truth for node CIDRs
3. *Multi Pool*: Multiple IP pools, different pods can draw from dfferent pools, e.g. by namespace or annotation
4. *Cloud Provider*: Pods get read VPC IPs, that means pods are first-class citizens on the cloud network, often used by managed clusters. E.g. AWS ENI or Azure IPAM

For *1* and *3* cilium allocates the IPs and for *2* and *4* the Kubernetes or the Cloud provider allocates the IPs. We need to make sure we use the correctly IPAM from the beginning, it is not changed easily afterwards.


