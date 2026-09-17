## Overview Namespace 

A Kubernetes Namespace provides a logical boundary for Kubernetes resources within a cluster.

---
### Namespaces are mainly used to:

* Organize Kubernetes resources
* Separate applications and teams
* Apply access control using RBAC
* Apply resource limits and quotas
* Apply security policies
* Separate environments such as development, staging, and production
* Avoid naming conflicts between resources

A namespace does not create a separate Kubernetes cluster. All namespaces share the same underlying cluster control plane and worker nodes.

---
### Why Do We Need Namespaces?

Without namespaces, resources are created in a single default namespace.

For example:

```text
Kubernetes Cluster
│
├── Pod
├── Deployment
├── Service
├── ConfigMap
├── Secret
└── ...
```

As the cluster grows, managing hundreds or thousands of resources becomes difficult.

Namespaces provide logical separation:

```text
Kubernetes Cluster
│
├── development
│   ├── vote
│   ├── result
│   └── redis
│
├── staging
│   ├── vote
│   ├── result
│   └── redis
│
└── production
    ├── vote
    ├── result
    └── redis
```

The same resource name can exist in different namespaces.

For example:

```text
development/vote
staging/vote
production/vote
```

These are different resources because they belong to different namespaces.

---
### Namespace vs Cluster

A namespace is **not a security boundary equivalent to a separate cluster**.

For example:

```text
Cluster A
│
├── dev namespace
├── staging namespace
└── production namespace
```

All three namespaces use the same cluster.

A stronger isolation model would be:

```text
Dev Cluster
Staging Cluster
Production Cluster
```

---
### Namespace Isolation

Namespaces provide:

* Logical isolation
* Resource organization
* RBAC boundaries
* ResourceQuota
* LimitRange
* Policy targeting

Namespaces do **not** automatically provide:

* Complete network isolation
* Separate worker nodes
* Separate control planes
* Complete security isolation

For stronger workload isolation, Kubernetes features such as **NetworkPolicy**, node taints and tolerations, RBAC, Pod Security controls, and separate clusters may be used.

### Creating a Namespace

#### Using `kubectl`

Create a namespace using:

```bash
kubectl create namespace development
```

#### Verify the Namespace

List all namespaces:

```bash
kubectl get namespaces
```

Short form:

```bash
kubectl get ns
```

#### Expected Output

```text
NAME              STATUS   AGE
default           Active   ...
kube-system       Active   ...
kube-public       Active   ...
kube-node-lease   Active   ...
development       Active   ...
```

### Creating a Namespace Using YAML

Create a file named:

```text
namespaces.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: development
    managed-by: kubectl
```

## Apply the Namespace

```bash
kubectl apply -f namespaces.yaml
```

## Verify the Namespace

```bash
kubectl get namespace development
```

Expected output:

```text
NAME          STATUS   AGE
development   Active   ...
```

---
## Namespace Labels

Namespaces can have **labels** that identify or categorize the namespace.

Example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
```

Labels can later be used by Kubernetes policies, automation, monitoring, and other tools.

## Check Namespace Labels

```bash
kubectl get namespace production --show-labels
```

Example output:

```text
NAME         STATUS   AGE   LABELS
production   Active   ...   environment=production,team=platform
```

---
## Deploying Resources Into a Namespace

A Kubernetes resource can specify the target namespace directly in its manifest using the `metadata.namespace` field.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: development
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

## Apply the Resource

```bash
kubectl apply -f pod.yaml
```

## Verify the Pod

```bash
kubectl get pods -n development
```

Example output:

```text
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          ...
```

---
## Namespace-Specific Resource Names

Resource names only need to be unique **within a namespace**.

For example:

```text
development
└── nginx

