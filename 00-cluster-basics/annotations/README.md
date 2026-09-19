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

---
## 2. Labels vs Annotations

This is one of the most important concepts to understand when working with Kubernetes metadata.

| Feature                 | Labels                     | Annotations                        |
| ----------------------- | -------------------------- | ---------------------------------- |
| Purpose                 | Identify and group objects | Store additional metadata          |
| Used by selectors       | Yes                        | No                                 |
| Used for filtering      | Yes                        | No                                 |
| Used by Services        | Yes                        | No                                 |
| Used by Deployments     | Yes                        | No                                 |
| Can store configuration | Limited                    | Yes                                |
| Intended for querying   | Yes                        | No                                 |
| Typical use             | `app=voting`               | `description=Frontend application` |

### Simple Rule

```text
Labels      → Identify / Select
Annotations → Describe / Configure
```

### Example

```yaml
metadata:
  labels:
    app: voting

  annotations:
    description: "Voting frontend application"
```

Here:

```text
app=voting
```

can be used by a **Service selector**.

However:

```text
description=Voting frontend application
```

cannot be used by a Service selector.

### Key Point

**Labels** are designed for identification, grouping, filtering, and selection.

**Annotations** are designed to store additional information, configuration, or metadata that does not need to be used for object selection.

---
## 3. Why Do We Need Annotations?

Annotations allow applications and Kubernetes components to attach additional information to resources without affecting resource selection.

Common uses include:

* Ingress configuration
* Load balancer configuration
* Certificate configuration
* Monitoring configuration
* Deployment metadata
* Ownership information
* Documentation
* External system configuration
* Controller-specific settings

> **Key Point:** Annotations provide a flexible way to attach additional metadata or configuration to Kubernetes resources without affecting labels, selectors, or resource selection.

---
## 4. Basic Annotation Example

Annotations can be added to Kubernetes objects using the `metadata.annotations` field.

Example:

```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  annotations:
    description: "Nginx web server"
    owner: "platform-team"
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

### Create the Pod

```bash 
kubectl apply -f annotations.yaml
```

### Check Annotations

```bash 
kubectl describe pod nginx
```

You will see:

```text 
Annotations:
  description: Nginx web server
  owner: platform-team
```

> **Key Point:** Annotations can store additional metadata about a Kubernetes resource without affecting how the resource is selected by Kubernetes selectors.

---
## 5. Multiple Annotations

A Kubernetes object can have **multiple annotations**.

Example:

```yaml 
metadata:
  annotations:
    description: "Voting frontend"
    owner: "platform-team"
    environment: "development"
    documentation: "https://example.com/voting"
```

The annotations can be visualized as:

```text 
Annotations
│
├── description
├── owner
├── environment
└── documentation
```

Annotations are useful when multiple tools or systems need to attach additional metadata to the same Kubernetes resource.

> **Key Point:** Multiple annotations can coexist on a single Kubernetes object, allowing different tools and platform components to store relevant metadata without affecting resource selection.

---
## 6. Annotation Keys

Annotation keys commonly use a **DNS-style prefix** to provide a unique and organized naming convention.

### Example

```yaml
metadata:
  annotations:
    example.com/owner: "platform-team"
```

Another example:

```yaml
metadata:
  annotations:
    platform.example.com/team: "devops"
```

### Annotation Key Format

The general format is:

```text
prefix/name
```

For example:

```text
example.com/owner
```

Where:

```text
example.com
```

is the **prefix** and:

```text
owner
```

is the **annotation name**.

### Structure

```text
example.com/owner
     │       │
     │       └── Annotation name
     │
     └────────── Prefix
```

> **Key Point:** Using a DNS-style prefix helps avoid naming conflicts when annotations are defined or consumed by different tools, controllers, or organizations.

---
## 7. Kubernetes and Controller Annotations

Many Kubernetes controllers use **annotations** to configure their behavior.

For example, an Ingress controller may use annotations to configure:

* SSL/TLS
* Redirects
* Load balancer behavior
* Rewrite rules
* Authentication
* Timeouts

Example:

```yaml 
metadata:
  annotations:
    example.com/ssl-redirect: "true"
