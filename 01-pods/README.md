# Kubernetes Pods

## 1. Overview

A **Pod** is the smallest deployable unit in Kubernetes.

A Pod represents one or more containers that are deployed and managed together on the same Kubernetes node.

Containers inside the same Pod share:

* Network namespace
* IP address
* Port space
* Volumes
* Pod lifecycle

A Pod normally contains a single application container, but Kubernetes also supports multi-container Pods when the containers need to work closely together.

---
## 2. Pod Architecture

A basic Pod looks like:

```text
Kubernetes Cluster
        │
        ▼
      Node
        │
        ▼
       Pod
        │
        ├── Container
        │
        └── Volume
```

A multi-container Pod:

```text
Pod
│
├── Application Container
│
├── Sidecar Container
│
└── Shared Volume
```

All containers inside the Pod share the same network namespace.

For example:

```text
Container A ──┐
              ├── Pod IP
Container B ──┘
```

They can communicate using:

```text
localhost
```

---
## 3. Pod Lifecycle

A simplified Pod lifecycle is:

```text
Pending
   │
   ▼
Running
   │
   ├── Completed
   │
   └── Failed
```

Check Pod status:

```bash
kubectl get pods
```

Detailed information:

```bash
kubectl describe pod <pod-name>
```

---
## 4. Pod Manifests in This Lab

This directory contains the following examples:

```text
pods/
│
├── basic-pod.yaml
├── pod-with-resources.yaml
├── pod-with-probes.yaml
├── pod-with-security-context.yaml
├── multi-container-pod.yaml
└── init-container.yaml
```

Each manifest demonstrates a specific Kubernetes Pod concept.

---
## 5. basic-pod.yaml

### Purpose

Demonstrates the simplest Kubernetes Pod configuration.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: basic-pod
  labels:
    app: basic-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f basic-pod.yaml
```

Verify:

```bash
kubectl get pods
```

Detailed information:

```bash
kubectl describe pod basic-pod
```

Check the Pod IP:

```bash
kubectl get pod basic-pod -o wide
```

---
## 6. Pod Lifecycle Commands

View Pods:

```bash
kubectl get pods
```

Watch Pod status:

```bash
kubectl get pods -w
```

Describe a Pod:

```bash
kubectl describe pod basic-pod
```

View logs:

```bash
kubectl logs basic-pod
```

Execute a command:

```bash
kubectl exec -it basic-pod -- /bin/bash
```

Delete:

```bash
kubectl delete pod basic-pod
```

---
## 7. pod-with-resources.yaml

### Purpose

Demonstrates Kubernetes CPU and memory resource management.

Containers should define:

```text
requests
limits
```

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

---
## 8. Requests vs Limits

### Requests

A request represents the amount of CPU or memory Kubernetes uses when scheduling the Pod.

Example:

```yaml
requests:
  cpu: "100m"
  memory: "128Mi"
```

The scheduler uses these values to determine whether the Pod can fit on a node.

---
### Limits

A limit defines the maximum resource amount that the container is allowed to consume.

Example:

```yaml
limits:
  cpu: "500m"
  memory: "512Mi"
```

Simplified model:

```text
             Container
                │
       ┌────────┴────────┐
       │                 │
    Request            Limit
       │                 │
       ▼                 ▼
  Scheduling        Maximum usage
```

---
## 9. CPU Units

Kubernetes CPU is measured in CPU cores.

Examples:

```text
1 CPU  = 1 core
500m   = 0.5 CPU
250m   = 0.25 CPU
100m   = 0.1 CPU
```

Therefore:

```yaml
cpu: "500m"
```

means:

```text
0.5 CPU
```

---
## 10. Memory Units

Common memory units:

```text
Mi = Mebibytes
Gi = Gibibytes
```

Example:

```yaml
memory: "128Mi"
```

means approximately:

```text
128 MiB
```

---
## 11. Why Resources Matter

Without resource requests and limits, workloads can consume unpredictable amounts of node resources.

Production clusters should normally define appropriate resource requests and limits.

This enables:

* Better scheduling
* Capacity planning
* Resource isolation
* ResourceQuota enforcement
* More predictable workloads

---
## 12. pod-with-probes.yaml

### Purpose

Demonstrates Kubernetes health probes.

Kubernetes supports:

```text
Liveness Probe
Readiness Probe
Startup Probe
```

---
## 13. Liveness Probe

A liveness probe determines whether a container is still healthy.

If the liveness probe repeatedly fails, Kubernetes can restart the container.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 10
```

