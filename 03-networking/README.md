# Part 3 - Kubernetes Networking

Hands-on notes for the 22 networking topics, practised on **minikube** in the namespace `production-labs`.

Every topic has: a simple definition, a small flow diagram, the commands (and YAML) to run, and the **output you should see**.

> **About the outputs:** outputs marked **Output (from my lab)** were copied from my own terminal. Outputs marked **Expected output** show the format you will get. Names like `10.244.x.x` and ages will be different in your cluster.

---

## Topics in this part

| No | Topic | No | Topic |
|----|-------|----|-------|
| 1 | ClusterIP Service | 12 | HTTP Routing |
| 2 | NodePort Service | 13 | Load Balancing |
| 3 | LoadBalancer Service | 14 | NetworkPolicy |
| 4 | Cluster DNS | 15 | External Traffic |
| 5 | CoreDNS | 16 | Pod-to-Pod Connectivity |
| 6 | Service Discovery | 17 | Pod-to-External-Service Connectivity |
| 7 | EndpointSlices | 18 | Request-Flow Tracing |
| 8 | Ingress Resources | 19 | Troubleshooting: Incorrect Service Selector |
| 9 | Ingress Controllers | 20 | Troubleshooting: Incorrect Target Port |
| 10 | TLS Termination | 21 | Troubleshooting: DNS Lookup Failure |
| 11 | Certificate Management | 22 | Troubleshooting: TLS Configuration Error |

---

## Setup

Set the default namespace so you do not need `-n` in every command:

```bash
cd ~/kubernetes-production-labs/03-networking
kubectl create namespace production-labs        # only if it does not exist
kubectl config set-context --current --namespace=production-labs
```

Create two small test pods. They stay alive for 24 hours so you can use `kubectl exec` on them:

```bash
kubectl run clusterip-client --image=busybox:1.36 --restart=Never -- sleep 86400
kubectl run lb-test-client   --image=busybox:1.36 --restart=Never -- sleep 86400
kubectl get pods
```

> If a test pod shows `Completed`, its sleep time ended. Delete it and create it again.
>
> Pod IPs and the minikube node IP (`192.168.49.2`) can change. Check the node IP with `minikube ip` and use your own value.

### The app used in all topics

`clusterip-deployment.yaml` - 2 nginx pods with the label `app: clusterip-demo`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusterip-demo
  namespace: production-labs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: clusterip-demo
  template:
    metadata:
      labels:
        app: clusterip-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f clusterip-deployment.yaml
kubectl get pods -o wide
```

Expected output:

```text
NAME                              READY   STATUS    RESTARTS   AGE   IP             NODE
clusterip-demo-xxxxxxxxxx-aaaaa   1/1     Running   0          1m    10.244.x.x     minikube
clusterip-demo-xxxxxxxxxx-bbbbb   1/1     Running   0          1m    10.244.x.x     minikube
```

---

## 1. ClusterIP Service

**In simple words:** A ClusterIP Service gives a group of pods one stable internal IP. It works only inside the cluster.

```text
Client pod
    |
    v
ClusterIP Service :80
    |
    v
nginx pod :80
```

`clusterip-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-demo-svc
  namespace: production-labs
spec:
  type: ClusterIP
  selector:
    app: clusterip-demo      # finds pods with this label
  ports:
    - port: 80               # port of the Service
      targetPort: 80         # port of the container
```

```bash
kubectl apply -f clusterip-service.yaml
kubectl get service clusterip-demo-svc
kubectl exec clusterip-client -- wget -qO- http://clusterip-demo-svc
```

**Output (from my lab):**

```text
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
clusterip-demo-svc   ClusterIP   10.96.114.183  <none>        80/TCP    43h
```

The `wget` command prints the nginx welcome page:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<h1>Welcome to nginx!</h1>
```

`EXTERNAL-IP` is `<none>`, so this Service works only inside the cluster.

---

## 2. NodePort Service

**In simple words:** A NodePort Service opens one port on every node, so you can reach the app from outside the cluster using `NodeIP:port`.

```text
Browser / curl
    |
    v
NodeIP:30080
    |
    v
NodePort Service :80
    |
    v
nginx pod :80
```

`nodeport-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-demo-svc
  namespace: production-labs
spec:
  type: NodePort
  selector:
    app: clusterip-demo
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080        # port opened on the node (30000-32767)
```

```bash
kubectl apply -f nodeport-service.yaml
kubectl get service nodeport-demo-svc
minikube ip
curl -i http://$(minikube ip):30080
```

