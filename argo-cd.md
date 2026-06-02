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


# Use Argo CD Application (and ApplicationSets) — Annotated Example

The fundamentals doc introduced the `Application` object. This one covers how
you actually *use* it: the different **source tool types**, scaling to many apps
with **App-of-Apps** and **ApplicationSets**, and the **generators** that make
ApplicationSets powerful.

## Source tool types (auto-detected by Argo CD)

One `Application`, but `source` differs by tool. Argo CD inspects the repo path
and picks the right one automatically.

```yaml
# ---- (a) Plain directory of manifests ----
source:
  repoURL: https://github.com/example/app.git
  targetRevision: main
  path: manifests
  directory:
    recurse: true                    # Include manifests in subfolders.

# ---- (b) Helm chart ----
source:
  repoURL: https://github.com/example/app.git
  targetRevision: main
  path: chart
  helm:
    releaseName: web
    valueFiles:
      - values-prod.yaml
    parameters:                      # Override individual chart values.
      - name: image.tag
        value: "1.8.0"
    # values: |                      # Inline values block (alternative to files).
    #   replicaCount: 3

# ---- (c) Kustomize overlay ----
source:
  repoURL: https://github.com/example/app.git
  targetRevision: main
  path: overlays/prod
  kustomize:
    namePrefix: prod-
    images:
      - example/web:1.8.0            # Override image tags.

# ---- (d) Helm chart from a Helm REPO (not Git) ----
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: nginx                       # `chart:` instead of `path:` for a Helm repo.
  targetRevision: 15.0.0             # Chart version.
```

## Multi-source Application

Pull manifests from one repo and Helm values from another (common for separating
config from charts). `$ref` lets one source reference another.

```yaml
spec:
  sources:
    - repoURL: https://github.com/example/charts.git
      targetRevision: main
      path: charts/web
      helm:
        valueFiles:
          - $values/envs/prod/values.yaml    # pulled from the ref below
    - repoURL: https://github.com/example/config.git
      targetRevision: main
      ref: values                            # named reference used above
```

## Useful Application fields

```yaml
metadata:
  finalizers:
    # Cascading delete: removing the Application also prunes its live resources.
    - resources-finalizer.argocd.argoproj.io
spec:
  # Tolerate fields mutated at runtime so the app doesn't show OutOfSync forever.
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas                       # e.g. ignore HPA-managed replica count
  revisionHistoryLimit: 10                      # How many past syncs to keep for rollback.
```

## Scaling pattern 1: App-of-Apps (older)

One parent Application whose Git source contains *other Application manifests*.
Syncing the parent creates the children.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap                    # The "root" app.
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/cluster-apps.git
    targetRevision: main
    path: apps                       # This folder holds child Application YAMLs.
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

## Scaling pattern 2: ApplicationSet (modern, preferred)

An `ApplicationSet` **templates** Applications from a **generator**. One object →
many Applications.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: web-per-env
  namespace: argocd
spec:
  goTemplate: true                   # Use Go templating in the template block.
  generators:
    # LIST generator: one Application per listed element.
    - list:
        elements:
          - env: dev
            cluster: https://kubernetes.default.svc
          - env: prod
            cluster: https://prod-cluster.example.com
  template:                          # Rendered once per generated element.
    metadata:
      name: 'web-{{.env}}'           # Values come from the generator element.
    spec:
      project: default
      source:
        repoURL: https://github.com/example/app.git
        targetRevision: main
        path: 'overlays/{{.env}}'
      destination:
        server: '{{.cluster}}'
        namespace: 'web-{{.env}}'
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

## ApplicationSet generators (know these)

| Generator | Produces one Application per… | Typical use |
|-----------|-------------------------------|-------------|
| **List** | hardcoded element | small fixed set of envs/clusters |
| **Cluster** | cluster registered in Argo CD | deploy an app to every cluster |
| **Git (directories)** | directory in a repo | folder-per-microservice |
| **Git (files)** | config file in a repo | values-file-per-environment |
| **Matrix** | combination of two generators | every app × every cluster |
| **Merge** | merged/overlaid generators | base list + per-item overrides |
| **SCM Provider** | repo in a GitHub/GitLab org | onboard every repo automatically |
| **Pull Request** | open pull request | ephemeral PR preview environments |