Conceptually:

```text
Application
    │
    ▼
Liveness Probe
    │
    ├── Healthy → Continue running
    │
    └── Failed repeatedly → Restart container
```

---
## 14. Readiness Probe

A readiness probe determines whether a Pod is ready to receive traffic.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
```

Conceptually:

```text
Pod
 │
 ▼
Readiness Probe
 │
 ├── Ready
 │      │
 │      ▼
 │   Service
 │      │
 │      ▼
 │   Traffic
 │
 └── Not Ready
        │
        ▼
   No Service traffic
```

A failed readiness probe does **not** normally restart the container.

---
## 15. Startup Probe

Startup probes are useful for applications that take a long time to start.

Example:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  failureThreshold: 30
  periodSeconds: 10
```

The startup probe gives the application time to initialize before liveness/readiness checks become important.

Conceptually:

```text
Container starts
      │
      ▼
Startup Probe
      │
      ▼
Application initialized
      │
      ├──────────────┐
      ▼              ▼
Liveness          Readiness
Probe             Probe
```

---
## 16. Probe Types

Kubernetes supports different probe mechanisms.

### HTTP

```yaml
httpGet:
  path: /health
  port: 8080
```

### TCP

```yaml
tcpSocket:
  port: 8080
```

### Command

```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

---
## 17. Important Probe Difference

Remember:

```text
Liveness
    ↓
"Should this container be restarted?"

Readiness
    ↓
"Should this Pod receive traffic?"

Startup
    ↓
"Has this application finished starting?"
```

This distinction is extremely important in production troubleshooting.

---
## 18. pod-with-security-context.yaml

### Purpose

Demonstrates Kubernetes security context configuration.

Security contexts can control:

* User/group IDs
* Privilege escalation
* Root execution
* Linux capabilities
* Filesystem permissions
* Security-related container behavior

Example:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
```

---
## 19. runAsNonRoot

Example:

```yaml
securityContext:
  runAsNonRoot: true
```

This tells Kubernetes that the container should not run as the root user.

This is an important production security practice.

---
## 20. runAsUser and runAsGroup

Example:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
```

The container process runs using the specified UID and GID.

---
## 21. readOnlyRootFilesystem

Example:

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

This makes the container's root filesystem read-only.

Applications that need temporary writable storage can use an `emptyDir` volume.

Example:

```yaml
volumes:
  - name: tmp
    emptyDir: {}
```

---
## 22. Capabilities

Linux capabilities can be dropped to reduce container privileges.

Example:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

A more restrictive container might combine:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

This follows the principle of least privilege.

---
## 23. Multi-Level Security Context

Security context can be configured at:

```text
Pod level
    │
    ▼
Pod SecurityContext

Container level
    │
    ▼
Container SecurityContext
```

Example:

```yaml
spec:
  securityContext:
    runAsNonRoot: true

  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
```

Pod-level settings apply to the Pod unless overridden or supplemented at the container level where supported.

---
## 24. multi-container-pod.yaml

### Purpose

Demonstrates a Pod containing multiple containers.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: app
      image: nginx:1.27

    - name: sidecar
      image: busybox:1.36
      command:
        - sh
        - -c
        - "while true; do echo sidecar running; sleep 10; done"
```

Architecture:

```text
Pod
│
├── Application Container
│
└── Sidecar Container
```

---
## 25. Why Use Multiple Containers?

Multi-container Pods are useful when containers need to operate together as a single unit.

Common patterns include:

### Sidecar

```text
Application
     │
     └── Sidecar
```

Example uses:

* Log processing
* Proxy
* Service mesh proxy
* Configuration synchronization

### Ambassador

A helper container acts as a proxy to an external service.

### Adapter

A helper container transforms application output into a format required by another system.

---
## 26. Shared Network

Containers inside the same Pod share the Pod network namespace.

Therefore, containers can communicate using:

```text
localhost
```

Example:

```text
Container A
    │
    │ localhost:8080
    ▼
Container B
```

They share the same Pod IP.

---
## 27. Shared Volumes

Containers in the same Pod can share volumes.

Example:

```yaml
volumes:
  - name: shared-data
    emptyDir: {}
```

Mount it into both containers:

```yaml
volumeMounts:
  - name: shared-data
    mountPath: /shared
```

Architecture:

```text
Application Container
        │
        ▼
   /shared
        ▲
        │
Sidecar Container
```

