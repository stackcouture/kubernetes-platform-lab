# Kubernetes QoS Classes

## 1. Overview

**QoS = Quality of Service**

Kubernetes assigns a **Quality of Service (QoS) class** to every Pod based primarily on its CPU and memory `requests` and `limits`.

QoS classification is important when a node experiences **resource pressure**, especially **memory pressure**.

Kubernetes has three QoS classes:

```text
1. Guaranteed
2. Burstable
3. BestEffort
```

The basic relationship is:

```text
CPU / Memory Requests & Limits
              │
              ▼
        QoS Classification
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
 Guaranteed Burstable BestEffort
```

---
## 2. Important Concept

QoS does **not** mean that Kubernetes guarantees a Pod a fixed amount of CPU at all times.

For example, a Pod with:

```yaml
requests:
  cpu: "500m"

limits:
  cpu: "500m"
```

is classified as `Guaranteed`, but this does not mean the Pod continuously receives exactly `500m` of CPU.

Instead:

* **Requests** influence scheduling and resource accounting.
* **Limits** constrain resource consumption.
* **QoS class** influences how Kubernetes treats Pods during resource pressure.

---
## 3. QoS Classes

| QoS Class  | Resource Configuration                                                      | General Behavior           |
| ---------- | --------------------------------------------------------------------------- | -------------------------- |
| Guaranteed | CPU & memory requests equal limits for every container                      | Highest QoS classification |
| Burstable  | Some resource requests/limits exist, but Guaranteed requirements aren't met | Intermediate               |
| BestEffort | No CPU or memory requests/limits                                            | Lowest QoS classification  |

---

## 4. Guaranteed QoS

A Pod receives the **Guaranteed** QoS class when the required CPU and memory configuration is satisfied for every container.

For every container:

```text
CPU request = CPU limit
Memory request = Memory limit
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

Here:

```text
CPU:
  request = 500m
  limit   = 500m

Memory:
  request = 512Mi
  limit   = 512Mi
```

Therefore:

```text
QoS = Guaranteed
```

---
## 5. Checking QoS Class

Use:

```bash
kubectl get pod guaranteed-pod \
  -o jsonpath='{.status.qosClass}'
```

Output:

```text
Guaranteed
```

You can also use:

```bash
kubectl get pod guaranteed-pod \
  -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass"
```

---
## 6. Guaranteed QoS and Resource Usage

Suppose a node has:

```text
CPU:     4 CPU
Memory:  8 GiB
```

The Pod has:

```text
CPU:
  request = 500m
  limit   = 500m

Memory:
  request = 512Mi
  limit   = 512Mi
```

The scheduler considers the resource request when determining whether the Pod can fit on a node.

For CPU:

```text
request = 500m
limit   = 500m
```

The container's CPU usage is constrained by the CPU limit.

For memory:

```text
request = 512Mi
limit   = 512Mi
```

If the container attempts to exceed its memory limit, it can be terminated with an `OOMKilled` result.

---
## 7. Burstable QoS

A Pod receives **Burstable** QoS when it has some CPU or memory requests/limits but does not satisfy the requirements for Guaranteed QoS.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "1"
          memory: "512Mi"
```

Here:

```text
CPU:
  request = 250m
  limit   = 1 CPU

Memory:
  request = 256Mi
  limit   = 512Mi
```

Because:

```text
request != limit
```

the Pod is not Guaranteed.

Therefore:

```text
QoS = Burstable
```

---
## 8. Burstable Workload Behavior

Burstable workloads can use resources above their requests when resources are available, subject to their configured limits.

Example:

```text
CPU request: 250m
CPU limit:   1 CPU
```

The application might normally use:

```text
200m
```

and temporarily increase to:

```text
700m
```

provided the node and container constraints allow it.

The maximum configured CPU usage is:

```text
1 CPU
```

This model is useful for applications whose resource consumption changes over time.

---
## 9. BestEffort QoS

A Pod receives **BestEffort** QoS when none of its containers specify CPU or memory requests or limits.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
```

There is no:

```yaml
resources:
```

configuration.

Therefore:

```text
QoS = BestEffort
```

Check:

```bash
kubectl get pod besteffort-pod \
  -o jsonpath='{.status.qosClass}'
```

Output:

```text
BestEffort
```

---
## 10. BestEffort Resource Behavior

A BestEffort Pod does not have explicitly configured CPU or memory requests/limits.

Therefore, it does not reserve resources through explicit resource requests for scheduling.

Example:

```text
Pod
│
├── CPU request:    Not specified
├── CPU limit:      Not specified
├── Memory request: Not specified
└── Memory limit:   Not specified
```

This makes BestEffort suitable mainly for workloads where explicit resource guarantees are not required.

For production workloads, leaving resources completely unspecified can make capacity planning and resource isolation harder.

---
## 11. QoS Classification Flow

The classification can be visualized as:

```text
                Pod
                 │
                 ▼
      Check CPU/Memory resources
                 │
       ┌─────────┴─────────┐
       │                   │
       ▼                   ▼