```

The exact annotations supported depend on the **controller, platform, or cloud provider** being used.

### Important

> **Do not assume an annotation is universally supported by Kubernetes.**

Many annotations are specific to a particular controller or platform.

For example:

```text
Ingress
   │
   └── Controller-specific annotations
          │
          ├── SSL/TLS
          ├── Redirects
          ├── Load Balancer
          ├── Authentication
          └── Timeouts
```

> **Key Point:** Kubernetes provides the annotation mechanism, but the meaning and behavior of many annotations are defined by the controller or platform that consumes them.

---
## 8. Ingress Annotation Example

A common real-world use of annotations is configuring an **Ingress controller**.

Example:

```yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: voting-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: voting.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vote-service
                port:
                  number: 80
```

The annotation:

```text 
nginx.ingress.kubernetes.io/ssl-redirect
```

is interpreted by the **NGINX Ingress controller**.

It instructs the controller to handle HTTP-to-HTTPS redirection according to its supported configuration.

This annotation is **controller-specific** and is not a generic Kubernetes Service selector.

### Key Point

```text 
Kubernetes
   │
   └── Annotation
          │
          └── NGINX Ingress Controller
                    │
                    └── Interprets the annotation
```

> **Important:** Always refer to the documentation of the specific Ingress controller or platform to determine which annotations are supported and what behavior they provide.

---
## 9. Annotations for Ownership

Annotations can be used to store **ownership and organizational metadata** for Kubernetes resources.

Example:

```yaml 
metadata:
  annotations:
    platform.example.com/owner: "platform-team"
    platform.example.com/cost-center: "engineering"
```

This information can help platform teams identify:

* Resource ownership
* Responsible teams
* Cost centers
* Organizational metadata

### Example

```text 
Deployment
│
├── Labels
│   └── app=voting
│
└── Annotations
    ├── owner=platform-team
    └── cost-center=engineering
```

> **Key Point:** Labels are typically used to identify and select workloads, while annotations can store additional ownership, organizational, or operational metadata.

---
## 11. Annotations and GitOps

Annotations are frequently used by **GitOps tools and Kubernetes controllers** to store metadata about resources.

Example:

```yaml 
metadata:
  annotations:
    deployment.example.com/source: "git"
    deployment.example.com/repository: "platform-apps"
```

A typical GitOps workflow might look like:

```text 
Git Repository
      │
      ▼
GitOps Controller
      │
      ▼
Kubernetes Resource
      │
      ├── Labels
      └── Annotations
```

Annotations can be used to store information such as:

* Source repository
* Deployment metadata
* Application information
* Controller-specific configuration
* Synchronization or management metadata

The exact annotations and their meaning depend on the **GitOps tool or controller** being used.

> **Key Point:** Kubernetes provides the annotation mechanism, while GitOps tools and controllers define how specific annotations are interpreted and used.

---
## 12. Annotations and Monitoring

Annotations can also be used by **monitoring systems and integrations** to store or communicate monitoring-related metadata.

For example, a monitoring controller could use:

```yaml 
metadata:
  annotations:
    monitoring.example.com/scrape: "true"
```

A monitoring system could inspect this annotation and determine whether the resource should be monitored.

### Monitoring Workflow

```text 
Kubernetes Resource
        │
        └── Annotation
              │
              │ monitoring.example.com/scrape: "true"
              ▼
      Monitoring System
              │
              ▼
        Monitor Resource
```

### Modern Kubernetes Monitoring

Modern Kubernetes monitoring stacks often use dedicated Kubernetes resources such as:

* `ServiceMonitor`
* `PodMonitor`

These provide a more structured way to define monitoring targets and configuration rather than relying only on annotations.

> **Key Point:** Annotations can be used for monitoring integrations, but the exact behavior depends on the monitoring tool or controller. Dedicated resources such as `ServiceMonitor` and `PodMonitor` are commonly used in modern Kubernetes monitoring setups.

---
## 13. Annotations Are Not Selectors

This is a **critical distinction** between labels and annotations.

Suppose a Pod has:

```yaml 
metadata:
  labels:
    app: voting

  annotations:
    owner: platform-team