production
└── nginx
```

Both resources can have the same name:

```text
nginx
```

because they belong to different namespaces.

They are identified using their namespace and resource name:

```text
development/nginx
production/nginx
```

These are **different Kubernetes objects**.

### Key Point

A resource name must be unique within its namespace, but the same resource name can be reused across different namespaces.

---
## Working With Namespaces Using `kubectl`

### List Namespaces

```bash
kubectl get ns
```

### Get Resources in a Namespace

List pods in the `development` namespace:

```bash
kubectl get pods -n development
```

### Get All Common Resources

```bash
kubectl get all -n development
```

### Describe a Namespace

```bash
kubectl describe namespace development
```

### Delete a Namespace

```bash
kubectl delete namespace development
```

> **Warning:** Deleting a namespace also deletes the namespaced resources inside it.

---
## Current Namespace Context

By default, `kubectl` commands operate against the `default` namespace.

For example:

```bash
kubectl get pods
```

is equivalent to:

```bash
kubectl get pods -n default
```

### Specify a Namespace

You can explicitly specify a namespace using the `-n` or `--namespace` option:

```bash
kubectl get pods -n development
```

### Set the Current Namespace

For frequent work with a specific namespace, you can configure the namespace in the current `kubectl` context:

```bash
kubectl config set-context --current --namespace=development
```

After setting the namespace, commands such as:

```bash
kubectl get pods
```

will automatically operate against:

```text
development
```

instead of:

```text
default
```

### Verify the Current Namespace

```bash
kubectl config view --minify --output 'jsonpath={..namespace}'
```

Expected output:

```text
development
```

### Key Point

Setting the current namespace changes the default namespace used by `kubectl` commands for the **current context**. It does not create or modify the namespace itself.

---
## Namespace-Scoped vs Cluster-Scoped Resources

Not every Kubernetes resource belongs to a namespace.

Kubernetes resources are broadly categorized as either **namespace-scoped** or **cluster-scoped**.

### Namespace-Scoped Resources

Namespace-scoped resources belong to a specific namespace.

Examples:

```text
Pod
Deployment
Service
ConfigMap
Secret
ServiceAccount
Role
RoleBinding
ResourceQuota
LimitRange
```

These resources can be accessed by specifying a namespace:

```bash
kubectl get pods -n development
```

List namespace-scoped resources:

```bash
kubectl api-resources --namespaced=true
```

---

### Cluster-Scoped Resources

Cluster-scoped resources belong to the entire Kubernetes cluster and are not associated with a specific namespace.

Examples:

```text
Node
Namespace
PersistentVolume
ClusterRole
ClusterRoleBinding
StorageClass
CustomResourceDefinition
```

List cluster-scoped resources:

```bash
kubectl api-resources --namespaced=false
```

### Why This Distinction Matters

Understanding the difference between namespace-scoped and cluster-scoped resources is important when designing:

* RBAC
* Resource management
* Security policies
* Cluster architecture
* Multi-tenant Kubernetes environments

#### Key Point

**Namespace-scoped resources** exist within a specific namespace, while **cluster-scoped resources** apply across the Kubernetes cluster.

---
## Namespaces and RBAC

Namespaces are commonly used with **Role-Based Access Control (RBAC)** to control access to resources within a specific namespace.

For example:

```text id="r7n4ax"
Development Team
       │
       ▼
development namespace
       │
       ├── Pods
       ├── Deployments
       └── Services
```

A Kubernetes **Role** can grant a team permission to access specific resources within the `development` namespace.

### Example Role

```yaml id="5v4z8r"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
```

This Role allows the associated identity to:

* Get Pods and Services
* List Pods and Services
* Watch Pods and Services

within the `development` namespace.

### Important

The Role is **namespace-scoped** because it is defined with:

```yaml id="4d3m8j"
namespace: development
```

It does not automatically provide access to resources in other namespaces.

For example:

```text id="5k5y2c"
developer Role
      │
      └── development
          ├── Pods       ✓
          └── Services   ✓

      staging
          ├── Pods       ✗
          └── Services   ✗
