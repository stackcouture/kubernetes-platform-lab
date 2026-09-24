# Kubernetes Pod Security Context

## 1. Overview

A Kubernetes **Security Context** defines security-related settings for Pods and containers.

Think of the model like this:

```text
Kubernetes Pod
│
├── Pod Security Context
│   │
│   ├── runAsUser
│   ├── runAsGroup
│   ├── runAsNonRoot
│   └── fsGroup
│
└── Container Security Context
    │
    ├── allowPrivilegeEscalation
    ├── readOnlyRootFilesystem
    ├── capabilities
    └── seccompProfile
```

There are two important layers:

* **Pod-level security context** → defines defaults and identity/filesystem-related settings for containers in the Pod.
* **Container-level security context** → defines restrictions specifically for an individual container.

---
## 2. `runAsUser`

`runAsUser` determines the **Linux UID** under which the container's main process runs.

Example:

```yaml
securityContext:
  runAsUser: 1000
```

Instead of running as:

```text
root
UID 0
```

the process runs as:

```text
UID 1000
```

### Linux Identity Model

```text
Linux
│
├── UID 0
│   └── root
│
├── UID 1000
│   └── application user
│
└── UID 1001
    └── another application user
```

Kubernetes passes this identity to the container runtime.

---
### 2.1 Debug `runAsUser`

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: user-demo

spec:
  securityContext:
    runAsUser: 1000

  containers:
    - name: app
      image: nginx:1.27-alpine
```

Apply:

```bash
kubectl apply -f user-demo.yaml
```

Check the running identity:

```bash
kubectl exec -it user-demo -- id
```

You may see:

```text
uid=1000 gid=0(root) groups=0(root)
```

Notice:

> **UID and GID are different concepts.**

You changed:

```text
UID = 1000
```

but did not change the primary group.

That is where `runAsGroup` becomes important.

---
## 3. `runAsGroup`

`runAsGroup` defines the primary Linux group for the container process.

Example:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 3000
```

Now:

```text
UID = 1000
GID = 3000
```

Debug:

```bash
kubectl exec user-demo -- id
```

You should see something similar to:

```text
uid=1000 gid=3000 groups=3000
```

Conceptually:

```text
Process
│
├── User ID
│   └── 1000
│
└── Primary Group
    └── 3000
```

This matters when applications access files based on Linux ownership and permissions.

---
## 4. `runAsNonRoot`

`runAsNonRoot` is different from `runAsUser`.

Example:

```yaml
securityContext:
  runAsNonRoot: true
```

It tells Kubernetes:

> The container must not run as UID 0.

If the image attempts to run as root, Kubernetes can reject the container startup.

### Why is this useful?

Suppose an attacker exploits a vulnerability in your application.

If the application is running as:

```text
UID 0
root
```

the attacker starts from a highly privileged identity.

If the application runs as:

```text
UID 1000
non-root
```

the attacker's initial privileges are substantially reduced.

This is **defense in depth**, not a complete security boundary.

---
## 5. `runAsUser` vs `runAsNonRoot`

This distinction is frequently asked in interviews.

### `runAsUser`

Specifies **which UID** the process should run as.

```yaml
runAsUser: 1000
```

Means:

```text
Run as UID 1000
```

### `runAsNonRoot`

Specifies a **security requirement**.

```yaml
runAsNonRoot: true
```

Means:

```text
Do not run as UID 0
```

You can combine them:

```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
```

This explicitly defines both the intended UID and the non-root requirement.

---
## 6. `fsGroup`

`fsGroup` is frequently misunderstood.

Consider:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000
```

Now you have:

```text
Process
│
├── UID = 1000
├── GID = 3000
│
└── Filesystem group = 2000
```

`fsGroup` is primarily relevant to **volume ownership and permissions** for volumes that support the behavior.

---
### 6.1 `fsGroup` Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fs-demo

spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000

  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          id
          touch /data/test.txt
          ls -ln /data
          sleep 3600

      volumeMounts:
        - name: data
          mountPath: /data

  volumes:
    - name: data
      emptyDir: {}
```