**Output (from my lab):**

```text
NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nodeport-demo-svc   NodePort   10.101.114.153  <none>        80:30080/TCP   43h
```

Expected output of `curl`:

```text
HTTP/1.1 200 OK
Server: nginx/1.27.x
Content-Type: text/html
...
<title>Welcome to nginx!</title>
```

`80:30080/TCP` means: Service port 80, node port 30080.

---

## 3. LoadBalancer Service

**In simple words:** A LoadBalancer Service asks the cloud for an external load balancer. On minikube there is no cloud, so the external IP stays `<pending>` until you run `minikube tunnel`.

```text
External client
      |
      v
External IP (LoadBalancer)
      |
      v
Service :80
      |
      v
nginx pods
```

`loadbalancer-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-demo-svc
  namespace: production-labs
spec:
  type: LoadBalancer
  selector:
    app: clusterip-demo
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f loadbalancer-service.yaml
kubectl get service loadbalancer-demo-svc     # EXTERNAL-IP shows <pending>
minikube tunnel                                # keep this terminal open
```

**Output (from my lab), before the tunnel:**

```text
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
loadbalancer-demo-svc   LoadBalancer   10.105.223.161  <pending>     80:30887/TCP   43h
```

In a second terminal, while the tunnel is running:

```bash
kubectl get service loadbalancer-demo-svc
```

Expected output: `EXTERNAL-IP` now shows an IP address instead of `<pending>`. Then `curl http://<EXTERNAL-IP>` gives the nginx page.

---

## 4. Cluster DNS

**In simple words:** Cluster DNS changes a Service name into its ClusterIP, so apps use names and not IPs.

```text
Client pod
   |  http://clusterip-demo-svc
   v
Cluster DNS (10.96.0.10)
   |  gives ClusterIP
   v
Service -> nginx pod
```

Full name of a Service:

```text
<service-name>.<namespace>.svc.cluster.local
clusterip-demo-svc.production-labs.svc.cluster.local
```

```bash
kubectl exec clusterip-client -- cat /etc/resolv.conf
kubectl exec clusterip-client -- nslookup clusterip-demo-svc
kubectl exec clusterip-client -- nslookup clusterip-demo-svc.production-labs.svc.cluster.local
kubectl get service clusterip-demo-svc        # compare the CLUSTER-IP with the DNS answer
```

**Output (from my lab):** `/etc/resolv.conf`

```text
search production-labs.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

**Output (from my lab):** `nslookup clusterip-demo-svc`

```text
Server:		10.96.0.10
Address:	10.96.0.10:53

** server can't find clusterip-demo-svc.cluster.local: NXDOMAIN

Name:	clusterip-demo-svc.production-labs.svc.cluster.local
Address: 10.96.114.183

** server can't find clusterip-demo-svc.svc.cluster.local: NXDOMAIN
```

How to read it:

- `Address: 10.96.114.183` is the ClusterIP of the Service. It is the same IP as in `kubectl get service`.
- `nameserver 10.96.0.10` is the cluster DNS IP. It is the same for all pods in all namespaces.
- `search` lets the short name work in the same namespace. For another namespace use `service-name.namespace`.
- The `NXDOMAIN` lines are normal. They are search-domain tries that do not exist. The line with the correct full name gives the IP.

---

## 5. CoreDNS

**In simple words:** CoreDNS is the software (pods in `kube-system`) that answers the DNS questions. The `kube-dns` Service gives these pods one stable IP.

```text
Pod DNS question
      |
      v
kube-dns Service (10.96.0.10)
      |
      v
CoreDNS pods
      |-- name ends with cluster.local --> answer from Kubernetes Services
      '-- any other name --------------> forward to outside DNS
```

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get service kube-dns -n kube-system
kubectl get deployment coredns -n kube-system
kubectl get configmap coredns -n kube-system -o yaml
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20
```

**Output (from my lab):** CoreDNS pods and the `kube-dns` Service

```text
NAME                       READY   STATUS    RESTARTS      AGE
coredns-559f6c778d-2ncsx   1/1     Running   4 (74m ago)   7d18h
coredns-559f6c778d-pcz8z   1/1     Running   4 (74m ago)   7d18h

NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   7d23h
```

**Output (from my lab):** the Corefile in the `coredns` ConfigMap

