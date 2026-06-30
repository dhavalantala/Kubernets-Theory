# 18 — RBAC 🔑

> The promise from Chapter 1's namespace-deletion warning, finally cashed
> in — proven by a pod that genuinely could not delete even itself.

---

## 📁 Folder Structure

```
18-rbac/
├── readonly-sa.yaml          # ServiceAccount, zero permissions by default
├── pod-reader-role.yaml      # Role: get/list/watch on pods only
├── pod-reader-binding.yaml   # RoleBinding: attaches the Role to the SA
├── rbac-test-pod.yaml        # real pod running AS that ServiceAccount
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why RBAC Exists — Cashing In an Earlier Warning

Chapter 1 noted, after seeing `kubectl delete namespace staging` wipe
everything instantly with no confirmation: "this is why real companies
use RBAC to prevent junior engineers from having permission to delete
production namespaces." This chapter proves that mechanism directly.

```
Everything done in this journey so far ran as the cluster ADMIN —
minikube's default kubectl context has unrestricted access.

RBAC = Role-Based Access Control
     = defines WHO can do WHAT, on WHICH resources, in WHICH namespace
```

---

## 🔑 Four Objects, Two Axes

```
Role / ClusterRole
  = WHAT is allowed — the permission list itself
  Role        → scoped to ONE namespace
  ClusterRole → cluster-wide, or reusable across namespaces

RoleBinding / ClusterRoleBinding
  = WHO gets that permission, and WHERE
  RoleBinding        → grants within ONE namespace
  ClusterRoleBinding → grants cluster-wide
```

A Role alone does nothing — same relationship as an Ingress object
without a controller (Chapter 09). The permission list and the actual
grant are deliberately separate objects.

## 🔑 Testing Against a ServiceAccount, Not a User

Pods authenticate to the K8s API as a **ServiceAccount**, not as a human
user. If application code inside a pod ever calls `kubectl` or the K8s
API directly, RBAC bound to that pod's ServiceAccount is exactly what
governs what it's allowed to do. Every namespace's `default`
ServiceAccount starts with almost no permissions — RBAC is what grants
any.

---

## 📄 Files

### `readonly-sa.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: readonly-sa
  namespace: dev-backend
```

### `pod-reader-role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev-backend
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

`apiGroups: [""]` means the core API group (pods, services, etc.).
Verbs are deliberately read-only — no `create`, `update`, or `delete`.

### `pod-reader-binding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: dev-backend
subjects:
- kind: ServiceAccount
  name: readonly-sa
  namespace: dev-backend
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### `rbac-test-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: dev-backend
spec:
  serviceAccountName: readonly-sa
  containers:
  - name: app
    image: bitnami/kubectl
    command: ["sleep", "3600"]
```

A real pod, running `kubectl` itself, authenticated as `readonly-sa` —
not the `--as` simulation, the genuine in-cluster credential path.

---

## 🔧 All Commands

```bash
# ── Apply ────────────────────────────────────────────────
kubectl apply -f readonly-sa.yaml
kubectl apply -f pod-reader-role.yaml
kubectl apply -f pod-reader-binding.yaml
kubectl apply -f rbac-test-pod.yaml

# ── Simulated checks (fast, no real risk) ───────────────
kubectl auth can-i get pods --as=system:serviceaccount:dev-backend:readonly-sa -n dev-backend
kubectl auth can-i delete pods --as=system:serviceaccount:dev-backend:readonly-sa -n dev-backend
kubectl auth can-i get secrets --as=system:serviceaccount:dev-backend:readonly-sa -n dev-backend
kubectl auth can-i get pods --as=system:serviceaccount:dev-backend:readonly-sa -n dev-frontend

# ── Real proof, from inside a live pod ──────────────────
kubectl exec -it rbac-test-pod -n dev-backend -- kubectl get pods
kubectl exec -it rbac-test-pod -n dev-backend -- kubectl delete pod rbac-test-pod

