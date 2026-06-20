# 08 — Services 🔌

> How Kubernetes gives a changing set of Pods one stable address.
> Pod IPs are disposable. Services are not.

---

## 📁 Folder Structure

```
08-services/
├── backend-deployment.yaml   # 3-replica backend, target of the Service
├── backend-service.yaml      # ClusterIP Service in front of it
└── README.md
```

Namespaces used (from `_infra/namespaces.yaml`, applied once, never per-chapter):
```
dev-backend     ← backend Deployment + Service live here
dev-frontend    ← debug/test pod launched here to prove cross-namespace calls
```

---

## 🧠 Why Services Exist — The Problem First

Every Pod gets its own IP (seen back in Chapter 02 — `nginx-pod IP: 10.244.0.9`).

```
Deployment manages 3 backend pods
Pod 1 dies → Deployment creates a replacement
New pod gets a BRAND NEW IP — never the old one
```

```
If a frontend called a pod's IP directly:
pod dies and gets replaced → frontend is now calling a dead IP ❌
→ would require manually updating every caller, constantly
```

Pod IPs are disposable, exactly like the Pod itself. A Service sits in
front of a changing set of Pods and gives callers one address that
never changes — no matter how many times the Pods behind it die,
restart, or get rescheduled.

---

## 🔑 The Three Service Types — Mapped to Real Problems

```
ClusterIP   (default, used in this chapter)
  → only reachable INSIDE the cluster
  → use case: backend ↔ database, frontend ↔ backend
  → ~90% of Services in any real system

NodePort
  → opens a specific port on every node directly
  → mostly for quick local testing, rarely used in real production

LoadBalancer
  → asks the cloud provider to provision a real external load balancer
  → use case: production-facing edge services on AWS/GCP/Azure
```

**Blast-radius reasoning:** `dev-backend` has no business being reachable
from outside the cluster, so it should default to `ClusterIP` unless
there's a specific reason otherwise — same instinct as keeping namespaces
split by environment.

---

## 📄 Files

### `backend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: dev-backend
  labels:
    app: backend
    tier: backend
    env: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
        env: dev
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo
        args:
          - "-text=Hello from backend pod"
        ports:
        - containerPort: 5678
```

`hashicorp/http-echo` is used deliberately — a minimal image that just
echoes text over HTTP, so the focus stays on networking, not application
logic.

### `backend-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: dev-backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 5678
```

| Field | Meaning |
|-------|---------|
| `selector: app=backend` | Service watches for ANY pod with this label — never points at fixed IPs |
| `port` | What other things INSIDE the cluster dial to reach this Service |
| `targetPort` | What the actual container is listening on — does NOT have to match `port` |

---

## 🔧 All Commands

```bash
# ── Apply ────────────────────────────────────────────────
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

# ── Check ────────────────────────────────────────────────
kubectl get pods -n dev-backend -o wide
kubectl get svc -n dev-backend
kubectl get endpoints backend-svc -n dev-backend

# ── Cross-namespace test ─────────────────────────────────
kubectl run test-curl -n dev-frontend --rm -it --image=busybox -- sh
# inside the shell:
wget -O- backend-svc.dev-backend.svc.cluster.local:8080

# ── Prove ClusterIP stability ────────────────────────────
kubectl get svc backend-svc -n dev-backend     # note ClusterIP
kubectl delete pod <backend-pod-name> -n dev-backend
kubectl get svc backend-svc -n dev-backend     # ClusterIP unchanged
kubectl get endpoints backend-svc -n dev-backend  # IP list updated

# ── Cleanup (scoped, never deletes the namespace) ───────
kubectl delete -f backend-service.yaml
kubectl delete -f backend-deployment.yaml
```

---

## 🧪 What Actually Happened — Real Results

### Service created with a stable ClusterIP

```
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
backend-svc   ClusterIP   10.110.87.99   <none>        8080/TCP   0s
```

### The mistake that proved `port` actually matters

First attempt connected on the wrong port — by default, plain
`wget -O- backend-svc.dev-backend.svc.cluster.local` assumes port 80:

```
Connecting to backend-svc.dev-backend.svc.cluster.local (10.110.87.99:80)
```

But the Service's `port` field had been set to `8080`, not `80`. The
connection hung — Kubernetes wasn't ignoring the request, the Service
genuinely wasn't listening on port 80 anymore. This wasn't a bug to fix
quietly; it was direct proof that `port` is a real, independent setting
that changes what callers must dial.

### `port` ≠ `targetPort` — confirmed directly from the live Service

```bash
kubectl get svc backend-svc -n dev-backend -o yaml | grep -A3 ports
```

```yaml
ports:
- port: 8080
  protocol: TCP
  targetPort: 5678
```

Two different numbers, living on the same Service object. `8080` is what
something inside the cluster dials. `5678` is what the container is
actually listening on. The Service translates between them — proving
they are independent fields, not aliases of each other.

---

## 💡 Key Takeaways

1. **Services solve the exact same problem Docker custom networks solved** — stable name instead of a disposable IP
2. **Selectors, not fixed IPs** — a Service auto-tracks whichever pods currently match its label selector
3. **ClusterIP never changes**, even as the Pods behind it are destroyed and recreated — only `Endpoints` updates
4. **`port` and `targetPort` are genuinely independent** — confirmed by both a real connection failure and the raw YAML
5. **Cross-namespace DNS pattern**: `<service>.<namespace>.svc.cluster.local` — this is the string a real frontend config would use, and it never changes regardless of pod churn
6. **Default to `ClusterIP`** unless there's a specific reason to expose something further — same blast-radius thinking as namespace design

---

## 🏋️ Exercises Completed

- [x] Deployed `backend` with 3 replicas, noted differing pod IPs
- [x] Created Service — confirmed stable `CLUSTER-IP: 10.110.87.99`
- [x] Checked `Endpoints` — confirmed pod IPs listed behind the Service
- [x] Deleted a backend pod — confirmed ClusterIP unchanged, Endpoints updated
- [x] Called backend from a `dev-frontend` debug pod via full cross-namespace DNS
- [x] Bonus: discovered `port` ≠ `targetPort` the hard way — wrong-port hang, then confirmed via YAML that `port: 8080` and `targetPort: 5678` are independent values

---

*Status: ✅ Complete*
*Next: 09 — Ingress*
*Platform: macOS · minikube · Kubernetes*
*Namespaces: `dev-backend` (workload), `dev-frontend` (cross-namespace test client)*