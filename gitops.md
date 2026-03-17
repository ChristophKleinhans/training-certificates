# GitOps Associate

## GitOps Terminology

### Continuous

The system will always keep attempting to reach the desired states, even after failures. It DOES NOT mean "instantaneous" or "real-time". The reconcilation loop runs constantly in the background. The agent observes the actual state and compares it with the desired state stored in Git. Is there a drift, it corrects it.

### Declarative Description

Imperative would be describing of HOW to get to the desired state. In the declarative description, all we do is to tell WHAT the result should look like.

### Desired State

What should the system look like. Git is the single source of truth. We control the state by changing what is written in the repo.

### State Drift

the actual state differs from the desired state which is defined in git. F.i. someone did a `kubectl delete pod/...` . The agents will continuously watch for drifts and correct it automaticall. It is important to have alerts, so in a case that something is deleted and created again, we will be alerted, otherwise the agent will simply create again and does not care if there is maybe a data loss.


### State Reconciliation

Its the loop which does automatically correct the actual state to match the desired state. Its pull based (not push-based, like a CI pipeline pushes to cluster). The GitOps Agent, which has cluster access, pulls the desired state from git.
- ArgoCD (3 minutes) and Flux (1 minute) are polling git in 1 to 3 minutes periods to checks for new commits. 
- The most GitOps agents do also support event driven notification using webhook.


### GitOps Managed Software System

Its a complete set of components that form a system where git is the single source of truth. There are three main components:

1. State Store (Git repository)
2. The reconciliation engine (GitOps agent) - continously comparing and correcting
3. Runtime Environment - the live system which is being managed

### State Store

There are 4 requirements to a state store
1. Immutable - it cannot be silently changed after a state has been commited
2. Versioned - every change has an unique indentifier (commit hash). a timestamp, an author and a message explaining the change
3. Retains complete vesrion history - we can always do rollbacks
4. Access controlled - no everyone can change the state

### Feedback Loop

Gives us visibility into the state of the system. The feedback loop answers:
- Did my commit actually get applied ?
- Is the system healthy
- Is there a drift right now
- Did reconciliation succeed or fail

In ArgoCD/Flux it would be *sync and health status*

### Rollback

The commit history in Git allows rollback to a previous commit.
Dont use `git reset ...` because it rewrites history and removes commits. Instead use:
```bash
git revert abc1234
# or
git checkout ghi9012 -- deployment.yaml
```

## GitOps Pricinpals

The four Principles are *Declarative*, *Versioned and Immutable*, *Pulled automatically* and *Continously Reconciled*

### 1. Declerative

We define the end state we want, and the reconciliation engine figures out how to match the desired state. We are not describing it imperative of HOW to get to the state we want, iterative scripts are also not possible to version control.
- git is single source of truth
- same declaration produces the same result (Reproducibility)
- When reading the manifest we automatically understand the state we want
- Drift detection possible

### 2. Versioned and Immutable

- Versioned is that every change is tracked and has a history
- Immutable is that once a version is write it cannot silently changed or be overwritten

- Using tags like *latest* are an anti-pattern in GitOps

### 3. Pulled automatically

- polling git between 1 and 3 minutes (can be adjusted) or event-based using a webhook

### 4. Continously Reconciled

- Loop of `observe current state` -> `compare with desired state` -> `Difference Found? Act!` -> `Correct Drift`
- The loop never stops running and enforeces the other three principles

## Related Practices

### Configuration as Code (CaC)

- managing system and application configurations in version controlled files
- f.i. configmaps stored in YAML in git
- secrets could be stored in Git using a secrets operator
- Helm values.yaml
- Proetheus alert rules
- nginx, istion, etc. configuration in Git

### Infrastructure as Code (IaC)

  - `writing Terraform/Helm files` -> `store in Git` -> `IaC tools applies automatically`

### DevOps and DevSecOps

- `DevOps vs. DevSecOps`: DevOps delivers sofware faster and more reliable by breaking down the wall between developers and operations teams, like CI/CD pipelines, automation, collaboration, fast feedback loops.
While DevSecOps follows the `shift left` appraoch where we have all the Security practices applied in each stage of DevOps:

```bash
  Plan → Code → Build → Test → Release → Deploy → Operate
    ↑      ↑      ↑       ↑        ↑        ↑        ↑
  Security integrated at EVERY stage (shift left!)
```

### CI and CD
- Continuous integration (CI) is the practice of automatically building and testing the code every time a developer is pushing a change to git
-  Continuous Delivery (code is ready to deploy and waiting for human approaval) and Continous Deployment (code is automatically deployed to production without human approval)

=> Its push-based, means the CI/CD pipeline reaches out and is directly pusing the changes into the cluster, like executing `kubectl apply ...`

## GitOps Pattern

### Deployment and release patterns

Blue Green Deployment: Two identical environments, Blue the current and green the new. Deploy the newest release to Green, test it without exposing it to real users (smoke test which tests critical paths automatically, Mirror/Shadown traffic from the real environment but the real users never see the responses). Now switch ALL customer at once to the new environment.

