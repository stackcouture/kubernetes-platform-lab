# Kubernetes Labels and Selectors

## Overview

**Labels** are key-value pairs attached to Kubernetes objects.

They are used to identify and organize resources.

Example:

```yaml
labels:
  app: voting
  component: frontend
  environment: development
```

A **selector** is used to find or target Kubernetes objects based on their labels.

The relationship is:

```text
Labels
   │
   │ identify
   ▼
Kubernetes Objects
   │
   │ selected by
   ▼
Selectors
```
---
## Why Labels Are Important

Labels are heavily used in real Kubernetes environments to identify, organize, and manage resources.

Common use cases include:

* Service-to-Pod selection
* Deployment management
* Monitoring
* NetworkPolicies
* Scheduling
* GitOps
* Resource organization
* Environment separation
* Application and component identification

For example:

```yaml
app: voting
component: frontend
environment: production
```

These labels allow Kubernetes and other platform tools to identify and target workloads without relying on the Pod name.

> **Key Point:** Labels provide a flexible way to identify Kubernetes resources based on their characteristics rather than their individual names.

---
## Label Structure

A Kubernetes label consists of a **key-value pair**:

```text
key=value
```

### Example

```yaml
labels:
  app: voting
```

Multiple labels can be added to the same Kubernetes object:

```yaml
labels:
  app: voting
  component: frontend
  environment: development
  tier: web
```

Each label provides additional information that can be used to identify, organize, and select Kubernetes resources.

---
# Labels vs Names

A **resource name** identifies one specific Kubernetes object.

Example:

```text
vote-7f8c9d
```

A **label** identifies the role or characteristics of an object.

Example:

```text
app=voting
component=frontend
```

## Why Labels Are Important

Pod names can change when a Deployment creates or recreates Pods.

For example:

```text
vote-7f8c9d
vote-6a4b2c
vote-9d7e1f
```

These Pods may represent the same application workload.

Therefore, Kubernetes components generally use **labels and selectors** rather than relying on individual Pod names.

```text
Label
  │
  │ app=voting
  ▼
Multiple Pods
  ├── vote-7f8c9d
  ├── vote-6a4b2c
  └── vote-9d7e1f
```

> **Key Point:** Names identify individual objects, while labels identify the role or characteristics of objects and allow Kubernetes to work with groups of resources.

---
## Example Pod With Labels

The following Pod manifest demonstrates how to attach multiple labels to a Kubernetes Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vote
  labels:
    app: voting
    component: frontend
    environment: development
spec:
  containers:
    - name: vote
      image: nginx:1.27
      ports:
        - containerPort: 80
```

### Create the Pod

```bash
kubectl apply -f labels-selectors.yaml
```

### Verify Labels

```bash
kubectl get pods --show-labels
```

Example output:

```text
NAME    READY   STATUS    RESTARTS   AGE   LABELS
vote    1/1     Running   0          ...   app=voting,component=frontend,environment=development
```

---




