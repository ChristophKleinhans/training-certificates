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

### Canary Deployment

Rollout the new version to a small percentage of users first, then gradually increase if healthy. Real production testing with limited blast radius, but complex traffic splitting and monitoring

### A/B Testing

Same as canary but driven by business logic rather than percentages, f.i. premium and free users, users from germany or users from france, etc.

### Rolling Update

Gradually replace old instances with new ones, one by one or in batches.

### Feature Flag (Feature Toggles)

Simply deploy it with a feture Flag OFF and switch it to ON as soon as we want to go live with it.