#### Canary Deployment

Rollout the new version to a small percentage of users first, then gradually increase if healthy. Real production testing with limited blast radius, but complex traffic splitting and monitoring

#### A/B Testing

Same as canary but driven by business logic rather than percentages, f.i. premium and free users, users from germany or users from france, etc.

#### Rolling Update

Gradually replace old instances with new ones, one by one or in batches.

#### Feature Flag (Feature Toggles)

Simply deploy it with a feture Flag OFF and switch it to ON as soon as we want to go live with it.

### Progressive Delivery Patterns

- Its the evolution of Continuous Delivery: It is extending CD by adding fine-grained control over who gets the new software and when. Deliver progressively, savely and with automatic guardrails.
- Progressive Delivery essentially takes Canary, Blue/Green and Feature Flags and adds automation and intelligence on top — making rollout decisions based on real metrics, not just human judgement.
1. Automated Metric Analysis (key difference to classic CD): The system automatically decides, whether to proceed, pause or rollback based on live metrics. F.i. 
```bash
Deploy v2 to 10% traffic
        ↓
Measure: error rate, latency, CPU, business KPIs
        ↓
Error rate < 1%? ✅  → Proceed to 25%
Error rate > 5%? ❌  → Automatic rollback to v1
```

2. Progressive Traffic Shifting: Traffic is moved gradually in controlled steps, while each step is gated by the metric analysis from step 1.
```bash
Step 1:   5% → v2,  95% → v1   (observe)
Step 2:  25% → v2,  75% → v1   (observe)
Step 3:  50% → v2,  50% → v1   (observe)
Step 4: 100% → v2,   0% → v1   ✅ done
```

3. Automatic Rollback: If metrics cross a defined threshold at any step, the system automatically rolls back
```bash
Step 2:  25% → v2  ← error rate spikes to 8% ❌
              ↓
     Automatic rollback
              ↓
       100% → v1 restored ✅
     (without any human intervention)
```

For the exam, remember:
- Progressive Delivery = Continuous Delivery + automated guardrails
- The key differentiator is automated metric-based promotion and rollback
- *Argo Rollouts* is the primary GitOps-native tool for this
- Progressive Delivery makes Canary deployments intelligent and self-driving

### GitOps Architecture Patterns

1. *In cluster reconciler* means the gitops agent runs in the same cluster (most common):
```bash
┌─────────────────────────────────┐
│         Kubernetes Cluster      │
│                                 │
│  ┌─────────────┐                │
│  │  Argo CD /  │ ←── pulls ──── Git Repo
│  │    Flux     │                │
│  └──────┬──────┘                │
│         │ reconciles            │
│         ▼                       │
│  ┌─────────────┐                │
│  │  Workloads  │                │
│  │  (your app) │                │
│  └─────────────┘                │
└─────────────────────────────────┘
```

2. *external reconciler* means the gitops agent runs outside the cluster, could be also created as Hub-Spoke Pattern with hundreds of clusters:
```bash
┌──────────────────────┐          ┌─────────────────────┐
│   Management Cluster │          │  Target Cluster 1   │
│                      │          │  ┌───────────────┐  │
│  ┌────────────────┐  │──────────►  │   Workloads   │  │
│  │   Argo CD /    │  │          │  └───────────────┘  │
│  │     Flux       │  │          └─────────────────────┘
│  └───────┬────────┘  │
│          │           │          ┌─────────────────────┐
│          │ pulls     │          │  Target Cluster 2   │
└──────────┼───────────┘          │  ┌───────────────┐  │
           │                      │  │   Workloads   │  │
           ▼                      │  └───────────────┘  │
        Git Repo                  └─────────────────────┘
```

3. *State Store Management* in gitops its always git
4. *Apps of Apps pattern* is a gitops way to manage many applications decelaratively. There is a root app, that in turn manage all other apps
- App-of-Apps is a valid pattern but has the scalability/crowding limitation
- ApplicationSet is the modern solution that solves the crowding limitation
- ApplicationSet uses generators to dynamically create Applications
- Both patterns keep everything defined in Git

## Tooling

### Manifest Format and Packaging

A manifest simply declares the desired state of a resource.

- *Helm*: The most widley used Kubernetes package manager, like apt or npm. While a *chart* is a package of Kubernetes manifest with templating. *values* customize the chart, a *release* is a deployed instance of a chart and the *repository* is the collection of charts (like npm registry)
- *Kustomize*: Using *overlays* built in kubectl natively. There is one *base* config and we simply have overlays f.i. for each stage like dev, qa, prod. Kustomize simply merges the overlay with the *base*.
- *OCI (open container initiative) Artifacts*. First OCI only standardized container images (Docker images). But the format was so good, that it got extended to any kind of artifact, not just container images. Before OCI, Helm charts were stored in dedicated Helm repos, now in a OCI registry. One registry for Container images, helm charts, Kubernetes manifest, Terraform modules, WASM binaries, Software signatures, etc. .
