# Assignment 05 — Adewale Alli

**GitHub username:** Adewalealli  
**Date completed:** 2026-07-19

> **Environment note:** The assignment examples use a Kind cluster named `k8s-lab`. My existing cluster is named `neom-k8s`, so my context is `kind-neom-k8s` and my control-plane container is `neom-k8s-control-plane`.

## 1. Answers to the 10 questions

### Q1 — Control-plane components and the per-node bucket

- **kube-apiserver:** Exposes the Kubernetes API, authenticates and authorizes requests, runs admission, and is the front door through which cluster state is read or changed.
- **etcd:** Stores the cluster's persistent source-of-truth data, including workloads, configuration, Secrets, RBAC objects, and controller state.
- **kube-scheduler:** Watches for Pods without a node assignment, selects an eligible node, and records that decision in `spec.nodeName`.
- **kube-controller-manager:** Runs reconciliation controllers that continuously compare desired state with observed state and create, update, or delete objects to close the difference.

`kube-proxy` and `kindnet` run once per node in my cluster. Kubernetes uses a **DaemonSet** when it needs one Pod on every eligible node. Kindnet provides Pod-network connectivity and node routes, while kube-proxy programs Linux Service rules. Kube-proxy does not personally carry every packet; the Linux kernel applies the rules and performs the actual forwarding and NAT.

### Q2 — Static Pods and the bootstrap chicken-and-egg problem

The **kubelet** watches `/etc/kubernetes/manifests` on the control-plane node. When it finds the static Pod manifests, it asks the container runtime to start the API server, etcd, scheduler, and controller manager directly.

This solves the bootstrap problem because the kubelet does not need an already-working API server to start those control-plane components. Linux starts the kubelet through systemd, and the kubelet starts the static Pods from local files.

The API server shows a mirror object for each static Pod. If I delete a mirror Pod with `kubectl`, the local manifest is still the source of truth, so the kubelet recreates the mirror object. To actually stop a static Pod, its manifest must be moved or removed from the watched directory.

### Q3 — etcd quorum and a stateless API server

etcd requires a majority quorum:

```text
quorum = floor(member count / 2) + 1
```

A one-member cluster requires one vote and tolerates zero member failures. A two-member cluster requires both votes, so it also tolerates zero failures: losing either member breaks quorum. Adding a second member therefore adds another failure point without increasing failure tolerance. A three-member cluster requires two votes and can tolerate one failure, which is why odd member counts are preferred.

The API server is stateless because it does not keep the authoritative cluster database inside its own process. If etcd were destroyed, Kubernetes would lose stored desired state and configuration such as Deployments, Pods, Services, Secrets, ConfigMaps, RBAC, and controller metadata.

Already-running containers could continue temporarily as long as their nodes, kubelets, container runtimes, and networking remained healthy. Existing Pod network namespaces, Pod IPs, CNI routes, and kube-proxy rules would not instantly disappear. However, the cluster could not reliably reconcile changes, schedule replacements, restore deleted objects, or rebuild state after node and process restarts.

### Q4 — Contexts and context drift

The field that changed `kubectl get pods` was the context's default **namespace**. Both `kind-neom-k8s` and `k8s-lab-system` pointed to the same cluster and user, but `k8s-lab-system` defaulted to `kube-system`.

A concrete context-drift accident would be intending to delete a test Deployment but still being pointed at a production cluster or namespace, causing a production workload to be deleted or modified.

My prevention habit is to check the active context before mutating anything:

```bash
kubectl config current-context
```

When namespace matters, I also verify it:

```bash
kubectl config view --minify -o jsonpath='{..namespace}{"\n"}'
```

### Q5 — Request flow: authentication, authorization, admission, persistence, and dry-run limits

The important request sent for my Deployment was:

```text
POST /apis/apps/v1/namespaces/pod-lab/deployments
```

The request flow was:

1. **Authentication:** The API server used my kubeconfig client certificate to identify me as `kubernetes-admin`.
2. **Authorization:** Kubernetes checked whether that identity could create an `apps/v1` Deployment in the `pod-lab` namespace.
3. **Admission:** Admission logic validated the object and could reject or mutate it according to cluster policies.
4. **Persistence:** After the request passed those stages, the API server stored the Deployment state in etcd and returned `201 Created`.

The earlier `--dry-run=client` command validated the command arguments and constructed a structurally valid Deployment object locally without sending a create request to the API server. It could not catch server-side conditions such as an RBAC denial, an admission-policy rejection, a resource-quota violation, or a missing namespace.

Network flow for the real apply:

```text
kubectl
→ TCP/TLS connection
→ HTTPS packets to kube-apiserver
→ authentication/authorization/admission
→ write to etcd
→ response packets return to kubectl
```

### Q6 — Observe, diff, act

The **ReplicaSet controller**, running inside `kube-controller-manager`, observed that the ReplicaSet desired three Pods but had fewer than three live Pods after one was deleted.

- **Observe:** Read the ReplicaSet and its current Pods.
- **Diff:** Desired replicas `3` minus available/current replicas `2` equals one missing Pod.
- **Act:** Create one replacement Pod object.

Desired state did not change when I deleted the individual Pod. Deleting the Pod changed actual state only. The controller restored actual state to the already-declared desired count.

### Q7 — Scheduler-down blast radius

The **ReplicaSet controller inside kube-controller-manager** created the two new Pod objects after the Deployment was scaled from three to five replicas.

The scheduler's missing contribution was the node assignment:

```text
spec.nodeName
```

While the scheduler was unavailable, the two new Pods existed in the API and etcd, but remained `Pending` with an empty `spec.nodeName`.

Their flow stopped before the data plane could start them:

```text
ReplicaSet creates Pod object
→ no scheduler assignment
→ no kubelet claims the Pod
→ no pause container / Pod sandbox
→ no CNI-assigned Pod IP
→ no application container
```

The already-running Pods would continue serving traffic while their nodes, kubelets, runtimes, and network remained healthy. This shows that the data plane is not continuously dependent on the control plane for every packet or every running process. However, the control plane is required to schedule new Pods, replace failed workloads, and react to desired-state changes.

### Q8 — kubelet as systemd, kube-proxy, and the pause container

**(a) Kubelet as a systemd service**

The kubelet must exist before ordinary Pods and static control-plane Pods can be started. If kubelet were itself an ordinary Pod, it would depend on itself to start. Linux therefore starts it as a systemd-managed service. The kubelet can then watch static manifests and ask containerd to create Pods.

**(b) kube-proxy manipulates iptables rather than routing packets itself**

Kube-proxy watches Services and EndpointSlices and programs Linux packet-processing rules. When a packet is sent to a Service IP, the **Linux kernel** matches the packet against those rules, performs DNAT from the virtual Service address to a real Pod IP and port, and forwards the packet using Linux routing and CNI-provided connectivity.

```text
kube-proxy = programs the traffic rules
Linux kernel = applies NAT and forwards packets
CNI/kindnet = provides Pod interfaces and routes
```

**(c) Pause container**

The pause container is the Pod sandbox that holds the Pod's shared network namespace and stable sandbox identity. The application containers join that namespace, similar to Assignment 04's `--network container:api` mechanism.

Because they join one network namespace, containers in the same Pod share:

- one Pod IP;
- the same localhost interface;
- the same network ports.

Without the pause container or equivalent sandbox mechanism, containers could receive separate network namespaces, `localhost` communication between them would fail, they would not share one stable Pod IP, and restarting whichever container owned the namespace could destroy the network environment for the rest of the Pod.

### Q9 — `kubectl top` and the aggregation layer

`kubectl get pods` worked because Pods are native Kubernetes resources served by the kube-apiserver through the core `v1` API. Their object state is stored through the API server in etcd.