```yaml
# Example: Git directories generator (one app per folder under apps/)
  generators:
    - git:
        repoURL: https://github.com/example/monorepo.git
        revision: main
        directories:
          - path: apps/*             # each match becomes {{.path.basename}}
```

## Quick reference

| Concept | Object / field | Purpose |
|---------|----------------|---------|
| Manifest source | `source` / `sources` | Where + how to get desired state |
| Helm config | `source.helm` | valueFiles, parameters, releaseName |
| Kustomize config | `source.kustomize` | namePrefix, images, common labels |
| Multi-source | `spec.sources` + `ref`/`$ref` | combine chart + external values |
| Cascading delete | `resources-finalizer...` finalizer | prune resources when app deleted |
| Drift tolerance | `ignoreDifferences` | ignore runtime-mutated fields |
| Many apps (old) | App-of-Apps (parent Application) | parent syncs child Applications |
| Many apps (new) | `ApplicationSet` + generators | template N apps from parameters |

## Exam-relevant points

1. **Source type is auto-detected.** Directory, Helm, Kustomize, Jsonnet — same
   Application, different `source` sub-block. `chart:` (Helm repo) vs `path:` (Git).
2. **App-of-Apps vs ApplicationSet.** Both scale to many apps; App-of-Apps nests
   Application manifests in Git, ApplicationSet *templates* them from generators
   (the modern, preferred approach).
3. **Know the generators.** Especially List, Cluster, Git (dirs/files), and Matrix.
   Pull Request = ephemeral preview envs; SCM Provider = per-repo onboarding.
4. **The finalizer enables cascading delete.** Without
   `resources-finalizer.argocd.argoproj.io`, deleting an Application leaves its
   resources running.
5. **Multi-source apps** combine a chart from one repo with values from another
   via a named `ref`.

# Configure Argo CD with Helm and Kustomize — Annotated Example

Argo CD natively supports **Helm** and **Kustomize** as config-management tools.
The critical thing to understand is *how* Argo CD drives them — it differs from
running them by hand.

## The #1 concept: Argo CD TEMPLATES, it does not install

| Tool | What Argo CD runs | What it does NOT do |
|------|-------------------|---------------------|
| Helm | `helm template` → applies output | no `helm install`, no Tiller, no release object, `helm list` shows nothing |
| Kustomize | `kustomize build` → applies output | nothing special; output is plain manifests |

Consequences for Helm:
- Helm is used as a **templating engine only**; Argo CD applies the rendered YAML.
- **Helm lifecycle hooks** (`helm.sh/hook`) don't run via `helm install`. Argo CD
  converts the common ones to **Argo CD sync hooks** instead.
- No release history in the cluster — **Argo CD** (Git + its own history) is the
  source of truth and the rollback mechanism, not `helm rollback`.

## Helm source configuration

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/app.git
    targetRevision: main
    path: chart                      # Chart dir in Git. (Use `chart:` for a Helm repo/OCI — see below.)
    helm:
      releaseName: web               # Sets .Release.Name during templating.
      valueFiles:                    # Equivalent of `-f`. Order matters (later overrides earlier).
        - values.yaml
        - values-prod.yaml
      ignoreMissingValueFiles: true  # Don't fail if a listed values file is absent.
      parameters:                    # Equivalent of `--set` (type-aware).
        - name: image.tag
          value: "1.8.0"
        - name: replicaCount
          value: "3"
      fileParameters:                # Equivalent of `--set-file` (value read from a file).
        - name: config.ini
          path: files/config.ini
      values: |                      # Inline values (equivalent of an extra -f). 
        service:
          type: ClusterIP
      # valuesObject: { ... }        # Structured inline values (preferred over the string form).
      skipCrds: false                # true = pass --skip-crds to `helm template`.
      passCredentials: false         # Pass repo creds to dependency chart repos.
  destination:
    server: https://kubernetes.default.svc
    namespace: web