```

>

---
## Namespaces and ResourceQuota

Namespaces can have **ResourceQuotas** to limit the amount of resources that workloads can consume within a namespace.

### Example ResourceQuota

```yaml id="x7r2pv"
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

This quota limits the `development` namespace to:

| Resource        | Maximum |
| --------------- | ------: |
| CPU Requests    |   4 CPU |
| Memory Requests |    8 Gi |
| CPU Limits      |   8 CPU |
| Memory Limits   |   16 Gi |
| Pods            |      20 |

ResourceQuota helps prevent a namespace from consuming unlimited cluster resources.

### Check ResourceQuota

```bash
kubectl get resourcequota -n development
```

### Describe ResourceQuota

```bash
kubectl describe resourcequota development-quota -n development
```

### Key Point

**ResourceQuota controls the total resource consumption of a namespace**, helping maintain fair resource allocation and protecting the cluster from excessive resource usage.

---
## Namespaces and NetworkPolicy

Namespaces can be used as part of **network segmentation** in Kubernetes.

Example architecture:

```text
production
│
├── frontend
├── backend
└── database
```

A **NetworkPolicy** can restrict which workloads are allowed to communicate with other workloads.

For example:

```text
frontend
   │
   │ ✓ Allowed
   ▼
backend
   │
   │ ✓ Allowed
   ▼
database
```

Unwanted communication can be blocked using appropriate `NetworkPolicy` rules.

### Important

> **Creating a namespace alone does not block network traffic between namespaces.**

By default, namespaces do not provide network isolation.

Network isolation requires:

* A networking implementation that supports `NetworkPolicy`
* Appropriate `NetworkPolicy` resources
* Correct pod labels and selectors
* Proper ingress and egress rules

### Key Point

**Namespaces provide logical organization, while NetworkPolicy provides network-level traffic control.**

Using them together allows Kubernetes administrators to implement stronger workload and network segmentation.

---
## Namespaces and Pod Security

Namespaces can be used to apply **Pod Security Standards (PSS)** to workloads within a specific namespace.

For example:

```yaml id="w7q0jd"
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

The `pod-security.kubernetes.io/enforce` label tells Kubernetes to enforce the **`restricted` Pod Security Standard** for the `production` namespace.

### Pod Security Standards

Kubernetes provides three Pod Security Standards:

* `privileged` — Provides unrestricted permissions.
* `baseline` — Prevents known privilege-escalation configurations.
* `restricted` — Enforces stronger security practices for pods.

For production workloads, the `restricted` standard can be used when applications are compatible with its security requirements.

### Key Point

**Namespaces provide the scope for applying Pod Security Standards, while Pod Security Admission enforces the selected security level within that namespace.**

---
## Recommended Namespace Structure

The namespace strategy should be based on the organization's **team structure, security requirements, deployment model, and cluster architecture**.

### Small Platform

For a small platform, environments can be separated into dedicated namespaces:

```text
Kubernetes Cluster
│
├── development
├── staging
├── production
└── monitoring
```

This approach provides a simple and easy-to-manage structure.

### Larger Platform

For a larger platform, namespaces can be organized based on application, data, monitoring, ingress, and security responsibilities:

```text
Kubernetes Cluster
│
├── applications
│   ├── vote
│   ├── result
│   └── worker
│
├── data
│   ├── redis
│   └── postgres
│
├── monitoring
│   ├── prometheus
│   └── grafana
│
├── ingress
│
└── security
```

### Namespace Strategy

There is no single namespace structure that works for every Kubernetes environment.

The exact strategy depends on:

* Team structure
* Security requirements
* Application architecture
* Deployment model
* Resource management requirements
* RBAC requirements
* Network segmentation
* Cluster architecture

> **Key Point:** Choose a namespace strategy that provides clear ownership, appropriate isolation, manageable access control, and efficient resource management.

---
## Namespace Naming Best Practices

Use **simple, predictable, and consistent** namespace names.

### Recommended Names

Good examples:

```text
development
staging
production
monitoring
ingress
security
```

Avoid unnecessarily complicated names such as:

```text
production-environment-application-services
```

### Naming Guidelines

Follow these practices:

* Use lowercase names
* Keep names short and descriptive
* Use consistent naming conventions
* Avoid unnecessary special characters
* Avoid overly long names
* Choose names that clearly represent their purpose

### Benefits of Consistent Naming

A consistent namespace naming strategy makes:

* `kubectl` commands easier to use
* RBAC configuration easier to manage
* Monitoring and observability easier
* GitOps configuration easier to maintain
* Troubleshooting easier

> **Key Point:** Namespace names should be simple, predictable, and meaningful so that administrators and automation tools can work with them consistently.

---
## Important Namespace Limitations

Namespaces are useful for **logical organization and resource isolation**, but they are not a complete isolation mechanism by themselves.

### What a Namespace Does Not Automatically Provide

A namespace does **not** automatically provide:

```text
❌ Network isolation
❌ Dedicated nodes
❌ Dedicated control plane
❌ Dedicated CPU
❌ Dedicated memory
❌ Complete security isolation
```

These capabilities require additional Kubernetes features or underlying infrastructure.

### Additional Isolation Mechanisms

A stronger workload separation model can combine namespaces with:

```text
Namespace
   │
   ├── RBAC
   ├── ResourceQuota
   ├── LimitRange
   ├── NetworkPolicy
   ├── Pod Security
   └── Node Scheduling