`kubectl top nodes` initially failed because it requests the separate `metrics.k8s.io` API group, and the fresh cluster had no Metrics Server or `APIService` registered to serve that group.

The aggregation layer allows an extension API server to appear beneath the main Kubernetes API URL. After Metrics Server was installed, the `v1beta1.metrics.k8s.io` `APIService` told the kube-apiserver to proxy requests for that API group to the `kube-system/metrics-server` Service.

The flow is:

```text
kubectl top
→ kube-apiserver
→ aggregation layer recognizes metrics.k8s.io
→ request is proxied through the metrics-server Service
→ Service rules select the metrics-server Pod
→ metrics-server returns CPU and memory measurements
→ response returns through kube-apiserver to kubectl
```

Metrics Server gathers current resource measurements from node kubelets. These are transient measurements provided by an aggregated API, not ordinary Kubernetes objects natively served and persisted in etcd.

### Q10 — Three ways to retrieve the Deployment image

```bash
kubectl describe deployment drill | grep -i 'Image:'
```

```bash
kubectl get deployment drill \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

```bash
kubectl get deployment drill -o yaml | grep 'image:'
```

All three returned:

```text
nginx:1.27-alpine
```

For a script, I would use JSONPath because it selects the exact structured field and returns only the required value. `describe` is human-oriented output whose formatting can change, and YAML piped into `grep` can accidentally match unrelated image fields.

## 2. Cluster survey

### Nodes

```bash
kubectl get nodes -o wide
```

```text
NAME                     STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION                     CONTAINER-RUNTIME
neom-k8s-control-plane   Ready    control-plane   82d   v1.32.0   172.18.0.4    <none>        Debian GNU/Linux 12 (bookworm)   6.6.87.2-microsoft-standard-WSL2   containerd://1.7.24
neom-k8s-worker          Ready    <none>          82d   v1.32.0   172.18.0.5    <none>        Debian GNU/Linux 12 (bookworm)   6.6.87.2-microsoft-standard-WSL2   containerd://1.7.24
neom-k8s-worker2         Ready    <none>          82d   v1.32.0   172.18.0.6    <none>        Debian GNU/Linux 12 (bookworm)   6.6.87.2-microsoft-standard-WSL2   containerd://1.7.24
```

### kube-system Pods

```bash
kubectl get pods -n kube-system -o wide
```

```text
NAME                                             READY   STATUS    RESTARTS      AGE    IP            NODE
coredns-668d6bf9bc-4b7kn                         1/1     Running   3 (15d ago)   82d    10.244.0.4    neom-k8s-control-plane
coredns-668d6bf9bc-jzrxw                         1/1     Running   3 (15d ago)   82d    10.244.0.3    neom-k8s-control-plane
etcd-neom-k8s-control-plane                      1/1     Running   2 (15d ago)   67d    172.18.0.4    neom-k8s-control-plane
kindnet-2q4rd                                    1/1     Running   3 (15d ago)   82d    172.18.0.5    neom-k8s-worker
kindnet-lp7sb                                    1/1     Running   3 (15d ago)   82d    172.18.0.6    neom-k8s-worker2
kindnet-rqmr6                                    1/1     Running   3 (15d ago)   82d    172.18.0.4    neom-k8s-control-plane
kube-apiserver-neom-k8s-control-plane            1/1     Running   2 (15d ago)   67d    172.18.0.4    neom-k8s-control-plane
kube-controller-manager-neom-k8s-control-plane   1/1     Running   3 (15d ago)   82d    172.18.0.4    neom-k8s-control-plane
kube-proxy-f5tgv                                 1/1     Running   3 (15d ago)   82d    172.18.0.6    neom-k8s-worker2
kube-proxy-jnhd7                                 1/1     Running   3 (15d ago)   82d    172.18.0.4    neom-k8s-control-plane
kube-proxy-kkdbs                                 1/1     Running   3 (15d ago)   82d    172.18.0.5    neom-k8s-worker
kube-scheduler-neom-k8s-control-plane            1/1     Running   0             34h    172.18.0.4    neom-k8s-control-plane
metrics-server-56fb9549f4-v9bd5                  1/1     Running   0             119m   10.244.1.13   neom-k8s-worker
```

**Three-bucket classification**

1. **Static control-plane Pods:** `etcd`, `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler`. The kubelet starts them from `/etc/kubernetes/manifests`.
2. **Per-node networking agents:** `kindnet` and `kube-proxy`. One copy of each runs on every node through DaemonSets. Kindnet establishes Pod-network connectivity and routes; kube-proxy programs Service rules.
3. **Cluster add-ons:** `coredns` and `metrics-server`. CoreDNS provides cluster DNS, while Metrics Server provides the aggregated resource-metrics API.

The ordinary add-on Pods have `10.244.x.x` Pod-network addresses. The static control-plane and networking Pods use the nodes' `172.18.0.x` host-network addresses.

### Static Pod manifest directory

```bash
docker exec neom-k8s-control-plane \
  ls /etc/kubernetes/manifests