```

### Helm chart straight from a Helm repo or OCI registry

```yaml
  source:
    repoURL: https://charts.bitnami.com/bitnami   # or oci://registry.example.com/charts
    chart: nginx                     # `chart:` (NOT `path:`) signals a Helm repo source.
    targetRevision: 15.0.0           # Here targetRevision is the CHART VERSION.
    helm:
      valueFiles: [ values.yaml ]
```

## Kustomize source configuration

```yaml
  source:
    repoURL: https://github.com/example/app.git
    targetRevision: main
    path: overlays/prod              # Folder containing kustomization.yaml
    kustomize:
      namePrefix: prod-              # Prepend to resource names.
      nameSuffix: -v2                # Append to resource names.
      images:                        # Override image names/tags.
        - example/web:1.8.0
      replicas:
        - name: web
          count: 3
      commonLabels:                  # Labels added to every resource.
        app.kubernetes.io/managed-by: argocd
      commonAnnotations:
        team: platform
      namespace: web                 # Set/override namespace on resources.
      patches:                       # Strategic-merge / JSON6902 patches.
        - target: { kind: Deployment, name: web }
          patch: |
            - op: add
              path: /spec/template/metadata/labels/tier
              value: frontend
```

## Combining: Kustomize inflating a Helm chart

Kustomize can render a Helm chart itself (the `helmCharts:` field), enabled in
Argo CD via the build option below.

```yaml
  source:
    path: overlays/prod
    kustomize:
      # requires kustomize built with --enable-helm (set on the repo-server / via flag)
      # kustomization.yaml then uses a `helmCharts:` block to inflate the chart,
      # and the overlay patches the rendered output.
```

## Config Management Plugins (CMP) — beyond the built-ins

For tools Argo CD doesn't support natively (helmfile, jsonnet bundlers, cdk8s,
etc.), a **CMP** sidecar on the repo-server runs your command and Argo CD applies
whatever manifests it prints to **stdout**.

```yaml
# A plugin is referenced from the Application like this:
  source:
    plugin:
      name: helmfile
      parameters:
        - name: environment
          string: prod
```

## Build environment variables

Available to Helm parameter substitution, Kustomize, and plugins:

| Variable | Meaning |
|----------|---------|
| `ARGOCD_APP_NAME` | The Application's name |
| `ARGOCD_APP_NAMESPACE` | Destination namespace |
| `ARGOCD_APP_REVISION` | Resolved Git revision |
| `ARGOCD_APP_SOURCE_PATH` | The `source.path` |
| `ARGOCD_APP_SOURCE_TARGET_REVISION` | The configured target revision |

```yaml
      parameters:
        - name: fullnameOverride
          value: "$ARGOCD_APP_NAME"   # substituted from the build environment
