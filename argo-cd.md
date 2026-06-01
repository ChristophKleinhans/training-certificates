# Understand Argo CD Fundamentals — Annotated Example

Argo CD is a **declarative, GitOps continuous delivery** tool for Kubernetes
(a CNCF graduated project). Where Argo Workflows *runs jobs to completion*,
Argo CD *continuously keeps a cluster matching what Git declares*.

Core principles:
- **Git is the single source of truth** for the desired cluster state.
- **Pull-based**: an agent runs *inside* the cluster and pulls from Git
  (vs. traditional CI/CD that pushes `kubectl apply` from outside).
- **Continuous reconciliation**: Argo CD compares the **target state** (Git)
  against the **live state** (cluster) and can drive live → target automatically.

```
   Git repo (desired state)            Kubernetes cluster (live state)
   ┌──────────────────────┐   pull    ┌──────────────────────────────┐
   │ manifests / Helm /    │ ───────▶  │  Argo CD (Application         │
   │ Kustomize             │  compare  │  Controller) reconciles       │
   └──────────────────────┘ ◀───────  │  live state toward Git        │
                              detect   └──────────────────────────────┘
                              drift
```

## The core object: an Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd                  # Argo CD's Application objects live in the argocd namespace.
spec:
  # ── Which AppProject this belongs to (guardrails / multi-tenancy). ──
  project: default

  # ── SOURCE: what to deploy and from where (the desired state in Git). ──
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD             # Branch, tag, or commit. e.g. main, v1.2.0, HEAD.
    path: guestbook                  # Folder within the repo containing the manifests.
    # Argo CD auto-detects the source type. For Helm/Kustomize you configure here:
    # helm:
    #   valueFiles: [ values-prod.yaml ]
    #   parameters: [ { name: image.tag, value: "1.4.0" } ]
    # kustomize:
    #   namePrefix: prod-

  # ── DESTINATION: which cluster + namespace to deploy into (the live target). ──
  destination:
    server: https://kubernetes.default.svc   # In-cluster API. Use `name:` for a registered remote cluster.
    namespace: guestbook

  # ── SYNC POLICY: how live state is kept in step with Git. ──
  syncPolicy:
    automated:                       # Omit this block entirely for MANUAL sync.
      prune: true                    # Delete resources that were removed from Git.
      selfHeal: true                 # Revert manual cluster changes back to Git's state.
    syncOptions:
      - CreateNamespace=true         # Create the destination namespace if absent.
      # - ApplyOutOfSyncOnly=true    # Only apply resources detected as OutOfSync.
      # - PrunePropagationPolicy=foreground
    retry:
      limit: 3
      backoff: { duration: 5s, factor: 2, maxDuration: 3m }
```

## Two statuses you must NOT conflate

Argo CD reports sync and health **independently**.

| Status type | Question it answers | Common values |
|-------------|---------------------|---------------|
| **Sync status** | Does live match Git? | `Synced`, `OutOfSync`, `Unknown` |
| **Health status** | Are the resources actually working? | `Healthy`, `Progressing`, `Degraded`, `Missing`, `Suspended`, `Unknown` |

Examples of the combinations:
- `Synced` + `Degraded` → Git applied correctly, but pods are crash-looping.
- `OutOfSync` + `Healthy` → app runs fine, but Git has changed (drift) and hasn't been applied.

## Sync vs. Refresh (don't mix these up)

| Action | What it does |
|--------|--------------|
| **Refresh** | Re-reads Git and recomputes the diff. Does **not** change the cluster. |
| **Sync** | Actually **applies** manifests to move live state toward the target. |
| **Hard Refresh** | Refresh that also bypasses the manifest-generation cache. |

## Architecture components

| Component | Responsibility |
|-----------|----------------|
| **API Server** | Backs the Web UI, CLI, and CI/CD calls; auth, RBAC, app/repo/cluster management |
| **Repository Server** | Clones Git, caches it, and **generates manifests** (runs Helm/Kustomize) |
| **Application Controller** | The reconciliation engine: compares states, performs syncs, runs hooks |
| **Redis** | Caching layer for the controller and repo server |
| **Dex** (optional) | SSO / external identity provider integration |
| **ApplicationSet controller** | Templates and generates many Applications at once |

## AppProject (grouping + guardrails)

An `AppProject` constrains a set of Applications — the basis for multi-tenancy.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:                       # Which Git repos apps in this project may use.
    - "https://github.com/team-a/*"
  destinations:                      # Which clusters/namespaces they may deploy to.
    - server: https://kubernetes.default.svc
      namespace: "team-a-*"
  clusterResourceWhitelist:          # Which cluster-scoped kinds are allowed.
    - group: ""
      kind: Namespace
```

## Exam-relevant points

1. **GitOps + pull-based.** Git is the source of truth; the in-cluster agent
   pulls and reconciles. This is the defining contrast with push-based CI/CD.
2. **Sync status ≠ health status.** They're independent; know the difference and
   the values each can take.
3. **`prune` vs `selfHeal`.** `prune` removes resources deleted from Git;
   `selfHeal` reverts manual live changes. Both belong to `syncPolicy.automated`.
4. **Application = source + destination.** Source (repoURL/path/targetRevision)
   declares *what*; destination (server/name + namespace) declares *where*.
5. **AppProject = guardrails.** Restricts allowed repos, destinations, and
   resource kinds — the multi-tenancy boundary.
6. **Who generates manifests?** The **Repository Server** (running Helm/Kustomize),
   not the controller. The **Application Controller** does the reconciling.
