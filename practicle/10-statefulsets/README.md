# 10 — StatefulSets 🗄️

> Stable pod identity. Stable, per-pod storage. Ordered startup.
> Proved this time with a real database, not a text-echo stand-in.

---

## 📁 Folder Structure

```
10-statefulsets/
├── headless-service.yaml    # clusterIP: None — individual pod DNS
├── statefulset.yaml         # PostgreSQL StatefulSet, 3 replicas
└── README.md
```

Namespace used: `dev-database` (from `_infra/namespaces.yaml`).

---

## 🧠 Why StatefulSets Exist — The Problem First

Run the same self-healing test from Chapter 03, but imagine the workload
is a database instead of `nginx`:

```
Deployment manages a 3-pod database
Delete one pod → Deployment creates a replacement
  → new pod gets a BRAND NEW name and IP
  → if it had its own volume, nothing guarantees it reattaches
    to the SAME disk it had before
```

For a stateless web server this is fine — any pod is interchangeable.
For a database, it's fatal. A 3-node database needs:

```
✅ db-0 to ALWAYS be db-0 — many databases elect a leader by identity
✅ db-0's disk to ALWAYS reattach to db-0, never get mixed with db-1's
✅ db-0 to start BEFORE db-1, which starts BEFORE db-2
```

A Deployment gives none of these guarantees by design. StatefulSet exists
specifically to provide stable identity, stable per-pod storage, and
ordered startup.

---

## 🔑 Why Postgres Instead of a Fake Echo Server

An initial pass at this chapter used `hashicorp/http-echo` to prove the
mechanics cheaply. Re-running it with real PostgreSQL surfaced two things
an echo server simply can't show:

```
1. A REAL readiness gap — Postgres takes several seconds to initialize
   internally. http-echo is "ready" the instant the process starts,
   which hides whether ordered startup is actually being ENFORCED
   or just happens to look ordered because nothing takes any time.

2. REAL data — a text file write is a fine proof, but a row in an
   actual SQL table, queried back out after full pod destruction,
   is a far more convincing and realistic demonstration of exactly
   what this object exists to protect.
```

---

## 📄 Files

### `headless-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
  namespace: dev-database
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

**Why `clusterIP: None` is the entire point:** a normal Service gives one
shared, load-balanced address. A headless Service gives DNS for each
individual pod separately — `db-0.db-headless...` resolves to db-0
specifically, not to whichever pod K8s feels like picking.

### `statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: dev-database
spec:
  serviceName: db-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        env:
        - name: POSTGRES_USER
          value: "appuser"
        - name: POSTGRES_PASSWORD
          value: "apppassword"
        - name: POSTGRES_DB
          value: "appdb"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser"]
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: db-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

| Field | Why it's here |
|-------|---------------|
| `serviceName: db-headless` | Required link to the headless Service — StatefulSet can't create stable identities without it |
| `volumeClaimTemplates` | Auto-creates ONE PVC per pod (`db-data-postgres-0`, `-1`, `-2`) — doesn't exist on Deployment at all |
| `subPath: pgdata` | Postgres needs a clean subfolder of the mount, not the volume root, in some setups — a real-world gotcha worth knowing |
| `readinessProbe: pg_isready` | Without this, K8s thinks the pod is ready the instant the container *starts*, even though Postgres is still initializing — this is what actually enforces ordered startup for a real workload |

---

## 🔧 All Commands

