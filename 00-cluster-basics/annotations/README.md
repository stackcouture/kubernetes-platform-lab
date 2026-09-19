# Kubernetes Annotations

## 1. Overview

**Annotations** are key-value metadata attached to Kubernetes objects.

They are primarily used to store **additional information, configuration, or metadata** that is not intended to be used for selecting or grouping objects.

Example:

```yaml
metadata:
  annotations:
    description: "Frontend application"
    owner: "platform-team"
```

Annotations can contain information used by:

* Kubernetes controllers
* Ingress controllers
* Cloud integrations
* Monitoring tools
* GitOps tools
* CI/CD systems
* Custom operators
* Platform automation

> **Key Point:** Annotations store additional metadata or configuration for Kubernetes objects, while labels are primarily used for identification, grouping, and selection.
