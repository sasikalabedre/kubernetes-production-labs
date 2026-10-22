# Part 4 - Kubernetes Resources and Scaling

Hands-on notes for the 21 resources and scaling topics, practised on **minikube** in the namespace `production-labs`.

Every topic has: a simple definition, a small flow diagram, the commands (and YAML) to run, and the **output you should see**.

> **About the outputs:** outputs marked **Output (from my lab)** were copied from my own terminal. Outputs marked **Expected output** show the format you will get. Names like `xxxxx`, IPs and ages will be different in your cluster.
>
> These are learning values. For a real workload, set requests, limits, autoscaling and replicas from measured usage, and do not copy the numbers blindly.

---

## Topics in this part

| No | Topic | No | Topic |
|----|-------|----|-------|
| 1 | Resource Requests | 12 | HPA Resource Metrics Requirements |
| 2 | Resource Limits | 13 | Vertical Pod Autoscaler (VPA) |
| 3 | CPU Throttling | 14 | VPA Recommendations |
| 4 | Memory Pressure | 15 | Node Autoscaling |
| 5 | Kubernetes Scheduler | 16 | Load Testing |
| 6 | Pending Pods Troubleshooting | 17 | Resource Consumption Monitoring |
| 7 | Taints and Tolerations | 18 | Capacity Planning |
| 8 | Node Affinity | 19 | Scaling Deployments |
| 9 | Pod Affinity and Anti-Affinity | 20 | Troubleshooting: Insufficient CPU |
| 10 | Topology Spread Constraints | 21 | Troubleshooting: Poor Resource Requests |
| 11 | Horizontal Pod Autoscaler (HPA) | | |

### Learning flow

```text
Requests and limits
        |
        v
Scheduler and pending pods
        |
        v
Placement rules (taints, affinity, topology spread)
        |
        v
Autoscaling (HPA, VPA, nodes)
        |
        v
Load testing, monitoring, capacity planning
        |
        v
Troubleshooting
```

---

## Setup

```bash
cd ~/kubernetes-production-labs/04-resources-and-scaling
kubectl config set-context --current --namespace=production-labs
kubectl get nodes -o wide
```

Resource numbers (`kubectl top`) need the **metrics server**:

```bash
minikube addons enable metrics-server
kubectl get pods -n kube-system | grep metrics-server     # wait for 1/1 Running
kubectl top nodes
```

The metrics server needs 1 to 2 minutes to become ready. Before that, `kubectl top` shows:

```text
error: Metrics API not available
```

**Output (from my lab):** while the metrics server was starting

```text
metrics-server-768f9f6999-l7cgl           0/1     Running   0               25s
```

`0/1` is normal at the start. Wait until it shows `1/1`.

### Units

| Value | Meaning |
|-------|---------|
| `1` or `1000m` | 1 CPU core |
| `500m` | 0.5 CPU core |
| `250m` | 0.25 CPU core |
| `100m` | 0.1 CPU core |
| `128Mi` | 128 mebibytes of memory (about 128 MB) |
| `1Gi` | 1 gibibyte of memory |

`m` means milli-CPU (one thousandth of a core).

---

## 1. Resource Requests

**In simple words:** A request is the **minimum** CPU and memory a container needs. The scheduler uses it to choose a node. It places the pod only on a node that still has this much free.

```text
Pod needs 250m CPU + 128Mi memory
        |
        v
Scheduler looks for a node with this much free
        |
        v
Pod is placed on that node
```

`resource-requests-demo.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-requests-demo
  namespace: production-labs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resource-requests-demo
  template:
    metadata:
      labels:
        app: resource-requests-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: "250m"        # minimum CPU (0.25 core)
              memory: "128Mi"    # minimum memory
```

```bash
kubectl apply -f resource-requests-demo.yaml
kubectl get pods -l app=resource-requests-demo
kubectl describe pod -l app=resource-requests-demo
```

Expected output of `kubectl describe pod` (the container part):

```text
Requests:
  cpu:        250m
  memory:     128Mi
```

A request is **not** a maximum. The pod can use more if the node has free resources. This file has no limit, so the pod has a guaranteed minimum but no ceiling.

---

## 2. Resource Limits

**In simple words:** A limit is the **maximum** a container may use. Over the CPU limit, the container is slowed down. Over the memory limit, the container is killed.

```text
CPU:     250m (minimum) ........ 500m (maximum)
Memory:  128Mi (minimum) ....... 256Mi (maximum)

Uses more CPU than the limit    --> throttled (slow, still running)
Uses more memory than the limit --> OOMKilled (killed and restarted)
```

`resource-limits-demo.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-limits-demo
  namespace: production-labs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resource-limits-demo
  template:
    metadata:
      labels:
        app: resource-limits-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: "250m"
              memory: "128Mi"
            limits:
              cpu: "500m"        # maximum CPU
              memory: "256Mi"    # maximum memory
```

