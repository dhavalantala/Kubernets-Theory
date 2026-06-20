# 07 — Storage: PersistentVolume & PersistentVolumeClaim 💾

> How Kubernetes decouples storage from Pod lifecycle.
> Pods are disposable. Data shouldn't be.

---

## 🧠 Why PV/PVC Exist — The Problem First

```
Pod runs a database container
Pod writes data to /data inside the container
Pod crashes or gets deleted → Deployment recreates it
New pod = brand new container filesystem
→ ALL DATA GONE ❌
```

Pods are ephemeral by design. Kubernetes needs a storage layer that
exists **independently** of any single Pod's lifecycle.

---

## 🔑 Why Two Objects Instead of One?

```
PersistentVolume (PV)
  = the actual storage resource
  = managed by platform/infra team (or auto-provisioned by cloud)
  = knows the real details: disk type, size, physical location

PersistentVolumeClaim (PVC)
  = a request for storage, made by an application team
  = says "I need 1Gi, ReadWriteOnce" — doesn't care HOW it's provisioned
  = binds to a matching PV automatically
```

**Why the separation matters:**
```
App teams         → write PVCs only, no knowledge of underlying disk
Platform/infra     → manages PVs, decides AWS EBS vs GCP PD vs local disk

Same separation of concerns as namespaces split by environment —
infrastructure concerns kept apart from application concerns.
```

---

## 📄 Files

### `pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv
  labels:
    tier: database
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/db-pv
```

| Field | Meaning |
|-------|---------|
| `capacity.storage` | Max space this PV provides — PVCs requesting more won't bind |
| `accessModes: ReadWriteOnce` | Only one node can mount read-write at a time |
| `persistentVolumeReclaimPolicy: Retain` | Data is KEPT when PVC is deleted — production-safe default |
| `hostPath` | minikube-only mechanism — never use in real production clusters |

### `pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
  namespace: dev-database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**Why it doesn't name the PV directly:** K8s auto-matches PVC to PV based
on `accessModes` + requested size ≤ PV capacity. This decouples app teams
from needing to know which exact PV they'll get.

### `db-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  namespace: dev-database
  labels:
    tier: database
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'I survived' >> /data/proof.txt && sleep 3600"]
    volumeMounts:
    - name: db-storage
      mountPath: /data
  volumes:
  - name: db-storage
    persistentVolumeClaim:
      claimName: db-pvc
```

| Section | Purpose |
|---------|---------|
| `spec.volumes` | Declares WHAT storage is available to this pod |
| `container.volumeMounts` | Declares WHERE inside the container it appears |

---

## 🔧 All Commands

```bash
# ── Apply ────────────────────────────────────────────────
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f db-pod.yaml

# ── Check binding ────────────────────────────────────────
kubectl get pv
kubectl get pvc -n dev-database
kubectl get pods -n dev-database

# ── Prove persistence ────────────────────────────────────
kubectl exec -it db-pod -n dev-database -- cat /data/proof.txt
kubectl delete pod db-pod -n dev-database
kubectl apply -f db-pod.yaml
kubectl exec -it db-pod -n dev-database -- cat /data/proof.txt

# ── Cleanup (scoped — never delete the namespace itself) ─
kubectl delete -f db-pod.yaml
kubectl delete pod db-pod -n dev-database     # must delete pod before PVC if still running
kubectl delete -f pvc.yaml
kubectl delete -f pv.yaml
```

---

## 🧪 What Actually Happened — Real Results

### Proof of persistence across pod deletion

```
Before kubectl delete pod:    "I survived" × 3
After delete + reapply:       "I survived" × 4
```

A brand-new pod, brand-new container filesystem — yet the previous lines
were already there before this pod's command even ran. The PVC/PV kept
the data alive across full pod destruction and recreation.

### PVC delete blocked while pod was still running

```bash
kubectl delete -f pvc.yaml
# hung — did not complete
```

```bash
kubectl get pods -n dev-database
# db-pod   1/1   Running   ← still mounting the PVC
```

**This is a safety mechanism, not a bug.** Kubernetes refuses to finalize
deletion of a PVC that's actively mounted by a running pod. The delete
only completed after the pod was explicitly deleted first.

### PV state after PVC deletion — `Released`, not destroyed

```
kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM
db-pv   1Gi        RWO            Retain           Released   dev-database/db-pvc

kubectl get pvc -n dev-database
No resources found in dev-database namespace.
```

The PVC is fully gone. The PV still exists with `STATUS: Released` —
**not** `Available`, **not** deleted. The `CLAIM` column still shows the
old PVC reference as a stale breadcrumb.

**Why this matters:** a `Released` PV will NOT automatically bind to a new
PVC, even one with identical specs, until a human manually clears the
`claimRef`. This manual gate is the entire point of `Retain` — a person
must consciously decide the data is safe to reuse or wipe. K8s will never
make that call on its own.

---

## 💡 Key Takeaways

1. **Pods are disposable, PVs are not** — that's the entire reason this object pair exists
2. **PVC and PV are intentionally decoupled** — app teams request, infra teams provide
3. **`Retain` is the production-safe default** — data survives accidental PVC deletion
4. **K8s blocks PVC deletion while a pod is mounting it** — a built-in safety net
5. **`Released` ≠ deleted** — it means "unbound, data intact, awaiting human decision"
6. **`hostPath` is local-only** — real clusters use cloud-backed PVs (EBS, PD, etc.), usually via a StorageClass (next chapter)

---

## 🏋️ Exercises Completed

- [x] Create PV — confirmed `STATUS: Available`
- [x] Create PVC in `dev-database` — confirmed `STATUS: Bound` on both sides
- [x] Mount PVC in a pod, write to file, delete pod, recreate, prove data survived (3 → 4 lines)
- [x] Attempted to delete PVC while pod still running — observed the hang/block
- [x] Deleted pod first, then PVC succeeded — confirmed PV landed in `Released`

---

*Status: ✅ Complete*
*Next: 08 — Services*
*Platform: macOS · minikube · Kubernetes*
*Namespace convention: `_infra/namespaces.yaml` applied once, `dev-database` used for this chapter's workloads*