Debug:

```bash
kubectl exec fs-demo -- id
```

Then:

```bash
kubectl exec fs-demo -- ls -ln /data
```

Inspect the ownership and permissions.

### Mental Model

```text
runAsUser
    ↓
Who is the process?

runAsGroup
    ↓
What is the process's primary group?

fsGroup
    ↓
What filesystem group should supported mounted volumes use?
```

---
## 7. `allowPrivilegeEscalation: false`

`allowPrivilegeEscalation` is a **container-level** security setting.

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

It prevents a process from gaining more privileges than its parent process.

At the Linux level, this relates to privilege transitions such as **setuid/setgid binaries** and related mechanisms.

### Security Model

Without the control:

```text
Application
    ↓
Vulnerable process
    ↓
Privilege escalation attempt
    ↓
Root
```

With the control:

```text
Application
    ↓
Privilege escalation attempt
    ↓
BLOCKED
```

This does not mean the container can never become root under every imaginable circumstance. It is one security control within a broader defense-in-depth strategy.

---
## 8. Debug `allowPrivilegeEscalation`

Example:

```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```

Inspect the Pod:

```bash
kubectl get pod secure-pod -o yaml
```

Look for:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

You can also inspect the running process:

```bash
kubectl exec secure-pod -- cat /proc/1/status
```

Look for:

```text
NoNewPrivs:
```

When `allowPrivilegeEscalation: false` is successfully applied, Linux's `no_new_privs` mechanism is used, and the state can be inspected through `/proc`.

This is a useful senior-level debugging technique.

---
## 9. `readOnlyRootFilesystem`

This is an important container-hardening control.

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

It makes the container's root filesystem read-only.

Conceptually:

```text
/
├── bin
├── etc
├── usr
├── app
└── ...
```

The container cannot normally modify these locations.

---

### 9.1 Why Use It?

Suppose an attacker gets code execution.

Without a read-only root filesystem:

```text
Attacker
   ↓
Write malicious file
   ↓
/tmp/backdoor
/application/malware
/etc/...
```

With:

```yaml
readOnlyRootFilesystem: true
```

many filesystem modification attempts fail.

---
## 10. The Classic `readOnlyRootFilesystem` Problem

You enable:

```yaml
readOnlyRootFilesystem: true
```

and suddenly your application crashes.

You may see:

```text
Read-only file system
```

Why?

The application needs to write somewhere.

Common locations include:

```text
/var/cache
/var/log
/tmp
/run
```

The solution is **not automatically to disable the security control**.

Instead:

1. Identify the directories that must be writable.
2. Mount writable volumes only at those locations.

Example:

```yaml
securityContext:
  readOnlyRootFilesystem: true

volumeMounts:
  - name: tmp
    mountPath: /tmp

volumes:
  - name: tmp
    emptyDir: {}
```

Architecture:

```text
Container filesystem
│
├── /usr       READ ONLY
├── /etc       READ ONLY
├── /app       READ ONLY
│
└── /tmp       WRITABLE
              │
              └── emptyDir
```

This provides a much tighter security boundary.

---
## 11. Debugging `readOnlyRootFilesystem`

Suppose the application is failing.

First check logs:

```bash
kubectl logs secure-pod
```

Look for:

```text
Read-only file system
```

Inspect mounts:

```bash
kubectl exec secure-pod -- mount
```

or:

```bash
kubectl exec secure-pod -- cat /proc/mounts
```

Test the filesystem:

```bash
kubectl exec -it secure-pod -- sh
```

Then:

```bash
touch /test.txt
```

You should get something similar to:

```text
touch: /test.txt: Read-only file system
```

But:

```bash
touch /tmp/test.txt
```

should work if `/tmp` has been mounted as a writable `emptyDir`.

---
## 12. Linux Capabilities

Linux traditionally provided the `root` user with extensive privileges.

Modern Linux divides many privileges into **capabilities**.

Examples include:

