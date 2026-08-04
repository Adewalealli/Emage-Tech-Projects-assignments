# Assignment 06 — Adewale Alli

**GitHub username:** Adewalealli  
**Date completed:** 2026-08-03

## 1. Answers to the 10 questions

### Q1 — spec vs status + defaulting + last-applied

The user, an automation tool, or another authorized controller writes an object's `spec`, which represents desired state. In this experiment, I changed the Deployment's desired state through `kubectl`. Kubernetes controllers and kubelets observe the cluster and report current reality through `status`. The Deployment controller calculated the Deployment status from its ReplicaSet and Pod availability.

When I scaled the Deployment from two to four replicas, `spec.replicas` changed first, while `status.availableReplicas` temporarily remained at two, increased to three, and finally reached four:

```text
4 desired / 2 available
4 desired / 3 available
4 desired / 4 available
```

This demonstrated that status trails spec while Kubernetes reconciles desired state.

I did not explicitly configure `strategy`, `progressDeadlineSeconds`, `revisionHistoryLimit`, `restartPolicy`, or several other fields. Kubernetes added those defaults during API request processing before storing the object in etcd. The Deployment controller also added the `deployment.kubernetes.io/revision` annotation to track rollout revisions.

Client-side `kubectl apply` uses the `kubectl.kubernetes.io/last-applied-configuration` annotation to remember the previously applied declarative configuration. It can compare the previous configuration, the new local configuration, and the current live object when deciding what to add, modify, or remove. `kubectl create` performs a one-time creation, so it does not need that previous applied configuration.

### Q2 — labels vs annotations design rule

Labels are short, structured identifiers that Kubernetes and its controllers can select and act on. Using the luggage-tag analogy, labels are like routing tags used by the system to decide where an object belongs.

Kubernetes uses labels when ReplicaSets identify and count the Pods they manage and when Services select the ready Pods that should receive traffic. Labels are also used by `kubectl -l`, NetworkPolicies, and scheduling rules.

Annotations are supporting metadata attached to an object but are not used by label selectors. Information such as a Git commit SHA, runbook URL, build details, monitoring configuration, or human notes belongs in annotations.

Label values are limited to 63 characters because labels use a constrained identifier format intended for selection and grouping. Annotation values do not have that same 63-character limit because they are intended for larger or less structured supporting information, although the Kubernetes object still has an overall size limit.

### Q3 — relabel experiment + orphaned pod

The ReplicaSet controller continuously observed Pods matching its selector, `app=anatomy`. Before the label change, it saw four matching Pods, matching the desired replica count of four.

When I changed one Pod's label from `app=anatomy` to `app=escaped`, the ReplicaSet controller observed only three matching Pods. It compared the current count of three with the desired count of four, detected a deficit of one, and created a replacement Pod.

The relabeled Pod continued running, but the ReplicaSet released its owner reference because the Pod no longer matched its selector. It became unmanaged, while the replacement Pod had a ReplicaSet owner reference.

A legitimate use is temporarily removing a malfunctioning Pod from controller management so an engineer can inspect its filesystem, processes, networking, or logs without the controller immediately replacing it. The risk is that the Pod is no longer self-healing, may continue consuming resources indefinitely, and may accidentally be excluded from Services, NetworkPolicies, or other label-driven behavior.

### Q4 — the missing controller / operator pattern

The missing half of the operator pattern is a custom controller. The CRD taught the API server how to validate, default, store, and return `Backup` objects, but it did not provide behavior that performs a backup.

For `postgres-nightly`, a Backup controller would:

1. **Watch:** Observe new and changed `Backup` resources.
2. **Diff:** Compare the requested source, schedule, and retention policy with existing cluster resources.
3. **Act:** Create or update the necessary CronJob, Job, credentials, or storage configuration.
4. **Update:** Observe success or failure and write the result into the `Backup` object's status.

Two real operator examples are cert-manager, whose controllers watch resources such as `Certificate`, and Argo CD, whose controller watches `Application` resources.

### Q5 — CRD deletion blast radius

When I deleted the `Backup` CRD, the `postgres-nightly` and `redis-hourly` custom resources were removed, and the API server could no longer list the `Backup` resource type.

Deleting a CRD is dangerous because it removes both the API definition and its stored custom-resource instances. In production, these objects may be the source of truth for certificates, databases, deployments, or other infrastructure.

For example, an Argo CD `Application` describes applications and resources managed by Argo CD. Removing its CRD destroys those application records and can leave workloads unmanaged or interact dangerously with finalizers and controller cleanup behavior.