```bash
kubectl apply -f resource-limits-demo.yaml
kubectl describe pod -l app=resource-limits-demo
kubectl get pod -l app=resource-limits-demo \
  -o custom-columns='POD:.metadata.name,CPU-REQUEST:.spec.containers[0].resources.requests.cpu,CPU-LIMIT:.spec.containers[0].resources.limits.cpu,MEM-REQUEST:.spec.containers[0].resources.requests.memory,MEM-LIMIT:.spec.containers[0].resources.limits.memory'
```

Expected output:

```text
POD                                    CPU-REQUEST   CPU-LIMIT   MEM-REQUEST   MEM-LIMIT
resource-limits-demo-xxxxxxxxxx-xxxxx  250m          500m        128Mi         256Mi
```

---

## 3. CPU Throttling

**In simple words:** When a container wants more CPU than its limit, Linux **slows it down**. The container does not crash. It just gets less CPU time.

```text
Loop wants 1000m CPU
        |
        v
Limit is only 200m
        |
        v
Container is throttled: it runs, but is held at about 200m
```

`cpu-load-test.yaml` - a pod that burns CPU in an endless loop, with a 200m limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-load-test
  namespace: production-labs
spec:
  restartPolicy: Never           # do not restart the pod
  containers:
    - name: busybox
      image: busybox:1.36
      command: ["sh", "-c", "while true; do :; done"]   # endless loop, uses all CPU it can get
      resources:
        requests:
          cpu: "100m"
          memory: "16Mi"
        limits:
          cpu: "200m"            # the loop wants 1000m, but is held at 200m
          memory: "32Mi"
```

> The old `kubectl run --requests/--limits` flags were removed from newer kubectl, so use a YAML file. Using the flags gives `error: unknown flag: --requests`.

```bash
kubectl apply -f cpu-load-test.yaml
kubectl get pod cpu-load-test
kubectl top pod cpu-load-test
kubectl exec cpu-load-test -- cat /sys/fs/cgroup/cpu.stat
```

**Output (from my lab):**

```text
NAME            READY   STATUS    RESTARTS   AGE
cpu-load-test   1/1     Running   0          23s
```

Expected output of `kubectl top` (after the metrics server is ready):

```text
NAME            CPU(cores)   MEMORY(bytes)
cpu-load-test   200m         1Mi
```

The CPU stays near `200m`, although the loop wants a full core. That is the limit throttling it. In `cpu.stat`, the numbers `nr_throttled` and `throttled_usec` keep growing.

Stop the load when done:

```bash
kubectl delete pod cpu-load-test
```

---

## 4. Memory Pressure

**In simple words:** Memory cannot be slowed down like CPU. If a container goes over its memory limit, Kubernetes **kills** it with the reason `OOMKilled`, and then restarts it.

```text
Container uses more memory than its limit
        |
        v
Container is killed: OOMKilled (exit code 137)
        |
        v
Kubernetes restarts it --> restart count goes up
```

Optional demo `memory-limit-demo.yaml` - the container tries to use 150M, but the limit is 100Mi:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-limit-demo
  namespace: production-labs
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]   # try to use 150M
      resources:
        requests:
          memory: "50Mi"
        limits:
          memory: "100Mi"        # smaller than 150M, so the container is killed
```

```bash
kubectl apply -f memory-limit-demo.yaml
kubectl get pod memory-limit-demo -w          # Ctrl+C to stop
kubectl describe pod memory-limit-demo
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes
```

Expected output of `kubectl get pod -w`:

```text
NAME                READY   STATUS      RESTARTS   AGE
memory-limit-demo   0/1     OOMKilled   0          5s
memory-limit-demo   0/1     CrashLoopBackOff   1   12s
```

Expected output of `kubectl describe pod` (the container part):

```text
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

Exit code `137` means the container was killed. Clean up: `kubectl delete pod memory-limit-demo`. Do not exhaust the memory of a shared machine on purpose.

---

## 5. Kubernetes Scheduler

**In simple words:** The scheduler chooses a node for every new pod. It looks at the resource requests, node labels, affinity, taints and spread rules.

```text
Pod created
    |
    v
Scheduler checks requests and rules
    |
    |---- suitable node found --> pod is assigned to that node
    |
    '---- no suitable node -----> pod stays Pending --> read the Events