```

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

## 3. Evidence

### Part 2.2 — etcd key for `etcd-canary`

```bash
kubectl exec -n kube-system etcd-neom-k8s-control-plane -- \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/default \
  --prefix \
  --keys-only | grep etcd-canary
```

```text
/registry/pods/default/etcd-canary
```

### Part 3.2 — `kubectl get pods` under `k8s-lab-system`

```bash
kubectl config use-context k8s-lab-system
kubectl get pods
```

```text
Switched to context "k8s-lab-system".

NAME                                             READY   STATUS    RESTARTS      AGE
coredns-668d6bf9bc-4b7kn                         1/1     Running   3 (13d ago)   80d
coredns-668d6bf9bc-jzrxw                         1/1     Running   3 (13d ago)   80d
etcd-neom-k8s-control-plane                      1/1     Running   2 (13d ago)   64d
kindnet-2q4rd                                    1/1     Running   3 (13d ago)   80d
kindnet-lp7sb                                    1/1     Running   3 (13d ago)   80d
kindnet-rqmr6                                    1/1     Running   3 (13d ago)   80d
kube-apiserver-neom-k8s-control-plane            1/1     Running   2 (13d ago)   64d
kube-controller-manager-neom-k8s-control-plane   1/1     Running   3 (13d ago)   80d
kube-proxy-f5tgv                                 1/1     Running   3 (13d ago)   80d
kube-proxy-jnhd7                                 1/1     Running   3 (13d ago)   80d
kube-proxy-kkdbs                                 1/1     Running   3 (13d ago)   80d
kube-scheduler-neom-k8s-control-plane            1/1     Running   3 (13d ago)   80d
```

### Part 4.2 — request from `kubectl apply -v=8`

```text
"Request" verb="POST" url="https://127.0.0.1:42885/apis/apps/v1/namespaces/pod-lab/deployments?fieldManager=kubectl-client-side-apply&fieldValidation=Strict"
"Response" status="201 Created"
```

Relevant method and path:

```text
POST /apis/apps/v1/namespaces/pod-lab/deployments
```

### Part 5.1 — deleted Pod replaced

```bash
kubectl get pods -l app=flow-demo -w
```

```text
NAME                         READY   STATUS              RESTARTS   AGE
flow-demo-7b574c4775-z9jwd   1/1     Running             0          15m
flow-demo-7b574c4775-4chbk   0/1     Pending             0          0s
flow-demo-7b574c4775-fvz48   0/1     Pending             0          0s
flow-demo-7b574c4775-4chbk   0/1     ContainerCreating   0          0s
flow-demo-7b574c4775-fvz48   0/1     ContainerCreating   0          0s
flow-demo-7b574c4775-4chbk   0/1     Terminating         0          0s
flow-demo-7b574c4775-mmkj5   0/1     Pending             0          0s
flow-demo-7b574c4775-mmkj5   0/1     ContainerCreating   0          0s
flow-demo-7b574c4775-fvz48   1/1     Running             0          1s
flow-demo-7b574c4775-4chbk   1/1     Terminating         0          5s
flow-demo-7b574c4775-mmkj5   1/1     Running             0          5s
flow-demo-7b574c4775-4chbk   0/1     Completed           0          6s
```

The terminating `4chbk` Pod was replaced by the newly created `mmkj5` Pod, which became Running.

### Part 5.2 — Pending Pods with empty `spec.nodeName`

```bash
kubectl scale deployment flow-demo -n pod-lab --replicas=5
kubectl get pods -n pod-lab -l app=flow-demo -o wide
```

```text
NAME                         READY   STATUS    RESTARTS   AGE   IP            NODE
flow-demo-7b574c4775-7jfpv   0/1     Pending   0          9s    <none>        <none>
flow-demo-7b574c4775-8vxl8   1/1     Running   0          50s   10.244.1.14   neom-k8s-worker
flow-demo-7b574c4775-mmkj5   1/1     Running   0          34h   10.244.2.9    neom-k8s-worker2
flow-demo-7b574c4775-nz7pf   0/1     Pending   0          9s    <none>        <none>
flow-demo-7b574c4775-sm489   1/1     Running   0          34h   10.244.1.11   neom-k8s-worker
```

```bash
kubectl get pod flow-demo-7b574c4775-7jfpv -n pod-lab \
  -o jsonpath='name={.metadata.name}{"\n"}phase={.status.phase}{"\n"}nodeName={.spec.nodeName}{"\n"}'