### Q6 — phase/state table + explanations

| Pod | Pod phase | Container state and reason | Restarts |
|---|---|---|---:|
| `broken-image` | `Pending` | `Waiting` — `ImagePullBackOff` | 0 |
| `one-shot` | `Succeeded` | `Terminated` — completed with exit code 0 | 0 |
| `one-shot-fail` | `Failed` | `Terminated` — error with exit code 1 | 0 |
| `crasher` | `Running` | `Waiting` — `CrashLoopBackOff`; previous exit code 1 | 5 |

`broken-image` remained `Pending` even though the scheduler successfully assigned it to a node because the kubelet could not pull the nonexistent image. The container never started, so there was no running process to restart.

`one-shot-fail` entered the `Failed` phase because it exited with code 1 and had `restartPolicy: Never`. The `crasher` Pod used the default `restartPolicy: Always`, so the kubelet continued trying to restart its failing container. The restart policy explains their different behavior.

`BackOff` means Kubernetes progressively increases the delay between repeated failed operations. In `ImagePullBackOff`, the kubelet delays image-pull attempts. In `CrashLoopBackOff`, the kubelet delays container restart attempts.

### Q7 — readiness vs liveness, who unplugged the pod

The kubelet executed the readiness probe and reported the affected Pod as not ready. The EndpointSlice controller then observed the Pod's readiness condition and removed its IP address from the Service's available backends.

The restart count remained zero because readiness failures control traffic eligibility and do not restart containers. If the liveness probe had repeatedly failed instead, the kubelet would have killed and restarted the container.

A production Pod may be alive but not ready while warming a cache, loading application data, experiencing a long garbage-collection pause, or temporarily unable to reach an essential upstream dependency.

### Q8 — startup probe mechanics vs initialDelaySeconds

While a startup probe has not yet succeeded, the kubelet does not execute the container's liveness or readiness probes. This prevents a slow-starting application from being declared unhealthy and restarted before it finishes booting.

The startup probe allowed up to 120 seconds because `failureThreshold: 12` multiplied by `periodSeconds: 10` equals 120 seconds. The application opened port 8080 after approximately 90 seconds, the startup probe succeeded, and normal liveness checking began. The fixed container reached `Running` with zero restarts.

This is better than setting `initialDelaySeconds: 120` on the liveness probe because a fixed delay disables liveness protection for the entire 120 seconds. If the application normally starts in 20 seconds, a startup probe succeeds around that time and enables liveness checks immediately instead of leaving the application unchecked for another 100 seconds.

### Q9 — init containers vs sleep vs in-app retry

The `ordered-app` Pod remained in the `Pending` phase with `Initialized: False` while the `wait-for-db` init container could not resolve or connect to `fake-db`. Kubernetes did not start the main nginx container until the init container completed successfully.

An init container is better than a fixed `sleep 30` because it waits for the actual dependency condition rather than guessing how long startup will take. Thirty seconds could be unnecessarily long when the database is ready quickly or too short during a slow recovery.

Init containers provide explicit startup ordering, separate logs, and visible initialization status. Retry logic may still belong inside the application when the dependency can disappear after startup, because init containers only control the initial Pod startup sequence.

### Q10 — QoS derivation + compressible vs non-compressible + eviction order

Kubernetes derived each QoS class from the Pods' resource configuration:

- **Guaranteed:** Every container specified CPU and memory requests and limits, and each request equaled its corresponding limit.
- **Burstable:** The Pod specified resource requests or limits but did not satisfy all Guaranteed requirements. Its requests were lower than its limits.
- **BestEffort:** The Pod specified no CPU or memory requests or limits.

CPU is compressible because Linux can restrict how much CPU time a container receives. When a container reaches its CPU limit, its processes are throttled and run more slowly, but they can continue operating.

Memory is non-compressible because the kernel cannot make an existing memory allocation run more slowly. When `oom-victim` exceeded its 100 MiB cgroup memory limit, the kernel killed the process. Kubernetes reported `OOMKilled` with exit code 137.

During node memory pressure, and assuming comparable Pod priority, BestEffort workloads are normally the first eviction candidates because they have no reserved memory. Burstable Pods exceeding their requests are considered before Guaranteed Pods. Guaranteed workloads receive the strongest protection and are generally evicted last.

