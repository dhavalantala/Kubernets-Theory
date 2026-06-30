# 15 — Taints & Tolerations 🚫

> Repulsion instead of attraction — a node deciding which pods it
> will NOT accept, by default.

---

## 📁 Folder Structure

```
15-taints-tolerations/
├── before-taint.yaml          # deployed before any taint exists
├── no-toleration.yaml         # should stick Pending once tainted
├── with-toleration.yaml       # matching toleration, should schedule
├── about-to-be-evicted.yaml   # proves NoExecute is retroactive
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why Taints/Tolerations Exist — The Problem First

Every scheduling decision up to this chapter worked by **attraction** —
a pod's labels matching a selector, deciding where it lands. Nothing so
far let the **node itself** refuse pods by default.

```
Taint       = applied to a NODE
              "by default, REPEL pods from scheduling here"

Toleration  = applied to a POD
              "I am explicitly OK with this specific repulsion"

Analogy: Taint = a "Staff Only" sign. Toleration = a staff badge.
The sign doesn't grant access — it just doesn't apply to the badge holder.
Everyone else stays blocked by default.
```

This is the inverse default from labels/selectors, which work by
attraction ("come here if you match"). Taints work by repulsion ("stay
away unless explicitly cleared").

### Real-world uses

```
GPU nodes           → taint so only GPU-tolerant workloads land there
Dedicated DB nodes   → taint to keep general app pods away
Maintenance mode     → taint a node before draining it
Control-plane nodes  → tainted by default on real multi-node clusters,
                        specifically so application workloads never
                        accidentally land on the control plane
```

---

## 🔑 Three Taint Effects

```
NoSchedule        → blocks NEW pods without a matching toleration
                     pods ALREADY running are left alone — not retroactive

PreferNoSchedule   → soft version — scheduler tries to avoid this node,
                     but will still use it if there's no alternative

NoExecute          → strictest — blocks NEW pods AND evicts EXISTING
                     pods without a toleration, even ones already
                     running before the taint was applied
```

`NoExecute` is the one with real teeth — the only effect of the three
that actively removes pods that were already there.

---

## 📄 Files

### `before-taint.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: before-taint
  namespace: dev-backend
  labels:
    app: before-taint
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
```

### `no-toleration.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration
  namespace: dev-backend
  labels:
    app: no-toleration
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
```

### `with-toleration.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
  namespace: dev-backend
  labels:
    app: with-toleration
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "backend-only"
    effect: "NoSchedule"
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
```

Each field in `tolerations` must match the taint exactly: `key`,
`value` (only checked when `operator: Equal`), and `effect`.
`operator: Exists` would instead tolerate the key regardless of value.

### `about-to-be-evicted.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: about-to-be-evicted
  namespace: dev-backend
  labels:
    app: about-to-be-evicted
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
```

---

## 🔧 All Commands

```bash
# ── Check existing taints first ──────────────────────────
kubectl describe node minikube | grep -A2 Taints

# ── Apply / remove a taint ───────────────────────────────
kubectl taint nodes minikube dedicated=backend-only:NoSchedule
kubectl taint nodes minikube dedicated=backend-only:NoSchedule-   # trailing - removes it

kubectl taint nodes minikube maintenance=true:NoExecute
kubectl taint nodes minikube maintenance=true:NoExecute-

# ── Apply pods ───────────────────────────────────────────
kubectl apply -f before-taint.yaml
kubectl apply -f no-toleration.yaml
kubectl apply -f with-toleration.yaml
kubectl apply -f about-to-be-evicted.yaml

# ── Watch / debug ────────────────────────────────────────
kubectl get pods -n dev-backend -l app=<name> -w
kubectl describe pod <name> -n dev-backend

# ── Cleanup ──────────────────────────────────────────────
kubectl delete -f before-taint.yaml -f no-toleration.yaml -f with-toleration.yaml -f about-to-be-evicted.yaml
kubectl describe node minikube | grep -A2 Taints
```

---

## 🧪 What Actually Happened — Real Results

### Confirmed: minikube ships with zero taints by default

```bash
kubectl describe node minikube | grep -A2 Taints
```
```
Taints:             <none>
Unschedulable:      false
```

This is worth noting as a deliberate contrast: on a real multi-node
cluster, the control-plane node is typically tainted automatically
(commonly `node-role.kubernetes.io/control-plane:NoSchedule`),
specifically to keep ordinary application workloads off it. A
single-node minikube cluster has nowhere else to run regular workloads,
so this taint is skipped by default — confirmed directly rather than
assumed.

### Remaining steps — in progress

The rest of this chapter (applying a `NoSchedule` taint, confirming a
new pod sticks `Pending`, confirming an already-running pod is
unaffected, adding a matching toleration to unblock scheduling, and
proving `NoExecute` evicts existing pods) was interrupted mid-session
due to usage limits. **Documented honestly as incomplete** rather than
assuming the expected results — consistent with how Chapter 13's
OOMKill test and Chapter 14's readiness self-recovery were both held to
"don't claim a result you haven't actually seen" until they were
re-run and confirmed.

---

## 💡 Key Takeaways (confirmed so far)

1. **Taints are the inverse of everything learned before this chapter** — repulsion by default, not attraction via matching labels
2. **A real multi-node cluster typically taints its control-plane node automatically** — minikube's single-node setup deliberately skips this so it can still run regular workloads, confirmed directly rather than assumed to be the same everywhere
3. **`NoSchedule` is expected to be non-retroactive, `NoExecute` is expected to be retroactive** — this distinction is the core of the chapter, pending direct confirmation on this cluster

