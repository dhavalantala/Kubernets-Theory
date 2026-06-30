# 16 — HPA (Horizontal Pod Autoscaler) 📈

> Manual scaling (Chapter 03's `kubectl scale`) replaced with automatic,
> load-driven scaling — proven end to end on real CPU metrics.

---

## 📁 Folder Structure

```
16-hpa-vpa/
├── hpa-demo-deployment.yaml   # the workload HPA scales
├── load-generator.yaml        # Service fronting it, used to drive load
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why HPA Exists — The Problem First

Every scaling action in this journey so far has been manual:

```
Chapter 03  → kubectl scale deployment --replicas=5
              a human decides the number, every time
```

HPA automates that decision based on real, observed load:

```
HPA (Horizontal Pod Autoscaler)
  = adds/removes PODS based on CPU/memory usage
  = "traffic is high → automatically spin up more replicas"
  = "traffic drops → automatically scale back down"

VPA (Vertical Pod Autoscaler) — not covered hands-on this chapter
  = adjusts a pod's resource REQUESTS/LIMITS automatically
  = solves a different problem (right-sizing one pod, not adding more)
  = rarely combined with HPA on the same workload without care,
    since the two can conflict over the same scaling decision
```

This chapter focuses on HPA specifically, since it's the far more common
tool in real production use.

---

## 📄 Files

### `hpa-demo-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
  namespace: dev-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
      - name: app
        image: registry.k8s.io/hpa-example
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
        ports:
        - containerPort: 80
```

`registry.k8s.io/hpa-example` is a purpose-built image that deliberately
burns CPU when its endpoint is hit — designed specifically for this
exercise.

### `load-generator.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo-svc
  namespace: dev-backend
spec:
  selector:
    app: hpa-demo
  ports:
    - port: 80
      targetPort: 80
```

---

## 🔧 All Commands

```bash
# ── Prerequisite ─────────────────────────────────────────
minikube addons enable metrics-server
kubectl get pods -n kube-system | grep metrics-server

# ── Apply ────────────────────────────────────────────────
kubectl apply -f hpa-demo-deployment.yaml
kubectl apply -f load-generator.yaml

# ── Create the HPA ───────────────────────────────────────
kubectl autoscale deployment hpa-demo -n dev-backend --cpu-percent=50 --min=1 --max=5

# ── Check metrics independently of HPA ──────────────────
kubectl top pods -n dev-backend

# ── Watch HPA react ──────────────────────────────────────
kubectl get hpa -n dev-backend -w
kubectl describe hpa hpa-demo -n dev-backend

# ── Generate real load (separate terminal) ──────────────
kubectl run load-gen -n dev-backend --image=busybox --rm -it -- /bin/sh -c "while true; do wget -q -O- http://hpa-demo-svc; done"

# ── Cleanup ──────────────────────────────────────────────
kubectl delete hpa hpa-demo -n dev-backend
kubectl delete -f hpa-demo-deployment.yaml -f load-generator.yaml
```

---

## 🧪 What Actually Happened — Real Results

### Mistake #1 — commented-out CPU request, proving the bonus question directly

First deployment attempt had `resources.requests.cpu` commented out
entirely:

```bash
kubectl get hpa -n dev-backend
```
```
TARGETS              REPLICAS
cpu: <unknown>/50%   0
```

This answered the planned bonus question without needing to ask it
separately: **HPA's percentage calculation is `current usage ÷
requested amount × 100`. With no request defined, there is no
denominator — HPA cannot compute any percentage at all, regardless of
real load.** It doesn't default to 0%; it sits permanently at
`<unknown>`.

### Mistake #2 — deprecated image registry, a second, unrelated cause for `REPLICAS: 0`

`k8s.gcr.io/hpa-example` failed to pull — this registry path is
deprecated in favor of `registry.k8s.io`. Switching the image, alongside
uncommenting the resources block, fixed pod scheduling. Worth noting
these were two genuinely separate problems stacked together, not one
problem with two symptoms — confirmed by checking `kubectl get pods`
independently before assuming the resources fix alone would resolve
everything.

### `<unknown>` persisted briefly even with everything correctly configured

After fixing both issues, `kubectl top pods` immediately showed real
data:
```
hpa-demo-79c86f55dd-bv6wx   6m   23Mi
```
but `kubectl get hpa` still showed `<unknown>` for nearly 3 minutes.
`kubectl describe hpa` revealed the precise reason via its Events:

```
missing request for cpu in container app of Pod hpa-demo-669b4549c5-4cql5
no metrics returned from resource metrics API
did not receive metrics for targeted pods (pods might be unready)
```

That first event referenced a **different pod hash** (`669b4549c5`)
than the currently running one (`79c86f55dd`) — evidence of a stale
ReplicaSet briefly being evaluated from the earlier, resource-less
version of the Deployment. The remaining events were a genuine warm-up
delay: `kubectl top` querying metrics-server directly is a faster path
than the HPA controller's own internal sync loop, which runs on its own
polling interval and can lag behind by a couple of minutes after
creation, independent of whether metrics-server itself is healthy.

After waiting the full warm-up period:
```bash
kubectl get hpa -n dev-backend
```
```
TARGETS       REPLICAS
cpu: 0%/50%   1
```

### Real load test — full scale-up confirmed

```bash
kubectl run load-gen ... -- /bin/sh -c "while true; do wget -q -O- http://hpa-demo-svc; done"
```

```
kubectl get hpa -n dev-backend -w

cpu: 0%/50%     REPLICAS: 1
cpu: 35%/50%    REPLICAS: 1
cpu: 249%/50%   REPLICAS: 1
...
REPLICAS: 4
REPLICAS: 5   ← hit the configured max
```

CPU usage climbed to roughly 5x the target (`249%` vs `50%`), and the
HPA controller responded by scaling the Deployment from 1 replica all
the way to the configured ceiling of 5 — entirely automatically, with
no manual `kubectl scale` command issued at any point during the load
test.

---

## 💡 Key Takeaways

1. **HPA cannot function without `resources.requests.cpu` defined on the container** — confirmed directly, not just stated; the percentage calculation has no denominator without it
2. **`kubectl top` working does not guarantee the HPA controller has synced yet** — they query through related but separate paths, and a multi-minute warm-up gap after HPA creation is normal, not a sign of a broken setup
3. **Stale ReplicaSets from earlier YAML edits can briefly surface in HPA's own error events** — worth checking `kubectl get rs` if an error message references a pod hash that doesn't match the currently running pod
4. **`registry.k8s.io` has replaced `k8s.gcr.io`** for official Kubernetes example images — a deprecated registry path is a distinct failure mode from a missing resource request, even though both can show up as `REPLICAS: 0`
5. **HPA reacted to genuinely extreme load (249% vs a 50% target) by scaling to its configured maximum** — the full automatic loop, from real CPU pressure to pod count increase, confirmed end to end on this cluster

---

## 🏋️ Exercises Completed

- [x] Confirmed metrics-server running before starting
- [x] Deployed `hpa-demo`, created the HPA, hit and resolved two real configuration issues along the way (missing CPU request, deprecated image registry)
- [x] Diagnosed a lingering `<unknown>` state via `kubectl describe hpa`'s Events, distinguishing a stale-ReplicaSet artifact from a genuine warm-up delay
- [x] Generated real load — confirmed `TARGETS` climbing from `0%` to `249%`
- [x] Confirmed `REPLICAS` scaling automatically from `1` to `5` (the configured max), entirely without manual intervention
- [x] Bonus: confirmed directly that HPA cannot calculate a percentage with no CPU request defined — sits at `<unknown>` permanently, not `0%`

---

*Status: ✅ Complete*
*Next: 17 — Node Affinity*
*Platform: macOS · minikube · Kubernetes*
*Namespace: `dev-backend`*