```

## Quick reference

| Need | Helm | Kustomize |
|------|------|-----------|
| Pick values | `valueFiles`, `values`, `valuesObject` | (overlay folders) |
| Override a value | `parameters` (`--set`) | `images`, `replicas`, `patches` |
| Name munging | `releaseName` | `namePrefix`, `nameSuffix` |
| Labels everywhere | values-dependent | `commonLabels`, `commonAnnotations` |
| Chart from registry | `chart:` + version in `targetRevision` | n/a |
| Skip CRDs | `skipCrds: true` | n/a |

## Exam-relevant points

1. **`helm template`, not `helm install`.** No Tiller, no release object, no
   `helm list`/`helm rollback`. Helm is just a renderer; Argo CD applies the YAML.
2. **Helm hooks ≠ helm-managed here.** Argo CD maps common Helm hooks onto its own
   sync hooks; they don't execute through a Helm release lifecycle.
3. **`path:` vs `chart:`.** `path:` = chart in Git; `chart:` = chart from a Helm
   repo/OCI registry, with `targetRevision` meaning the chart version.
4. **Kustomize overlays declaratively.** `images`, `namePrefix`, `replicas`,
   `commonLabels`, `patches` — all expressible in the `kustomize` block.
5. **CMP for everything else.** A plugin sidecar lets Argo CD apply any tool's
   stdout manifests (helmfile, jsonnet, cdk8s, …).
6. **Build env vars** like `ARGOCD_APP_NAME` are usable in parameters and plugins.

# Identify Common Reconciliation Patterns — Annotated Notes

Reconciliation is the controller pattern underneath the *entire* Argo ecosystem.
Once you recognize it, Argo CD, Rollouts, and Workflows all look like the same
machine aimed at different problems.

## The reconciliation loop

A controller runs this loop continuously, forever:

```
        ┌──────────────────────────────────────────────┐
        │                                                │
        ▼                                                │
   1. OBSERVE        2. COMPARE (diff)        3. ACT      │
   read actual   →   desired vs actual?   →   converge ──┘
   (live) state      compute the gap          toward desired
```

- **Desired state** — what you declared (Git manifests, a Workflow spec, a
  Rollout strategy). The "target."
- **Observed/actual state** — what is really running in the cluster.
- The controller's only job: **drive actual → desired and keep it there.**

## THE key distinction: level-triggered vs edge-triggered

| | Edge-triggered | Level-triggered ✅ (Kubernetes / Argo) |
|--|----------------|----------------------------------------|
| Reacts to | **events** ("a change happened") | **state** ("the world currently looks like X") |
| Missed signal | stays wrong forever | self-corrects on the next loop |
| Robustness | fragile | resilient |
| Example | "on push, deploy" | "make the cluster match Git, always" |

Level-triggered design is *why* Argo controllers self-heal and tolerate missed
events: every loop re-reads reality from scratch, so a dropped notification just
gets fixed on the next pass.

## Patterns that fall out of this design

| Pattern | Meaning | Where you see it |
|---------|---------|------------------|
| **Idempotency** | running the loop twice = same result (re-applying a correct manifest is a no-op) | every Argo controller |
| **Eventual consistency** | system converges over repeated loops, not instantly | sync waves waiting for health; rollout steps |
| **Drift detection** | loop notices live ≠ desired | Argo CD `OutOfSync` status |
| **Self-healing** | loop actively reverts drift to desired | Argo CD `selfHeal: true` |
| **Continuous reconciliation** | watch + periodic resync interval keep it current | controllers re-queue on a timer |
| **Convergence** | repeated correction until actual == desired | all of the above |

## The same loop across Argo

| Tool | Desired state | Observed state | Reconcile action |
|------|---------------|----------------|------------------|
| **Argo CD** | Git manifests | live cluster objects | sync / prune / self-heal toward Git |
| **Argo Rollouts** | Rollout strategy + steps | current ReplicaSets/traffic | advance/pause the rollout step |
| **Argo Workflows** | Workflow spec (DAG/steps) | pod/node statuses | schedule next runnable node |
| **Argo Events** ⚠️ | n/a (edge-driven) | incoming events | fire trigger on event (NOT a level loop) |

> ⚠️ **Argo Events is the odd one out.** It is genuinely **event/edge-driven**
> (event arrives → sensor → trigger), not a level-triggered reconciliation loop.
> A common exam contrast.

## Lifecycle plumbing that supports reconciliation

```yaml
# ── Owner references: enable cascading deletion of child resources ──
metadata:
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: web-rs
      uid: "..."
      controller: true               # deleting the owner garbage-collects this child

# ── Finalizers: run cleanup BEFORE the object is actually removed ──
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # Argo CD prunes app resources on delete
```

```yaml
# ── Argo CD: the self-heal half of the reconcile loop ──
spec:
  syncPolicy:
    automated:
      selfHeal: true                 # actively revert live drift back to Git (correction)
      prune: true                    # delete resources no longer in desired state