Both containers can access the shared volume.

---
## 28. init-container.yaml

### Purpose

An **init container** runs before the main application containers start.

Init containers are useful for initialization tasks such as:

* Preparing configuration
* Waiting for dependencies
* Database initialization
* Downloading files
* Setting permissions
* Performing pre-start checks

Architecture:

```text
Pod Created
    │
    ▼
Init Container
    │
    ├── Success
    │      │
    │      ▼
    │  Application Container
    │
    └── Failure
           │
           ▼
       Retry / Pod not ready
```

---
## 29. Init Container Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container-demo
spec:

  initContainers:
    - name: init
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Initializing application..."
          sleep 5
          echo "Initialization complete"

  containers:
    - name: app
      image: nginx:1.27
```

The init container must complete successfully before the application container starts.

---
## 30. Multiple Init Containers

A Pod can have multiple init containers.

Example:

```yaml
initContainers:

  - name: init-config
    image: busybox:1.36

  - name: init-permissions
    image: busybox:1.36

containers:

  - name: application
    image: nginx:1.27
```

They execute sequentially:

```text
init-config
    │
    ▼
init-permissions
    │
    ▼
application
```

If an init container fails, subsequent init containers do not run until the failed one succeeds.

---
## 31. Init Container vs Sidecar

| Feature                 | Init Container | Sidecar                       |
| ----------------------- | -------------- | ----------------------------- |
| Runs before app         | Yes            | Not necessarily               |
| Runs during application | No             | Yes                           |
| Main purpose            | Initialization | Supporting application        |
| Lifecycle               | Completes      | Usually runs with application |
| Example                 | Prepare config | Log collector                 |

Simple rule:

```text
Init Container
    ↓
Prepare

Sidecar
    ↓
Support
```

---
## 32. Pod Networking

Every Pod receives its own IP address.

Example:

```text
Pod
│
├── Container A
├── Container B
│
└── Pod IP: 10.10.1.25
```

Both containers share:

```text
Pod IP
Network namespace
Port space
```

Therefore, two containers in the same Pod cannot normally bind the same port.

---
## 33. Pod Storage

Containers have ephemeral writable storage by default.

For sharing data between containers, use volumes.

Example:

```yaml
volumes:
  - name: shared-data
    emptyDir: {}
```

`emptyDir` exists for the lifetime of the Pod.

```text
Pod Created
     │
     ▼
emptyDir created
     │
     ▼
Containers use volume
     │
     ▼
Pod deleted
     │
     ▼
emptyDir deleted
```

---
## 34. Important Pod Design Principle

A Pod should contain multiple containers only when those containers are **tightly coupled**.

Good example:

```text
Application
     +
Log Sidecar
```

Bad example:

```text
Pod
├── Frontend
├── Backend
├── Database
└── Redis
```

These components normally have different:

* Scaling requirements
* Lifecycle
* Availability requirements
* Resource requirements
* Deployment schedules

They should generally be separate workloads.

---
## 35. Production Pod Checklist

Use this checklist when creating or reviewing Kubernetes workloads.

* [ ] Resource requests
* [ ] Resource limits
* [ ] Liveness probe
* [ ] Readiness probe
* [ ] Startup probe where required
* [ ] SecurityContext
* [ ] Non-root execution
* [ ] Capability restrictions
* [ ] Image version
* [ ] Image security scanning
* [ ] Appropriate labels
* [ ] Appropriate annotations
* [ ] Secrets management
* [ ] ConfigMap configuration
* [ ] NetworkPolicy

---
## 36. Hands-On Lab

### Step 1 — Deploy Basic Pod

```bash
kubectl apply -f basic-pod.yaml
```

Check:

```bash
kubectl get pods
```

### Step 2 — Test Resource Configuration

```bash
kubectl apply -f pod-with-resources.yaml
```

Check:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Requests
Limits
```

### Step 3 — Test Probes

```bash
kubectl apply -f pod-with-probes.yaml
```

Check:

```bash
kubectl describe pod <pod-name>
```

Look at:

```text
Liveness
Readiness
Startup
```

### Step 4 — Test Security Context

```bash
kubectl apply -f pod-with-security-context.yaml
```

Check:

```bash
kubectl exec -it <pod-name> -- id
```

Verify the container user.


### Step 5 — Test Multi-Container Pod

```bash
kubectl apply -f multi-container-pod.yaml
```