```

```bash
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods -o wide
kubectl describe pod <pod-name>
```

Expected output of `kubectl get pods -o wide`:

```text
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE
resource-requests-demo-x   1/1     Running   0          2m    10.244.x.x    minikube
```

In `kubectl describe pod`, the `Events` part shows the scheduler's decision:

```text
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  2m    default-scheduler  Successfully assigned production-labs/resource-requests-demo-x to minikube
```

In `kubectl describe nodes`, the part `Allocated resources` shows how much of the node is already reserved by requests.

---

## 6. Pending Pods Troubleshooting

**In simple words:** `Pending` is a symptom, not the cause. The cause is always in the **Events** of the pod.

```text
Pod is Pending
    |
    v
kubectl describe pod  -->  read the Events at the bottom
    |
    v
Fix the reason: capacity, taint, affinity, node selector, PVC ...
```

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe nodes
```

What the `FailedScheduling` messages mean:

| Message | Meaning |
|---------|---------|
| `Insufficient cpu` / `Insufficient memory` | Requests are bigger than the free space on every node |
| `node(s) had untolerated taint` | The node has a taint the pod has no toleration for |
| `didn't match Pod's node affinity/selector` | No node has the required label |
| `didn't match pod anti-affinity rules` | The anti-affinity rule blocks every node |
| `unbound immediate PersistentVolumeClaims` | The PVC is not bound yet |

Examples are in topics 9 and 20.

---

## 7. Taints and Tolerations

**In simple words:** A **taint** is a "keep out" sign on a node. A **toleration** is a "pass" on a pod. A pod with the matching pass is allowed on that node.

```text
Node with taint team=database:NoSchedule  (keep out)
        |
        |-- normal pod (no pass)      --> not allowed
        '-- taint-test-pod (has pass) --> allowed
```

A toleration only **allows** a pod on a tainted node. It does not force the pod to go there.

`taint-test-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: taint-test-pod
  namespace: production-labs
spec:
  tolerations:                       # the passes of this pod
    - key: "team"                    # name of the taint it matches
      operator: "Equal"              # key and value must both match
      value: "database"
      effect: "NoSchedule"           # type of the taint it matches
  containers:
    - name: nginx
      image: nginx:1.27
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
```

Put the taint on the nodes (use only on a test cluster):

```bash
kubectl taint nodes --all team=database:NoSchedule
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

Expected output:

```text
node/minikube tainted
node/minikube-m02 tainted

NAME           TAINTS
minikube       [map[effect:NoSchedule key:team value:database]]
minikube-m02   [map[effect:NoSchedule key:team value:database]]
```

Test a pod with the toleration and a pod without it:

```bash
kubectl apply -f taint-test-pod.yaml
kubectl run no-toleration-pod --image=nginx:1.27 -n production-labs
kubectl get pods -o wide
kubectl describe pod no-toleration-pod
```

Expected output:

```text
NAME                READY   STATUS    NODE
taint-test-pod      1/1     Running   minikube
no-toleration-pod   0/1     Pending   <none>
```

Event of the pending pod:

```text
Warning  FailedScheduling  ...  0/2 nodes are available: 2 node(s) had untolerated taint {team: database}.
```

Remove the taint and the test pod afterwards:

```bash
kubectl taint nodes --all team=database:NoSchedule-
kubectl delete pod no-toleration-pod taint-test-pod
```

The `-` at the end of the taint removes it.

---

## 8. Node Affinity

**In simple words:** Node affinity **pulls** a pod towards a node that matches a label rule. A taint pushes pods away, affinity chooses where the pod goes.

```text
Pod says: "I must run on the node with hostname=minikube"
        |
        v
Scheduler checks node labels
        |
        |-- label matches --> pod is placed here
        '-- no match    --> pod is not placed here
```

`node-affinity-demo.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-demo
  namespace: production-labs
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # a MUST. Not moved later if the label changes
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/hostname             # every node has this label
                operator: In                            # value must be one of the list
                values:
                  - minikube
  containers:
    - name: nginx
      image: nginx:1.27
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
```

```bash
kubectl get nodes --show-labels
kubectl apply -f node-affinity-demo.yaml
kubectl get pod node-affinity-demo -o wide
```

Expected output:

```text
NAME                 READY   STATUS    RESTARTS   AGE   IP           NODE
node-affinity-demo   1/1     Running   0          10s   10.244.x.x   minikube
```

The NODE column shows `minikube`, even when `minikube-m02` has free space.

Two kinds of node affinity:

| Type | Meaning |
|------|---------|
| `requiredDuringSchedulingIgnoredDuringExecution` | A must. If no node matches, the pod stays `Pending` |
| `preferredDuringSchedulingIgnoredDuringExecution` | A wish. The scheduler tries, but can place the pod elsewhere |

---

## 9. Pod Affinity and Anti-Affinity

**In simple words:** Pod affinity places a pod **near** other pods. Pod anti-affinity keeps a pod **away** from other pods. "Near" and "away" are measured by the `topologyKey`. Here it is the node.

```text
anti-affinity: do NOT place this pod on a node that already has a pod with app=backend

