# 01 — Namespaces 🗂️

> How Kubernetes isolates workloads between teams, environments, and projects.
> The first concept to understand before deploying anything to K8s.

---

## 📁 Folder Structure

```
01-namespaces/
├── namespace.yaml    # Namespace definition via YAML
├── pod.yaml          # Pod deployed into a specific namespace
└── README.md
```

---

## 🧠 Why Namespaces Exist

Without Namespaces, every team deploys into the same cluster:

```
❌ Pod name conflicts    — two teams both name their pod "my-app"
❌ Accidental deletion   — one team deletes another's pods
❌ No resource limits    — one team consumes all cluster resources
❌ No isolation          — dev workloads mix with production
```

With Namespaces, each team gets a virtual cluster:

```
team-a/my-app     ← completely isolated ✅
team-b/my-app     ← completely isolated ✅
team-c/my-app     ← completely isolated ✅
```

Think of Namespaces like **folders on your Mac** —
same filename can exist in different folders without conflict.

---

## 🏗️ Default Namespaces K8s Creates

```bash
kubectl get namespaces
```

| Namespace | Purpose |
|-----------|---------|
| `default` | Where pods go if you don't specify a namespace |
| `kube-system` | K8s internal components (DNS, scheduler, etcd) |
| `kube-public` | Readable by everyone, rarely used |
| `kube-node-lease` | Node heartbeat tracking |

---

## 📄 Files

### `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
    team: backend
```

### `pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-dev
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:alpine
```

---

## 🔧 All Commands

```bash
# ── View ─────────────────────────────────────────────────
kubectl get namespaces               # list all namespaces
kubectl get ns                       # short form
kubectl get pods -n dev              # pods in dev namespace
kubectl get pods --all-namespaces    # pods in ALL namespaces
kubectl get pods -A                  # short form

# ── Create ───────────────────────────────────────────────
kubectl apply -f namespace.yaml      # create via YAML
kubectl create namespace staging     # create via CLI

# ── Switch context ───────────────────────────────────────
kubectl config set-context --current --namespace=dev       # set default ns
kubectl config set-context --current --namespace=default   # switch back

# ── Delete ───────────────────────────────────────────────
kubectl delete namespace staging           # deletes namespace + everything in it
kubectl delete -f namespace.yaml           # delete via YAML
kubectl delete pod nginx-dev -n dev        # delete specific pod in namespace
```

---

## ⚠️ YAML Rules Learned (hard way)

```yaml
# ❌ WRONG — tabs not allowed in YAML
metadata:
	name: dev      ← TAB character = error

# ✅ CORRECT — always use 2 spaces
metadata:
  name: dev      ← 2 SPACES = correct
```

**Fix in VS Code:**
- `Cmd + Shift + P` → "Convert Indentation to Spaces"
- Bottom right → click "Tab" → select "Spaces: 2"

**Permanent fix — add to VS Code settings.json:**
```json
"[yaml]": {
    "editor.insertSpaces": true,
    "editor.tabSize": 2,
    "editor.detectIndentation": false
}
```

---

## 💡 Key Takeaways

1. **Never deploy to `default`** in real companies — always create a namespace first
2. **Deleting a namespace deletes everything inside it** — pods, services, deployments
3. **`-n` flag** specifies which namespace to target — forget it and you see nothing
4. **`-A` flag** shows all namespaces at once — use this to get the full picture
5. **Context switching** saves typing `-n` every time — set your working namespace

---

## 🏋️ Exercises Completed

- [x] Create namespace via YAML file
- [x] Create namespaces via CLI (`kubectl create namespace`)
- [x] Deploy pod into specific namespace
- [x] Verify pod is NOT visible in default namespace
- [x] Use `kubectl get pods -A` to see all namespaces
- [x] Switch context to dev namespace
- [x] Switch back to default namespace
- [x] Delete namespaces and verify pods are gone

---

## 🔍 Real World Usage

| Namespace | Use case |
|-----------|---------|
| `dev` | Development environment |
| `staging` | Pre-production testing |
| `production` | Live traffic |
| `monitoring` | Prometheus, Grafana |
| `logging` | ELK stack |
| `team-frontend` | Frontend team isolation |

---

*Status: ✅ Complete*
*Next: 02 — Pods*
*Platform: macOS · minikube v1.38.1 · Kubernetes v1.35.1*