```

## Watch + resync (how the loop is actually driven)

- **Watch**: controllers subscribe to API-server change notifications, so they
  reconcile *promptly* when something changes (efficient, near-real-time).
- **Periodic resync**: they ALSO re-reconcile on a timer regardless of events —
  this is the safety net that makes the system level-triggered (catches anything
  a watch missed). In Argo CD this is the app reconciliation interval.

## Exam-relevant points

1. **Reconciliation loop = observe → diff → act, repeated forever.** Drive actual
   state toward declared desired state and hold it there.
2. **Level-triggered, not edge-triggered.** React to *state*, not *events*; this
   is what makes Argo controllers self-healing and resilient to missed signals.
3. **Idempotency + eventual consistency.** Re-running is safe; the system
   converges over repeated loops rather than in one shot.
4. **Self-heal vs prune.** Self-heal corrects drift back to desired; prune removes
   what's no longer desired — both are the reconcile loop acting.
5. **Argo Events is the exception** — event/edge-driven, not a level loop.
6. **Owner references** drive cascading deletion; **finalizers** gate deletion to
   allow cleanup first.

# Argo Rollouts — CAPA Study Notes

Covers the three exam subtopics:
1. Understand Argo Rollouts Fundamentals
2. Use Common Progressive Rollout Strategies
3. Describe Analysis Template and AnalysisRun

---

# 1) Understand Argo Rollouts Fundamentals

**Argo Rollouts** is a Kubernetes controller + set of CRDs that provides
**progressive delivery** — advanced deployment strategies (blue-green, canary)
with traffic shaping and automated, metric-driven promotion/rollback that the
built-in Kubernetes `Deployment` cannot do.

## Why not just a Deployment?

A native `Deployment` only supports `RollingUpdate` / `Recreate`, and a readiness
probe only tells you a container *started* — not that the new version behaves
correctly under real traffic. Rollouts adds **fine-grained traffic control** and
**analysis-based gating** so a bad version is caught and rolled back automatically.

## The core object: `Rollout` (a drop-in Deployment replacement)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout                        # Replaces `kind: Deployment`.
metadata:
  name: guestbook
spec:
  replicas: 5
  selector:
    matchLabels:
      app: guestbook
  template:                          # Pod template — IDENTICAL to a Deployment's.
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
        - name: guestbook
          image: example/guestbook:v2
  # The ONLY real difference vs a Deployment: the strategy block.
  strategy:
    canary:                          # or `blueGreen:` — see section 2
      steps:
        - setWeight: 20
        - pause: { duration: 5m }
```

Key facts:
- A `Rollout` manages **ReplicaSets** under the hood, just like a Deployment.
- You can **migrate an existing Deployment** by either changing `kind` to
  `Rollout` (and adding `strategy`), or by referencing it via
  `workloadRef` so the Deployment stays and the Rollout drives it.
- Rollouts has its own **kubectl plugin** (`kubectl argo rollouts get rollout …`,
  `… promote`, `… abort`, `… retry`) and a **dashboard UI**.
- It integrates with **service meshes / ingress** (Istio, NGINX, ALB, SMI,
  Gateway API, etc.) for precise traffic weighting.

## Key terms

| Term | Meaning |
|------|---------|
| **Stable** | The currently good/known version (old ReplicaSet) |
| **Canary** | The new version receiving a controlled slice of traffic |
| **Active service** | (blue-green) Service pointing at the live version |
| **Preview service** | (blue-green) Service pointing at the new version pre-promotion |
| **Promotion** | Advancing the new version toward 100% / making it stable |
| **Abort** | Stopping a rollout and reverting traffic to stable |

---

# 2) Use Common Progressive Rollout Strategies

Two strategies: **Canary** and **Blue-Green**.

## Canary — shift traffic gradually

Slowly increases traffic to the new version through a series of **steps**,
optionally pausing and analyzing between increments.