```text
.:53 {
    log
    errors
    health { lameduck 5s }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    hosts {
       192.168.49.1 host.minikube.internal
       fallthrough
    }
    forward . /etc/resolv.conf { max_concurrent 1000 }
    cache 30 { disable success cluster.local
               disable denial cluster.local }
    loop
    reload
    loadbalance
}
```

Main lines of the Corefile:

| Line | Meaning |
|------|---------|
| `kubernetes cluster.local ...` | Answers Service and pod names inside the cluster |
| `forward . /etc/resolv.conf` | Sends outside names (like google.com) to outside DNS |
| `cache 30` | Remembers answers for 30 seconds (not for `cluster.local` names) |
| `log` | Prints every lookup in the logs |
| `health`, `ready` | Tell Kubernetes CoreDNS is alive and ready |
| `reload` | Reads the Corefile again when it changes |

**Output (from my lab):** CoreDNS logs

```text
[INFO] 10.244.120.93:47665 - 63200 "A IN clusterip-demo-svc.production-labs.svc.cluster.local. udp 70 false 512" NOERROR qr,aa,rd 138 0.000131046s
[INFO] 10.244.120.93:52551 - 41475 "A IN clusterip-demo-svc.cluster.local. udp 50 false 512" NXDOMAIN qr,aa,rd 143 0.000179785s
[INFO] 10.244.120.93:35742 - 24550 "A IN google.com. udp 28 false 512" NOERROR qr,rd,ra 54 0.003972595s
```

How to read the logs:

- `10.244.120.93` is the pod that asked. `A IN <name>` is the question.
- `NOERROR` means the name was found. `NXDOMAIN` means the name does not exist.
- `google.com` is answered by the `forward` line. `...svc.cluster.local` is answered by the `kubernetes` block.

Check both kinds of lookup:

```bash
kubectl exec clusterip-client -- nslookup kubernetes.default     # inside the cluster
kubectl exec clusterip-client -- nslookup google.com             # outside the cluster
```

Both give an IP address, which proves CoreDNS answers cluster names and outside names.

---

## 6. Service Discovery

**In simple words:** Service discovery means finding a Service by its name, not by its IP.

```text
Client pod --- name: clusterip-demo-svc ---> CoreDNS gives ClusterIP   (discovery)
Client pod --- request to ClusterIP ------> Service -> nginx pod       (connection)
```

```bash
kubectl get services
kubectl exec clusterip-client -- wget -qO- http://clusterip-demo-svc
```

**Output (from my lab):**

```text
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
app2                    ClusterIP      10.107.97.212    <none>        80/TCP         42h
clusterip-demo-svc      ClusterIP      10.96.114.183    <none>        80/TCP         43h
loadbalancer-demo-svc   LoadBalancer   10.105.223.161   <pending>     80:30887/TCP   43h
nodeport-demo-svc       NodePort       10.101.114.153   <none>        80:30080/TCP   43h
web-headless            ClusterIP      None             <none>        80/TCP         2d
```

The `wget` command prints the nginx welcome page. The client used only the name, and DNS found the IP.

`web-headless` has CLUSTER-IP `None`. It is a headless Service, a special Service without its own ClusterIP.

---

## 7. EndpointSlices

**In simple words:** An EndpointSlice is the list of pod IPs behind a Service. These are the pods that **receive** the traffic. Only ready pods are in the list.

```text
Service selector (app=clusterip-demo)
        |
        v
Matching ready pods
        |
        v
EndpointSlice = pod IPs + ports
```

```bash
kubectl get endpointslices
kubectl get endpointslices -l kubernetes.io/service-name=clusterip-demo-svc
kubectl describe endpointslices -l kubernetes.io/service-name=clusterip-demo-svc
kubectl get pods -l app=clusterip-demo -o wide      # the pod IPs match the EndpointSlice
```

Expected output:

```text
NAME                       ADDRESSTYPE   PORTS   ENDPOINTS                    AGE
clusterip-demo-svc-xxxxx   IPv4          80      10.244.x.x,10.244.x.x        43h
```

The two IPs in `ENDPOINTS` are the same as the pod IPs of the two nginx pods in `kubectl get pods -o wide`. The `clusterip-client` pod is not in the list, because it only calls the Service.

---

## 8. Ingress Resources

**In simple words:** An Ingress is a set of HTTP routing rules for traffic coming from outside the cluster.

```text
Client request for web.local
          |
          v
Ingress rule
          |
          v
clusterip-demo-svc :80
          |
          v
nginx pod
```