Node 1 (has backend pod) --> X  not allowed
Node 2 (no backend pod)  --> OK allowed
```

First create the pod that the rule points to:

```bash
kubectl run backend-pod --image=nginx:1.27 --labels=app=backend -n production-labs
kubectl get pod backend-pod -o wide
```

`anti-affinity-demo.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: anti-affinity-demo
  namespace: production-labs
  labels:
    app: isolated
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # a MUST
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - backend                 # stay away from pods with app=backend
          topologyKey: kubernetes.io/hostname   # "away" means a different node
  containers:
    - name: nginx
      image: nginx:1.27
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
```

```bash
kubectl apply -f anti-affinity-demo.yaml
kubectl get pods -o wide --show-labels
kubectl describe pod anti-affinity-demo
```

**With 2 nodes:** `anti-affinity-demo` runs on the node that does **not** have `backend-pod`.

```text
NAME                 READY   STATUS    NODE
backend-pod          1/1     Running   minikube
anti-affinity-demo   1/1     Running   minikube-m02
```

**With only 1 node:** the pod stays `Pending`, because the only node already has a `backend` pod.

```text
NAME                 READY   STATUS    NODE
anti-affinity-demo   0/1     Pending   <none>
```

```text
Warning  FailedScheduling  ...  0/1 nodes are available: 1 node(s) didn't match pod anti-affinity rules.
```

Use `required...` only when strict separation is needed and there are enough nodes. For a softer rule, use `preferred...` with a `weight`.

Clean up: `kubectl delete pod backend-pod anti-affinity-demo`.

---

## 10. Topology Spread Constraints

**In simple words:** Topology spread constraints **spread the pods of one app evenly** across nodes (or zones), so they do not all pile up in one place.

```text
3 pods on 2 nodes, maxSkew 1:

OK  :  node 1: 2 pods,  node 2: 1 pod     (difference 1)
Not :  node 1: 3 pods,  node 2: 0 pods    (difference 3)
```

`topology-spread-demo.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: topology-spread-demo
  namespace: production-labs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: topology-spread-demo
  template:
    metadata:
      labels:
        app: topology-spread-demo
    spec:
      topologySpreadConstraints:
        - maxSkew: 1                              # max difference in pod count between nodes
          topologyKey: kubernetes.io/hostname     # each node is one place
          whenUnsatisfiable: DoNotSchedule        # a MUST. ScheduleAnyway would be a wish
          labelSelector:
            matchLabels:
              app: topology-spread-demo           # count only the pods of this app
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
```

```bash
kubectl apply -f topology-spread-demo.yaml
kubectl get pods -l app=topology-spread-demo -o wide
```

Expected output with 2 nodes:

```text
NAME                                    READY   STATUS    NODE
topology-spread-demo-xxxxxxxxxx-aaaaa   1/1     Running   minikube
topology-spread-demo-xxxxxxxxxx-bbbbb   1/1     Running   minikube-m02
topology-spread-demo-xxxxxxxxxx-ccccc   1/1     Running   minikube
```

With only 1 node, all 3 pods go on `minikube`, because the difference is always 0.

Running pods are **not moved** when you add a node. To spread them after adding a node, restart the Deployment:

```bash
kubectl rollout restart deployment topology-spread-demo
```

Compared with anti-affinity: anti-affinity is strict (no two pods together), topology spread is flexible (pods can share a node, but the counts stay nearly equal).

---

## 11. Horizontal Pod Autoscaler (HPA)

**In simple words:** HPA changes the **number of pods** automatically, based on CPU usage.

```text
CPU usage measured
       |
       v
HPA compares usage with the target
       |
       |--- above target --> more pods (up to max)
       '--- below target --> fewer pods (down to min)
```

`hpa-demo.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
  namespace: production-labs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: 100m          # HPA calculates the percentage from this request
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
```

Create the HPA:

```bash
kubectl apply -f hpa-demo.yaml
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=2 --max=5
kubectl get hpa hpa-demo
kubectl describe hpa hpa-demo
```

- `--cpu-percent=50`: keep the average CPU at 50% of the **request**
- `--min=2`: never fewer than 2 pods
- `--max=5`: never more than 5 pods

Expected output:

```text
NAME       REFERENCE             TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   cpu: 2%/50%   2         5         2          5m
```

**How the numbers work** (my reading in the lab was `2% (2m) / 50%`):

```text
Request per pod    = 100m
Target             = 50% of 100m = 50m per pod
Usage per pod      = 2m  -->  2m / 100m = 2%
```

Usage (2%) is far below the target (50%), so there is no reason to add pods.

```text
desired pods = current pods x (current usage / target usage), rounded up

