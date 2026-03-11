# GitOps Associate

## GitOps Terminology

### Continuous

The system will always keep attempting to reach the desired states, even after failures. It DOES NOT mean "instantaneous" or "real-time". The reconcilation loop runs constantly in the background. The agent observes the actual state and compares it with the desired state stored in Git. Is there a drift, it corrects it.
