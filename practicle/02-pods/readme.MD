# 02 — Pods 🐳

> The smallest deployable unit in Kubernetes.
> K8s never runs containers directly — always wraps them in a Pod.

---

## 📁 Folder Structure

```
02-pods/
├── pod.yaml          # Basic nginx pod
├── crash-pod.yaml    # Pod that crashes — shows CrashLoopBackOff
├── multi-pod.yaml    # Two containers sharing same network
└── README.md
```

---

## 🧠 Why Pods Exist — Not Just Containers?

You already know Docker containers. So why does K8s wrap them in a Pod?

```
Docker world:              K8s world:
──────────────             ──────────────
docker run nginx    →      Pod → contains → nginx container
```

A Pod gives the container:

```
✅ A unique IP address inside the cluster
✅ Shared storage (volumes between containers)
✅ Shared network (containers in same pod share localhost)
✅ Lifecycle management (restart policy)
✅ Labels for selection by Services
```

Real world analogy:
```
Container = a person
Pod       = that person's office

The office has:
- Its own address (IP)
- Shared resources (printer, wifi)
- Multiple people can share one office (multi-container pod)
```

---

## 📄 Files

### `pod.yaml` — Basic nginx Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: default
  labels:
    app: nginx
    environment: dev
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
```

**Why each field:**

| Field | Purpose |
|-------|---------|
| `apiVersion: v1` | Pod is a core K8s object |
| `kind: Pod` | What we're creating |
| `metadata.name` | Unique name in this namespace |
| `metadata.labels` | Key-value tags — Services use these to find pods |
| `spec.containers` | List of containers inside the pod |
| `image` | Same Docker image you already know |
| `ports` | Documents which port the container listens on |

---

### `crash-pod.yaml` — Pod that crashes intentionally

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
spec:
  containers:
  - name: crash
    image: busybox
    command: ["sh", "-c", "echo Pod started && sleep 5 && exit 1"]
```

---

### `multi-pod.yaml` — Two containers in one Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
  - name: logger
    image: busybox
    command: ["sh", "-c", "while true; do echo logging at $(date); sleep 5; done"]
```

---

## 🔧 All Commands

```bash
# ── Create ───────────────────────────────────────────────
kubectl apply -f pod.yaml

# ── View ─────────────────────────────────────────────────
kubectl get pods                        # basic list
kubectl get pods -o wide                # shows IP + node
kubectl get pods -w                     # watch live updates
kubectl describe pod nginx-pod          # full details + events
kubectl get pod nginx-pod -o yaml       # full YAML output
kubectl get pod nginx-pod -o json       # full JSON output

# ── Interact ─────────────────────────────────────────────
kubectl exec -it nginx-pod -- sh                    # shell into pod
kubectl exec -it multi-pod -c nginx -- sh           # shell into specific container
kubectl logs nginx-pod                              # view logs
kubectl logs -f nginx-pod                           # follow live logs
kubectl logs multi-pod -c nginx                     # logs of specific container
kubectl logs multi-pod -c logger                    # logs of logger container

# ── Delete ───────────────────────────────────────────────
kubectl delete pod nginx-pod
kubectl delete -f pod.yaml
kubectl delete pod nginx-pod crash-pod multi-pod    # delete multiple
```

---

## 🔍 What kubectl describe Shows

```
Name:         nginx-pod
Namespace:    default
Labels:       app=nginx
              environment=dev
Status:       Running
IP:           10.244.0.9          ← pod's IP inside cluster
Node:         minikube            ← which node it runs on

Containers:
  nginx:
    Image:    nginx:alpine
    Port:     80/TCP

Events:
  Normal  Scheduled  → Successfully assigned to minikube
  Normal  Pulled     → Image already present on machine
  Normal  Created    → Container created
  Normal  Started    → Container started
```

---

## 💥 CrashLoopBackOff — What it means

```
Pod crashes → K8s restarts it
Crashes again → K8s restarts it
Crashes again → K8s waits longer before restarting
Crashes again → K8s waits even longer...

This is CrashLoopBackOff:
"I keep restarting this but it keeps dying
I'll wait longer between each restart
But I'll NEVER give up trying"
```

Seen live in this drill:
```
crash-pod    RESTARTS: 7    ← K8s restarted 7 times automatically
crash-pod1   RESTARTS: 3
```

**This is exactly why Deployment exists** — to manage pods that need
to stay healthy at scale with proper update strategies.

---

## 🔀 Multi-container Pod — How containers share network

```
multi-pod
├── nginx container   (port 80)
└── logger container

Both containers share:
✅ Same IP address
✅ Same localhost
✅ Same volumes

nginx can call logger via: localhost:port
logger can call nginx via: localhost:80
```

Verified by running inside nginx container:
```bash
wget -O- localhost
# Returns nginx HTML page ✅
```

---

## 💡 Key Takeaways

1. **Pod = wrapper around containers** — never run containers directly in K8s
2. **Every pod gets its own IP** — unique inside the cluster
3. **CrashLoopBackOff** = pod keeps crashing, K8s keeps retrying with backoff
4. **Multi-container pods** share network and storage — use for sidecar patterns
5. **Labels are critical** — Services and Deployments use them to find pods
6. **Pods alone have no self-healing at scale** — that's why Deployment exists

---

## 🐳 Docker → K8s Translation

| Docker | Kubernetes |
|--------|-----------|
| `docker run nginx` | `kubectl apply -f pod.yaml` |
| `docker exec -it container bash` | `kubectl exec -it pod -- sh` |
| `docker logs container` | `kubectl logs pod` |
| `docker inspect container` | `kubectl describe pod` |
| `docker ps` | `kubectl get pods` |
| `docker stop container` | `kubectl delete pod` |

---

## 🏋️ Exercises Completed

- [x] Create nginx-pod — note IP with `kubectl get pods -o wide`
- [x] Shell into pod — run `wget -O- localhost` — got nginx response ✅
- [x] Read Events section in `kubectl describe pod`
- [x] Create crash-pod — watched 7 restarts with `-w` flag
- [x] Create multi-pod — checked logs of each container separately
- [x] Found `status.phase: Running` in YAML output

---

*Status: ✅ Complete*
*Next: 03 — Deployments*
*Platform: macOS · minikube v1.38.1 · Kubernetes v1.35.1*