# 11 — ConfigMaps 🗂️

> Separating configuration from the workload that uses it.
> Proved, directly against the cluster, that env vars freeze at pod startup — file mounts don't.

---

## 📁 Folder Structure

```
11-configmaps/
├── app-config.yaml          # literal key-value ConfigMap
├── app-deployment.yaml      # consumes it via configMapKeyRef, then envFrom
├── nginx-config.yaml        # ConfigMap holding a file's contents
├── config-file-pod.yaml     # mounts that ConfigMap as a real file
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why ConfigMaps Exist — The Problem First

Chapter 10's StatefulSet hardcoded config directly into the workload spec:

```yaml
env:
- name: POSTGRES_DB
  value: "appdb"
```

In a real team, dev/staging/prod each need different values for the same
setting. With everything hardcoded:

```
→ near-duplicate YAML files per environment
→ changing one config value means editing and reapplying the
  ENTIRE workload definition
→ config changes get buried inside workload changes during review
```

ConfigMap separates **what the workload looks like** (image, replicas,
ports) from **how it's configured** (env vars, files) — the same
separation-of-concerns instinct already used for PV/PVC and for
environment-scoped namespaces.

---

## 🔑 Three Ways to Consume a ConfigMap — Mapped to Real Use Cases

```
1. configMapKeyRef   → pick specific keys as named env vars
2. envFrom            → pull ALL keys in as env vars at once
3. volumeMount         → mount as an actual file on disk —
                          the ONLY option with live updates
```

---

## 📄 Files

### `app-config.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev-backend
data:
  APP_ENV: "development"
  LOG_LEVEL: "debug"
  FEATURE_FLAG_NEW_UI: "true"
```

### `app-deployment.yaml` — selective keys via `configMapKeyRef`

```yaml
env:
- name: APP_ENV
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_ENV
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: LOG_LEVEL
```

Only the two explicitly wired keys appear in the pod's environment —
`FEATURE_FLAG_NEW_UI` is deliberately left out to prove this method is
selective, not all-or-nothing.

### Same Deployment — switched to `envFrom` to pull everything

```yaml
envFrom:
- configMapRef:
    name: app-config
```

All three keys appear automatically this time, with zero per-key wiring.

### `nginx-config.yaml` + `config-file-pod.yaml` — mounted as a file

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: dev-backend
data:
  message.txt: |
    Hello from version 1 of the config file
```

```yaml
volumeMounts:
- name: config-volume
  mountPath: /etc/config
volumes:
- name: config-volume
  configMap:
    name: nginx-conf
```

---

## 🔧 All Commands

```bash
# ── Create ───────────────────────────────────────────────
kubectl apply -f app-config.yaml
kubectl apply -f app-deployment.yaml
kubectl apply -f nginx-config.yaml
kubectl apply -f config-file-pod.yaml

# ── Inspect ──────────────────────────────────────────────
kubectl describe configmap app-config -n dev-backend
kubectl get configmap app-config -n dev-backend -o yaml

# ── Check env vars inside a running pod ─────────────────
kubectl exec -it -n dev-backend $(kubectl get pod -n dev-backend -l app=config-demo -o jsonpath='{.items[0].metadata.name}') -- env | grep APP_ENV

# ── Check mounted file ───────────────────────────────────
kubectl exec -it config-file-demo -n dev-backend -- cat /etc/config/message.txt

# ── Edit a ConfigMap live ───────────────────────────────
kubectl edit configmap app-config -n dev-backend
kubectl edit configmap nginx-conf -n dev-backend

# ── Cleanup ──────────────────────────────────────────────
kubectl delete -f app-deployment.yaml
kubectl delete -f app-config.yaml
kubectl delete -f config-file-pod.yaml
kubectl delete -f nginx-config.yaml
```

---

## 🧪 What Actually Happened — Real Results

### Unrelated mid-chapter blocker: cluster-level DNS failure

Mid-exercise, a completely unrelated problem surfaced — `busybox` failed
to pull with:

```
Failed to pull image "busybox": Error response from daemon:
Get "https://registry-1.docker.io/v2/": dial tcp:
lookup registry-1.docker.io on 192.168.65.254:53: no such host
```

This was the minikube node itself losing the ability to resolve external
DNS — not a YAML mistake, not an image name typo, and not related to
ConfigMaps at all. The same `busybox` image had already pulled
successfully in earlier chapters, confirming this was new, environment-
level breakage rather than a recurring error. Diagnosed via
`kubectl describe pod` events showing `ErrImagePull` →
`ImagePullBackOff`, then resolved with a `minikube stop` / `minikube start`
cycle (preserves workloads, unlike `minikube delete` used in Chapter 09).

**Lesson reinforced:** when a pull fails, read the actual Events section
before assuming the YAML is wrong — the real cause is often
infrastructure-level, not application-level.

### The core proof — env vars are snapshotted, file mounts are not

**ConfigMap updated directly:**
```bash
kubectl get configmap app-config -n dev-backend -o yaml | grep APP_ENV
```
```
APP_ENV: PRODUCTION-CHANGED
```

**Same key, checked inside the still-running pod, with no restart:**
```bash
kubectl exec -it ... -- env | grep APP_ENV
```
```
APP_ENV=development
```

The ConfigMap's live value is `PRODUCTION-CHANGED`. The running pod's
environment variable still says `development` — the original value from
when the pod started. This is direct, cluster-verified proof that env
vars sourced from a ConfigMap are copied in once at pod startup and never
re-read afterward, no matter how many times the underlying ConfigMap
changes.

(Note: the `last-applied-configuration` annotation visible in the raw
YAML also showed `development` — that's a separate, internal kubectl
bookkeeping field used for diffing future applies, not a second "live"
value. Worth not confusing the two when reading raw ConfigMap YAML.)

---

## 💡 Key Takeaways

1. **ConfigMaps separate configuration from workload shape** — same separation-of-concerns principle as PV/PVC and namespace-by-environment
2. **`configMapKeyRef` is selective; `envFrom` is all-or-nothing** — choose based on how many settings the app actually needs
3. **Env vars from a ConfigMap are frozen at pod startup** — proven directly: changing the ConfigMap did not change the running pod's environment
4. **File mounts ARE live-updated** — the same kind of change, applied via a volume mount instead, propagates into the running pod without a restart
5. **A failed image pull is an infrastructure signal, not necessarily a YAML bug** — `kubectl describe pod`'s Events section names the real cause; don't guess at the manifest first
6. **`minikube stop` / `start` is a lighter recovery option than `delete`** — worth trying first for node-level glitches, since it preserves existing workloads

---

## 🏋️ Exercises Completed

- [x] Created ConfigMap — confirmed all 3 keys via `describe`
- [x] Wired 2 of 3 keys via `configMapKeyRef` — confirmed only those 2 appeared in env
- [x] Switched to `envFrom` — confirmed all 3 keys appeared automatically
- [x] Mounted ConfigMap as a file — read it via `exec`
- [x] Edited the ConfigMap, confirmed the mounted file updates live
- [x] Bonus: edited `app-config` while the env-var pod was still running — confirmed via direct `exec` that the running pod's env var did NOT change, proving the snapshot behavior empirically rather than by assertion

---

*Status: ✅ Complete*
*Next: 12 — Secrets*
*Platform: macOS · minikube · Kubernetes*
*Namespace: `dev-backend`*