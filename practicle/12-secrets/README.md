# 12 — Secrets 🔐

> Encoding is not encryption. Proved directly, including correcting
> a wrong claim made earlier in this same chapter.

---

## 📁 Folder Structure

```
12-secrets/
├── app-secret.yaml          # Opaque Secret via stringData
├── secret-deployment.yaml   # consumes it via secretKeyRef
├── secret-file-pod.yaml     # mounts it as a file
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why Secrets Exist — The Problem First

Chapter 11 proved ConfigMaps work well for `APP_ENV`, `LOG_LEVEL`, feature
flags. Apply the same approach to a password:

```yaml
data:
  DB_PASSWORD: "apppassword123"
```

```bash
kubectl get configmap -o yaml
```

Fully readable plain text — visible to anyone with `kubectl get` access,
in any YAML export, in any cluster backup. Fine for config, not fine for
passwords, API keys, tokens, or certs. Secret exists for that second
category specifically.

---

## ⚠️ The Critical Distinction — Encoding ≠ Encryption

```
Secret data is base64 ENCODED, not encrypted.
Base64 is fully, trivially reversible — it is a text-safe binary
format, not a security mechanism.
```

Proved directly on this cluster:

```bash
kubectl get secret app-secret -n dev-backend -o yaml
```
```yaml
data:
  DB_PASSWORD: YXBwcGFzc3dvcmQxMjM=
```
```bash
echo "YXBwcGFzc3dvcmQxMjM=" | base64 -d
```
```
apppassword123
```

One command, no special tooling, instantly recovers the real value.
Anyone who can read the Secret object can run this exact command next.

### What Secret actually provides, stated precisely

```
✅ Not shown in plain text by default in kubectl get/describe output
   — reduces accidental exposure on a screen, in a casual log line
✅ CAN be encrypted at rest in etcd — but only if the cluster operator
   explicitly configures encryption-at-rest. minikube does NOT do
   this by default.
❌ Access is still governed entirely by RBAC — anyone with read
   access to Secrets in a namespace can decode them as easily as
   a ConfigMap
```

**Honest summary:** Secret is a signal and a convention, not a vault,
unless the cluster is specifically configured for encryption-at-rest
and locked down with RBAC. Real production secret management (Vault,
AWS Secrets Manager, Sealed Secrets) exists precisely because plain K8s
Secrets aren't sufficient alone for serious production use.

---

## 📄 Files

### `app-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: dev-backend
type: Opaque
stringData:
  DB_PASSWORD: "apppassword123"
  API_KEY: "sk-test-abc123xyz"
```

`stringData` lets you write the plain value directly — Kubernetes
base64-encodes it automatically on creation, rather than requiring you
to pre-encode it yourself.

### `secret-deployment.yaml` — consumed identically to a ConfigMap

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: DB_PASSWORD
```

Deliberately near-identical API to `configMapKeyRef` from Chapter 11 —
the consumption pattern is the same; only the storage/visibility
treatment differs.

### `secret-file-pod.yaml`

```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/secret
  readOnly: true
volumes:
- name: secret-volume
  secret:
    secretName: app-secret
```

---

## 🔧 All Commands

```bash
# ── Create ───────────────────────────────────────────────
kubectl apply -f app-secret.yaml
kubectl apply -f secret-deployment.yaml
kubectl apply -f secret-file-pod.yaml

# ── Inspect raw storage ──────────────────────────────────
kubectl get secret app-secret -n dev-backend -o yaml

# ── Decode manually ──────────────────────────────────────
echo "<base64-value-from-above>" | base64 -d

# ── Check env var inside a running pod ──────────────────
kubectl exec -it -n dev-backend $(kubectl get pod -n dev-backend -l app=secret-demo -o jsonpath='{.items[0].metadata.name}') -- env | grep DB_PASSWORD

# ── Check mounted file + structure ──────────────────────
kubectl exec -it secret-file-demo -n dev-backend -- ls -la /etc/secret/
kubectl exec -it secret-file-demo -n dev-backend -- cat /etc/secret/DB_PASSWORD

# ── Cleanup ──────────────────────────────────────────────
kubectl delete -f secret-deployment.yaml
kubectl delete -f secret-file-pod.yaml
kubectl delete -f app-secret.yaml
```