`web-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: production-labs
spec:
  ingressClassName: nginx          # which Ingress controller handles this
  rules:
    - host: web.local              # rule applies to this host name
      http:
        paths:
          - path: /                # any path
            pathType: Prefix
            backend:
              service:
                name: clusterip-demo-svc
                port:
                  number: 80
```

```bash
kubectl apply -f web-ingress.yaml
kubectl get ingress
kubectl describe ingress web-ingress
curl -i --resolve web.local:80:192.168.49.2 http://web.local/
```

Expected output of `kubectl get ingress`:

```text
NAME          CLASS   HOSTS       ADDRESS        PORTS   AGE
web-ingress   nginx   web.local   192.168.49.2   80      1m
```

**Output (from my lab):** `curl`

```text
HTTP/1.1 200 OK
Date: Wed, 21 Oct 2026 06:12:54 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
Last-Modified: Wed, 16 Apr 2025 12:01:11 GMT
ETag: "67ff9c07-267"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

`--resolve` tells curl which IP to use for `web.local` for this one request. It does not change system DNS.

> The nginx controller does not allow two Ingresses with the same host and path. Before applying the TLS Ingress in topic 10, delete this one: `kubectl delete ingress web-ingress`. If you forget, you get this error:
>
> ```text
> Error from server (BadRequest): admission webhook "validate.nginx.ingress.kubernetes.io" denied the request: host "web.local" and path "/" is already defined in ingress production-labs/web-ingress
> ```

---

## 9. Ingress Controllers

**In simple words:** The Ingress is only the rule. The Ingress controller is the software that follows the rule. Without a controller, an Ingress does nothing.

```text
Ingress YAML rules
       |
       v
nginx Ingress controller
       |
       v
Service -> Pod
```

Install on minikube:

```bash
minikube addons enable ingress
```

```bash
kubectl get pods -n ingress-nginx          # controller must be 1/1 Running
kubectl get services -n ingress-nginx
kubectl get ingressclass
```

**Output (from my lab):**

```text
NAME                                       READY   STATUS      RESTARTS       AGE
ingress-nginx-admission-create-h2m8w       0/1     Completed   0              8d
ingress-nginx-admission-patch-n7jrk        0/1     Completed   1 (8d ago)     8d
ingress-nginx-controller-d7cd8c989-t5t52   1/1     Running     4 (106m ago)   8d
```

The two `Completed` pods are one-time setup jobs and are normal. The `ingress-nginx-controller` pod is the one that follows the Ingress rules.

Expected output of `kubectl get ingressclass`:

```text
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       8d
```

If you run `minikube delete`, enable the addon again.

---

## 10. TLS Termination

**In simple words:** TLS makes `http` into `https` (encrypted). TLS termination means the encryption ends (is decrypted) at the Ingress controller. After that, plain HTTP goes to the Service.

```text
Client --- HTTPS (encrypted) ---> Ingress controller
                                      |  TLS ends here
                                      v  plain HTTP
                                  Service :80
                                      |
                                      v
                                  nginx pod
```

`web-tls-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-tls-ingress
  namespace: production-labs
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - web.local
      secretName: web-local-tls     # Secret with the certificate and private key
  rules:
    - host: web.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: clusterip-demo-svc
                port:
                  number: 80
```

The Secret `web-local-tls` must exist first (see topic 11).

```bash
kubectl delete ingress web-ingress                 # same host and path, so remove the old one
kubectl apply -f web-tls-ingress.yaml
kubectl get ingress web-tls-ingress
curl -k -i --resolve web.local:443:192.168.49.2 https://web.local/
```

Expected output of `kubectl get ingress`:

```text
NAME              CLASS   HOSTS       ADDRESS        PORTS     AGE
web-tls-ingress   nginx   web.local   192.168.49.2   80, 443   1m
```

`PORTS` shows `80, 443`. Port 443 is HTTPS, which means TLS is on.

Expected output of `curl`:

```text
HTTP/2 200
server: nginx
content-type: text/html
...
<title>Welcome to nginx!</title>
```

- `-k` accepts the self-signed certificate. Use it only in this lab.
- `--resolve web.local:443:192.168.49.2` sends the HTTPS request to the minikube node.

---

## 11. Certificate Management

**In simple words:** A TLS certificate proves the website is real. It comes with a private key. Both are stored in a Kubernetes Secret of type `kubernetes.io/tls`.

```text
openssl creates certificate + private key
        |
        v
kubectl creates TLS Secret (web-local-tls)
        |
        v