idle:        2 x (2 / 50)   = 0.08 --> 1, but --min=2 keeps 2 pods
heavy load:  2 x (100 / 50) = 4    --> scale to 4 pods (max is 5)
```

Watch the scaling (stop with `Ctrl+C`):

```bash
kubectl get hpa hpa-demo --watch
kubectl get deployment hpa-demo --watch
```

---

## 12. HPA Resource Metrics Requirements

**In simple words:** HPA needs two things: the **metrics server** (to read CPU usage) and a **CPU request** on the container (to calculate the percentage). Without them the HPA cannot work.

```text
Metrics server --> gives CPU usage numbers
CPU request    --> gives the 100% base
        |
        v
HPA can calculate "2% of request"
```

```bash
kubectl top nodes
kubectl top pods -l app=hpa-demo
kubectl get hpa hpa-demo
kubectl describe hpa hpa-demo
```

Expected output when everything is fine:

```text
NAME       REFERENCE             TARGETS       MINPODS   MAXPODS   REPLICAS
hpa-demo   Deployment/hpa-demo   cpu: 2%/50%   2         5         2
```

When something is missing, the TARGETS column shows `<unknown>`:

```text
hpa-demo   Deployment/hpa-demo   cpu: <unknown>/50%   2   5   2
```

Check these:

```bash
kubectl get pods -n kube-system | grep metrics-server       # must be 1/1 Running
kubectl describe hpa hpa-demo                                # read the Events and Conditions
kubectl get events --sort-by=.metadata.creationTimestamp
```

| Symptom | Reason |
|---------|--------|
| `error: Metrics API not available` | Metrics server not enabled or not ready yet |
| `cpu: <unknown>/50%` | No metrics yet, or the container has no CPU request |

Fix for the first one: `minikube addons enable metrics-server`, then wait 1 to 2 minutes. If you run `minikube delete`, enable it again.

---

## 13. Vertical Pod Autoscaler (VPA)

**In simple words:** HPA adds more pods. VPA changes the **size** of a pod, meaning its CPU and memory **requests**. VPA is not part of minikube by default, so you must install its components first.

```text
VPA watches real usage
        |
        v
Recommends better requests (or changes them, depending on the mode)
```

Check whether VPA is installed:

```bash
kubectl get deployments -n kube-system | grep vpa
kubectl get crd | grep verticalpodautoscaler
```

If VPA is not installed, `kubectl apply` of the file below fails with:

```text
error: resource mapping not found ... no matches for kind "VerticalPodAutoscaler"
```

`vpa-demo.yaml` - recommendation only:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hpa-demo-vpa
  namespace: production-labs
spec:
  targetRef:                  # which workload to watch
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  updatePolicy:
    updateMode: "Off"         # recommendations only, nothing is changed
```

```bash
kubectl apply -f vpa-demo.yaml
kubectl describe vpa hpa-demo-vpa
kubectl get deployment hpa-demo \
  -o jsonpath='{.spec.template.spec.containers[0].resources.requests}'
echo
```

Expected output of the last command (requests are **not** changed):

```text
{"cpu":"100m","memory":"64Mi"}
```

`updateMode: "Off"` means VPA only recommends. It does not change the Deployment.

---

## 14. VPA Recommendations

**In simple words:** VPA reads the real usage and tells you what CPU and memory requests would fit better.

```bash
kubectl describe vpa hpa-demo-vpa
```

Expected output (the numbers will be different):

```text
Recommendation:
  Container Recommendations:
    Container Name:  nginx
    Lower Bound:
      Cpu:     25m
      Memory:  262144k
    Target:
      Cpu:     25m
      Memory:  262144k
    Upper Bound:
      Cpu:     100m
      Memory:  500Mi
```

| Field | Meaning |
|-------|---------|
| **Target** | The recommended requests |
| **Lower Bound** | The lowest estimate that is still suitable |
| **Upper Bound** | The highest estimate that may be needed |
| **Update Mode** | `Off` = only recommend, other modes can apply changes |

The recommendation appears only after VPA has watched the pods for a few minutes. Use measured behaviour and testing before you change the requests, and do not copy a recommendation blindly into production.

---

## 15. Node Autoscaling

**In simple words:** Node autoscaling adds or removes **nodes** automatically when pods cannot be scheduled. It needs a Cluster Autoscaler and real infrastructure (a cloud). Minikube does not have it. `minikube node add` only adds a node by hand.

```text
Pod cannot be scheduled (no capacity)
        |
        v
Cluster Autoscaler notices it
        |
        v
Infrastructure creates a new node
        |
        v
Node becomes Ready --> scheduler places the pod
```

Add a node by hand for multi-node practice:

```bash
kubectl get nodes -o wide
minikube node list
minikube node add
kubectl get nodes -w          # Ctrl+C when the node is Ready
```

