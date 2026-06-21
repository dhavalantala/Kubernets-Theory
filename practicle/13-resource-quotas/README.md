# 13 — Resource Quotas & Limits 📏

> Two different layers of enforcement, hit live and unintentionally —
> both proving the cluster blocks bad requests before they ever run.

---

## 📁 Folder Structure

```
13-resource-quotas/
├── resource-quota.yaml      # namespace-wide hard caps
├── limit-range.yaml         # per-container defaults, min, max
├── no-resources-pod.yaml    # proves LimitRange injects defaults
├── quota-buster.yaml        # deliberately exceeds the quota
├── memory-hog.yaml          # intended to trigger OOMKilled
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why This Chapter Exists — The Problem First

Nothing covered so far stops a single careless pod from doing this:

```yaml
command: ["sh", "-c", "while true; do : ; done"]
```

With no resource limit set anywhere, this pod can consume as much CPU
as the node physically has — starving every other pod on that node,
including workloads in completely different namespaces. Namespaces
isolate *what* can be touched; they say nothing about *how much
compute* anything is allowed to consume. Resource Quotas and Limits
close that second gap.

---

## 🔑 Two Objects, Two Different Scopes

```
LimitRange
  = default/max/min resources per CONTAINER
  = applies automatically, even to pods that never mention resources

ResourceQuota
  = a TOTAL cap for an ENTIRE NAMESPACE, summed across all pods

Analogy:
  LimitRange    = "no single employee can expense more than $500/meal"
  ResourceQuota = "the whole department's total budget is $50,000/month"
```

## 🔑 CPU vs Memory — Why They Fail Differently

```
CPU      → compressible — exceeding the limit just THROTTLES the pod
Memory   → incompressible — exceeding the limit KILLS the container
           outright (OOMKilled)
```

## 🔑 Requests vs Limits

```
requests = guaranteed minimum, used by the SCHEDULER to place the pod
limits   = the ceiling — pod can burst up to this, never beyond
```

---

## 📄 Files

### `resource-quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-quota
  namespace: dev-backend
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "10"
```

### `limit-range.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: backend-limits
  namespace: dev-backend
spec:
  limits:
  - default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "500m"
      memory: "512Mi"
    min:
      cpu: "50m"
      memory: "64Mi"
    type: Container
```

### `memory-hog.yaml` (intended OOMKill trigger)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
  namespace: dev-backend
spec:
  containers:
  - name: app
    image: polinux/stress
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
```

---

## 🔧 All Commands

```bash
# ── Apply ────────────────────────────────────────────────
kubectl apply -f resource-quota.yaml
kubectl apply -f limit-range.yaml
kubectl apply -f no-resources-pod.yaml
kubectl apply -f quota-buster.yaml
kubectl apply -f memory-hog.yaml

# ── Inspect ──────────────────────────────────────────────
kubectl describe resourcequota backend-quota -n dev-backend
kubectl get pod no-resources-pod -n dev-backend -o yaml | grep -A4 resources
kubectl get pods -n dev-backend
kubectl describe deployment quota-buster -n dev-backend
kubectl describe pod memory-hog -n dev-backend

# ── Cleanup ──────────────────────────────────────────────
kubectl delete -f quota-buster.yaml
kubectl delete -f no-resources-pod.yaml
kubectl delete -f memory-hog.yaml
kubectl delete -f limit-range.yaml
kubectl delete -f resource-quota.yaml
```

---

## 🧪 What Actually Happened — Real Results

### Rejection #1 — LimitRange minimum, enforced at admission time

First attempt at `memory-hog.yaml` requested `50Mi`:

```
Error from server (Forbidden): error when creating "memory-hog.yaml":
pods "memory-hog" is forbidden: minimum memory usage per Container
is 64Mi, but request is 50Mi
```

This is the `min: memory: "64Mi"` field from `limit-range.yaml`, set
earlier in this same chapter, actively rejecting a pod that violates
its own floor. Notably, this failed **before the pod object was even
created** — not after scheduling, not after starting. This is
admission-time enforcement, distinct from the runtime-level OOMKill
behavior the test was originally designed to demonstrate.

**Fix applied:** raised the request to `64Mi`, satisfying the floor.

### Rejection #2 — ResourceQuota pod-count cap, also admission-time

Second attempt, after fixing the memory value, hit a completely
different limit:

```
Error from server (Forbidden): error when creating "memory-hog.yaml":
pods "memory-hog" is forbidden: exceeded quota: backend-quota,
requested: pods=1, used: pods=10, limited: pods=10
```

`dev-backend` had accumulated exactly 10 pods (largely from
`quota-buster`'s `replicas: 5`, layered on top of other pods already
present in the namespace from earlier steps and prior chapters), hitting
the `pods: "10"` hard cap from `resource-quota.yaml` with zero headroom
left. This is the ResourceQuota actively blocking pod creation purely
on COUNT, independent of how much CPU/memory the new pod actually
requested.

### OOMKill test — attempted, not yet confirmed

The actual memory-limit-exceeded → `OOMKilled` behavior this chapter set
out to demonstrate was not yet observed, due to the two admission-time
rejections above consuming the available attempts. **This is documented
honestly rather than assumed** — the expected behavior (memory limits
cause a hard kill, unlike CPU's throttling) remains correct based on how
Kubernetes works, but has not yet been verified against this specific
cluster the way every other claim in this learning journey has been.
Revisiting this test with a cleaned-up `dev-backend` namespace (freed
pod count) is a clear next step.

---

## 💡 Key Takeaways

1. **Both LimitRange and ResourceQuota enforce at admission time, not runtime** — a violating pod is rejected by the API server before it's even created, not after it starts misbehaving
2. **These two objects fail for completely independent reasons** — a memory value can be perfectly fine while a pod COUNT cap blocks creation anyway; check both whenever a pod is unexpectedly rejected
3. **`kubectl get pods -n <namespace>` is the first command to run on any quota-related rejection** — the error message names limits, not always the actual current usage breakdown
4. **Accumulated pods from earlier chapters silently eat into later quota budgets** — namespace hygiene (cleaning up after each chapter) directly affects whether later exercises in the SAME namespace will even be able to run
5. **Not every exercise in this journey has gone to plan on the first attempt — and that's fine.** Documenting an attempted-but-unconfirmed result honestly is more valuable than asserting a clean result that was never actually seen

---

## 🏋️ Exercises Completed

- [x] Applied ResourceQuota — confirmed via `describe`
- [x] Applied LimitRange — confirmed default injection on a pod with zero resource fields
- [x] Hit a real LimitRange minimum rejection live, debugged and fixed it
- [x] Hit a real ResourceQuota pod-count rejection live, root-caused it to accumulated pods
- [ ] OOMKilled confirmation — attempted, blocked by the two rejections above, not yet directly observed

---

*Status: 🔄 Mostly complete — one test still pending re-run*
*Next: 14 — Probes*
*Platform: macOS · minikube · Kubernetes*
*Namespace: `dev-backend`*