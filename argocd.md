# Certified Argo Project Associate

## Argo Workflows

### Argo Workflow fundamentals

- Argo Workflows is implemented as a Kubernetes CRD
- The `Workflow` is a live object
- Each step runs a single K8s pod (not a Job Pod, because Job pods would handle retries. Argo manages the retries itself). Each pod has multiple container inside it. At least these containers are present:
  1. **InitContainer**: Copies the `argoexec` binary into a shared volume so the other containers can use it and also downloads any input artifacts
  2. **Container**: This is the main container. It runs what we define in the YAML
  3. **Wait**: This is a sidecar which runs alongside the main container using the argoexec. It monitors the main container, collects output parameters and artifacts. When it finishes, it reports status back to the workflow controller.

<img width="650" height="308" alt="image" src="https://github.com/user-attachments/assets/6aa1776f-23bb-49f0-99e3-94139ab54f8f" />

- Argo Workflows is using `spec.templates` to define 9 different types of templates:

| Template | Type | Spawns pod | Key field | Use when |
|---|---|---|---|---|
| `steps` | Orchestrator | No | `steps:` | Sequential or parallel multi-step flow |
| `dag` | Orchestrator | No | `dag:` | Complex dependency graphs with explicit dependencies |
| `container` | Worker | Yes | `container:` | Run any container image |
| `script` | Worker | Yes | `script:` | Write inline code without building an image |
| `resource` | Worker | No | `resource:` | Create, patch or delete Kubernetes resources |
| `suspend` | Worker | No | `suspend:` | Manual approval gates or timed pauses |
| `http` | Worker | No | `http:` | Trigger webhooks or call external REST APIs |
| `plugin` | Worker | No | `plugin:` | Delegate to a custom executor plugin |
| `containerSet` | Worker | Yes (shared pod) | `containerSet:` | Multiple containers with dependencies in one pod |