**Output (from my lab):** right after the node joined

```text
NAME           STATUS     ROLES           AGE   VERSION
minikube       Ready      control-plane   8d    v1.37.0
minikube-m02   NotReady   <none>          8s    v1.37.0
```

`NotReady` is normal for the first 1 to 2 minutes, while the node starts its network pod. Then it shows `Ready`.

A manual node is **not** proof that autoscaling works. Real node autoscaling happens only with a Cluster Autoscaler on supported infrastructure (for example EKS or GKE).

### If `minikube node add` does not work

What I faced in the lab, and the fixes:

| Problem | Fix |
|---------|-----|
| `Failed to create control group inotify object: Too many open files` | Raise the inotify limits (below) |
| `minikube node add` says "Successfully added" but `kubectl get nodes` does not show it | The node did not join. Check `minikube status`, and `kubelet` must be `Running` |
| Kubelet log: `open /var/lib/kubelet/config.yaml: no such file or directory` | The node never joined. Remove all traces of the node (below) and add it again |
| `minikube node add` prints `container is missing, will recreate` | An old node record is still saved. A fresh add never prints this |

Raise the inotify limits (the real cause of the first failure):

```bash
sudo sysctl fs.inotify.max_user_instances=8192
sudo sysctl fs.inotify.max_user_watches=524288
echo "fs.inotify.max_user_instances=8192" | sudo tee /etc/sysctl.d/99-inotify.conf
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.d/99-inotify.conf
```

Remove every trace of a failed node, then add it again:

```bash
docker rm -f minikube-m02 minikube-m03
docker volume rm minikube-m02 minikube-m03
rm -rf ~/.minikube/machines/minikube-m02 ~/.minikube/machines/minikube-m03
jq '.Nodes |= map(select(.Name != "m02" and .Name != "m03"))' ~/.minikube/profiles/minikube/config.json > /tmp/config.json
cp /tmp/config.json ~/.minikube/profiles/minikube/config.json
minikube node list          # only "minikube" must be left
minikube node add
```

The clean option: `minikube delete` and then `minikube start --nodes 2`. It removes the pods and addons, so apply your YAML files again.

A second node uses about 3 GB more memory. Remove it when you finish: `minikube node delete m02`.

---

## 16. Load Testing

**In simple words:** A load test sends many requests to an app, to see how the CPU grows and how the HPA reacts.

```text
Load generator sends requests
        |
        v
App CPU goes up
        |
        v
HPA compares CPU with the target
        |
        v
More pods are created
        |
Stop the load --> CPU falls --> pods are removed slowly
```

The load test needs the `hpa-demo` Deployment, its HPA and a Service:

```bash
kubectl get deployment hpa-demo
kubectl get services
kubectl get hpa hpa-demo
```

If no Service named `hpa-demo` exists, create it:

```bash
kubectl expose deployment hpa-demo \
  --name=hpa-demo --port=80 --target-port=80 --type=ClusterIP
```

Start a temporary load generator:

```bash
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sh -c 'while true; do wget -q -O- http://hpa-demo; done'
```

Watch the load and the scaling (in two terminals):

```bash
kubectl get pod load-generator -o wide
kubectl top pods
kubectl get hpa hpa-demo --watch
kubectl get deployment hpa-demo --watch
```

Expected output of the HPA watch:

```text
NAME       REFERENCE             TARGETS        MINPODS   MAXPODS   REPLICAS
hpa-demo   Deployment/hpa-demo   cpu: 2%/50%    2         5         2
hpa-demo   Deployment/hpa-demo   cpu: 85%/50%   2         5         2
hpa-demo   Deployment/hpa-demo   cpu: 85%/50%   2         5         4
hpa-demo   Deployment/hpa-demo   cpu: 40%/50%   2         5         4
```

The `REPLICAS` column grows from 2 to 4 while the load is high.

If the logs show `bad address 'hpa-demo'`, check that the Service exists in the same namespace:

```bash
kubectl get services
kubectl get endpoints hpa-demo
```

Stop the load:

```bash
kubectl delete pod load-generator
kubectl get hpa hpa-demo --watch
```

Scale-down is slow. It can take about 5 minutes, because the HPA waits (stabilization) before it removes pods.

---

## 17. Resource Consumption Monitoring

**In simple words:** Monitoring means checking how much CPU and memory the nodes and pods use right now.

```text
kubectl top nodes   --> node usage now
kubectl top pods    --> pod usage now
kubectl describe    --> requests, limits, status, events
Compare             --> find high usage or oversized requests
```

