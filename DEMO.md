# Live Demo Script — Assignment 9 (Kubernetes + Helm)

Run these commands **in order**, in this folder:
```bash
cd /Users/shreeyashpatil/clg/devops/ass9/assignment9-k8s-helm
```

Before starting, make sure you're pointed at the right cluster (this assignment uses
a cluster named `assignment9`, not any other cluster you may have for other
assignments):
```bash
kubectl config use-context kind-assignment9
```

---

## Part 1 — Prove the cluster and tools exist

**1.1 Show tool versions**
```bash
docker version
kubectl version --client
helm version
```
*Say: "These are the three tools involved — Docker runs containers, kubectl talks to
Kubernetes, Helm packages and installs applications on Kubernetes."*

**1.2 Show the cluster is alive**
```bash
kubectl cluster-info
kubectl get nodes -o wide
```
*Say: "This is a local 1-node Kubernetes cluster running inside Docker, created with
`kind`."*

**1.3 Show Kubernetes' own internal components**
```bash
kubectl get pods -n kube-system -o wide
```
*Say: "These pods ARE Kubernetes' control plane — API server, etcd, scheduler,
controller manager, plus kube-proxy and the DNS service (coredns). They're the
architecture, running live."*

---

## Part 2 — Show the Helm chart

**2.1 Create the namespace for this demo**
```bash
kubectl create namespace demo
```

**2.2 Show the chart's three key files**
```bash
cat webapp/Chart.yaml
cat webapp/values.yaml
ls webapp/templates/
```
*Say: "Chart.yaml is metadata about the chart. values.yaml holds the configurable
settings — 3 replicas, httpd image, ClusterIP service on port 80. templates/ has the
actual Kubernetes YAML files that use `{{ .Values.x }}` to pull from values.yaml."*

**2.3 Validate the chart**
```bash
helm lint ./webapp
```
*Say: "This checks the chart for mistakes before we use it. 0 charts failed = it's
valid."*

**2.4 Preview the rendered YAML (no install yet)**
```bash
helm template webrelease ./webapp -n demo | less
```
Press `q` to exit `less`. *Say: "This shows exactly what Kubernetes YAML Helm will
send, without actually deploying anything — a dry run."*

**2.5 Prove `--set` overrides values.yaml**
```bash
helm template webrelease ./webapp -n demo > /tmp/default.yaml
helm template webrelease ./webapp -n demo --set replicaCount=5 --set image.tag=2.4.62 > /tmp/override.yaml
diff /tmp/default.yaml /tmp/override.yaml
```
*Say: "The diff shows replicas changed 3 to 5 and the image tag changed — proving
command-line flags override the chart's defaults."*

---

## Part 3 — Install and test the app

**3.1 Install**
```bash
helm install webrelease ./webapp -n demo
```

**3.2 Confirm it's running**
```bash
helm list -n demo
kubectl get pods,deployments,services -n demo
```
*Say: "3/3 pods are Running, matching replicaCount: 3 from values.yaml."*

**3.3 Actually test the app (not just "Running" on paper)**
```bash
kubectl port-forward svc/webrelease-webapp 8080:80 -n demo &
sleep 2
curl -i localhost:8080
kill %1
```
*Say: "A real HTTP 200 response from the Apache httpd server proves the app is
actually reachable through the Kubernetes Service, not just listed as running."*

---

## Part 4 — Upgrade and roll back visibility

**4.1 Upgrade the release (scale down to 2 replicas)**
```bash
helm upgrade webrelease ./webapp -n demo --set replicaCount=2
kubectl get pods -n demo
```
*Say: "Helm upgrade changed the running release without deleting it — Kubernetes
scaled 3 pods down to 2 automatically."*

**4.2 Show revision history**
```bash
helm history webrelease -n demo
```
*Say: "Helm keeps a history of every revision — revision 1 was the install, revision 2
is this upgrade. This is also what lets you `helm rollback` if something breaks."*

---

## Part 5 — Clean removal

**5.1 Uninstall**
```bash
helm uninstall webrelease -n demo
helm list -n demo
kubectl get all -n demo
```
*Say: "Uninstall removes every Kubernetes object the chart created — pods, deployment,
service — in one command."*

**5.2 Delete the namespace**
```bash
kubectl delete namespace demo
```

---

## If faculty asks "what if I change a value?"
Live-edit `webapp/values.yaml` (e.g. change `replicaCount: 3` to `replicaCount: 4`),
save, then re-run:
```bash
helm template webrelease ./webapp -n demo | grep replicas
```
to show the rendered output picks up the new value immediately — no need to touch any
template file.

---

## Full teardown (after the demo is fully done)
Only run this once you are completely done and don't need the cluster anymore — it
deletes the entire local Kubernetes cluster and frees up the Docker resources it was
using:
```bash
kind delete cluster --name assignment9
```
