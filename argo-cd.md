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

# Synchronize Applications Using Argo CD — Annotated Example

A **sync** is the operation that applies Git's desired state to the cluster,
moving live state toward target. This domain is about *how* sync runs, *when*
it triggers, and *in what order* resources are applied.

## Manual vs. Automated sync

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/web-app.git
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: web

  syncPolicy:
    # ── AUTOMATED: omit this whole block for MANUAL sync ──
    automated:
      prune: true                    # Delete live resources removed from Git.
      selfHeal: true                 # Revert manual cluster edits back to Git.
      allowEmpty: false              # Refuse to sync down to zero resources (safety).

    # Applies to manual AND automated retries (automated does NOT retry without this).
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2                    # 5s, 10s, 20s, ...
        maxDuration: 3m

    syncOptions:
      - CreateNamespace=true         # Create destination namespace if missing.
      - ApplyOutOfSyncOnly=true      # Only apply resources that are OutOfSync (faster).
      - PruneLast=true               # Prune only after other resources sync successfully.
      - Validate=false               # Skip `kubectl apply --validate`.
      # - Replace=true               # Use `kubectl replace` instead of `apply`.
```

- **Manual**: a human/CI triggers `argocd app sync web-app` after reviewing the diff.
- **Automated**: Argo CD syncs itself on detected drift. `prune` and `selfHeal`
  control destructive/corrective behavior; `selfHeal` waits a short reconciliation
  interval before reverting manual changes.

## Two INDEPENDENT ordering mechanisms

### 1) Sync PHASES — the coarse timeline of one sync

```
PreSync  ──▶  Sync  ──▶  PostSync
                 │
                 └─(on failure)─▶  SyncFail
```

| Phase | Typical use |
|-------|-------------|
| **PreSync** | DB migration, schema setup — runs *before* the main apply |
| **Sync** | The application's actual manifests |
| **PostSync** | Smoke tests, notifications — runs *after* the app is healthy |
| **SyncFail** | Cleanup that runs only if the sync failed |

### 2) Sync WAVES — fine ordering WITHIN a phase

Resources carry an annotation; Argo applies them in **ascending** wave order and
waits for each wave to be **healthy** before the next. Default wave is `0`;
negatives run first.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # e.g. CRDs / namespaces first
```

> Full ordering rule: **phase first, then wave within the phase**, then by
> kind/name. So a PostSync resource always runs after every Sync-phase wave.

## Resource HOOKS (attach work to phases)

A hook is a normal resource (usually a `Job`) annotated to run during a phase.
This is Argo CD's GitOps-native replacement for Helm hooks.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    # WHEN to run: PreSync | Sync | PostSync | SyncFail | Skip
    argocd.argoproj.io/hook: PreSync
    # WHEN to delete the hook resource afterward:
    #   HookSucceeded | HookFailed | BeforeHookCreation
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    argocd.argoproj.io/sync-wave: "0"    # Hooks honor waves too.
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: migrate/migrate:latest
          args: ["-path=/migrations", "-database=$(DB_URL)", "up"]
```

| Hook phase | Runs… |
|------------|-------|
| `PreSync` | before applying the main manifests (e.g. migrations) |
| `Sync` | alongside the main resources, within the Sync phase |
| `PostSync` | after all Sync resources are healthy (e.g. tests) |
| `SyncFail` | only when the sync operation fails |
| `Skip` | tells Argo CD NOT to apply this manifest |

| Hook delete policy | Hook resource deleted… |
|--------------------|------------------------|
| `HookSucceeded` | after the hook completes successfully |
| `HookFailed` | after the hook fails |
| `BeforeHookCreation` | right before a new run creates it (keeps last run for debugging) |

## Per-resource sync controls (annotations on the manifest)

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false   # Never prune this resource.
    # Other options: Delete=false, SkipDryRunOnMissingResource=true,
    #                Validate=false, ServerSideApply=true
    argocd.argoproj.io/compare-options: IgnoreExtraneous  # Don't flag as OutOfSync if extra.
```

## Selective / partial sync

```bash
# Sync only specific resources instead of the whole app:
argocd app sync web-app --resource apps:Deployment:web

# Preview without applying:
argocd app sync web-app --dry-run

# Force replace + prune:
argocd app sync web-app --replace --prune
```

## Quick reference

| Concept | Where it lives | Purpose |
|---------|----------------|---------|
| Manual sync | trigger via UI/CLI/API | human-reviewed apply |
| Automated sync | `syncPolicy.automated` | auto-apply on drift |
| `prune` | `syncPolicy.automated` | delete resources gone from Git |
| `selfHeal` | `syncPolicy.automated` | revert manual cluster edits |
| `retry` | `syncPolicy.retry` | retry failed syncs with backoff |
| Sync phases | hook annotation | PreSync → Sync → PostSync (+ SyncFail) |
| Sync waves | `sync-wave` annotation | ordering within a phase (ascending) |
| Hooks | `hook` annotation | run Jobs at a phase |
| Hook cleanup | `hook-delete-policy` annotation | when to delete the hook |
| Sync options | `syncPolicy.syncOptions` or per-resource | CreateNamespace, PruneLast, Replace, etc. |

## Exam-relevant points

1. **Manual vs automated, and the two flags.** `prune` = delete-on-removal,
   `selfHeal` = revert-manual-edits. Both are under `syncPolicy.automated`.
2. **Phases vs waves are different axes.** Phases = PreSync/Sync/PostSync/SyncFail
   (coarse). Waves = ascending ordering *within* a phase. Phase wins first.
3. **Hooks attach to phases.** PreSync for migrations, PostSync for tests; the
   GitOps-native alternative to Helm hooks. `hook-delete-policy` controls cleanup.
4. **Automated sync doesn't retry by default.** Add a `retry` block for backoff.
5. **`PruneLast` and `Prune=false`.** `PruneLast` defers pruning until after a
   successful sync; the per-resource `Prune=false` annotation exempts a resource.