```bash
kubectl top nodes
kubectl top pods
kubectl top pods -A
kubectl top pods -l app=hpa-demo
kubectl describe deployment hpa-demo
kubectl get pods -l app=hpa-demo \
  -o custom-columns='POD:.metadata.name,CPU-REQUEST:.spec.containers[0].resources.requests.cpu,CPU-LIMIT:.spec.containers[0].resources.limits.cpu,MEM-REQUEST:.spec.containers[0].resources.requests.memory,MEM-LIMIT:.spec.containers[0].resources.limits.memory'
```

Expected output:

```text
NAME       CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
minikube   350m         4%       1800Mi          23%

NAME                        CPU(cores)   MEMORY(bytes)
hpa-demo-xxxxxxxxxx-aaaaa   2m           4Mi
hpa-demo-xxxxxxxxxx-bbbbb   2m           4Mi
```

`kubectl top` shows the **current** usage only. For history and alerts, you need tools like Prometheus and Grafana.

---

## 18. Capacity Planning

**In simple words:** Capacity planning means checking whether the nodes have enough room for the pods, now and when the app grows.

```text
Measure usage and growth
        |
        v
Review requests, limits and replicas
        |
        v
Compare the requests with node allocatable capacity
        |
        v
Adjust resources, replicas or nodes
        |
        v
Monitor and repeat
```

```bash
kubectl get nodes -o wide
kubectl describe node minikube
kubectl top nodes
kubectl top pods
kubectl get pods -o wide
```

Expected output of `kubectl describe node minikube` (the important part):

```text
Allocatable:
  cpu:     12
  memory:  7900Mi
Allocated resources:
  Resource   Requests      Limits
  --------   --------      ------
  cpu        1450m (12%)   2200m (18%)
  memory     780Mi (9%)    1200Mi (15%)
```

- **Allocatable** is what the node can give to pods.
- **Requests** is what pods have already reserved. The scheduler compares this with Allocatable.

Review for each workload:

- Current and peak CPU and memory use
- Requests and limits
- Replica count and expected growth
- Node allocatable capacity and room for system pods
- Scheduling failures and HPA or VPA recommendations

Do not plan from usage alone. The scheduler uses **requests**, and the nodes also need free room for system components.

---

## 19. Scaling Deployments

**In simple words:** Manual scaling means you set the number of replicas yourself.

```text
kubectl scale --replicas=N
        |
        v
Deployment desired replicas change
        |
        v
ReplicaSet creates or removes pods
        |
        v
Check the READY count
```

```bash
kubectl get deployment hpa-demo
kubectl scale deployment hpa-demo --replicas=4
kubectl get deployment hpa-demo
kubectl get pods -l app=hpa-demo
kubectl scale deployment hpa-demo --replicas=2
kubectl get deployment hpa-demo
```

Expected output after scaling up:

```text
deployment.apps/hpa-demo scaled

NAME       READY   UP-TO-DATE   AVAILABLE   AGE
hpa-demo   4/4     4            4           30m
```

If an HPA targets this Deployment, the HPA can change the replica count again from the metrics. Manual scaling does not stay as a permanent override.

---

## 20. Troubleshooting: Insufficient CPU

**Problem:** A pod asks for more CPU than any node has free, so it stays `Pending`.

```text
Pod requests 100 CPU cores
        |
        v
No node has 100 free cores
        |
        v
FailedScheduling: Insufficient cpu --> Pending
```

`insufficient-cpu-pod.yaml` - the wrong version:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: insufficient-cpu-pod
  namespace: production-labs
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      resources:
        requests:
          cpu: "100"          # 100 full cores, far more than the node has
          memory: "64Mi"
```

```bash
kubectl apply -f insufficient-cpu-pod.yaml
kubectl get pod insufficient-cpu-pod
kubectl describe pod insufficient-cpu-pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

Expected output:

```text
NAME                   READY   STATUS    RESTARTS   AGE
insufficient-cpu-pod   0/1     Pending   0          20s
```

```text
Warning  FailedScheduling  ...  0/2 nodes are available: 2 Insufficient cpu.
```

**Fix:** use a realistic request. Most pod spec fields cannot be changed after creation, so delete the pod and create it again with this corrected file:

```yaml
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
```

```bash
kubectl delete pod insufficient-cpu-pod
kubectl apply -f insufficient-cpu-pod.yaml
kubectl get pod insufficient-cpu-pod
```

Expected output after the fix: `1/1 Running`.

Important: a missing **limit** never causes `Insufficient cpu`. Requests affect scheduling, and limits affect runtime.

---

## 21. Troubleshooting: Poor Resource Requests

**Problem:** A request that is too big reserves node capacity on paper, even if the pod uses almost nothing. It wastes space and can block other pods.

```text
Request 2 cores + 2Gi, real usage 2m + 4Mi
        |
        v
Node capacity is reserved but not used
        |
        v
Other pods may not fit --> wasted capacity
```