```

### Purpose of Each Mechanism

| Feature         | Purpose                                     |
| --------------- | ------------------------------------------- |
| Namespace       | Logical resource organization and scope     |
| RBAC            | Controls access to Kubernetes resources     |
| ResourceQuota   | Limits total resource consumption           |
| LimitRange      | Controls resource requirements and defaults |
| NetworkPolicy   | Controls network communication              |
| Pod Security    | Enforces pod security requirements          |
| Node Scheduling | Controls where workloads run                |

> **Key Point:** A namespace alone provides limited logical separation. Combining namespaces with RBAC, resource controls, network policies, pod security, and appropriate node scheduling provides much stronger workload separation.

---
## Real-World Platform Example

A production **GKE/EKS** cluster can be organized using separate namespaces for applications, data services, and platform components.

Example:

```text id="5q0j4f"
Cluster
│
├── app-vote
├── app-result
├── app-worker
│
├── redis
├── postgres
│
├── monitoring
├── logging
├── ingress
└── security
```

Each namespace can have its own configuration and policies, such as:

```text id="1e3p4k"
RBAC
ResourceQuota
LimitRange
NetworkPolicy
Pod Security
Secrets
ServiceAccounts
```

This approach provides:

* Logical separation of workloads
* Namespace-level access control
* Resource management
* Network segmentation
* Security policy enforcement
* Better organization of platform components
* Easier monitoring and troubleshooting

> **Key Point:** Combining namespaces with RBAC, resource controls, network policies, pod security, and service identities creates a more controlled and manageable multi-team Kubernetes environment.

---
## Useful Commands

### List Namespaces

```bash
kubectl get ns
```

### Create Namespace

```bash
kubectl create ns development
```

### Delete Namespace

```bash
kubectl delete ns development
```

### List Pods in a Namespace

```bash
kubectl get pods -n development
```

### List Deployments

```bash
kubectl get deployments -n development
```

### List Services

```bash
kubectl get services -n development
```

### List All Resources

```bash
kubectl get all -n development
```

### Describe Namespace

```bash
kubectl describe ns development
```

### Set Current Namespace

Set `development` as the default namespace for the current `kubectl` context:

```bash
kubectl config set-context --current --namespace=development
```

### Find Namespace-Scoped Resources

```bash
kubectl api-resources --namespaced=true
```

### Find Cluster-Scoped Resources

```bash
kubectl api-resources --namespaced=false
```
---
## Interview Questions

### Q1. What is a Kubernetes Namespace?

A **Namespace** provides a logical boundary for organizing and managing Kubernetes resources within a cluster.

### Q2. Why do we use Namespaces?

Namespaces are used to:

* Organize Kubernetes resources
* Define boundaries for RBAC
* Apply resource quotas and limits
* Target security and network policies
* Separate teams and environments
* Support multi-tenant cluster management

### Q3. Does a Namespace provide network isolation?

**No.**

Creating a namespace does not automatically isolate network traffic.

Network isolation requires **NetworkPolicy** and a networking implementation that supports it.

### Q4. Can two namespaces have resources with the same name?

**Yes.**

For example:

```text
development/nginx
production/nginx
```

These are different Kubernetes resources because they belong to different namespaces.

### Q5. What happens when you delete a Namespace?

Deleting a namespace deletes the namespace and its **namespaced resources**.

> **Warning:** Deleting a namespace can result in the deletion of workloads and other resources contained within it.

### Q6. Are Nodes namespace-scoped?

**No.**

Nodes are **cluster-scoped** resources.

```text
Node → Cluster-scoped
```

### Q7. Are Pods namespace-scoped?

**Yes.**

Pods are **namespace-scoped** resources.

```text
Pod → Namespace-scoped
```

### Q8. Can RBAC permissions be restricted to a Namespace?

**Yes.**

A `Role` and `RoleBinding` can grant permissions within a specific namespace.

For example:

```text
Role
 │
 └── RoleBinding
       │
       └── development namespace