```text
CAP_NET_ADMIN
CAP_NET_RAW
CAP_SYS_ADMIN
CAP_CHOWN
CAP_SETUID
CAP_SETGID
```

Instead of giving an application broad root privileges, you can grant only the specific capabilities it requires.

Conceptually:

```text
Root
 │
 ├── Capability A
 ├── Capability B
 ├── Capability C
 └── ...
```

This supports the **principle of least privilege**.

---
## 13. `capabilities.drop: ALL`

Example:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

This removes all Linux capabilities from the container.

Instead of:

```text
Container
   ↓
Many Linux capabilities
```

you aim for:

```text
Container
   ↓
No unnecessary capabilities
```

This follows the principle of least privilege.

---
## 14. Add Only What You Need

Suppose an application genuinely requires:

```text
CAP_NET_BIND_SERVICE
```

You can configure:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

The resulting model becomes:

```text
Capabilities
│
├── CAP_NET_ADMIN          ❌
├── CAP_SYS_ADMIN          ❌
├── CAP_SYS_PTRACE         ❌
├── CAP_NET_RAW            ❌
└── CAP_NET_BIND_SERVICE   ✅
```

This is much better than blindly granting:

```yaml
privileged: true
```

Only add a capability when the workload genuinely requires it.

---
## 15. Debug Linux Capabilities

From inside the container:

```bash
kubectl exec secure-pod -- cat /proc/1/status
```

Look for:

```text
CapInh:
CapPrm:
CapEff:
CapBnd:
```

You can decode capability masks on Linux using tools such as:

```bash
capsh --decode=<hex-value>
```

if `libcap` / `capsh` is available.

### Production Debugging Principle

```text
Application fails
       ↓
Determine required Linux privilege
       ↓
Identify required capability
       ↓
Grant only that capability
```

Not:

```text
Application fails
       ↓
privileged: true
```

The second approach is a poor security practice.

---
## 16. `seccompProfile`

Now we move from Linux privileges to **system calls**.

Applications communicate with the Linux kernel through system calls.

```text
Application
     │
     │ system call
     ▼
Linux Kernel
```

Examples include:

```text
open
read
write
socket
clone
execve
mount
ptrace
```

A compromised application could potentially attempt dangerous system calls.

**seccomp = Secure Computing Mode**

Seccomp allows system calls to be restricted.

Conceptually:

```text
Application
     │
     ▼
seccomp filter
     │
     ├── Allowed syscall ──────► Kernel
     │
     └── Blocked syscall ──────► DENIED
```

---
## 17. `RuntimeDefault`

Kubernetes can use the container runtime's default seccomp profile:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

This is a practical production baseline.

Conceptually:

```text
Application
     │
     ▼
RuntimeDefault seccomp profile
     │
     ├── Allowed system calls → Kernel
     │
     └── Blocked system calls → Denied
```

---
## 18. Debug seccomp

Inspect the Pod:

```bash
kubectl get pod secure-pod -o yaml
```

Look for:

```yaml
seccompProfile:
  type: RuntimeDefault
```

You can inspect the process:

```bash
kubectl exec secure-pod -- cat /proc/1/status
```

Depending on the kernel and container environment, look for:

```text
Seccomp:
Seccomp_filters:
```

For example:

```text
Seccomp: 2
```

generally indicates that seccomp filtering is active for the process.

---
## 19. Putting Everything Together

A hardened application Pod can look like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod

spec:

  securityContext:

    # Run as non-root
    runAsNonRoot: true

    # Linux UID
    runAsUser: 1000

    # Linux primary group
    runAsGroup: 3000

    # Volume filesystem group
    fsGroup: 2000

    # Default seccomp profile
    seccompProfile:
      type: RuntimeDefault

  containers:

    - name: app
      image: nginx:1.27-alpine

      securityContext:

        # Prevent privilege escalation
        allowPrivilegeEscalation: false

        # Make root filesystem read-only
        readOnlyRootFilesystem: true

        # Remove unnecessary capabilities
        capabilities:
          drop:
            - ALL

      volumeMounts:

        - name: tmp
          mountPath: /tmp

        - name: nginx-cache
          mountPath: /var/cache/nginx

        - name: nginx-run
          mountPath: /var/run

  volumes:

    - name: tmp
      emptyDir: {}

    - name: nginx-cache
      emptyDir: {}

    - name: nginx-run
      emptyDir: {}
