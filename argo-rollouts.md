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
