# What is this project? (Explain-to-anyone version)

## In one line
This project shows how **Kubernetes** manages containers automatically, and how
**Helm** (a packaging tool for Kubernetes) is used to install, upgrade, and remove an
application on a Kubernetes cluster.

## Why does this exist?
Running one container is easy (`docker run ...`). But real applications need many
copies of a container running at once, need to survive crashes, need to scale up/down,
and need a stable address so other services can find them. Doing all of this by hand is
painful — **Kubernetes** does it automatically. **Helm** then makes it easy to package,
configure, and release an application on Kubernetes using one command instead of
applying many YAML files by hand.

## What did I actually build?
1. A local Kubernetes cluster (using `kind`, which runs Kubernetes inside Docker on my
   own laptop — no cloud account needed).
2. A Helm chart called `webapp` that describes: "run 3 copies of the Apache httpd web
   server, and expose them through a Service on port 80."
3. I ran the full Helm lifecycle on that chart — checked it for errors, previewed the
   exact YAML it would generate, installed it, tested that it actually serves a web
   page, upgraded it to change the replica count, checked its revision history, and
   finally removed it cleanly.

## What do the key pieces mean?

| Term | What it means here |
|---|---|
| **Cluster** | A group of machines (here, just 1, via `kind`) managed by Kubernetes |
| **Node** | A machine (worker) in the cluster that runs containers |
| **Pod** | The smallest unit Kubernetes runs — one or more containers running together |
| **Deployment** | Tells Kubernetes "keep N copies of this Pod running at all times" |
| **Service** | A stable network address that routes traffic to the right Pods |
| **Namespace** | A way to separate resources into folders (`demo` is where my app lives) |
| **Helm Chart** | A packaged template of Kubernetes YAML files for one application |
| **Chart.yaml** | Metadata about the chart (name, version, description) |
| **values.yaml** | The configurable settings (replica count, image, port, etc.) |
| **templates/** | The actual Kubernetes YAML files, with `{{ .Values.x }}` placeholders filled in from values.yaml |
| **Release** | One running instance of a Helm chart installed on the cluster (`webrelease`) |

## What does each Helm command do?

| Command | What it does |
|---|---|
| `helm create webapp` | Scaffolds a brand-new chart with default files |
| `helm lint ./webapp` | Checks the chart for mistakes before using it |
| `helm template ... ` | Renders the final Kubernetes YAML *without* installing anything — a "dry run preview" |
| `helm install webrelease ./webapp -n demo` | Actually deploys the chart to the cluster as a release named `webrelease` |
| `helm list -n demo` | Shows installed releases in that namespace |
| `helm upgrade webrelease ./webapp -n demo --set replicaCount=2` | Changes a running release's settings (here, drops from 3 to 2 copies) without deleting it first |
| `helm history webrelease -n demo` | Shows every past revision of this release (install, upgrade, etc.) |
| `helm uninstall webrelease -n demo` | Removes the release and all the Kubernetes objects it created |

## What do the key kubectl commands do?

| Command | What it does |
|---|---|
| `kubectl get nodes` | Lists the machines in the cluster |
| `kubectl get pods -n kube-system` | Shows Kubernetes' own internal components running as pods |
| `kubectl create namespace demo` | Creates an isolated "folder" for this assignment's resources |
| `kubectl get pods,deployments,services -n demo` | Shows what's running inside that namespace |
| `kubectl port-forward svc/webrelease-webapp 8080:80 -n demo` | Temporarily tunnels my laptop's port 8080 to the Service's port 80, so I can test it with `curl` |
| `kubectl delete namespace demo` | Deletes everything inside that namespace in one shot |

## Where is everything?
- `docs/architecture.md` — full Kubernetes architecture theory + diagram
- `docs/values-to-template.md` — shows values.yaml turning into real YAML
- `outputs/` — the actual terminal output captured from every command
- `webapp/` — the Helm chart itself
- `REPORT.md` — full lab write-up (Aim, Theory, Procedure, Observations, Conclusion)
- `DEMO.md` — step-by-step script to re-run everything live, e.g. in front of faculty
