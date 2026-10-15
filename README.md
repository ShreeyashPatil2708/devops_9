# Assignment 9 — Kubernetes Container Orchestration & Helm

This folder contains a complete, hands-on exploration of Kubernetes architecture and
Helm charts, done on a real local cluster (not simulated).

## What's inside

```
assignment9-k8s-helm/
├── README.md                     <- this file
├── REPORT.md                     <- lab write-up (Aim, Theory, Procedure, Observations, Conclusion)
├── docs/
│   ├── architecture.md           <- Kubernetes architecture theory + diagram + real cluster mapping
│   └── values-to-template.md     <- shows values.yaml -> templates/ -> rendered YAML
├── outputs/                       <- real terminal output captured from every command
│   ├── step1-architecture.txt
│   ├── step3-1-helm-lint.txt
│   ├── rendered-default.yaml
│   ├── rendered-override.yaml
│   ├── step3-2-template-diff.txt
│   ├── step3-3-install.txt
│   ├── step3-4-helm-list.txt
│   ├── step3-5-verify.txt
│   ├── step3-6-curl.txt
│   ├── step3-7-upgrade.txt
│   ├── step3-8-history.txt
│   └── step3-9-uninstall.txt
└── webapp/                        <- the Helm chart created for this assignment
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
```

## Environment used

- Docker Desktop (for the container runtime)
- `kind` (Kubernetes-in-Docker) to run a local 1-node cluster named `assignment9`
- `kubectl` v1.36 (client)
- `helm` v4.3.0

`minikube` was not installed, so `kind` was used instead (both are valid local cluster
tools for this kind of assignment).

## How to reproduce this yourself

```bash
# 1. Create a local cluster
kind create cluster --name assignment9
kubectl get nodes
kubectl create namespace demo

# 2. Create the Helm chart
helm create webapp
# then edit webapp/Chart.yaml and webapp/values.yaml as described in docs/

# 3. Validate and render the chart
helm lint ./webapp
helm template webrelease ./webapp -n demo > outputs/rendered-default.yaml
helm template webrelease ./webapp -n demo --set replicaCount=5 --set image.tag=2.4.62 > outputs/rendered-override.yaml
diff outputs/rendered-default.yaml outputs/rendered-override.yaml

# 4. Install and test
helm install webrelease ./webapp -n demo
kubectl get pods,deployments,services -n demo
kubectl port-forward svc/webrelease-webapp 8080:80 -n demo &
curl localhost:8080
kill %1

# 5. Upgrade and inspect history
helm upgrade webrelease ./webapp -n demo --set replicaCount=2
kubectl get pods -n demo
helm history webrelease -n demo

# 6. Clean up
helm uninstall webrelease -n demo
kubectl delete namespace demo
kind delete cluster --name assignment9   # optional, removes the whole cluster
```

## Key things changed in the chart

- `Chart.yaml`: `description` set to "Sample web application for Assignment 9",
  `appVersion` set to `"2.4"` (rest left as `helm create` default: apiVersion v2,
  type application, version 0.1.0).
- `values.yaml`: `replicaCount: 3`, `image.repository: httpd`, `image.tag: "2.4"`,
  `service.type: ClusterIP`, `service.port: 80` (ClusterIP/80 were already the chart's
  defaults, so no edit was needed there).

See `docs/architecture.md` and `docs/values-to-template.md` for the full explanation,
and `REPORT.md` for the complete lab write-up with observations.

## Full walkthrough

Run these commands **in order**, from this folder:
```bash
cd /Users/shreeyashpatil/clg/devops/ass9/assignment9-k8s-helm
```

Make sure you're pointed at the right cluster first (this assignment uses a cluster
named `assignment9`, not any other cluster you may have for other assignments):
```bash
kubectl config use-context kind-assignment9
```

### Part 1 — Prove the cluster and tools exist

**1.1 Show tool versions**
```bash
docker version
kubectl version --client
helm version
```
These are the three tools involved — Docker runs containers, kubectl talks to
Kubernetes, Helm packages and installs applications on Kubernetes.