A production workload should not ship as BestEffort because it has no guaranteed resource reservation, has the weakest protection during resource pressure, can be starved by other workloads, and is the easiest class to evict.

## 2. Files

### `backup-crd.yaml`

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.training.emagetech.io
spec:
  group: training.emagetech.io
  names:
    kind: Backup
    plural: backups
    singular: backup
    shortNames:
      - bk
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - source
                - schedule
              properties:
                source:
                  type: string
                  description: PVC or database to back up
                schedule:
                  type: string
                  description: Cron expression
                retentionDays:
                  type: integer
                  minimum: 1
                  default: 7
      additionalPrinterColumns:
        - name: Source
          type: string
          jsonPath: .spec.source
        - name: Schedule
          type: string
          jsonPath: .spec.schedule
```

### `probed.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probed
spec:
  replicas: 2
  selector:
    matchLabels:
      app: probed
  template:
    metadata:
      labels:
        app: probed
    spec:
      containers:
        - name: web
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          readinessProbe:
            exec:
              command: ["cat", "/tmp/ready"]
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
          lifecycle:
            postStart:
              exec:
                command: ["touch", "/tmp/ready"]
---
apiVersion: v1
kind: Service
metadata:
  name: probed
spec:
  selector:
    app: probed
  ports:
    - port: 80
```

### `slow-start-fixed.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-fixed
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 90 && httpd -f -p 8080 -h /tmp"]
      livenessProbe:
        tcpSocket:
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 3
      startupProbe:
        tcpSocket:
          port: 8080
        failureThreshold: 12
        periodSeconds: 10