```yaml
spec:
  strategy:
    canary:
      canaryService: guestbook-canary   # Service selecting ONLY canary pods
      stableService: guestbook-stable   # Service selecting ONLY stable pods
      trafficRouting:                    # Optional: precise weighting via mesh/ingress
        nginx:
          stableIngress: guestbook-ingress
      steps:
        - setWeight: 20                  # send 20% of traffic to canary
        - pause: { duration: 5m }        # wait 5 minutes (timed pause)
        - setWeight: 50
        - pause: {}                      # pause INDEFINITELY until manual promote
        - setWeight: 80
        - pause: { duration: 10m }
        # reaching the end => canary becomes stable (100%)
```

Step primitives you should know:
| Step | Effect |
|------|--------|
| `setWeight: N` | Route N% of traffic to the canary |
| `pause: { duration: 5m }` | Pause for a fixed time, then auto-continue |
| `pause: {}` | Pause **indefinitely** until a manual `promote` |
| `analysis:` | Run an AnalysisTemplate as a gate (see section 3) |
| `setCanaryScale:` | Control canary replica count independent of weight |
| `experiment:` | Launch a temporary experiment (extra ReplicaSets) |

**Traffic weighting — two levels (exam point):**
- **Replica-level (no mesh):** weight is approximated by pod counts. Coarse, and
  inaccurate at low replica counts.
- **Mesh/ingress-level (Istio, ALB, NGINX, SMI, Gateway API):** precise,
  per-request weighting independent of pod count. Preferred for production.

## Blue-Green — switch all at once

Runs the new version (green) fully in parallel, then flips the Service selector
from old (blue) to new in one cut — with optional pre/post analysis gates.

```yaml
spec:
  strategy:
    blueGreen:
      activeService: guestbook-active     # live traffic Service
      previewService: guestbook-preview   # points at new version BEFORE promotion
      autoPromotionEnabled: false         # false => wait for manual promotion
      autoPromotionSeconds: 30            # or auto-promote after N seconds
      scaleDownDelaySeconds: 30           # keep old RS around briefly for fast rollback
      prePromotionAnalysis:               # gate BEFORE the traffic switch
        templates:
          - templateName: smoke-tests
      postPromotionAnalysis:              # gate AFTER the switch (auto-rollback on fail)
        templates:
          - templateName: success-rate
```

| Field | Purpose |
|-------|---------|
| `activeService` | Service serving live traffic |
| `previewService` | Lets you test the new version before cutover |
| `autoPromotionEnabled` | `false` = require manual promotion |
| `scaleDownDelaySeconds` | Delay scaling down old RS (enables instant rollback) |
| `prePromotionAnalysis` | Must pass before traffic switches |
| `postPromotionAnalysis` | If it fails, traffic switches back to stable (abort) |

## Canary vs Blue-Green at a glance

| | Canary | Blue-Green |
|--|--------|------------|
| Traffic shift | gradual (%) | all-at-once |
| Resource use | ~1 version + small canary | 2 full versions simultaneously |
| Blast radius | small (only canary %) | full (after switch) |
| Rollback | reduce weight / abort | flip selector back |
| Needs mesh for precision | yes (for fine weights) | no (selector switch) |

---

# 3) Describe Analysis Template and AnalysisRun

Analysis is what makes Rollouts **automated and metric-driven** rather than just
a fancier traffic shifter.

## The three objects

| Object | Role |
|--------|------|
| **AnalysisTemplate** | Reusable *definition* of metric queries + pass/fail conditions (namespaced) |
| **ClusterAnalysisTemplate** | Same, but cluster-scoped (shared across namespaces) |
| **AnalysisRun** | A *live execution* of a template; produces a verdict (Successful / Failed / Inconclusive / Error) the Rollout observes |

Relationship: a `Rollout` references an `AnalysisTemplate`; when triggered it
creates an `AnalysisRun` (an instance, like a Job is an instance of a CronJob).
Templates are reusable, so one "5xx error-rate check" can gate many services.