Ingress uses the Secret
```

Create a self-signed certificate for the lab:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout web-local.key -out web-local.crt \
  -subj "/CN=web.local" \
  -addext "subjectAltName=DNS:web.local"
```

Create the Secret:

```bash
kubectl create secret tls web-local-tls \
  --cert=web-local.crt --key=web-local.key -n production-labs
kubectl get secret web-local-tls
```

Expected output:

```text
NAME            TYPE                DATA   AGE
web-local-tls   kubernetes.io/tls   2      10s
```

`DATA 2` means the Secret holds 2 items: the certificate and the key.

Look inside the certificate:

```bash
openssl x509 -in web-local.crt -noout -subject -issuer -dates
openssl x509 -in web-local.crt -noout -ext subjectAltName
```

Expected output:

```text
subject=CN = web.local
issuer=CN = web.local
notBefore=Oct 21 06:00:00 2026 GMT
notAfter=Oct 21 06:00:00 2027 GMT

X509v3 Subject Alternative Name:
    DNS:web.local
```

`subject` and `issuer` are the same, because the certificate is self-signed.

Check that the certificate and the private key belong together. Both hashes must be the same:

```bash
openssl x509 -in web-local.crt -pubkey -noout | sha256sum
openssl pkey -in web-local.key -pubout | sha256sum
```

Expected output:

```text
3f2a...9c1b  -
3f2a...9c1b  -
```

Same hash on both lines means the certificate and the key are a pair. A different hash means they do not match.

> **Never push the private key to GitHub.** Add `*.key` to `.gitignore`.

---

## 12. HTTP Routing

**In simple words:** The URL path decides which Service gets the traffic.

```text
routing.local/app1 --> Ingress --> clusterip-demo-svc --> nginx pods
routing.local/app2 --> Ingress --> app2               --> app2 pods
```

`http-routing-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: http-routing-ingress
  namespace: production-labs
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # cut the path to / before sending to the pod
spec:
  ingressClassName: nginx
  rules:
    - host: routing.local
      http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: clusterip-demo-svc
                port:
                  number: 80
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app2
                port:
                  number: 80
```

```bash
kubectl apply -f http-routing-ingress.yaml
kubectl describe ingress http-routing-ingress
curl -i --resolve routing.local:80:192.168.49.2 http://routing.local/app1
curl -i --resolve routing.local:80:192.168.49.2 http://routing.local/app2
```

Expected output of `kubectl describe ingress` (rules part):

```text
Rules:
  Host           Path  Backends
  ----           ----  --------
  routing.local
                 /app1   clusterip-demo-svc:80 (10.244.x.x:80,10.244.x.x:80)
                 /app2   app2:80 (10.244.x.x:80)
```

Expected output of both `curl` commands: `HTTP/1.1 200 OK` and a page from the app behind each path.

**Why the annotation:** without it, the pod receives `/app1`, nginx has only `/`, and the answer is a 404:

```text
HTTP/1.1 404 Not Found
```

`rewrite-target: /` changes the path to `/` before it reaches the pod, so the 404 goes away.

---

## 13. Load Balancing

**In simple words:** A Service shares requests between all its ready pods.

```text
                 +--> nginx pod 1
Client --> Service
                 +--> nginx pod 2
```

Send 10 requests:

```bash
kubectl exec clusterip-client -- sh -c 'for i in $(seq 1 10); do wget -qO- http://clusterip-demo-svc > /dev/null; done'
```

Both pods show the same page, so the page cannot tell which pod answered. Check the logs of both pods instead:

```bash
kubectl logs -l app=clusterip-demo --prefix --tail=10
```

Expected output:

```text
[pod/clusterip-demo-xxxxxxxxxx-aaaaa/nginx] 10.244.x.x - - [21/Oct/2026:06:30:01 +0000] "GET / HTTP/1.1" 200 615 "-" "Wget" "-"
[pod/clusterip-demo-xxxxxxxxxx-aaaaa/nginx] 10.244.x.x - - [21/Oct/2026:06:30:01 +0000] "GET / HTTP/1.1" 200 615 "-" "Wget" "-"
[pod/clusterip-demo-xxxxxxxxxx-bbbbb/nginx] 10.244.x.x - - [21/Oct/2026:06:30:01 +0000] "GET / HTTP/1.1" 200 615 "-" "Wget" "-"
...
```

`GET / HTTP/1.1` lines appear in **both** pods, which shows that the Service shared the 10 requests. The split is not always exactly 5 and 5.

---