```

## 3. Evidence

### Part 1 — status trailing spec

```text
2 desired / 2 available
4 desired / 2 available
4 desired / 2 available
4 desired / 3 available
4 desired / 3 available
4 desired / 4 available
```

```text
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  anatomy-d89479458 (0/0 replicas created)
NewReplicaSet:   anatomy-8644478ddd (4/4 replicas created)
```

### Part 2.2 — selector drills

#### Equality: all production Pods

```bash
kubectl get pods -l environment=production --show-labels
```

```text
NAME         READY   STATUS    RESTARTS   LABELS
api-prod-1   1/1     Running   0          app=api,environment=production
web-prod-1   1/1     Running   0          app=web,environment=production
web-prod-2   1/1     Running   0          app=web,deprecated=true,environment=production
```

#### Set-based: web Pods in production or QA

```bash
kubectl get pods -l 'app=web,environment in (production,qa)'
```

```text
NAME         READY   STATUS    RESTARTS
web-prod-1   1/1     Running   0
web-prod-2   1/1     Running   0
web-qa-1     1/1     Running   0
```

#### Production Pods without the `deprecated` label

```bash
kubectl get pods -l 'environment=production,!deprecated'
```

```text
NAME         READY   STATUS    RESTARTS
api-prod-1   1/1     Running   0
web-prod-1   1/1     Running   0
```

### Part 2.3 — annotations are not selectors

```bash
kubectl get pods -l emagetech.io/git-sha=abc1234
```

```text
No resources found in a06 namespace.
```

```json
{
  "emagetech.io/git-sha": "abc1234",
  "emagetech.io/runbook": "https://wiki.example.com/web"
}
```

### Part 2.4 — relabel replacement and orphan

Before relabeling, four Pods matched `app=anatomy`.

After relabeling one Pod:

```text
anatomy-8644478ddd-44cv8   1/1   Running             app=escaped
anatomy-8644478ddd-gkgch   1/1   Running             app=anatomy
anatomy-8644478ddd-hjxxl   0/1   ContainerCreating   app=anatomy
anatomy-8644478ddd-rhbcc   1/1   Running             app=anatomy
anatomy-8644478ddd-vdx6w   1/1   Running             app=anatomy
```

After the replacement started:

```text
anatomy-8644478ddd-44cv8   1/1   Running   app=escaped
anatomy-8644478ddd-gkgch   1/1   Running   app=anatomy
anatomy-8644478ddd-hjxxl   1/1   Running   app=anatomy
anatomy-8644478ddd-rhbcc   1/1   Running   app=anatomy
anatomy-8644478ddd-vdx6w   1/1   Running   app=anatomy
```

The escaped Pod returned no owner reference:

```text
<empty>
```

A managed replacement returned a ReplicaSet owner reference:

```json
[
  {
    "apiVersion": "apps/v1",
    "controller": true,
    "kind": "ReplicaSet",
    "name": "anatomy-8644478ddd"
  }
]
```

### Part 3.2 — custom resources, printer columns, and defaulting

```text
NAME               SOURCE              SCHEDULE
postgres-nightly   pvc/postgres-data   0 2 * * *
redis-hourly       pvc/redis-data      0 * * * *
```

The `bk` short name returned the same objects. The omitted default was stored:

```text
7
```

### Part 3.3 — schema rejection

```text
The Backup "bad-backup" is invalid: spec.retentionDays: Invalid value: 0: spec.retentionDays in body should be greater than or equal to 1
```

### Part 3.4 — no controller status and CRD deletion

```text
kubectl get backups -o yaml | grep -c "status:"
0
```

After deleting the CRD:

```text
Error from server (NotFound): Unable to list "training.emagetech.io/v1, Resource=backups": the server could not find the requested resource (get backups.training.emagetech.io)
```

### Part 4 — Pod phase and container-state evidence

#### Broken image

```text
NAME           READY   STATUS             RESTARTS
broken-image   0/1     ImagePullBackOff   0
```

```text
Pending | Waiting | ImagePullBackOff | Restarts: 0
```

Events confirmed that the scheduler assigned the Pod, but the kubelet could not pull the image.

#### One-shot Pods

```text
NAME            READY   STATUS      RESTARTS
one-shot        0/1     Completed   0
one-shot-fail   0/1     Error       0
```

#### Crashing Pod

```text
NAME      READY   STATUS             RESTARTS
crasher   0/1     CrashLoopBackOff   4
```

```text
Phase: Running | Current state: Waiting — CrashLoopBackOff | Last exit code: 1 | Restarts: 5
```

### Part 5.2 — readiness before, during, and after

Before failure:

```text
NAME     ENDPOINTS
probed   10.244.1.19:80,10.244.2.17:80
```

During failure:

```text
NAME                      READY   STATUS    RESTARTS
probed-85d9c77896-749jl   0/1     Running   0
probed-85d9c77896-bjmkh   1/1     Running   0
```

```text
NAME     ENDPOINTS
probed   10.244.2.17:80
```

```text
Ready: false | Restarts: 0
```

After restoring readiness:

```text
NAME                      READY   STATUS    RESTARTS
probed-85d9c77896-749jl   1/1     Running   0
probed-85d9c77896-bjmkh   1/1     Running   0
```

```text
NAME     ENDPOINTS
probed   10.244.1.19:80,10.244.2.17:80
```

### Part 5.3 — liveness restart loop and startup-probe fix

The slow-starting Pod failed liveness repeatedly:

```text
Normal   Started    32s (x3 over 2m31s)  kubelet  Started container app
Warning  Unhealthy  5s (x8 over 2m12s)   kubelet  Liveness probe failed: connect: connection refused
```

The Pod reached at least two restarts before the application could complete its 90-second startup:

```text
Phase: Running | Restarts: 2
```

The startup-probe version survived startup:

```text
NAME               READY   STATUS    RESTARTS   AGE
slow-start-fixed   1/1     Running   0          103s
```

```text
Phase: Running | Ready: true | Restarts: 0
```

### Part 6 — init-container ordering

Before `fake-db` existed:

```text
NAME          READY   STATUS     RESTARTS
ordered-app   0/1     Init:0/1   0
```

```text
nc: bad address 'fake-db'
waiting for db
Phase: Pending | Initialized: False
```

After creating the Pod and Service:

```text
NAME      ENDPOINTS
fake-db   10.244.2.18:5432
```

```text
NAME          READY   STATUS    RESTARTS
ordered-app   1/1     Running   0
```

```text
Phase: Running | Initialized: True | Ready: True
```

### Part 7.1 — QoS classes

```text
qos-besteffort   BestEffort
qos-burstable    Burstable
qos-guaranteed   Guaranteed
```

### Part 7.2 — OOMKilled

```text
NAME         READY   STATUS      RESTARTS
oom-victim   0/1     OOMKilled   0
```

```text
Phase: Failed | Reason: OOMKilled | Exit: 137
```

## 4. One trade-off I had to make

I used a startup probe rather than increasing the liveness probe's initial delay. A large initial delay would be simpler, but it would leave the container without liveness protection for the full delay even when the application starts quickly. The startup probe adapts to the actual startup time while still enforcing a maximum startup window.

## 5. One thing I'm still unsure about

I am still unsure exactly when Kubernetes resets the increasing container-restart backoff after a previously crashing container remains healthy for a period of time.
