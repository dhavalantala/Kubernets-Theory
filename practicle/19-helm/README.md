# 19 — Helm 📦

> A package manager for Kubernetes — the same separation-of-config-from-workload
> principle as Chapter 11's ConfigMaps, applied to an entire set of YAML files at once.

---

## 📁 Folder Structure

```
19-helm/
└── myapp/
    ├── Chart.yaml              # chart metadata (name, version)
    ├── values.yaml             # default injectable values
    └── templates/
        ├── deployment.yaml     # the actual templated resource
        └── _helpers.tpl        # shared template helpers (auto-generated)
```

---

## 🧠 Why Helm Exists — The Problem First

Looking back across all 18 chapters, every YAML file had hardcoded values:
namespace names, image tags, replica counts, resource limits — written by hand
every time. For one environment that's manageable. For dev, staging, and prod:

```
Without Helm:
  → 3 near-duplicate YAML sets per environment
  → changing one shared setting means editing 3 places
  → no versioning of "what was actually deployed last Tuesday"

Helm:
  = bundle YAML templates into a "Chart"
  = inject DIFFERENT VALUES per environment into the SAME templates
  = track releases with built-in rollback
```

This is the direct evolution of Chapter 11's ConfigMap lesson — separating
config from workload shape — except Helm does it for the **entire set of
YAML files at once**, not just env vars inside one Deployment.

---

## 🔑 Two Core Concepts

```
Chart    = the package — templates + default values + metadata
Release  = one deployed instance of a chart with specific values applied

You can have multiple releases of the same chart:
  helm install myapp-dev   ./myapp   → release in dev-backend
  helm install myapp-prod  ./myapp   → release in prod-backend
  Same chart, different values, different names, independent lifecycles.
```

---

## 🔑 Important Namespace Distinction — Learned Live

```
values.yaml: namespace: dev-backend
  → tells Kubernetes WHERE to put the actual resources (Deployment, pods)

helm install (no -n flag)
  → Helm stores its own release metadata in the DEFAULT namespace
    (or whichever namespace the current kubectl context uses)
```

These two namespaces serve completely different purposes and CAN be different:
- `kubectl get pods -n dev-backend` → finds the actual running pods
- `helm list` (no `-n`) → finds the release metadata tracked by Helm
- `helm history myapp-dev -n dev-backend` → WRONG namespace, release not found
- `helm history myapp-dev` (no `-n`) → correct, release metadata in `default`

To keep these aligned in real use, specify `helm install myapp-dev ./myapp -n dev-backend`
explicitly — that way Helm tracks the release AND deploys resources into the same namespace.

---

## 📄 Files

### `myapp/values.yaml`

```yaml
replicaCount: 2

image:
  repository: busybox
  tag: latest
  pullPolicy: IfNotPresent

namespace: dev-backend

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### `myapp/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-myapp
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-myapp
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-myapp
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
```

`{{ .Release.Name }}` is automatically supplied by Helm at install time —
the same chart can produce differently-named resources per release.

---

## 🔧 All Commands

```bash
# ── Install Helm ─────────────────────────────────────────
brew install helm
helm version

# ── Create a chart ───────────────────────────────────────
helm create myapp

# ── Preview rendered YAML (no cluster interaction) ──────
helm template myapp-dev ./myapp

# ── Install ──────────────────────────────────────────────
helm install myapp-dev ./myapp

# ── Inspect release ──────────────────────────────────────
helm list
helm history myapp-dev

# ── Upgrade (change a value) ─────────────────────────────
helm upgrade myapp-dev ./myapp --set replicaCount=4

# ── Rollback ─────────────────────────────────────────────
helm rollback myapp-dev 1

# ── Cleanup ──────────────────────────────────────────────
helm uninstall myapp-dev
```

---

## 🧪 What Actually Happened — Real Results

### Scaffolded templates referenced values that no longer existed

`helm create` scaffolded more templates than expected in this version.
Deleting `service.yaml`, `ingress.yaml`, `hpa.yaml`, `serviceaccount.yaml`
wasn't enough — `httproute.yaml` and `NOTES.txt` also referenced
`.Values.httpRoute.enabled`, which had been stripped from the simplified
`values.yaml`. Both were discovered and removed via the explicit error
messages from `helm template`, which named the exact file and line each time.

### `helm template` output — confirmed all placeholders resolved correctly

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-dev-myapp
  namespace: dev-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp-dev-myapp
  ...
  image: "busybox:latest"
```

Every `{{ .Values.X }}` placeholder replaced with real values.
Zero cluster interaction during this step.

### Install, upgrade, rollback — 3 revisions confirmed

```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete     ← replicaCount: 2
2         superseded  Upgrade complete     ← replicaCount: 4 (--set)
3         deployed    Rollback to 1        ← replicaCount: 2 again
```

Rollback creates a NEW revision (3) rather than rewriting the history —
same append-only audit trail principle as `kubectl rollout history`.

### Pods caught mid-rollback — 2 Terminating, 2 Running

```
NAME                              READY   STATUS        RESTARTS
myapp-dev-myapp-669df5cfd-9g7lf   1/1     Terminating   0
myapp-dev-myapp-669df5cfd-fsgpw   1/1     Terminating   0
myapp-dev-myapp-669df5cfd-gkkfd   1/1     Running       0
myapp-dev-myapp-669df5cfd-hlc2f   1/1     Running       0
```

This captured the rollback mid-flight: the 2 excess pods from revision 2's
`replicaCount: 4` scaling down to revision 1's `replicaCount: 2`, with
exactly 2 terminating and 2 remaining — the transition in progress.

---

## 💡 Key Takeaways

1. **Helm separates config from templates for the entire chart** — the same principle as ConfigMaps (Chapter 11), applied at the whole-release level
2. **`helm template` is the most important debugging command** — renders final YAML without touching the cluster; run this before every install/upgrade
3. **Helm tracks release metadata separately from where resources live** — `helm list` and `helm history` query Helm's own storage (by default in `default` namespace), not the namespace where pods actually run
4. **Rollback creates a new revision, not a history rewrite** — provides a clean audit trail, same behavior as `kubectl rollout history`
5. **`helm create` scaffolds more templates than you might expect** — any template file referencing a value that doesn't exist in `values.yaml` will break `helm template` with a precise error naming the file and line
6. **`{{ .Release.Name }}` in templates is what makes multiple releases from one chart independently named** — without it, two releases of the same chart in the same namespace would produce conflicting resource names

---

## 🏋️ Exercises Completed

- [x] Created chart, inspected `values.yaml` and template placeholders
- [x] Removed scaffolded templates that referenced stripped-out values (discovered via explicit `helm template` errors)
- [x] Confirmed `helm template` renders clean YAML with no cluster interaction
- [x] Installed release, confirmed Deployment exists in cluster
- [x] Upgraded with `--set replicaCount=4`, confirmed pod count changed
- [x] Rolled back to revision 1, caught the transition mid-flight (2 Terminating, 2 Running)
- [x] Confirmed 3-revision history with append-only audit trail
