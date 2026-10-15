# Kubernetes Architecture — Assignment 9

## 1. What is Container Orchestration and why do we need it?

A **container** packages an application with everything it needs to run (code, runtime,
libraries). Running one container by hand is easy. Running hundreds of containers across
many machines, making sure they stay healthy, can talk to each other, and can handle more
traffic — that is hard to do by hand. **Container orchestration** is the automated
management of containers at scale. A container orchestrator (like Kubernetes) takes care
of:

- **Deployment** — you describe "I want 3 copies of my app running" and the orchestrator
  starts them for you, without you manually running `docker run` three times.
- **Scaling** — if traffic increases, you can ask for more replicas (or it can be done
  automatically) and the orchestrator spreads the extra copies across the available
  machines.
- **Networking** — containers get IP addresses and can reach each other by name, even
  though they move between machines and restart often.
- **Service discovery** — other parts of the app (or outside users) can find a group of
  pods through a stable name/IP (a **Service**) instead of tracking each pod's changing
  IP address.
- **Load balancing** — incoming traffic is automatically spread across all healthy
  replicas of an application.
- **Self-healing** — if a container crashes or a machine dies, the orchestrator notices
  and automatically starts a replacement so the desired number of replicas is always
  running.

In short: orchestration lets a student (or a company) describe the *desired state*
("run 3 copies of httpd, keep them healthy, expose them on port 80") and the system keeps
making reality match that desired state, without a human watching it 24/7.

## 2. Kubernetes Architecture

Kubernetes has two main parts: the **Control Plane** (the "brain", usually on
master/control-plane nodes) and **Worker Nodes** (where the actual application
containers run).

```
                         CONTROL PLANE
        ┌───────────────────────────────────────────────┐
        │                                                 │
        │   ┌───────────┐   ┌────────────┐   ┌─────────┐ │
        │   │ API Server│──▶│    etcd    │   │Scheduler│ │
        │   │(kube-apiserver)│ (key-value │   │         │ │
        │   └─────┬─────┘   │   store)   │   └────┬────┘ │
        │         │          └────────────┘        │      │
        │         │        ┌───────────────────┐   │      │
        │         └───────▶│ Controller Manager │◀──┘      │
        │                  └───────────────────┘          │
        └───────────────────────────────────────────────┘
                              │
                 (watches / schedules / commands)
                              │
        ┌─────────────────────────────────────────────────┐
        │                  WORKER NODE                     │
        │   ┌─────────┐   ┌───────────┐   ┌─────────────┐  │
        │   │ kubelet │   │kube-proxy │   │  Container   │  │
        │   │         │   │           │   │   Runtime    │  │
        │   └────┬────┘   └─────┬─────┘   │ (containerd) │  │
        │        │              │          └──────┬───────┘  │
        │        └──────────────┴─────────────────┘          │
        │                      │                             │
        │              ┌───────▼────────┐                    │
        │              │  Pod   Pod  Pod │  (your containers) │
        │              └────────────────┘                    │
        └─────────────────────────────────────────────────┘
```

### Control plane components
- **API Server (`kube-apiserver`)** — the front door of Kubernetes. Every command
  (`kubectl`, Helm, dashboards) talks to the API server. It validates requests and
  stores the desired state.
- **etcd** — a small, reliable key-value database. This is where the *entire cluster
  state* (all objects: pods, deployments, services, configs) is actually stored.
- **Scheduler (`kube-scheduler`)** — decides *which worker node* a new Pod should run
  on, based on resources available, constraints, etc.
- **Controller Manager (`kube-controller-manager`)** — runs background "controllers"
  that constantly watch the actual state vs desired state and fix differences (e.g. if a
  Deployment says "3 replicas" but only 2 are running, it creates one more).

### Worker node components
- **kubelet** — the agent running on every worker node. It talks to the API server,
  receives instructions ("run this pod here"), and tells the container runtime to start
  containers. It also reports node/pod health back.
- **kube-proxy** — manages networking rules on each node so that Services can route
  traffic to the correct Pods (load balancing at the network level).
- **Container runtime (containerd)** — the software that actually pulls images and runs
  containers (the lowest-level piece that `docker run` normally does).
- **Pods** — the smallest deployable unit in Kubernetes; one or more containers that
  share network/storage, scheduled together on a node.

## 3. Mapping to my actual cluster

My cluster was created locally using **kind** (Kubernetes-in-Docker), which runs a
single-node cluster inside a Docker container named `assignment9-control-plane`. On a
kind cluster, control-plane and worker roles run on the *same* node, but you can still
see every component as a separate Pod/process:

| Component | Purpose | What I actually saw on my cluster |
|---|---|---|
| API Server | Front door for all cluster operations | Pod `kube-apiserver-assignment9-control-plane` (Running) |
| etcd | Stores all cluster state | Pod `etcd-assignment9-control-plane` (Running) |
| Scheduler | Decides which node runs a new Pod | Pod `kube-scheduler-assignment9-control-plane` (Running) |
| Controller Manager | Keeps actual state == desired state | Pod `kube-controller-manager-assignment9-control-plane` (Running) |
| kubelet | Node agent that starts/stops containers | Not a pod — runs as a process directly on the `assignment9-control-plane` node (kubelet manages the node itself, so it doesn't appear in `kubectl get pods`) |
| kube-proxy | Networking rules / load balancing on the node | Pod `kube-proxy-vbcr4` (Running) |
| Container runtime | Actually runs containers | `containerd://2.3.4` (seen in `kubectl get nodes -o wide` under CONTAINER-RUNTIME) |
| CNI (pod networking) | Gives pods IP addresses and lets them talk to each other | Pod `kindnet-dbtzg` (Running) — kind's built-in CNI plugin |
| DNS / Service discovery | Lets pods find each other by name | Pods `coredns-559f6c778d-*` (2 replicas, Running) |

Raw command output used for this table is saved in `outputs/step1-architecture.txt`.

## 4. What happens when I create a Deployment? (step-by-step)

When I run something like `helm install webrelease ./webapp -n demo` (which creates a
Deployment behind the scenes), this is the real sequence of events:

1. **kubectl/Helm → API Server**: The client sends the Deployment YAML as an HTTP
   request to the `kube-apiserver`.
2. **API Server validates & writes to etcd**: The API server checks the request is
   valid, then saves the new Deployment object into `etcd` as the new *desired state*.
3. **Controller Manager notices**: The Deployment controller (inside
   `kube-controller-manager`) is watching the API server for changes. It sees the new
   Deployment and creates the matching **ReplicaSet**, which in turn creates the
   requested number of **Pod** objects (still just entries in etcd at this point — no
   containers running yet).
4. **Scheduler assigns a node**: The `kube-scheduler` sees the new Pods have no node
   assigned yet. It picks a suitable worker node for each Pod and writes that decision
   back to the API server (and etcd).
5. **kubelet picks it up**: The `kubelet` on the chosen node is watching the API server
   for Pods assigned to its node. Once it sees a new Pod, it asks the **container
   runtime (containerd)** to pull the image and start the container(s).
6. **kube-proxy updates networking**: As Pods come up, `kube-proxy` updates the node's
   networking rules so the Service can route traffic to the new Pod IPs.
7. **Status flows back**: kubelet reports Pod status (Running, Ready) back to the API
   server, which updates etcd — so `kubectl get pods` shows the real current state.

This loop (watch → compare desired vs actual → act) is the core idea behind every
Kubernetes controller, and it's what makes self-healing and scaling automatic.