## 14. NetworkPolicy

**In simple words:** A NetworkPolicy is a firewall rule for pods. When a policy selects a pod, that pod accepts only the traffic the policy allows. Everything else is blocked.

> NetworkPolicy works only if the network plugin supports it. This lab uses minikube started with `--cni=calico`.

```text
Client pod --- allowed by policy? ---> selected nginx pod
                   yes: reaches the pod
                   no : blocked
```

**Policy 1 - block all incoming traffic.** There is no `ingress:` section, so the allowed list is empty.

`deny-clusterip-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-clusterip-ingress
  namespace: production-labs
spec:
  podSelector:
    matchLabels:
      app: clusterip-demo       # protects the nginx pods
  policyTypes:
    - Ingress                   # controls incoming traffic
```

**Policy 2 - allow only `lb-test-client` on TCP port 80.**

`allow-lb-client.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-lb-client
  namespace: production-labs
spec:
  podSelector:
    matchLabels:
      app: clusterip-demo
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              run: lb-test-client   # label added by "kubectl run lb-test-client"
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl get pods --show-labels
kubectl apply -f deny-clusterip-ingress.yaml
kubectl exec clusterip-client -- wget -T 5 -qO- http://clusterip-demo-svc     # blocked

kubectl apply -f allow-lb-client.yaml
kubectl exec lb-test-client -- wget -T 5 -qO- http://clusterip-demo-svc       # allowed
kubectl exec clusterip-client -- wget -T 5 -qO- http://clusterip-demo-svc     # still blocked
kubectl get networkpolicy
```

Expected output when blocked:

```text
wget: download timed out
command terminated with exit code 1
```

Expected output when allowed: the nginx welcome page (`<title>Welcome to nginx!</title>`).

Expected output of `kubectl get networkpolicy`:

```text
NAME                     POD-SELECTOR         AGE
allow-lb-client          app=clusterip-demo   1m
deny-clusterip-ingress   app=clusterip-demo   3m
```

Policies add up. Together, only `lb-test-client` can reach the nginx pods. The Ingress controller is in another namespace, so it is also blocked and `web.local` stops working while these policies are applied.

Remove them when the test is done:

```bash
kubectl delete networkpolicy deny-clusterip-ingress allow-lb-client
```

---

## 15. External Traffic

**In simple words:** Traffic from outside the cluster enters through a NodePort or through the Ingress.

```text
External client
   |                    \
   v                     v
NodeIP:30080          Ingress controller (host web.local)
   |                     |
   v                     v
NodePort Service      Service
   |                     |
   +--------> nginx pods <+
```

```bash
minikube ip
curl -i http://192.168.49.2:30080
curl -k -i --resolve web.local:443:192.168.49.2 https://web.local/
```

Expected output:

```text
192.168.49.2

HTTP/1.1 200 OK
...
<title>Welcome to nginx!</title>

HTTP/2 200
...
<title>Welcome to nginx!</title>
```

The first `curl` reaches the app through the NodePort. The second one reaches it through the Ingress with HTTPS.

---

## 16. Pod-to-Pod Connectivity

**In simple words:** One pod calls another pod directly by its pod IP. No Service and no DNS are used.

```text
Client pod ----------------> nginx pod IP :80
```

```bash
kubectl get pods -l app=clusterip-demo -o wide        # copy a pod IP
kubectl exec lb-test-client -- wget -qO- http://<POD_IP>
```

Replace `<POD_IP>` with a real IP from the command above.

Expected output: the nginx welcome page.

```html
<title>Welcome to nginx!</title>
```

If the pod IP works but the Service name does not, the problem is in the Service or DNS. If the pod IP also fails, check the network or a NetworkPolicy.

---

## 17. Pod-to-External-Service Connectivity

**In simple words:** A pod reaches a website outside the cluster. This uses DNS and outgoing traffic together.

```text
Pod
 |  1. CoreDNS forwards the name to outside DNS
 v
 |  2. request leaves the cluster
 v
example.com
```

```bash
kubectl exec lb-test-client -- wget -T 10 -qO- https://example.com
```

Expected output:

```text
wget: note: TLS certificate validation not implemented
<!doctype html>
<html>
<head>
    <title>Example Domain</title>
...
```

| Result | Meaning |
|--------|---------|
| The page comes back | External connection works |
| Times out after 10 seconds | Outgoing traffic is blocked or there is no internet |
| `bad address` | DNS failed, check CoreDNS |