`poor-resource-requests-pod.yaml` - an oversized request for comparison:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: poor-resource-requests-pod
  namespace: production-labs
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      resources:
        requests:
          cpu: "2"
          memory: "2Gi"
        limits:
          cpu: "2"
          memory: "2Gi"
```

```bash
kubectl apply -f poor-resource-requests-pod.yaml
kubectl get pod poor-resource-requests-pod
kubectl top pod poor-resource-requests-pod
kubectl describe pod poor-resource-requests-pod
```

Expected output of `kubectl top`:

```text
NAME                         CPU(cores)   MEMORY(bytes)
poor-resource-requests-pod   2m           4Mi
```

The pod uses `2m` CPU and `4Mi` memory, but it reserves `2` cores and `2Gi`. That is the problem.

A high request does **not** automatically make a pod `Pending`. It only stops scheduling when no node has enough free capacity. If the node has enough, the pod runs, and the extra space is wasted.

**Fix:** set the request close to the real usage, with some room, using the numbers from `kubectl top` or the VPA recommendation:

```yaml
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
```

---

## Cleanup

Delete only the temporary test resources:

```bash
kubectl delete pod insufficient-cpu-pod poor-resource-requests-pod --ignore-not-found
kubectl delete pod load-generator cpu-load-test memory-limit-demo --ignore-not-found
kubectl delete pod taint-test-pod node-affinity-demo anti-affinity-demo backend-pod --ignore-not-found
kubectl delete deployment topology-spread-demo resource-requests-demo resource-limits-demo cpu-throttling-demo --ignore-not-found
kubectl taint nodes --all team=database:NoSchedule- 2>/dev/null
kubectl delete -f vpa-demo.yaml --ignore-not-found       # only if you installed VPA
```

Keep the `hpa-demo` Deployment, its HPA and its Service if you want to reuse them. Remove the extra node when you finish: `minikube node delete m02`. Do not delete the whole minikube cluster as a normal cleanup.

---

## Quick command reference

```bash
# Nodes and metrics
kubectl get nodes -o wide
kubectl top nodes
kubectl top pods
kubectl top pods -A

# Workloads and placement
kubectl get deployments
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp

# Taints and labels
kubectl taint nodes <node-name> team=database:NoSchedule
kubectl taint nodes <node-name> team=database:NoSchedule-
kubectl get nodes --show-labels

# HPA
kubectl get hpa
kubectl describe hpa hpa-demo
kubectl get hpa hpa-demo --watch

# Manual scaling
kubectl scale deployment hpa-demo --replicas=4
kubectl scale deployment hpa-demo --replicas=2

# Service and load test
kubectl get endpoints hpa-demo
kubectl logs load-generator --tail=10
kubectl delete pod load-generator --ignore-not-found

# VPA
kubectl get crd | grep verticalpodautoscaler
kubectl describe vpa hpa-demo-vpa

# Minikube nodes
minikube node list
minikube node add
minikube node delete m02
```

---

## Troubleshooting table

| Symptom | First thing to check |
|---------|----------------------|
| Pod is `Pending` | `kubectl describe pod`, read the Events |
| `Insufficient cpu` or `memory` | Requests are too big, or add capacity |
| `untolerated taint` | Add a toleration or remove the taint |
| `didn't match pod anti-affinity rules` | Not enough nodes for a strict rule |
| `OOMKilled`, exit code 137 | Memory limit is too low, or the app leaks memory |
| App is slow but not crashing | CPU throttling, check the CPU limit |
| `error: Metrics API not available` | Enable the metrics server and wait 1 to 2 minutes |
| HPA shows `<unknown>` | Metrics server missing, or no CPU request |
| HPA does not scale down | Wait about 5 minutes (stabilization) |
| New node does not appear in `kubectl get nodes` | Check `minikube status` and the kubelet log, and the inotify limit |

---

## Key concepts to remember

- **Requests** decide where a pod is scheduled, and they are the base of the HPA percentage.
- **Limits** cap runtime use. CPU over the limit is throttled, memory over the limit is killed (`OOMKilled`).
- **HPA** changes the number of pods. **VPA** changes the size of the requests.
- **Cluster Autoscaler** adds nodes, but only on supported infrastructure. `minikube node add` is manual.
- **Taints** keep pods away, **tolerations** allow them, **affinity** chooses the node, **topology spread** spreads the pods.
- `kubectl top` shows the current usage only. It is not a history tool.
- `Pending` is a symptom, not the cause. Read the Events in `kubectl describe pod`.

---

## Part 4 Completion

All 21 resources and scaling topics were practised on minikube in the `production-labs` namespace: requests, limits, throttling, memory pressure, the scheduler, pending pods, taints, affinity, topology spread, HPA, VPA, node scaling, load testing, monitoring, capacity planning and 2 troubleshooting labs.
