# Argo Events — CAPA Study Notes

Covers the two exam subtopics:
1. Understand Argo Events Fundamentals
2. Understand Argo Events Components and Architecture

---

# 1) Understand Argo Events Fundamentals

**Argo Events** is a **Kubernetes-native, event-driven automation framework**.
It listens for events from many external sources and **triggers actions** in
response — start an Argo Workflow, create/patch a K8s resource, call an HTTP
endpoint, invoke a Lambda, post to Slack, trigger an Argo CD sync, etc.

## The key mental model: EDGE-driven, not level-driven

This is the big contrast with the rest of the Argo family (and a favorite exam
point). Argo CD / Rollouts / Workflows are **reconciliation loops** that drive
state toward a declared target. **Argo Events is different — it reacts to discrete
events as they arrive** ("when X happens, do Y"). It's the event/edge-driven
member of the family, not a continuous reconciler.

```
   external event            Argo Events                  action
   ┌──────────────┐        ┌─────────────┐           ┌──────────────────┐
   │ webhook / S3  │  ───▶  │ capture →   │   ───▶     │ Argo Workflow /   │
   │ Kafka / cron  │        │ route →     │           │ K8s resource /    │
   │ git / SNS ... │        │ evaluate    │           │ HTTP / Lambda ... │
   └──────────────┘        └─────────────┘           └──────────────────┘
```

## What it's for

- **Event-driven CI/CD**: a Git push or image push triggers a build Workflow or
  an Argo CD sync — no polling delay.
- **Data/ML pipelines**: a file landing in S3 kicks off processing.
- **Automation / ChatOps**: a Slack command or webhook performs an action.
- **Reacting to cluster state**: a Kubernetes resource change fires a trigger.

## CloudEvents

Events are normalized into the **CloudEvents** spec (a CNCF standard envelope)
before flowing through the system, so downstream components have a consistent
shape regardless of the original source.

---

# 2) Understand Argo Events Components and Architecture

Argo Events is built from a small set of CRDs plus a controller. The **three
core components** are **EventSource → EventBus → Sensor (→ Trigger)**.

```
  ┌─────────────┐     publish      ┌──────────────┐    subscribe   ┌──────────┐
  │ EventSource │ ───────────────▶ │   EventBus   │ ─────────────▶ │  Sensor  │
  │ (capture &  │   (CloudEvents)  │ (transport / │                │ (deps +  │
  │  normalize) │                  │  pub-sub)    │                │ triggers)│
  └─────────────┘                  └──────────────┘                └────┬─────┘
        ▲                                                                │ fires
   external system                                                       ▼
   (webhook, S3, …)                                                  ┌──────────┐
                                                                     │ Trigger  │
                                                                     │ (action) │
                                                                     └──────────┘
```

## Component 1: EventSource

Captures events from an external system and **converts them to CloudEvents**,
then publishes them to the EventBus. Runs as its own Kubernetes deployment/pod.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: webhook
  namespace: argo-events
spec:
  webhook:                           # the source TYPE (one of many)
    example:                         # an event NAME within this source
      port: "12000"
      endpoint: /example
      method: POST
```

Supported source types are numerous: **webhook**, **AWS S3 / SNS / SQS**,
**GCP Pub/Sub**, **Kafka**, **NATS**, **AMQP**, **MQTT**, **Redis**,
**Git/GitHub/GitLab**, **calendar/cron**, **resource** (K8s object changes),
**file**, and more. One EventSource can define multiple named events.

## Component 2: EventBus

The **transport layer** — a publish/subscribe backbone that **decouples**
EventSources from Sensors. EventSources publish to it; Sensors subscribe.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default                      # Sensors/EventSources use "default" unless told otherwise.
  namespace: argo-events
spec:
  jetstream:                         # the modern, recommended implementation
    replicas: 3                      # HA: run an odd number for quorum
  # nats:                            # the older NATS Streaming (STAN) — deprecated
  #   native: { replicas: 3 }
  # kafka: { ... }                   # Kafka-backed bus is also supported
```

Key points:
- Implementations: **JetStream** (recommended), **NATS Streaming (deprecated)**,
  and **Kafka**.
- Decoupling is the whole value: many Sensors can react to one source, and
  sources/sensors can be mixed and matched freely.