## AnalysisTemplate

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:                              # Parameters passed in by the Rollout.
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m                   # How often to sample. Omit => single measurement.
      count: 5                       # How many measurements before completing.
      # Verdict conditions evaluate the query result:
      successCondition: result[0] >= 0.95
      failureCondition: result[0] < 0.90
      failureLimit: 2                # Allow up to 2 failed measurements before failing.
      # inconclusiveLimit: 3         # measurements that are neither pass nor fail
      provider:                      # WHERE the metric comes from:
        prometheus:
          address: http://prometheus.example.com:9090
          query: |
            sum(irate(http_requests_total{service="{{args.service-name}}",code!~"5.."}[5m]))
            /
            sum(irate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

**Metric providers** (know that several exist): Prometheus, Datadog, New Relic,
Wavefront, CloudWatch, Graphite, InfluxDB, a **Kubernetes Job** (run custom test
logic; exit code = verdict), and a **Web** provider (HTTP request → check response).

**Verdict fields:**
| Field | Meaning |
|-------|---------|
| `successCondition` | expression that means the measurement passed |
| `failureCondition` | expression that means it failed |
| `failureLimit` | how many failures tolerated before the run fails (default 0) |
| `inconclusiveLimit` | how many inconclusive results tolerated |
| `count` / `interval` | number and spacing of measurements |

## How a Rollout triggers analysis

### (a) Inline analysis step (canary) — blocks the rollout
```yaml
strategy:
  canary:
    steps:
      - setWeight: 20
      - pause: { duration: 5m }
      - analysis:                    # runs here; pass => continue, fail => abort
          templates:
            - templateName: success-rate
          args:
            - name: service-name
              value: guestbook-canary.default.svc.cluster.local
```

### (b) Background analysis (canary) — runs alongside the steps
```yaml
strategy:
  canary:
    analysis:                        # NOTE: under `canary:`, not in `steps:`
      templates:
        - templateName: success-rate
      startingStep: 2                # begin monitoring once step index 2 is reached
      args:
        - name: service-name
          value: guestbook-canary.default.svc.cluster.local
    steps:
      - setWeight: 20
      - pause: { duration: 10m }
      - setWeight: 40
```

### (c) Blue-green — `prePromotionAnalysis` / `postPromotionAnalysis`
(see section 2). Pre gates the switch; post can auto-rollback after it.

## Passing arguments

Args defined in the Rollout are **merged** with the template's `args`. Values can
be literals or pulled from the live object via `valueFrom.fieldRef`
(e.g. `metadata.labels['env']`, `status.alb.canaryTargetGroup.name`).

## AnalysisRun outcomes

| Result | Effect on Rollout |
|--------|-------------------|
| **Successful** | proceed / promote |
| **Failed** | abort → revert to stable |
| **Inconclusive** | pause for manual decision |
| **Error** | provider/measurement error; treated like failure for gating |

---

# Exam-relevant points (all three subtopics)

1. **Rollout replaces Deployment** and differs only by the `strategy` block;
   it manages ReplicaSets and can wrap an existing Deployment via `workloadRef`.
2. **Two strategies:** canary (gradual `setWeight`/`pause`/`analysis` steps) vs
   blue-green (parallel green + selector switch, with preview service).
3. **`pause: {}` vs `pause: {duration}`** — indefinite (manual promote) vs timed.
4. **Traffic weighting precision** needs a mesh/ingress; without it weight is
   approximated by replica counts.
5. **AnalysisTemplate = reusable definition; AnalysisRun = its execution.**
   ClusterAnalysisTemplate is the cluster-scoped variant.
6. **successCondition / failureCondition / failureLimit** drive the verdict;
   providers include Prometheus, Datadog, Job, Web, and more.
7. **Inline vs background vs pre/post-promotion** analysis — know where each is
   configured (inside `steps`, under `canary.analysis`, or under `blueGreen`).
8. **A failed AnalysisRun aborts the rollout and reverts to stable** — this is
   the automated, metric-driven rollback that defines progressive delivery.