```

A Service can use the label as a selector:

```yaml 
selector:
  app: voting
```

However, this will **not** work:

```yaml 
selector:
  owner: platform-team
```

because:

```text 
owner=platform-team
```

is an **annotation**, not a label.

### Correct Architecture

```text 
Pod
│
├── Labels
│   └── app=voting
│          ▲
│          │
│       Service
│       selector
│
└── Annotations
    └── owner=platform-team
```

### Key Point

```text 
Labels      → Used for selection
Annotations → Used for metadata/configuration
```

A Kubernetes Service selector can select Pods using **labels**, but it cannot select Pods using annotations.

---
## 14. Viewing Annotations

Annotations attached to Kubernetes resources can be viewed using `kubectl`.

### Using `kubectl describe`

```bash
kubectl describe pod nginx
```

The output includes the annotations associated with the Pod.

### Using JSONPath

To display all annotations:

```bash
kubectl get pod nginx \
  -o jsonpath='{.metadata.annotations}'
```

To display a specific annotation:

```bash
kubectl get pod nginx \
  -o jsonpath='{.metadata.annotations.description}'
```

### Key Point

`kubectl describe` is useful for viewing annotations along with other resource details, while **JSONPath** is useful when you need to extract specific annotation values for scripts or automation.

---
## 15. Adding an Annotation

You can add an annotation to an existing Kubernetes resource using the `kubectl annotate` command.

### Add an Annotation

```bash 
kubectl annotate pod nginx \
  description="Nginx web server"
```

This adds the following annotation to the `nginx` Pod:

```text 
description: Nginx web server
```

### Verify the Annotation

```bash 
kubectl describe pod nginx
```

The annotation will appear under the **Annotations** section.

> **Key Point:** `kubectl annotate` allows you to add or update annotations on existing Kubernetes resources without modifying the resource manifest directly.

---
## 16. Updating an Annotation

If an annotation already exists, use the `--overwrite` option to update its value.

### Update an Annotation

```bash 
kubectl annotate pod nginx \
  description="Updated Nginx web server" \
  --overwrite
```

The existing annotation:

```text 
description: Nginx web server
```

will be updated to:

```text 
description: Updated Nginx web server
```

### Verify the Updated Annotation

```bash 
kubectl get pod nginx \
  -o jsonpath='{.metadata.annotations.description}'
```

Expected output:

```text
Updated Nginx web server
```

> **Key Point:** Use `--overwrite` when updating an annotation that already exists.

---
## 17. Removing an Annotation

An annotation can be removed using the `kubectl annotate` command by adding `-` after the annotation key.

### Remove an Annotation

```bash 
kubectl annotate pod nginx description-
```

This removes the `description` annotation from the `nginx` Pod.

### Verify

```bash 
kubectl get pod nginx \
  -o jsonpath='{.metadata.annotations}'
```

> **Key Point:** The `-` suffix tells `kubectl` to remove the specified annotation from the Kubernetes resource.

---
## 18. Annotations With Deployments

Annotations can be applied to **Deployments** to store additional metadata or configuration.

Example:

```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: voting
  annotations:
    platform.example.com/owner: "platform-team"
    platform.example.com/documentation: "https://docs.example.com/voting"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: voting
  template:
    metadata:
      labels:
        app: voting
    spec:
      containers:
        - name: voting
          image: nginx:1.27
```

### Deployment Metadata vs Pod Template Metadata

It is important to understand that the **Deployment metadata** and the **Pod template metadata** are separate.

```text 
Deployment
│
├── metadata
│   └── annotations
│
└── spec
    └── template
        └── metadata
            └── labels
```

The annotations under:

```yaml 
metadata:
  annotations:
