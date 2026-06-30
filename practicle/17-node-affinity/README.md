# 17 — Node Affinity 🎯

> The inverse of Chapter 15's taints — a pod actively pulling toward
> a specific kind of node, instead of a node repelling pods by default.

---

## 📁 Folder Structure

```
17-node-affinity/
├── nodeselector-demo.yaml     # matching nodeSelector — schedules fine
├── nodeselector-fail.yaml     # mismatched nodeSelector — sticks Pending
├── affinity-required.yaml     # nodeAffinity required, In [ssd, nvme]
├── affinity-preferred.yaml    # nodeAffinity preferred — soft, never blocks
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why This Exists — And How It Differs From Chapter 15

```
Taint/Toleration (Ch 15)  → NODE says "stay away unless cleared" (repulsion)
Node Affinity (this Ch)    → POD says "I want a SPECIFIC kind of node" (attraction)
```

Often used together in real clusters: a taint keeps *other* pods off a
specialized node (e.g. GPU), while affinity is what actually *pulls*
the right pod there. **Toleration alone only removes repulsion — it
does not attract.** Without affinity too, a tolerant pod could still
land on a completely different, untainted node by chance.

---

## 🔑 Two Mechanisms, Old and New

```
nodeSelector
  = oldest, simplest — exact key=value match only
  = zero flexibility — any mismatch blocks scheduling entirely

nodeAffinity
  = modern, richer — supports operators (In, NotIn, Exists...)
  = two modes:
    requiredDuringSchedulingIgnoredDuringExecution  → hard rule
    preferredDuringSchedulingIgnoredDuringExecution → soft rule, never blocks
```

---

## 📄 Files

### `nodeselector-demo.yaml` / `nodeselector-fail.yaml`

```yaml
spec:
  nodeSelector:
    disktype: ssd     # demo: matches → schedules
    # disktype: hdd   # fail: no node has this → Pending forever
```

### `affinity-required.yaml`

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
```

`operator: In` accepts multiple acceptable values — still a hard rule,
just with more flexibility than `nodeSelector`'s single exact match.

### `affinity-preferred.yaml`

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - nvme
```

---

## 🔧 All Commands

```bash
# ── Label the node ───────────────────────────────────────
kubectl label nodes minikube disktype=ssd
kubectl get nodes --show-labels | grep disktype

# ── Apply each pod ───────────────────────────────────────
kubectl apply -f nodeselector-demo.yaml
kubectl apply -f nodeselector-fail.yaml
kubectl apply -f affinity-required.yaml
kubectl apply -f affinity-preferred.yaml

# ── Watch / debug ────────────────────────────────────────
kubectl get pods -n dev-backend -l app=<name> -o wide
kubectl describe pod <name> -n dev-backend

# ── Cleanup ──────────────────────────────────────────────
kubectl delete -f nodeselector-demo.yaml -f nodeselector-fail.yaml -f affinity-required.yaml -f affinity-preferred.yaml
kubectl label nodes minikube disktype-
```

---

## 🧪 What Actually Happened — Real Results

### `nodeSelector` mismatch — stuck Pending, exact scheduler wording

```
NAME                READY   STATUS    RESTARTS   AGE
nodeselector-fail   0/1     Pending   0          5s
```

```
Warning  FailedScheduling  default-scheduler
0/1 nodes are available: 1 node(s) didn't match Pod's node
affinity/selector. no new claims to deallocate, preemption:
0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

Asked for `disktype=hdd`, the node only has `disktype=ssd` — zero
tolerance for mismatch, since `nodeSelector` has no operators at all.

### `nodeAffinity` required, `In [ssd, nvme]` — matched and scheduled

```
Normal  Scheduled  default-scheduler  Successfully assigned
                                       dev-backend/affinity-required to minikube
Normal  Started    kubelet            spec.containers{app}: Container started
```

The node's `disktype=ssd` satisfied the `In [ssd, nvme]` condition —
either listed value is acceptable. Still a hard requirement, just with
more flexibility than `nodeSelector`'s single exact match.

### `nodeAffinity` preferred, value with zero matching nodes — scheduled anyway

```
NAME                 READY   STATUS    RESTARTS   AGE   IP             NODE
affinity-preferred   1/1     Running   0          7s    10.244.0.142   minikube
```

No node in the cluster has `disktype=nvme` — yet the pod scheduled and
ran successfully on `minikube` regardless, in direct contrast to the
`nodeSelector` mismatch above failing outright on a comparable miss.

---

## 💡 Key Takeaways — Including the Bonus Answer

```
nodeSelector mismatch (Step 3):
  → ZERO tolerance — nodeSelector has no operators, just exact match
  → Pending forever

nodeAffinity required + In [ssd, nvme] (Step 4):
  → still a HARD rule, but allows MULTIPLE acceptable values
  → ssd was one of them → matched → Scheduled

nodeAffinity preferred + nvme, no match anywhere (Step 5):
  → would have stuck Pending under "required" — identical to Step 3
  → but "preferred" never blocks: "try to honor this, but if
    nothing matches, schedule it ANYWHERE rather than block"
  → Scheduled on minikube anyway, the unmet preference simply ignored
```

The real distinction isn't *how close* a match was — Step 4 matched
correctly via `In`, Step 5 had zero matching nodes at all. The
distinction is purely `required` vs `preferred` as a *policy*:
`required` blocks on any mismatch; `preferred` never blocks, no matter
how badly it misses.

1. **Affinity is attraction; taints/tolerations are repulsion + clearance** — the two are complementary, not interchangeable, and real GPU-node setups typically need both together
2. **`nodeSelector` has no flexibility at all** — any mismatch is fatal to scheduling
3. **`nodeAffinity`'s `In` operator allows a hard rule to still have multiple acceptable values** — more expressive than `nodeSelector` without giving up the requirement
4. **`preferred` vs `required` is a policy choice, not a strictness gradient on the match itself** — a `preferred` rule that matches nothing still succeeds; a `required` rule that matches nothing always fails

---

## 🏋️ Exercises Completed

- [x] Labeled the node, deployed matching `nodeSelector` pod — scheduled
- [x] Deployed mismatched `nodeSelector` pod — confirmed `Pending` forever with exact scheduler error text
- [x] Deployed `nodeAffinity` required + `In [ssd, nvme]` — confirmed scheduled via the flexible operator
- [x] Deployed `nodeAffinity` preferred with a value matching zero nodes — confirmed scheduled anyway
- [x] Bonus: explained precisely why Step 4 and Step 5 both succeeded despite very different match conditions — `required`-with-multiple-options vs `preferred`-with-zero-options