The TLS note comes from BusyBox `wget`. Use this only as a connectivity test.

---

## 18. Request-Flow Tracing

**In simple words:** Follow one request step by step, so you know where to look when it fails.

```text
Client
  |
  v
DNS / host name (web.local)
  |
  v
Ingress controller
  |
  v
Service (clusterip-demo-svc)
  |
  v
EndpointSlice (pod IPs)
  |
  v
nginx pod
```

Check every step:

```bash
curl -k -i --resolve web.local:443:192.168.49.2 https://web.local/    # 1. client request
kubectl get pods -n ingress-nginx                                      # 2. controller is running
kubectl describe ingress web-tls-ingress                               # 3. Ingress rule and backend
kubectl get service clusterip-demo-svc                                 # 4. Service
kubectl get endpointslices -l kubernetes.io/service-name=clusterip-demo-svc   # 5. pod IPs
kubectl get pods -l app=clusterip-demo -o wide                         # 6. pods are Running
```

What each step should show:

| Step | Good result |
|------|-------------|
| 1. curl | `200` and the nginx page |
| 2. controller | `ingress-nginx-controller` is `1/1 Running` |
| 3. Ingress | TLS secret `web-local-tls`, host `web.local`, backend `clusterip-demo-svc:80` with pod IPs |
| 4. Service | A ClusterIP on port 80 |
| 5. EndpointSlice | The IPs of the two nginx pods |
| 6. pods | Both pods `Running` with the same IPs as step 5 |

If one step fails, the problem is at that step.

---

## 19. Troubleshooting: Incorrect Service Selector

**Problem:** The Service selector does not match any pod label, so the Service has no pods behind it.

```text
Service selector: app=wrong-app
        |
        X   no pod has this label
        |
EndpointSlice: empty
```

`wrong-selector-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wrong-selector-svc
  namespace: production-labs
spec:
  type: ClusterIP
  selector:
    app: wrong-app          # wrong, no pod has this label
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f wrong-selector-service.yaml
kubectl get endpointslices -l kubernetes.io/service-name=wrong-selector-svc
kubectl exec lb-test-client -- wget -T 5 -qO- http://wrong-selector-svc
kubectl get pods --show-labels                                                  # real labels
```

Expected output (the problem):

```text
NAME                       ADDRESSTYPE   PORTS   ENDPOINTS   AGE
wrong-selector-svc-xxxxx   IPv4          80      <unset>     10s

wget: can't connect to remote host (10.x.x.x): Connection refused
```

`ENDPOINTS <unset>` means no pod is behind the Service.

**Fix:** change the selector to `app: clusterip-demo`, then:

```bash
kubectl apply -f wrong-selector-service.yaml
kubectl get endpointslices -l kubernetes.io/service-name=wrong-selector-svc
kubectl exec lb-test-client -- wget -T 5 -qO- http://wrong-selector-svc
```

Expected output after the fix: `ENDPOINTS` shows the two pod IPs, and `wget` prints the nginx page.

---

## 20. Troubleshooting: Incorrect Target Port

**Problem:** The Service finds the pods, but sends traffic to a port where nothing is listening.

```text
Service :80 --> pod IP :8080 --> nothing listening (nginx listens on 80)
```

`wrong-targetport-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wrong-targetport-svc
  namespace: production-labs
spec:
  type: ClusterIP
  selector:
    app: clusterip-demo     # correct, pods are found
  ports:
    - port: 80
      targetPort: 8080      # wrong, nginx listens on 80
```

```bash
kubectl apply -f wrong-targetport-service.yaml
kubectl get endpointslices -l kubernetes.io/service-name=wrong-targetport-svc
kubectl exec lb-test-client -- wget -T 5 -qO- http://wrong-targetport-svc
```

Expected output (the problem):

```text
NAME                         ADDRESSTYPE   PORTS   ENDPOINTS                          AGE
wrong-targetport-svc-xxxxx   IPv4          8080    10.244.x.x,10.244.x.x              10s

wget: can't connect to remote host (10.x.x.x): Connection refused
```

The pod IPs are there, but the port is `8080`. That is the clue.

**Fix:** set `targetPort: 80`, then:

```bash
kubectl apply -f wrong-targetport-service.yaml
kubectl exec lb-test-client -- wget -T 5 -qO- http://wrong-targetport-svc
```

Expected output after the fix: the nginx welcome page.

Difference from topic 19: with a wrong selector the EndpointSlice is empty. With a wrong target port the EndpointSlice has pod IPs, but the port is wrong.