**1.2 Show the cluster is alive**
```bash
kubectl cluster-info
kubectl get nodes -o wide
```
This is a local 1-node Kubernetes cluster running inside Docker, created with `kind`.

**1.3 Show Kubernetes' own internal components**
```bash
kubectl get pods -n kube-system -o wide
```
These pods ARE Kubernetes' control plane — API server, etcd, scheduler, controller
manager, plus kube-proxy and the DNS service (coredns). They're the architecture,
running live.

### Part 2 — Show the Helm chart

**2.1 Create the namespace**
```bash
kubectl create namespace demo
```

**2.2 Show the chart's three key files**
```bash
cat webapp/Chart.yaml
cat webapp/values.yaml
ls webapp/templates/
```
Chart.yaml is metadata about the chart. values.yaml holds the configurable settings —
3 replicas, httpd image, ClusterIP service on port 80. templates/ has the actual
Kubernetes YAML files that use `{{ .Values.x }}` to pull from values.yaml.

**2.3 Validate the chart**
```bash
helm lint ./webapp
```
This checks the chart for mistakes before using it. 0 charts failed = it's valid.

**2.4 Preview the rendered YAML (no install yet)**
```bash
helm template webrelease ./webapp -n demo | less
```
Press `q` to exit `less`. This shows exactly what Kubernetes YAML Helm will send,
without actually deploying anything — a dry run.

**2.5 Prove `--set` overrides values.yaml**
```bash
helm template webrelease ./webapp -n demo > /tmp/default.yaml
helm template webrelease ./webapp -n demo --set replicaCount=5 --set image.tag=2.4.62 > /tmp/override.yaml
diff /tmp/default.yaml /tmp/override.yaml
```
The diff shows replicas changed 3 to 5 and the image tag changed — proving
command-line flags override the chart's defaults.

### Part 3 — Install and test the app

**3.1 Install**
```bash
helm install webrelease ./webapp -n demo
```

**3.2 Confirm it's running**
```bash
helm list -n demo
kubectl get pods,deployments,services -n demo
```
3/3 pods are Running, matching `replicaCount: 3` from values.yaml.

**3.3 Actually test the app (not just "Running" on paper)**
```bash
kubectl port-forward svc/webrelease-webapp 8080:80 -n demo &
sleep 2
curl -i localhost:8080
kill %1
```
A real HTTP 200 response from the Apache httpd server proves the app is actually
reachable through the Kubernetes Service, not just listed as running.

### Part 4 — Upgrade and roll back visibility

**4.1 Upgrade the release (scale down to 2 replicas)**
```bash
helm upgrade webrelease ./webapp -n demo --set replicaCount=2
kubectl get pods -n demo
```
Helm upgrade changed the running release without deleting it — Kubernetes scaled 3
pods down to 2 automatically.

**4.2 Show revision history**
```bash
helm history webrelease -n demo
```
Helm keeps a history of every revision — revision 1 was the install, revision 2 is
this upgrade. This is also what lets you `helm rollback` if something breaks.

### Part 5 — Clean removal

**5.1 Uninstall**
```bash
helm uninstall webrelease -n demo
helm list -n demo
kubectl get all -n demo
```
Uninstall removes every Kubernetes object the chart created — pods, deployment,
service — in one command.

**5.2 Delete the namespace**
```bash
kubectl delete namespace demo
```

### What if a value changes?

Live-edit `webapp/values.yaml` (e.g. change `replicaCount: 3` to `replicaCount: 4`),
save, then re-run:
```bash
helm template webrelease ./webapp -n demo | grep replicas
```
to show the rendered output picks up the new value immediately — no need to touch any
template file.

### Full teardown

Only run this once you are completely done and don't need the cluster anymore — it
deletes the entire local Kubernetes cluster and frees up the Docker resources it was
using:
```bash
kind delete cluster --name assignment9
```
