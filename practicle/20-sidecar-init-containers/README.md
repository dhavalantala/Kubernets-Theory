# 20 — Sidecar / Init Containers 🔧

> Two patterns for multi-container pods — one sequential and blocking,
> one concurrent and permanent — each solving a genuinely different problem.

---

## 📁 Folder Structure

```
20-sidecar-init-containers/
├── init-demo.yaml       # init container writes file, main reads it
├── init-fail.yaml       # failing init blocks main permanently
├── sidecar-demo.yaml    # log-shipper sidecar alongside main app
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why These Exist — Two Different Problems

```
Init Container
  = runs BEFORE the main container starts
  = must complete SUCCESSFULLY (exit 0) before main is allowed to begin
  = used for: setup, waiting for a dependency, schema migrations

Sidecar Container
  = runs ALONGSIDE the main container, for the pod's entire lifetime
  = shares the same network and volumes
  = used for: log shipping, proxies, metrics collection

Key difference:
  Init   → sequential, blocking, temporary
  Sidecar → concurrent, permanent
```

---

## 📄 Files

### `init-demo.yaml` — sequential setup via shared volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
  namespace: dev-backend
  labels:
    app: init-demo
spec:
  initContainers:
  - name: init-setup
    image: busybox
    command: ["sh", "-c", "echo 'Init running...' && sleep 5 && echo 'Init done' > /shared/ready.txt"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  containers:
  - name: main-app
    image: busybox
    command: ["sh", "-c", "echo 'Main app started' && cat /shared/ready.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
```

### `init-fail.yaml` — failed init blocks main permanently

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-fail
  namespace: dev-backend
  labels:
    app: init-fail
spec:
  initContainers:
  - name: init-broken
    image: busybox
    command: ["sh", "-c", "echo 'Trying...' && exit 1"]
  containers:
  - name: main-app
    image: busybox
    command: ["sh", "-c", "echo 'I should never run' && sleep 3600"]
```

### `sidecar-demo.yaml` — log-shipper pattern

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
  namespace: dev-backend
  labels:
    app: sidecar-demo
spec:
  containers:
  - name: main-app
    image: busybox
    command: ["sh", "-c", "while true; do echo \"$(date): app log entry\" >> /logs/app.log; sleep 3; done"]
    volumeMounts:
    - name: log-volume
      mountPath: /logs
  - name: log-shipper
    image: busybox
    command: ["sh", "-c", "tail -f /logs/app.log"]
    volumeMounts:
    - name: log-volume
      mountPath: /logs
  volumes:
  - name: log-volume
    emptyDir: {}
```

---

## 🔧 All Commands

```bash
# ── Apply ────────────────────────────────────────────────
kubectl apply -f init-demo.yaml
kubectl apply -f init-fail.yaml
kubectl apply -f sidecar-demo.yaml

# ── Watch init transition ────────────────────────────────
kubectl get pods -n dev-backend -l app=init-demo -w
kubectl get pods -n dev-backend -l app=init-fail -w

# ── Check specific container logs ────────────────────────
kubectl logs init-demo -n dev-backend -c init-setup
kubectl logs init-demo -n dev-backend -c main-app
kubectl logs sidecar-demo -n dev-backend -c main-app
kubectl logs sidecar-demo -n dev-backend -c log-shipper

# ── Confirm both sidecar containers in live status ───────
kubectl get pod sidecar-demo -n dev-backend -o jsonpath='{.status.containerStatuses[*].name}'

# ── Cleanup ──────────────────────────────────────────────
kubectl delete -f init-demo.yaml -f init-fail.yaml -f sidecar-demo.yaml
```

---

## 🧪 What Actually Happened — Real Results

### Failing init container — main app never appeared

```
NAME        READY   STATUS                  RESTARTS
init-fail   0/1     Init:CrashLoopBackOff   1 (2s ago)
init-fail   0/1     Init:Error              2 (17s ago)
init-fail   0/1     Init:CrashLoopBackOff   2 (27s ago)
```

The main container column never appeared at all — not `0/1`, not
`Pending`, just permanently absent. This is the real value of init
containers: a broken dependency actively prevents the app from starting
in a broken state, rather than starting anyway and failing later in a
less obvious way.

### Sidecar — both containers alive, log-shipper tailing main-app's file

```
kubectl get pods -n dev-backend -l app=sidecar-demo
NAME           READY   STATUS    RESTARTS   AGE
sidecar-demo   2/2     Running   0          83s
```

```
kubectl get pod sidecar-demo -n dev-backend \
  -o jsonpath='{.status.containerStatuses[*].name}'
log-shipper main-app
```

Both confirmed in live pod status — same pod, same IP, same volume,
two independent processes.

### The detail that proved the sidecar pattern most precisely

```bash
kubectl logs sidecar-demo -n dev-backend -c main-app
# (empty — no output)

kubectl logs sidecar-demo -n dev-backend -c log-shipper
Fri Jul  3 12:21:07 UTC 2026: app log entry
Fri Jul  3 12:21:10 UTC 2026: app log entry
Fri Jul  3 12:21:13 UTC 2026: app log entry
Fri Jul  3 12:21:16 UTC 2026: app log entry
Fri Jul  3 12:21:19 UTC 2026: app log entry
```

`main-app` writes to a FILE (`/logs/app.log`), not to stdout — so
`kubectl logs` shows nothing for it. `log-shipper` tails that file and
sends the content to its OWN stdout, which is why its logs show all
the entries. The main app doesn't know anything is reading its logs —
they share purely through the volume. That's the log-shipping sidecar
pattern, working exactly as designed.

---

## 💡 Key Takeaways

1. **Init containers run sequentially before the main container** — a failed init actively prevents a broken app from starting rather than letting it fail more obscurely later
2. **`Init:CrashLoopBackOff` is the STATUS to watch for** — the main container's column never appears at all, not even as `0/1`
3. **Sidecar containers run for the entire pod lifetime** — `2/2 Running` is the tell; the number before the slash is ready containers, not total
4. **`kubectl logs` reads stdout** — if a container writes to a file instead of stdout, `kubectl logs` for that container shows nothing; the sidecar's job is precisely to bridge that gap to its own stdout
5. **Both patterns share the same volume mechanism** — `emptyDir` for temporary sharing within a pod's lifetime, same as Chapter 07's PV/PVC principle but scoped to a single pod
6. **Bonus answer:** deleting the pod and letting it recreate would lose all log entries in `emptyDir` — the volume lives with the pod, not independently like a PVC. Using a PVC instead would preserve logs across pod restarts, which is exactly what Chapter 07 taught

---

## 🏋️ Exercises Completed

- [x] Deployed `init-demo` — watched STATUS transition through `Init:0/1` → `Running`
- [x] Confirmed init wrote the file and main read it via separate container logs
- [x] Deployed `init-fail` — confirmed main container never started, `Init:CrashLoopBackOff` indefinitely
- [x] Deployed `sidecar-demo` — confirmed `2/2 Running`
- [x] Confirmed `log-shipper` tails what `main-app` writes, via empty vs populated logs from the two containers
- [x] Confirmed both container names in live pod status via jsonpath
- [x] Bonus: `emptyDir` volumes die with the pod — a PVC would be needed to survive pod deletion, tying directly back to Chapter 07