---

## 21. Troubleshooting: DNS Lookup Failure

**Problem:** A pod cannot turn a Service name into an IP.

```text
Client pod --> /etc/resolv.conf --> CoreDNS --> Service IP
```

Check each step:

```bash
kubectl exec clusterip-client -- cat /etc/resolv.conf                 # nameserver is 10.96.0.10
kubectl exec clusterip-client -- nslookup clusterip-demo-svc          # gives the Service IP
kubectl exec clusterip-client -- nslookup clusterip-demo-svc.production-labs.svc.cluster.local
kubectl get pods -n kube-system -l k8s-app=kube-dns                   # CoreDNS is Running
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20
```

**Output (from my lab) when DNS works:**

```text
Name:	clusterip-demo-svc.production-labs.svc.cluster.local
Address: 10.96.114.183
```

Expected output when the Service name is wrong (for example `nslookup wrong-name`):

```text
** server can't find wrong-name.production-labs.svc.cluster.local: NXDOMAIN
** server can't find wrong-name.svc.cluster.local: NXDOMAIN
** server can't find wrong-name.cluster.local: NXDOMAIN
```

| What you see | Meaning |
|--------------|---------|
| The Service IP comes back | DNS is fine, the problem is somewhere else |
| `NXDOMAIN` for **every** try | Wrong Service name or wrong namespace |
| No answer or a timeout | CoreDNS is down or the pod cannot reach it |

`NXDOMAIN` for some search-domain tries is normal when one of the tries gives the IP. Only worry when the correct full name also fails.

---

## 22. Troubleshooting: TLS Configuration Error

**Problem:** HTTPS gives a certificate error or the default fake certificate.

```text
HTTPS client --> Ingress host: web.local
                      |
                      v
              Secret: web-local-tls
                      |
                      v
              clusterip-demo-svc :80 --> nginx pods
```

Check each part:

```bash
kubectl get secret web-local-tls                         # must exist, type kubernetes.io/tls
kubectl describe ingress web-tls-ingress                 # TLS host and secret name
kubectl get ingress web-tls-ingress -o yaml
openssl x509 -in web-local.crt -noout -ext subjectAltName    # SAN must contain web.local
curl -k -i --resolve web.local:443:192.168.49.2 https://web.local/
```

Expected output of `kubectl describe ingress web-tls-ingress` (TLS part):

```text
TLS:
  web-local-tls terminates web.local
Rules:
  Host        Path  Backends
  ----        ----  --------
  web.local
              /     clusterip-demo-svc:80 (10.244.x.x:80,10.244.x.x:80)
```

Expected output of `curl`: `HTTP/2 200` and the nginx page.

Checklist:

- The Secret exists in the **same namespace** as the Ingress.
- The Secret type is `kubernetes.io/tls`.
- `tls.hosts` contains `web.local`.
- `secretName` is `web-local-tls`.
- The certificate and key match (compare the two `sha256sum` hashes from topic 11).
- The backend Service and its EndpointSlice are correct.

If the Secret is missing or wrong, the controller uses its default fake certificate. Then `curl` without `-k` shows a certificate error.

---

## Quick troubleshooting table

| Symptom | First thing to check |
|---------|----------------------|
| Service has no endpoints | Service selector and pod labels |
| Connection refused | `targetPort`, the container port, pod readiness |
| DNS lookup fails | Service name, namespace, `/etc/resolv.conf`, CoreDNS |
| Ingress gives 404 | Host, path, Ingress class, rewrite annotation, backend Service |
| Ingress apply is rejected | Another Ingress already uses the same host and path |
| TLS error | Secret, certificate SAN, host name and Secret name |
| Request times out | A NetworkPolicy may be blocking it |
| LoadBalancer IP is `<pending>` | Is `minikube tunnel` running? |
| `exec` says pod is completed | The test pod's sleep ended, create it again |

---

## Cleanup

Remove the temporary test resources:

```bash
kubectl delete service wrong-selector-svc wrong-targetport-svc
kubectl delete networkpolicy deny-clusterip-ingress allow-lb-client
kubectl delete pod clusterip-client lb-test-client
```

This does not delete the main `clusterip-demo-svc` or the nginx pods.

---

## Part 3 Completion

All 22 networking topics were practised on minikube in the `production-labs` namespace: Services, DNS, CoreDNS, EndpointSlices, Ingress, TLS, NetworkPolicy, connectivity tests, request-flow tracing and 4 failure simulations.