---

## 🧪 What Actually Happened — Real Results

### Base64 decode — proven, not assumed

```
DB_PASSWORD: YXBwcGFzc3dvcmQxMjM=
```
```bash
echo "YXBwcGFzc3dvcmQxMjM=" | base64 -d
```
```
apppassword123
```

Confirms the encoding-not-encryption claim directly.

### Env var consumption — works identically to ConfigMap

```bash
kubectl exec -it ... -- env | grep DB_PASSWORD
```
```
DB_PASSWORD=apppassword123
```

### File mount structure — and a correction to a wrong claim made earlier in this chapter

**A prediction made while planning this chapter was wrong:** it was
stated that Secret-mounted files would show more restrictive
permissions than ConfigMap-mounted files. The actual output:

```
drwxrwxrwt    3 root     root   ...   .
lrwxrwxrwx    1 root     root   ...   API_KEY -> ..data/API_KEY
lrwxrwxrwx    1 root     root   ...   DB_PASSWORD -> ..data/DB_PASSWORD
```

These permission bits are wide open, not restrictive — the original
claim does not hold up against this cluster's actual behavior and
should not have been stated with confidence before verifying it.

**What this output actually demonstrates instead, and it's genuinely
useful:** the double-symlink atomic-update structure.

```
..2026_06_21_06_47_00.341999136     ← real, timestamped directory
..data -> ..2026_06_21_06_47_00...  ← symlink pointing at it
DB_PASSWORD -> ..data/DB_PASSWORD    ← symlink pointing through ..data
```

When a mounted Secret (or ConfigMap) changes, Kubernetes builds a brand
new timestamped directory, then atomically swaps the `..data` symlink to
point at it. The application never sees a partially-written file mid-
update — only the complete old version or the complete new version.
This mechanism is shared between Secret and ConfigMap file mounts; it is
not something unique to Secrets, and is the real explanation for the
live-update behavior proved with ConfigMaps in Chapter 11.

---

## 💡 Key Takeaways

1. **Base64 is not encryption** — proven directly with a single `base64 -d` command, recovering the real password instantly
2. **Secret's real default protection is reduced accidental visibility, not cryptographic security** — encryption-at-rest is a separate, cluster-level opt-in
3. **RBAC, not Secret itself, is what actually gates who can decode a Secret** — anyone with read access can do exactly what was just demonstrated
4. **Secret and ConfigMap share the same consumption API on purpose** — `secretKeyRef` mirrors `configMapKeyRef` almost exactly
5. **The atomic double-symlink structure, not stricter permissions, is what file-mounted Secrets/ConfigMaps actually share** — a claim about permissions was made incorrectly and corrected after checking real output, the same discipline applied to the Chapter 09 Ingress mistake
6. **Typed Secrets (`kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`) are Secret's one genuinely unique capability** — structural validation for specific real-world shapes that a generic ConfigMap has no equivalent for

---

## 🏋️ Exercises Completed

- [x] Created Secret via `stringData` — confirmed base64 storage via `-o yaml`
- [x] Decoded a value manually with `base64 -d` — confirmed exact match to the original
- [x] Consumed via `secretKeyRef` — confirmed decoded value visible in running pod's `env`
- [x] Mounted as a file — inspected actual structure, corrected a wrong prediction about permissions, identified the real atomic-update mechanism instead

---

*Status: ✅ Complete*
*Next: 13 — Resource Quotas & Limits*
*Platform: macOS · minikube · Kubernetes*
*Namespace: `dev-backend`*