- Run with multiple **replicas** for high availability in production.

## Component 3: Sensor (and its Triggers)

The **brain**. A Sensor subscribes to the EventBus, evaluates **dependencies**
(which events it's waiting for, plus optional filters), and fires **triggers**
(the actions) when its conditions are met. Runs as its own deployment/pod.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook
  namespace: argo-events
spec:
  template:
    serviceAccountName: operate-workflow-sa   # RBAC the trigger needs to act
  dependencies:                      # WHAT events this sensor waits for
    - name: test-dep
      eventSourceName: webhook       # must match the EventSource name
      eventName: example             # must match the event name within it
      # filters:                     # optional: only accept matching events
      #   data:
      #     - path: body.action
      #       type: string
      #       value: ["opened"]
  triggers:                          # WHAT to do when dependencies are satisfied
    - template:
        name: launch-workflow
        k8s:                         # trigger TYPE: create a K8s resource
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow         # e.g. start an Argo Workflow
              metadata:
                generateName: webhook-
              spec:
                entrypoint: whalesay
                serviceAccountName: operate-workflow-sa
                templates:
                  - name: whalesay
                    container:
                      image: docker/whalesay
                      command: [cowsay]
                      args: ["hello"]
```

### Trigger types (know that several exist)
Argo Workflow, **K8s resource** (create/update/patch/delete any object),
**HTTP request**, **AWS Lambda**, **Kafka / NATS / Pulsar** message,
**Slack**, **Argo CD** (via resource), **OpenWhisk**, **log**, and **custom**.

### Trigger conditions, filters, and policies
- **Dependencies** can be combined with boolean **`conditions`** (e.g.
  `"dep-a && dep-b"`) so a trigger fires only on a combination of events.
- **Filters** (data, context, time, expr) drop events that don't match criteria.
- **Trigger parameterization**: values from the event payload can be injected
  into the trigger resource (e.g. pass a Git commit SHA into the Workflow).
- **Retry / policy** settings control re-attempts and success criteria.

## The controller (control plane)

A single **controller-manager** (the EventSource/Sensor/EventBus controller)
watches these CRDs and creates/manages the underlying deployments and pods. All
of this typically lives in the **`argo-events`** namespace, with a ServiceAccount
+ ClusterRoles granting the needed Kubernetes permissions.

## End-to-end flow (memorize this chain)

```
external event → EventSource (capture + CloudEvents) → EventBus (route)
   → Sensor (match dependencies + filters) → Trigger (perform action)
```

## Component summary

| Component | Kind | Role | Runs as |
|-----------|------|------|---------|
| **EventSource** | `EventSource` | capture external events → CloudEvents → publish | its own pod |
| **EventBus** | `EventBus` | pub/sub transport that decouples sources & sensors | JetStream/Kafka pods |
| **Sensor** | `Sensor` | subscribe, evaluate dependencies/filters, fire triggers | its own pod |
| **Trigger** | (inside Sensor) | the actual action executed | — |
| **Controller** | controller-manager | watches CRDs, manages the above | one deployment |

---

# Exam-relevant points

1. **Argo Events is event/edge-driven**, the exception to the reconciliation-loop
   pattern used by Argo CD / Rollouts / Workflows.
2. **Three core components: EventSource → EventBus → Sensor**, with **Triggers**
   defined inside the Sensor. Know the order and what each does.
3. **EventSource** captures + normalizes to **CloudEvents** and publishes;
   **EventBus** is the pub/sub transport that **decouples**; **Sensor** evaluates
   **dependencies** and fires **triggers**.
4. **EventBus implementations**: JetStream (recommended), NATS Streaming
   (deprecated), Kafka. Use replicas for HA.
5. **Dependencies vs triggers**: dependencies = the events awaited (+ filters);
   triggers = the actions taken. Boolean `conditions` combine dependencies.
6. **Trigger types are many** — most exam-relevant is launching an **Argo
   Workflow** or creating a **K8s resource**; also HTTP, Lambda, Slack, etc.
7. **The controller** manages all three CRDs; components run as their own pods,
   usually in the `argo-events` namespace.
8. **Common pairing**: Argo Events triggers **Argo Workflows** (event-driven CI)
   or **Argo CD syncs** (event-driven CD), removing polling delay.