```

---
## 20. Security Architecture

The resulting security model is:

```text
                     Kubernetes Pod
                           │
              ┌────────────┴────────────┐
              │                         │
       Pod Security              Container Security
              │                         │
       ┌──────┼───────┐        ┌────────┼──────────┐
       │      │       │        │        │          │
      UID    GID    fsGroup   No PE   ReadOnly   Capabilities
       │      │       │        │        │          │
      1000   3000    2000     false    true       DROP ALL
       │
       └─────────────────────────────┐
                                     │
                          seccomp RuntimeDefault
```

Each control addresses a different aspect of container security.

---
## 21. Systematic Security Debugging

Security hardening should not be debugged by randomly removing controls.

Imagine your application suddenly stops working after hardening the Pod.

Use this workflow:

```text
Pod fails
   │
   ▼
kubectl get pod
   │
   ▼
kubectl describe pod
   │
   ▼
kubectl logs
   │
   ▼
Identify failure category
   │
   ├── Permission denied
   │       ↓
   │   UID / GID / fsGroup
   │
   ├── Read-only filesystem
   │       ↓
   │   readOnlyRootFilesystem
   │
   ├── Operation not permitted
   │       ↓
   │   capabilities / privilege
   │
   ├── Container won't start
   │       ↓
   │   runAsNonRoot / image USER
   │
   └── System call denied
           ↓
         seccomp
```

---
## 22. Debug Scenario — Permission Denied

Suppose the application reports:

```text
Permission denied: /data/app.db
```

Do not immediately change:

```yaml
runAsUser: 0
```

That is a poor fix.

First determine the process identity:

```bash
kubectl exec pod -- id
```

Then inspect file ownership:

```bash
kubectl exec pod -- ls -ln /data
```

Check:

```text
UID
GID
ownership
permissions
```

Then inspect the Pod:

```bash
kubectl get pod pod -o yaml
```

Look for:

```yaml
runAsUser:
runAsGroup:
fsGroup:
```

### Debugging Chain

```text
Permission denied
       ↓
Who am I?
       ↓
id
       ↓
Who owns the file?
       ↓
ls -ln
       ↓
What group am I using?
       ↓
fsGroup / runAsGroup
```

---
## 23. Debug Scenario — Read-Only Filesystem

Suppose you receive:

```text
open /tmp/application.log: read-only file system
```

Check:

```bash
kubectl get pod pod -o yaml
```

You discover:

```yaml
readOnlyRootFilesystem: true
```

Determine:

```text
Does the application actually need /tmp?
```

If yes, mount a writable volume:

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp

volumes:
  - name: tmp
    emptyDir: {}
```

Do not disable the security control simply because the application was not designed for it.

---
## 24. Debug Scenario — `Operation not permitted`

Suppose the application reports:

```text
iptables: Operation not permitted
```

or:

```text
mount: Operation not permitted
```

Investigate capabilities:

```bash
kubectl exec pod -- cat /proc/1/status
```

Look at:

```text
CapEff
CapBnd
```

If you configured:

```yaml
capabilities:
  drop:
    - ALL
```

the application may legitimately lack the capability it requires.

Determine which capability is actually required.

For example:

```yaml
capabilities:
  drop:
    - ALL
  add:
    - NET_ADMIN
```

Only do this if the workload genuinely requires that capability.

---
## 25. Debug Scenario — Container Won't Start

Suppose:

```yaml
runAsNonRoot: true
```

but the image expects to run as root.

You might see an error similar to:

```text
container has runAsNonRoot and image will run as root
```

Investigate the image configuration:

```bash
docker image inspect <image>
```

Check the image's configured user.

Also inspect the Pod:

```bash
kubectl get pod pod -o yaml
```

Check:

```yaml
securityContext:
  runAsNonRoot: true
```

The proper fix is usually to make the image/application genuinely support a non-root user.

Do not simply change:

```yaml
runAsNonRoot: false
```

to make the error disappear.

---
## 26. Senior-Level Security Model

Mentally map the controls like this:

```text
                     Application
                          │
                          ▼
                   Non-root identity
                   runAsUser / UID
                          │
                          ▼
                    Group permissions
                 runAsGroup / fsGroup
                          │
                          ▼
                    Privilege escalation
                  allowPrivilegeEscalation
                          │
                          ▼
                    Linux capabilities
                       drop ALL
                          │
                          ▼
                       Filesystem
                 readOnlyRootFilesystem
                          │
                          ▼
                      System calls
                         seccomp
                          │
                          ▼
                        Kernel
```

Each layer addresses a different attack surface.

---
## 27. Runtime Security vs Admission Security

Do not confuse **runtime security controls** with **admission controls**.

Conceptually:

```text
                     Kubernetes Security
                            │
              ┌─────────────┴─────────────┐
              │                           │
       Admission / Policy           Runtime Security
              │                           │
        Pod Security                SecurityContext
        Kyverno                      runAsNonRoot
        Gatekeeper                   capabilities
        ValidatingAdmission          seccomp
                                    readOnlyFS
```

### Admission / Policy Controls

These determine whether a workload should be allowed into the cluster.

Examples:

```text
Pod Security
Kyverno
Gatekeeper
ValidatingAdmission
```

### Runtime Security Controls

These control how the container operates after it is admitted.

Examples:

```text
SecurityContext
runAsNonRoot
capabilities
seccomp
readOnlyRootFilesystem
```

---
## 28. Recommended Baseline

For production-oriented Kubernetes workloads, a strong baseline is:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

Where the application supports it, also use:

```yaml
readOnlyRootFilesystem: true
```

Additional controls such as `runAsUser`, `runAsGroup`, and `fsGroup` should be selected according to the application's actual user, group, and volume-permission requirements.

---
## 29. Quick Reference

| Security Control           | Purpose                                                  | Typical Scope   |
| -------------------------- | -------------------------------------------------------- | --------------- |
| `runAsUser`                | Defines Linux UID                                        | Pod / Container |
| `runAsGroup`               | Defines primary Linux GID                                | Pod / Container |
| `runAsNonRoot`             | Prevents UID 0 execution                                 | Pod / Container |
| `fsGroup`                  | Controls filesystem group behavior for supported volumes | Pod             |
| `allowPrivilegeEscalation` | Prevents privilege escalation                            | Container       |
| `readOnlyRootFilesystem`   | Makes root filesystem read-only                          | Container       |
| `capabilities.drop`        | Removes Linux capabilities                               | Container       |
| `capabilities.add`         | Adds only required capabilities                          | Container       |
| `seccompProfile`           | Restricts Linux system calls                             | Pod / Container |

---
## 30. Key Takeaways

Remember the security model:

```text
runAsUser
    ↓
Who is the process?

runAsGroup
    ↓
What is the process's primary group?

runAsNonRoot
    ↓
Must the process avoid UID 0?

fsGroup
    ↓
What filesystem group should supported
mounted volumes use?

allowPrivilegeEscalation
    ↓
Can the process gain additional privileges?

readOnlyRootFilesystem
    ↓
Can the container modify its root filesystem?

capabilities
    ↓
What Linux privileges does the process have?

seccomp
    ↓
What system calls can the process make?
```

The overall defense-in-depth model is:

```text
                    Container
                       │
                       ▼
                Non-root Identity
                       │
                       ▼
                 Group Controls
                       │
                       ▼
              No Privilege Escalation
                       │
                       ▼
              Drop Linux Capabilities
                       │
                       ▼
             Read-only Root Filesystem
                       │
                       ▼
                Seccomp Filtering
                       │
                       ▼
                     Kernel
```

The goal is **not** to make the container "secure" with one setting. The goal is to progressively reduce the privileges and attack surface available to the workload.