All containers       No CPU/memory
have matching        requests/limits
requests & limits         │
       │                  ▼
       ▼             BestEffort
  Guaranteed
       │
       │
       └────── otherwise ──────► Burstable
```

---
## 12. Guaranteed vs Burstable vs BestEffort

### Guaranteed

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Result:

```text
Guaranteed
```

---
### Burstable

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

Result:

```text
Burstable
```

---
### BestEffort

No resource configuration:

```yaml
containers:
  - name: app
    image: nginx:1.27
```

Result:

```text
BestEffort
```

---
## 13. Multi-Container Pod and QoS

QoS classification considers **all containers in the Pod**.

For example:

```yaml
containers:

  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"

  - name: sidecar
    image: busybox:1.36
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
```

Both containers have:

```text
CPU request = CPU limit
Memory request = Memory limit
```

Therefore:

```text
QoS = Guaranteed
```

If the sidecar instead has:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

then:

```text
QoS = Burstable
```

because the entire Pod no longer satisfies the Guaranteed requirements.

---
## 14. Memory Pressure and Eviction

QoS becomes particularly important when a node experiences resource pressure.

For example:

```text
Node
│
├── Pod A
├── Pod B
├── Pod C
└── Pod D
       │
       ▼
Memory pressure
```

Kubernetes may need to reclaim resources.

The kubelet uses eviction logic that considers factors including:

* Pod QoS class
* Whether a Pod is exceeding its requests
* Pod priority
* Resource pressure
* Other eviction-related conditions

A simplified mental model is:

```text
Resource Pressure
       │
       ▼
Eviction / Reclamation Decisions
       │
       ├── BestEffort workloads
       ├── Burstable workloads
       └── Guaranteed workloads
```

**Important:** Do not interpret QoS as a simple absolute eviction order in every situation. Actual eviction decisions depend on the resource under pressure, Pod resource usage relative to requests, Pod priority, and kubelet eviction behavior.

---
## 15. QoS Does Not Replace Priority

QoS and Pod Priority are different concepts.

### QoS

Determined primarily from:

```text
CPU requests
CPU limits
Memory requests
Memory limits
```

### Priority

Determined using:

```text
PriorityClass
```

Conceptually:

```text
Pod
│
├── QoS Class
│   ├── Guaranteed
│   ├── Burstable
│   └── BestEffort
│
└── Priority
    └── PriorityClass
```

Do not confuse the two.

---
## 16. QoS and CPU

CPU limits are enforced differently from memory limits.

If a container reaches its CPU limit, it can be **throttled**.

Example:

```text
CPU limit = 500m
```

If the application attempts to use significantly more CPU than the configured limit, the container may be throttled rather than killed.

Memory behaves differently.

If a container exceeds its memory limit, it can be terminated by the kernel's OOM mechanism and Kubernetes can report:

```text
OOMKilled
```

Therefore:

```text
CPU limit
    ↓
Throttling

Memory limit
    ↓
Possible OOM kill
```

---
## 17. QoS and Scheduling

Resource requests are important to Kubernetes scheduling.

For example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The scheduler uses these requests when determining whether a Pod can be placed on a node.

Conceptually:

```text
Pod Request
     │
     ▼
Scheduler
     │
     ▼
Node Capacity
     │
     ├── Enough resources → Schedule
     │
     └── Not enough → Consider another node
```

QoS itself is not a separate scheduling mechanism.

The resource requests and limits used to determine QoS also have their own scheduling and enforcement implications.

---
## 18. Practical Resource Strategy

A typical production application might use:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

This results in:

```text
QoS = Burstable
```

This allows:

* Predictable baseline resource requests
* Controlled maximum resource consumption
* Ability to burst when resources are available

However, the correct values should come from actual workload measurements and capacity planning rather than arbitrary numbers.

---
## 19. Hands-On Lab

Create three Pods to observe all QoS classes.

### Guaranteed Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

---
### Burstable Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "1"
          memory: "512Mi"
```

---
### BestEffort Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
```

---
## 20. Verify QoS Classes

Check all three:

```bash
kubectl get pods
```

Then:

```bash
kubectl get pod guaranteed-pod \
  -o jsonpath='{.status.qosClass}{"\n"}'
```

Expected:

```text
Guaranteed
```

Check Burstable:

```bash
kubectl get pod burstable-pod \
  -o jsonpath='{.status.qosClass}{"\n"}'
```

Expected:

```text
Burstable
```

Check BestEffort:

```bash
kubectl get pod besteffort-pod \
  -o jsonpath='{.status.qosClass}{"\n"}'
```

Expected:

```text
BestEffort
```

---
## 21. Display QoS for Multiple Pods

A convenient command:

```bash
kubectl get pods \
  -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass"
```

Example:

```text
NAME              QOS
guaranteed-pod    Guaranteed
burstable-pod     Burstable
besteffort-pod    BestEffort
```

---
## 22. Verify Using `describe`

You can also run:

```bash
kubectl describe pod guaranteed-pod
```

Look for the Pod's configuration and resource information.

For the authoritative QoS classification, use:

```bash
kubectl get pod guaranteed-pod \
  -o jsonpath='{.status.qosClass}'