```

belong to the **Deployment object**.

The metadata under:

```yaml 
spec:
  template:
    metadata:
```

belongs to the **Pod template** and is used when creating Pods.

> **Key Point:** Deployment metadata describes the Deployment itself, while Pod template metadata defines metadata for the Pods created by that Deployment.

---
## 19. Pod Template Annotations

Annotations can also be added directly to the **Pod template** inside a Deployment.

Example:

```yaml 
spec:
  template:
    metadata:
      labels:
        app: voting
      annotations:
        platform.example.com/logging: "enabled"
```

These annotations are applied to the **Pods created by the Deployment**.

### Example Structure

```text 
Deployment
│
└── spec
    └── template
        └── metadata
            ├── labels
            │   └── app=voting
            │
            └── annotations
                └── platform.example.com/logging=enabled
```

This is useful when a controller, monitoring system, logging agent, or other platform component needs information specifically associated with the Pods.

> **Key Point:** Annotations under `spec.template.metadata` become part of the metadata of the Pods created by the Deployment.

---
## 20. Important Deployment Example

A Deployment can have annotations at two different levels:

1. **Deployment metadata**
2. **Pod template metadata**

Example:

```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: voting
  annotations:
    platform.example.com/owner: "platform-team"

spec:
  replicas: 3

  selector:
    matchLabels:
      app: voting

  template:
    metadata:
      labels:
        app: voting
      annotations:
        platform.example.com/logging: "enabled"

    spec:
      containers:
        - name: voting
          image: nginx:1.27
```

### Deployment Annotation

The annotation under:

```yaml 
metadata:
  annotations:
    platform.example.com/owner: "platform-team"
```

belongs to the **Deployment object**.

```text 
Deployment annotation
        │
        ▼
Deployment metadata
```

### Pod Annotation

The annotation under:

```yaml 
spec:
  template:
    metadata:
      annotations:
        platform.example.com/logging: "enabled"
```

belongs to the **Pod template** and is applied to Pods created by the Deployment.

```text 
Pod annotation
        │
        ▼
Pod template metadata
```

### Structure

```text 
Deployment
│
├── metadata
│   └── annotations
│       └── owner=platform-team
│
└── spec
    └── template
        └── metadata
            ├── labels
            │   └── app=voting
            │
            └── annotations
                └── logging=enabled
```

> **Key Point:** Deployment metadata and Pod template metadata are separate. An annotation on the Deployment does not automatically become an annotation on the Pods. To add annotations to Pods, define them under `spec.template.metadata.annotations`.

---
## 21. Labels + Annotations Together

Production Kubernetes resources commonly use both **labels** and **annotations**.

Example:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: voting
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: voting-platform

  annotations:
    platform.example.com/owner: "platform-team"
    platform.example.com/documentation: "https://docs.example.com/voting"
```

### Labels vs Annotations

Think of it this way:

```text 
                 Kubernetes Object
                       │
              ┌────────┴────────┐
              │                 │
           Labels          Annotations
              │                 │
              ▼                 ▼
       Identify / Select   Describe / Configure
              │                 │
              ▼                 ▼
       Services, queries,   Controllers,
       policies, filtering  integrations, metadata
```

### Labels

Labels are primarily used to:

* Identify resources
* Group resources
* Select resources
* Filter resources
* Connect Services to Pods
* Support policies and automation

### Annotations

Annotations are primarily used to:

* Store additional metadata
* Provide controller-specific configuration
* Store ownership information
* Link documentation
* Support integrations and automation

> **Key Point:** Use **labels** when Kubernetes or another tool needs to identify or select a resource. Use **annotations** when additional metadata or configuration needs to be attached to the resource.

---
## 22. Hands-On Lab

This lab demonstrates how to create, view, add, update, remove, and work with Kubernetes annotations.

### Step 1 — Create a Pod

Create a file named `annotations.yaml`:

```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  labels:
    app: voting
    component: frontend
  annotations:
    description: "Kubernetes annotations demonstration"
    owner: "platform-team"
    documentation: "https://example.com/docs"
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply the manifest:

```bash
kubectl apply -f annotations.yaml
```


### Step 2 — Check the Pod

```bash 
kubectl get pod annotations-demo
```

Expected output:

```text 
NAME               READY   STATUS    RESTARTS   AGE
annotations-demo   1/1     Running   0          ...
```

### Step 3 — Display Labels

```bash 
kubectl get pod annotations-demo --show-labels
```

You should see labels such as:

```text 
app=voting
component=frontend
```

### Step 4 — Display Annotations

```bash 
kubectl describe pod annotations-demo
```

You should see:

```text 
Annotations:
  description: Kubernetes annotations demonstration
  owner: platform-team
  documentation: https://example.com/docs
```

### Step 5 — Query Using a Label

Use the `app=voting` label to select the Pod:

```bash 
kubectl get pods -l app=voting
```

The `annotations-demo` Pod should be returned.

```text 
NAME               READY   STATUS    RESTARTS   AGE
annotations-demo   1/1     Running   0          ...
```

### Step 6 — Try to Use an Annotation as a Selector

The following command is intentionally incorrect for selecting this Pod:

```bash 
kubectl get pods -l owner=platform-team
```

The Pod will not be selected because:

```text 
owner=platform-team
```

is an **annotation**, not a label.

```text 
Labels       → Can be selected with -l
Annotations  → Cannot be selected with -l
```

This demonstrates one of the main differences between **labels** and **annotations**.

### Step 7 — Add an Annotation

Add an `environment` annotation:

```bash 
kubectl annotate pod annotations-demo \
  environment="development"
```

Verify:

```bash 
kubectl describe pod annotations-demo
```

### Step 8 — Update the Annotation

Update the existing annotation using `--overwrite`:

```bash 
kubectl annotate pod annotations-demo \
  environment="production" \
  --overwrite
```

Verify:

```bash 
kubectl get pod annotations-demo \
  -o jsonpath='{.metadata.annotations.environment}'
```

Expected output:

```text 
production
```

### Step 9 — Remove the Annotation

Remove the `environment` annotation:

```bash 
kubectl annotate pod annotations-demo environment-
```

Verify:

```bash 
kubectl get pod annotations-demo \
  -o jsonpath='{.metadata.annotations}'
```

### Step 10 — Cleanup

Delete the Pod:

```bash 
kubectl delete pod annotations-demo
```

### Lab Summary

```text 
annotations.yaml
      │
      ▼
annotations-demo Pod
      │
      ├── Labels
      │     ├── app=voting
      │     └── component=frontend
      │
      └── Annotations
            ├── description
            ├── owner
            └── documentation