```

```text
name=flow-demo-7b574c4775-7jfpv
phase=Pending
nodeName=
```

### Part 5.3 — Pods Running after scheduler recovery

```bash
kubectl get pods -n pod-lab -l app=flow-demo -o wide
```

```text
NAME                         READY   STATUS    RESTARTS   AGE    IP            NODE
flow-demo-7b574c4775-7jfpv   1/1     Running   0          76s    10.244.2.13   neom-k8s-worker2
flow-demo-7b574c4775-8vxl8   1/1     Running   0          117s   10.244.1.14   neom-k8s-worker
flow-demo-7b574c4775-mmkj5   1/1     Running   0          34h    10.244.2.9    neom-k8s-worker2
flow-demo-7b574c4775-nz7pf   1/1     Running   0          76s    10.244.1.15   neom-k8s-worker
flow-demo-7b574c4775-sm489   1/1     Running   0          34h    10.244.1.11   neom-k8s-worker
```

### Part 7.3(a) — every Pod with its node

```bash
kubectl get pods -A \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
```

```text
chapter-05             ch5-web                                      neom-k8s-worker2
default                drill-7c7848ffb4-6ptcq                       neom-k8s-worker
default                drill-7c7848ffb4-cm552                       neom-k8s-worker2
ingress-nginx          ingress-nginx-controller-76f9746865-g4b4n    neom-k8s-worker2
kube-system            coredns-668d6bf9bc-4b7kn                     neom-k8s-control-plane
kube-system            coredns-668d6bf9bc-jzrxw                     neom-k8s-control-plane
kube-system            etcd-neom-k8s-control-plane                  neom-k8s-control-plane
kube-system            kindnet-2q4rd                                neom-k8s-worker
kube-system            kindnet-lp7sb                                neom-k8s-worker2
kube-system            kindnet-rqmr6                                neom-k8s-control-plane
kube-system            kube-apiserver-neom-k8s-control-plane        neom-k8s-control-plane
kube-system            kube-controller-manager-neom-k8s-control-plane neom-k8s-control-plane
kube-system            kube-proxy-f5tgv                             neom-k8s-worker2
kube-system            kube-proxy-jnhd7                             neom-k8s-control-plane
kube-system            kube-proxy-kkdbs                             neom-k8s-worker
kube-system            kube-scheduler-neom-k8s-control-plane        neom-k8s-control-plane
local-path-storage     local-path-provisioner-58cc7856b6-xhrml      neom-k8s-control-plane
neom-marketplace       api-6f57f46584-5jbzx                         neom-k8s-worker
neom-marketplace       api-6f57f46584-m7vh9                         neom-k8s-worker2
neom-marketplace       postgres-0                                   neom-k8s-worker
pod-lab                flow-demo-7b574c4775-4kxt2                   neom-k8s-worker2
pod-lab                flow-demo-7b574c4775-fvz48                   neom-k8s-worker
pod-lab                flow-demo-7b574c4775-mmkj5                   neom-k8s-worker2
pod-lab                flow-demo-7b574c4775-sm489                   neom-k8s-worker
pod-lab                flow-demo-7b574c4775-z9jwd                   neom-k8s-worker
pod-lab                hello                                        neom-k8s-worker2
pod-lab                hello-1                                      neom-k8s-worker
pod-lab                nginx                                        neom-k8s-worker
pod-lab                web-65d846d465-7nb8p                         neom-k8s-worker2
pod-lab                web-65d846d465-crws8                         neom-k8s-worker2
pod-lab                web-65d846d465-qgv64                         neom-k8s-worker
```

### Part 7.3(b) — unique images in kube-system

```bash
kubectl get pods -n kube-system \
  -o jsonpath='{.items[*].spec.containers[*].image}' \
  | tr ' ' '\n' | sort -u
