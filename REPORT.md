# Assignment 9 — Container Orchestration using Kubernetes and Helm

## Title
Explore the concept of Container Orchestration using Kubernetes, with Kubernetes
architecture and the key components of a Helm chart (Chart.yaml, values.yaml,
templates/). Helm commands: install, upgrade, uninstall, and template.

## Aim
To understand container orchestration and Kubernetes architecture, and to create and
operate a Helm chart (create, lint, template, install, upgrade, uninstall) on a real
local Kubernetes cluster.

## Objectives
1. Understand what container orchestration is and why it is needed.
2. Understand the Kubernetes architecture — control plane and worker node components.
3. Create a Helm chart and understand its three key parts: `Chart.yaml`, `values.yaml`,
   and `templates/`.
4. Practically run the core Helm commands (`lint`, `template`, `install`, `upgrade`,
   `history`, `uninstall`) and verify every result on a live cluster.

## Theory

### Container Orchestration
A container packages an app with its dependencies so it runs the same everywhere.
Running one container is simple; running many containers, across many machines,
reliably, is not. Container orchestration automates:
- **Deployment** — rolling out the desired number of app copies.
- **Scaling** — adding/removing copies as load changes.
- **Networking & service discovery** — containers can find and talk to each other via
  stable names even though their IPs change.
- **Load balancing** — traffic is spread across healthy replicas.
- **Self-healing** — crashed containers/nodes are automatically replaced.

Kubernetes is the most widely used container orchestrator.

### Kubernetes Architecture
Kubernetes is split into:
- **Control plane**: `kube-apiserver` (entry point for all operations), `etcd`
  (stores all cluster state), `kube-scheduler` (assigns Pods to nodes),
  `kube-controller-manager` (reconciles desired vs actual state).
- **Worker node**: `kubelet` (node agent that starts containers), `kube-proxy`
  (networking/load-balancing rules), the container runtime (e.g. containerd), and the
  **Pods** themselves.

Full diagram, component table mapped to the real cluster, and the Deployment-creation
walkthrough are in `docs/architecture.md`.

### Helm Chart Components
Helm is the package manager for Kubernetes. A chart has three key parts:
- **Chart.yaml** — metadata about the chart: its name, version, the app version it
  packages, and whether it is an `application` or `library` chart.
- **values.yaml** — the default configuration values (replica count, image, service
  type, etc.) that templates read from. These can be overridden at install/upgrade
  time with `--set` or a custom values file, without touching the chart itself.