List containers:

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.spec.containers[*].name}'
```

View logs for a specific container:

```bash
kubectl logs <pod-name> -c <container-name>
```

### Step 6 — Test Init Container

```bash
kubectl apply -f init-container.yaml
```

Check:

```bash
kubectl describe pod <pod-name>
```

View init container logs:

```bash
kubectl logs <pod-name> -c init
```

---
## 37. Useful Pod Commands

### List Pods

```bash
kubectl get pods
```

### Detailed information

```bash
kubectl describe pod <pod-name>
```

### Pod YAML

```bash
kubectl get pod <pod-name> -o yaml
```

### Pod IP and Node

```bash
kubectl get pods -o wide
```

### Logs

```bash
kubectl logs <pod-name>
```

### Previous container logs

```bash
kubectl logs <pod-name> --previous
```

### Specific container logs

```bash
kubectl logs <pod-name> -c <container-name>
```

### Execute command

```bash
kubectl exec -it <pod-name> -- sh
```

### Watch Pod status

```bash
kubectl get pods -w
```

### Delete Pod

```bash
kubectl delete pod <pod-name>
```

---
## 38. Common Interview Questions

### Q1. What is a Pod?

A Pod is the smallest deployable unit in Kubernetes and represents one or more containers that share network and storage resources.

### Q2. Why does Kubernetes use Pods instead of directly managing containers?

The Pod provides a higher-level abstraction for managing tightly coupled containers that share networking, storage, lifecycle, and scheduling.


### Q3. Can a Pod contain multiple containers?

Yes.

Multiple containers should generally be used when they need to share the same lifecycle, network namespace, or storage.

### Q4. What is a sidecar container?

A sidecar is a supporting container that runs alongside the primary application container in the same Pod.

### Q5. What is an init container?

An init container runs before the application containers and must complete successfully before the application containers start.

### Q6. Difference between readiness and liveness probes?

```text
Readiness → Should the Pod receive traffic?

Liveness  → Should the container be restarted?
```

### Q7. What happens if a readiness probe fails?

The Pod is marked as not ready and should be removed from Service traffic through the normal EndpointSlice mechanism.

The container is not automatically restarted just because readiness failed.

### Q8. What happens if a liveness probe fails?

After the configured failure conditions are met, Kubernetes restarts the affected container.

### Q9. What are resource requests?

Requests tell the Kubernetes scheduler how much CPU and memory a container requires for scheduling purposes.

### Q10. What are resource limits?

Limits define the maximum CPU and memory resources a container is allowed to consume.

### Q11. Why should containers run as non-root?

Running as non-root reduces the impact of a container compromise and follows the principle of least privilege.

### Q12. Do containers in the same Pod have separate IP addresses?

No.

Containers within the same Pod share the Pod's network namespace and IP address.

### Q13. Can two containers in the same Pod use the same port?

Normally no. Because they share the same network namespace, they cannot both bind the same IP/port combination.

### Q14. What happens to a Pod when the node fails?

The Pod on the failed node becomes unavailable. A controller such as a Deployment or StatefulSet can create a replacement Pod on another suitable node.

A standalone Pod itself does not provide replica management.

---
## 39. Pod vs Deployment

A common production mistake is deploying standalone Pods for applications that need high availability.

### Pod

```text
Pod
 │
 └── Application
```

If it disappears, Kubernetes does not automatically maintain a desired replica count.

### Deployment

```text
Deployment
    │
    ▼
ReplicaSet
    │
    ├── Pod
    ├── Pod
    └── Pod
```

For stateless production applications, a Deployment is generally used instead of manually managing individual Pods.

---
## 40. Key Takeaways

The most important concepts from this lab are:

```text
Pod
 │
 ├── Containers
 │
 ├── Networking
 │
 ├── Volumes
 │
 ├── Resource Requests/Limits
 │
 ├── Health Probes
 │
 ├── Security Context
 │
 ├── Sidecars
 │
 └── Init Containers
```

Remember:

```text
Resources
    ↓
How much CPU/memory does the workload need?

Readiness
    ↓
Can it receive traffic?

Liveness
    ↓
Should it be restarted?

Startup
    ↓
Has it finished starting?

SecurityContext
    ↓
How should it run?

Init Container
    ↓
What must happen before startup?

Sidecar
    ↓
What needs to run alongside it?
```

For production Kubernetes, don't stop at **"the Pod is running."** A production-ready workload needs appropriate resource management, health checks, security controls, configuration management, and a controller such as a Deployment or StatefulSet to manage its lifecycle.