# ── Cleanup ──────────────────────────────────────────────
kubectl delete -f rbac-test-pod.yaml -f pod-reader-binding.yaml -f pod-reader-role.yaml -f readonly-sa.yaml
```

---

## 🧪 What Actually Happened — Real Results

### A YAML gap surfaced a useful side-lesson before the main test

`rbac-test-pod.yaml`'s `metadata` only had `name` and `namespace` — no
`labels:` block. A `kubectl get pods -l app=rbac-test-pod` search
returned `No resources found`, not because the pod failed to create,
but because that label was never set at all. Confirmed the pod was
genuinely fine by querying its exact name directly instead:

```
NAME            READY   STATUS    RESTARTS   AGE
rbac-test-pod   1/1     Running   0          91s
```

A separate `container not found ("app")` error on an early `exec`
attempt was also explained once the full `describe` was checked — the
`bitnami/kubectl` image took 18 seconds to pull, and the exec attempt
had almost certainly landed during that creation window, before the
container existed yet, not as a result of any real failure.

### The real proof — from inside a genuinely live pod

```bash
kubectl exec -it rbac-test-pod -n dev-backend -- kubectl get pods
```
```
NAME                       READY   STATUS    RESTARTS      AGE
backend-588bbcd4d4-jtc5s   1/1     Running   1 (21h ago)   21h
backend-588bbcd4d4-v8dzt   1/1     Running   1 (21h ago)   21h
rbac-test-pod              1/1     Running   0             2m1s
```

Succeeded — real pod data returned, confirming `get/list/watch` works
from inside the live container, authenticated via the mounted
ServiceAccount token, not a simulated `--as` flag.

```bash
kubectl exec -it rbac-test-pod -n dev-backend -- kubectl delete pod rbac-test-pod
```
```
Error from server (Forbidden): pods "rbac-test-pod" is forbidden:
User "system:serviceaccount:dev-backend:readonly-sa" cannot delete
resource "pods" in API group "" in the namespace "dev-backend"
```

Blocked — and the denial message names every relevant piece explicitly:
the exact ServiceAccount, the exact verb (`delete`), the exact resource
(`pods`), the exact namespace. A pod running with valid, working
credentials was genuinely unable to delete even itself, because its
bound Role only ever granted read access.

---

## 💡 Key Takeaways

1. **A Role with no RoleBinding is completely inert** — same two-object pattern as Ingress + controller; the permission list and the grant are deliberately separate
2. **`kubectl auth can-i --as=...` is the fast way to simulate a permission check** — but testing from inside a real, live pod is the genuine proof, since it exercises the actual in-cluster credential path rather than a simulation
3. **RBAC denial messages are precise, not generic** — they name the exact subject, verb, resource, and namespace involved, making them genuinely useful for debugging rather than just a wall
4. **Pods authenticate as ServiceAccounts, not as the human who deployed them** — this is the real mechanism governing what application code can do if it ever calls the K8s API itself
5. **A missing `labels:` block doesn't break pod creation** — it only breaks label-based queries against it; worth checking the exact resource name directly when a label search unexpectedly returns nothing
6. **An `exec` failure during a slow image pull can look like a real RBAC or pod problem** — checking `kubectl describe pod` for actual `STATUS` and pull timing resolved the ambiguity immediately

---

## 🏋️ Exercises Completed

- [x] Created ServiceAccount — confirmed zero permissions via `auth can-i`
- [x] Created Role alone — confirmed still inert without a binding
- [x] Created RoleBinding — confirmed the SAME `can-i` command flipped from `no` to `yes`
- [x] Ran boundary checks (delete pods, get secrets, cross-namespace) — all confirmed `no`
- [x] Ran the real pod test — `get pods` succeeded, `delete pod` failed with a genuine RBAC-forbidden error, from inside a live container
- [x] Bonus: cross-namespace check failing despite the same ServiceAccount having permission in `dev-backend` confirms `Role` is namespace-scoped by design — the same permission set granted via `ClusterRole` instead would have succeeded across namespaces