- **templates/** — Kubernetes YAML manifests written with Go template syntax
  (`{{ .Values.xxx }}`) that get filled in with real values from `values.yaml` (and
  built-in objects like `.Release`, `.Chart`) to produce the final manifests Kubernetes
  applies.

Full field-by-field explanation is in `docs/architecture.md` (chart fields) and
`docs/values-to-template.md` (values → template → rendered output examples).

## Procedure

1. Verified `docker`, `kubectl`, `helm` were installed (versions: Docker 29.6.2,
   kubectl client v1.36.1, Helm v4.3.0).
2. No cluster was running, so created one with `kind create cluster --name assignment9`
   (minikube was not installed on this machine, kind was used instead — both are valid
   local single-node cluster tools).
3. Created namespace `demo` with `kubectl create namespace demo`.
4. Inspected the real cluster: `kubectl get nodes -o wide`, `kubectl get pods -n
   kube-system -o wide`, `kubectl cluster-info`.
5. Ran `helm create webapp` to scaffold the chart, then edited:
   - `Chart.yaml`: description → "Sample web application for Assignment 9",
     appVersion → "2.4" (apiVersion v2, type application, version 0.1.0 were already
     correct from `helm create`).
   - `values.yaml`: `replicaCount: 3`, `image.repository: httpd`, `image.tag: "2.4"`.
     `service.type: ClusterIP` and `service.port: 80` were already the chart defaults.
6. Ran `helm lint ./webapp` — passed with only an informational note (missing icon,
   not an error).
7. Rendered the chart twice with `helm template`: once with defaults, once with
   `--set replicaCount=5 --set image.tag=2.4.62`, saved both outputs, and diffed them.
8. Installed the chart with `helm install webrelease ./webapp -n demo`, waited for
   pods to become Ready, then verified pods/deployments/services.
9. Port-forwarded the service and used `curl` to confirm the app actually responds.
10. Upgraded the release with `helm upgrade ... --set replicaCount=2`, watched pods
    scale down, and checked `helm history`.
11. Uninstalled the release, confirmed it was gone from `helm list` and `kubectl get
    all`, then deleted the `demo` namespace.

## Observations (real captured output)

### helm lint
```
==> Linting ./webapp
[INFO] Chart.yaml: icon is recommended
1 chart(s) linted, 0 chart(s) failed
```

### helm template diff (proves values.yaml / --set change the rendered manifest)
```
51c51
<   replicas: 3
---
>   replicas: 5
68c68
<           image: "httpd:2.4"
---
>           image: "httpd:2.4.62"
```

### helm install
```
NAME: webrelease
NAMESPACE: demo
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
```

### kubectl get pods,deployments,services -n demo (after install)
```
pod/webrelease-webapp-545774697c-dk9rk   1/1   Running   0   21s
pod/webrelease-webapp-545774697c-jrh9s   1/1   Running   0   21s
pod/webrelease-webapp-545774697c-wp8jd   1/1   Running   0   21s

deployment.apps/webrelease-webapp   3/3   3   3   21s   httpd:2.4

service/webrelease-webapp   ClusterIP   10.96.20.75   <none>   80/TCP   21s
```

### curl localhost:8080 (via kubectl port-forward)
```
HTTP/1.1 200 OK
Server: Apache/2.4.69 (Unix)
Content-Type: text/html

<title>It works! Apache httpd</title>
<p>It works!</p>
```
This confirms the app deployed by the Helm chart was actually reachable, not just
"Running" on paper.

### helm upgrade --set replicaCount=2
```
Release "webrelease" has been upgraded. Happy Helming!
REVISION: 2
STATUS: deployed
```
Pods observed scaling down from 3 → 2 (one pod seen in `Terminating` state right after
the upgrade, then settled at exactly 2 `Running` pods).

### helm history
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         deployed    Upgrade complete
```

### helm uninstall
```
release "webrelease" uninstalled
```
`helm list -n demo` afterward returned no releases, and `kubectl get all -n demo`
showed the pods in `Terminating` and then gone — confirming full cleanup. The `demo`
namespace was then deleted with `kubectl delete namespace demo`.

Full raw output for every command above is saved in the `outputs/` folder.

## Conclusion
This assignment demonstrated, on a real local Kubernetes cluster (created with
`kind`), how Kubernetes orchestrates containers through its control plane (API
server, etcd, scheduler, controller manager) and worker node components (kubelet,
kube-proxy, container runtime). It also showed how Helm packages Kubernetes manifests
into a reusable chart using `Chart.yaml` (metadata), `values.yaml` (configurable
settings), and `templates/` (Go-templated manifests), and how the core Helm workflow
(`lint` → `template` → `install` → `upgrade` → `uninstall`) manages an application's
full lifecycle on the cluster.

## Learning Outcome
- Understood why container orchestration is needed and how Kubernetes solves
  deployment, scaling, networking, and self-healing problems automatically.
- Learned the role of each Kubernetes control-plane and worker-node component by
  observing them as real running pods/processes on a live cluster.
- Learned how a Helm chart is structured and how `values.yaml` feeds into
  `templates/` to produce real Kubernetes manifests, including how `--set` overrides
  defaults at install/upgrade time.
- Got hands-on experience with the full Helm release lifecycle: install, verify,
  upgrade (and see a rolling scale change happen), check history, and uninstall
  cleanly.