```

The permissions do not automatically apply to other namespaces.

### Q9. How can you limit CPU and memory usage per Namespace?

Use:

* **ResourceQuota** — Controls the total resource consumption of a namespace.
* **LimitRange** — Defines default, minimum, and maximum resource requirements for workloads within a namespace.

### Q10. Would you use Namespaces to separate production and development?

Namespaces **can** be used to separate development and production workloads when the required isolation level is appropriate.

For stronger isolation, separate clusters may be considered depending on:

* Security requirements
* Compliance requirements
* Availability requirements
* Operational requirements
* Multi-tenancy requirements
* Blast-radius considerations

> **Key Point:** Namespace separation provides logical boundaries, but it should not automatically be treated as equivalent to separate-cluster isolation.

---
## Hands-On Lab

This lab demonstrates the basic Kubernetes Namespace workflow, including creating a namespace, deploying a Pod, verifying resources, and cleaning up.

### Step 1 — Create Namespace

```bash
kubectl create namespace development
```

### Step 2 — Verify Namespace

```bash
kubectl get namespaces
```

### Step 3 — Deploy a Pod

Create a file named `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: development
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply the manifest:

```bash
kubectl apply -f pod.yaml
```

### Step 4 — Verify Pod

```bash
kubectl get pods -n development
```

## Step 5 — Describe Pod

```bash
kubectl describe pod nginx -n development
```

## Step 6 — Delete Pod

```bash
kubectl delete pod nginx -n development
```

## Step 7 — Delete Namespace

```bash
kubectl delete namespace development
```

> **Warning:** Deleting the namespace also deletes the namespaced resources contained within it.

---

# Key Takeaways

Remember these points for interviews:

```text
Namespace
   │
   ├── Logical resource boundary
   ├── Resource organization
   ├── RBAC boundary
   ├── ResourceQuota boundary
   ├── LimitRange boundary
   ├── Policy targeting
   └── Multi-team / multi-environment management
```

The most important concept is:

> **A namespace is a logical management boundary, not a complete isolation boundary.**

For production-grade workload separation, namespaces can be combined with:

```text
RBAC
+
NetworkPolicy
+
ResourceQuota
+
LimitRange
+
Pod Security
+
Node Scheduling
+
Secrets Management
```

This combination provides stronger control over **access, resources, networking, security, workload placement, and sensitive configuration**.