```bash
# ── Apply ────────────────────────────────────────────────
kubectl apply -f headless-service.yaml
kubectl apply -f statefulset.yaml

# ── Watch ordered startup ────────────────────────────────
kubectl get pods -n dev-database -w

# ── Connect ──────────────────────────────────────────────
kubectl exec -it postgres-0 -n dev-database -- psql -U appuser -d appdb

# ── Prove persistence ────────────────────────────────────
kubectl delete pod postgres-0 -n dev-database
kubectl get pods -n dev-database -w
kubectl exec -it postgres-0 -n dev-database -- psql -U appuser -d appdb -c "SELECT * FROM proof;"

# ── Prove independence between replicas ──────────────────
kubectl exec -it postgres-1 -n dev-database -- psql -U appuser -d appdb -c "SELECT * FROM proof;"

# ── Scaling (ordered both directions) ────────────────────
kubectl scale statefulset postgres -n dev-database --replicas=5
kubectl scale statefulset postgres -n dev-database --replicas=2

# ── Storage check ────────────────────────────────────────
kubectl get pvc -n dev-database

# ── Cleanup (PVCs survive by design — same as Chapter 07) ─
kubectl delete -f statefulset.yaml
kubectl delete -f headless-service.yaml
kubectl get pvc -n dev-database          # still there
kubectl delete pvc -l app=postgres -n dev-database   # explicit wipe, if wanted
```

---

## 🧪 What Actually Happened — Real Results

### Data written, then proven to survive full pod deletion

```sql
CREATE TABLE proof (id SERIAL PRIMARY KEY, note TEXT);
INSERT INTO proof (note) VALUES ('data written before pod deletion');
```

```bash
kubectl delete pod postgres-0 -n dev-database
# pod recreated as postgres-0 again — same name, not a new one
```

```bash
kubectl exec -it postgres-0 -n dev-database -- psql -U appuser -d appdb -c "SELECT * FROM proof;"
```

```
 id |               note
----+----------------------------------
  1 | data written before pod deletion
(1 row)
```

A fully destroyed and recreated pod, running a real Postgres instance,
with the row still intact. The PVC reattached correctly, exactly as
designed.

### Proof each replica is genuinely independent — no automatic replication

```bash
kubectl exec -it postgres-1 -n dev-database -- psql -U appuser -d appdb -c "SELECT * FROM proof;"
```

```
ERROR:  relation "proof" does not exist
LINE 1: SELECT * FROM proof;
                      ^
command terminated with exit code 1
```

This error is the correct, expected result — not a failure. `postgres-1`
never received that `INSERT`, because each pod has its own independent
PVC and its own independent Postgres instance. **A StatefulSet alone only
guarantees stable identity and stable storage — it does NOT set up data
replication between replicas.** Real multi-node Postgres clustering
requires additional tooling (streaming replication, Patroni, etc.),
deliberately out of scope here.

---

## 💡 Key Takeaways

1. **StatefulSet solves identity + storage stability, nothing else** — replication, leader election, and clustering logic are separate concerns, usually handled by the application or a dedicated operator
2. **A headless Service (`clusterIP: None`) is required**, not optional, for per-pod DNS addressing to work at all
3. **`volumeClaimTemplates` is the single biggest practical difference from Deployment** — automatic, predictable, one-PVC-per-pod provisioning
4. **Readiness probes are what actually enforce ordered startup for real workloads** — a process that starts instantly (like a fake echo server) can't demonstrate this; a real database with a genuine initialization delay can
5. **PVCs outlive the StatefulSet by default** — same `Retain`-style safety philosophy as Chapter 07, applied automatically here without needing to set a reclaim policy explicitly
6. **Independent storage means independent data** — don't assume a multi-replica StatefulSet is a clustered, replicated database unless you've explicitly set that up

---

## 🏋️ Exercises Completed

- [x] Created headless Service — confirmed `CLUSTER-IP: None`
- [x] Created StatefulSet — watched ordered, readiness-gated startup
- [x] Confirmed predictable pod naming (`postgres-0`, `-1`, `-2`)
- [x] Confirmed auto-created PVCs matching pod numbering
- [x] Connected via `psql`, created a real table, inserted a real row
- [x] Deleted `postgres-0` entirely, confirmed it returned as `postgres-0` with data intact
- [x] Queried `postgres-1` — confirmed it has no knowledge of `postgres-0`'s data, proving independent storage rather than assumed replication

---

*Status: ✅ Complete*
*Next: 11 — ConfigMaps*
*Platform: macOS · minikube · Kubernetes*
*Namespace: `dev-database`*