```

> **Key Point:** Labels are used to **identify and select resources**, while annotations are used to store **additional metadata and configuration**.

---
## 23. Common Real-World Uses

Annotations are commonly encountered in production Kubernetes environments and are frequently used by controllers, cloud integrations, monitoring systems, GitOps tools, and custom operators.

### Ingress Controllers

Ingress controllers commonly use controller-specific annotations.

For example, NGINX Ingress annotations use the following prefix:

```text 
nginx.ingress.kubernetes.io/*
```

These annotations can configure controller-specific behavior such as:

* SSL/TLS redirects
* Authentication
* Rewrite rules
* Timeouts
* Request handling


### Cloud Load Balancers

Cloud-specific controllers can use annotations to configure load balancer behavior.

Common configuration areas include:

```text 
Load balancer type
Health checks
Subnets
Security groups
Internal/external behavior
```

The exact annotations depend on the cloud provider and controller.

### Certificate Management

Certificate controllers may use annotations to trigger or configure certificate-related behavior.

Common use cases include:

* Certificate issuance
* Certificate configuration
* ACME integration
* TLS configuration

The supported annotations depend on the certificate management solution being used.


### Monitoring

Monitoring integrations may use annotations to configure resource discovery or monitoring behavior.

For example:

```text 
Kubernetes Resource
       │
       ▼
Monitoring Annotation
       │
       ▼
Monitoring Controller
       │
       ▼
Monitoring Configuration
```

Modern monitoring solutions may also use dedicated Kubernetes resources such as `ServiceMonitor` and `PodMonitor`.


### GitOps

GitOps controllers can use annotations to store metadata or controller-specific configuration.

Common use cases include:

```text 
Synchronization
Tracking
Resource Metadata
Controller Configuration
```

The exact annotations depend on the GitOps tool being used.


### Custom Operators

Custom Kubernetes operators and controllers frequently use annotations to store:

* Configuration
* Metadata
* Controller-specific settings
* Integration information
* Operational information

```text 
Kubernetes Resource
        │
        └── Annotations
              │
              ├── Controller Configuration
              ├── Metadata
              └── Integration Settings
```

> **Key Point:** Kubernetes provides the annotation mechanism, but the meaning and behavior of an annotation are usually defined by the specific controller, operator, cloud integration, or platform consuming it.

---
## 24. Important Caution

Do not blindly add annotations copied from the internet.

Annotations are often **controller-specific**, meaning their behavior depends on the controller, operator, or platform that consumes them.

For example:

```text 
nginx.ingress.kubernetes.io/*
```

belongs to the **NGINX Ingress ecosystem**.

Another Ingress controller or Kubernetes component may completely ignore these annotations.

### How Annotations Work

```text 
Annotation
    │
    ▼
Which controller understands it?
    │
    ├── NGINX Ingress
    ├── Cloud Controller
    ├── cert-manager
    ├── GitOps Controller
    └── Custom Operator
```

The controller that understands the annotation determines what behavior it produces.

### Best Practice

Before using an annotation:

1. Identify which controller or component consumes it.
2. Check the official documentation for that controller.
3. Verify that the annotation is supported by the installed version.
4. Confirm the expected behavior in a non-production environment.
5. Avoid copying annotations without understanding their purpose.

> **Key Point:** An annotation is only meaningful if the relevant controller or component recognizes and processes it. Always check the documentation of the controller that consumes the annotation.

---
## 25. Labels vs Annotations

> **Labels** are key-value metadata used to identify and group Kubernetes resources and are queried using selectors. **Annotations** are also key-value metadata, but they are intended for additional information or controller-specific configuration and are not used by Kubernetes selectors.

### Example

```yaml 
labels:
  app: voting
  environment: production

annotations:
  platform.example.com/owner: "platform-team"
  platform.example.com/documentation: "https://docs.example.com"
```

### Simple Rule

```text 
Labels
   │
   └── Selection and Identification


Annotations
   │
   └── Metadata and Configuration
```

| Feature                           | Labels                       | Annotations                |
| --------------------------------- | ---------------------------- | -------------------------- |
| Identify resources                | Yes                          | No                         |
| Group resources                   | Yes                          | No                         |
| Used by selectors                 | Yes                          | No                         |
| Store metadata                    | Limited                      | Yes                        |
| Controller-specific configuration | Limited                      | Yes                        |
| Typical purpose                   | Selection and identification | Metadata and configuration |

> **Key Point:** Use **labels** when resources need to be identified, grouped, or selected. Use **annotations** when additional metadata or controller-specific configuration needs to be attached to a resource.

---
## 26. Key Takeaways

Remember these four points:

```text
1. Labels identify resources.

2. Selectors query resources using labels.

3. Annotations store additional metadata or
   controller-specific configuration.

4. Annotations cannot be used as Service selectors.
```

### Simplest Mental Model

```text
LABEL
  ↓
"Which resources are these?"


ANNOTATION
  ↓
"What additional information or configuration
 should a controller or human know about them?"
```

### Quick Reference

| Concept               | Purpose                                    |
| --------------------- | ------------------------------------------ |
| **Labels**            | Identify and group resources               |
| **Selectors**         | Find resources using labels                |
| **Annotations**       | Store additional metadata or configuration |
| **Service Selectors** | Select Pods using labels                   |

> **Key Point:** **Labels identify and select. Annotations describe and configure.**