```

```text
docker.io/kindest/kindnetd:v20241212-9f82dd49
registry.k8s.io/coredns/coredns:v1.11.3
registry.k8s.io/etcd:3.5.16-0
registry.k8s.io/kube-apiserver:v1.32.0
registry.k8s.io/kube-controller-manager:v1.32.0
registry.k8s.io/kube-proxy:v1.32.0
registry.k8s.io/kube-scheduler:v1.32.0
```

### Part 7.3(c) — Pods not in Running phase

```bash
kubectl get pods -A --field-selector status.phase!=Running
```

```text
NAMESPACE   NAME    READY   STATUS    RESTARTS   AGE
pod-lab     nginx   0/1     Unknown   0          31d
```

### Part 7.3(d) — allocatable memory per node

```bash
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.memory}{"\n"}{end}'
```

```text
neom-k8s-control-plane   16011460Ki
neom-k8s-worker          16011460Ki
neom-k8s-worker2         16011460Ki
```

### Part 7.4 — Metrics Server repaired and `kubectl top` working

Before installation:

```text
error: Metrics API not available
```

After installing Metrics Server and patching it with `--kubelet-insecure-tls`:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
```

```text
NAME                     SERVICE                      AVAILABLE
v1beta1.metrics.k8s.io   kube-system/metrics-server   True
```

```bash
kubectl top nodes
```

```text
NAME                     CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
neom-k8s-control-plane   204m         1%       642Mi           4%
neom-k8s-worker          61m          0%       425Mi           2%
neom-k8s-worker2         57m          0%       567Mi           3%
```

## 4. One trade-off I had to make

I patched the Metrics Server Deployment to add `--kubelet-insecure-tls` instead of downloading and rewriting its full manifest. The patch was the smallest change required for the Kind lab and preserved the upstream manifest as the source of truth. Rewriting the manifest would have made the final configuration more explicit in one file, but it would also have introduced more copied configuration to maintain and more opportunity for unrelated mistakes.

## 5. One thing I'm still unsure about

I am still unsure about the exact boundary between kube-proxy's Service rules, the CNI plugin's routes, and the Linux kernel's packet forwarding during cross-node Service traffic.