```

---
## 23. Troubleshooting QoS

If you expected:

```text
Guaranteed
```

but Kubernetes reports:

```text
Burstable
```

check **every container**.

Example:

```text
Container A
CPU request = 500m
CPU limit   = 500m
Memory request = 512Mi
Memory limit   = 512Mi

Container B
CPU request = 100m
CPU limit   = 500m
```

Container B breaks the Guaranteed condition.

Therefore:

```text
QoS = Burstable
```

The key troubleshooting question is:

> **Does every container satisfy the Guaranteed resource requirements?**

---
## 24. Common Mistakes

### Mistake 1 — Setting only limits

Example:

```yaml
resources:
  limits:
    cpu: "1"
    memory: "512Mi"
```

This does not automatically make the Pod Guaranteed.

Kubernetes may assign default requests based on the resource configuration, but you should not rely on assumptions when designing QoS. Define the intended requests and limits explicitly.

---
### Mistake 2 — Forgetting the sidecar

You configure:

```text
Application → Guaranteed
```

but forget that:

```text
Sidecar → different resource configuration
```

QoS is calculated for the **Pod as a whole**.

---
### Mistake 3 — Thinking Guaranteed means guaranteed performance

This is incorrect.

Guaranteed QoS does not mean:

```text
"this Pod will always receive exactly X CPU"
```

It describes the Pod's resource configuration and its treatment under resource pressure.

---
### Mistake 4 — Confusing QoS with Priority

```text
QoS ≠ PriorityClass
```

They are separate Kubernetes concepts.

---
## 25. Interview Questions

### Q1. What is Kubernetes QoS?

QoS, or Quality of Service, is the classification Kubernetes assigns to Pods based primarily on their CPU and memory resource requests and limits.

---
### Q2. What are the three QoS classes?

```text
Guaranteed
Burstable
BestEffort
```

---
### Q3. How does a Pod become Guaranteed?

Every container must have CPU and memory requests and limits configured such that:

```text
CPU request = CPU limit
Memory request = Memory limit
```

---
### Q4. How does a Pod become BestEffort?

None of its containers have CPU or memory requests or limits.

---
### Q5. What is Burstable?

A Pod is Burstable when it has resource requests/limits but does not satisfy the conditions for Guaranteed.

---
### Q6. Does Guaranteed mean the Pod always gets its CPU?

No.

Resource requests and limits affect scheduling and resource enforcement, while QoS classification influences behavior during resource pressure.

---
### Q7. What happens if a container exceeds its memory limit?

It can be terminated by the kernel's OOM mechanism, and Kubernetes may report the container as:

```text
OOMKilled
```

---
### Q8. What happens when a container reaches its CPU limit?

CPU can be throttled rather than the container being killed simply because it reached its CPU limit.

---
### Q9. Does QoS determine Pod priority?

No.

Pod priority is controlled through `PriorityClass`.

---
### Q10. Why are resource requests important?

Requests influence scheduling and help Kubernetes determine how much capacity a workload needs.

---
## 26. Quick Reference

```text
┌──────────────────────────────────────────────┐
│              Kubernetes Pod                  │
└──────────────────────────────────────────────┘
                     │
                     ▼
          CPU / Memory Configuration
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
      Guaranteed  Burstable  BestEffort
          │          │          │
          │          │          │
     Request =    Some        No CPU/
       Limit      requests/   memory
                  limits      requests/
                              limits
```
### Guaranteed

```text
CPU request = CPU limit
Memory request = Memory limit
for every container
```

### Burstable

```text
Some CPU/memory resources configured
but Guaranteed requirements not met
```

### BestEffort

```text
No CPU/memory requests or limits
```

---
## 27. Key Takeaways

Remember these points for Kubernetes interviews and production troubleshooting:

```text
1. QoS has three classes:
   Guaranteed
   Burstable
   BestEffort

2. QoS is primarily determined by CPU and memory
   requests and limits.

3. Guaranteed requires matching CPU and memory
   requests and limits for every container.

4. Burstable has resource configuration but does
   not satisfy Guaranteed requirements.

5. BestEffort has no CPU or memory requests/limits.

6. Resource requests influence scheduling.

7. Resource limits constrain resource consumption.

8. CPU limit violations can result in throttling.

9. Memory limit violations can result in OOMKilled.

10. QoS is relevant during node resource pressure.

11. QoS does not replace PriorityClass.

12. QoS does not mean guaranteed performance.
```

The most useful mental model is:

```text
                CPU / Memory
                Requests + Limits
                       │
                       ▼
                  QoS Class
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     Guaranteed     Burstable    BestEffort
          │            │            │
          └────────────┼────────────┘
                       ▼
              Resource Pressure
                       │
                       ▼
             Eviction / Reclaim
              Decision Process
```
