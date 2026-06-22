# 14 — Probes 🩺

> The difference between a container being "alive" and an application
> actually working — and a genuinely strange invisible-character bug
> along the way.

---

## 📁 Folder Structure

```
14-probes/
├── fake-healthy.yaml      # no probes — proves the baseline gap
├── liveness-demo.yaml     # livenessProbe — destructive on failure
├── readiness-demo.yaml    # readinessProbe + Service — non-destructive
└── README.md
```

Namespace used: `dev-backend` (from `_infra/namespaces.yaml`).

---

## 🧠 Why Probes Exist — The Problem First

Chapter 10 used a real readiness probe (`pg_isready`) and saw it gate
StatefulSet startup correctly — proof probes *can* work. This chapter
asks the sharper question: what happens when an app is technically
running but actually broken?

```
Container process alive (status: Running)
Application inside could still be:
  → deadlocked, never responding
  → crashed internally while the process stays up
  → still booting, not yet able to handle real traffic

Without probes, K8s only checks "is the PROCESS alive?"
A deadlocked app that never crashes shows Running forever.
Traffic keeps getting routed to it regardless.
```

---

## 🔑 Three Probe Types, Three Different Questions

```
livenessProbe   → "Is this still alive, or should it be restarted?"
                   fails repeatedly → kubelet KILLS and RESTARTS it

readinessProbe  → "Is this ready for traffic RIGHT NOW?"
                   fails → REMOVED from Service Endpoints, NOT restarted

startupProbe    → "Has this slow-starting app finished booting?"
                   blocks liveness/readiness checks until it succeeds
```

**The critical distinction:** liveness failure is destructive (restart).
Readiness failure is non-destructive (just stop routing traffic here).
Using readiness logic as a liveness check can cause restart loops on an
app that's only briefly busy, not actually broken.

---

## 📄 Files

### `fake-healthy.yaml` — the baseline gap, no probes at all

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fake-healthy
  namespace: dev-backend
  labels:
    app: fake-healthy
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 3600"]
```

Deletes its own health marker after 30s, then sits broken forever —
simulating silent internal failure with no probe watching for it.

### `liveness-demo.yaml` — same break, but now detected and acted on

```yaml
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

| Field | Why |
|-------|-----|
| `initialDelaySeconds: 5` | wait before the first check — avoids killing something still booting |
| `periodSeconds: 10` | how often to recheck |
| `failureThreshold: 3` | require 3 consecutive failures before restarting — avoids one-off blips causing unnecessary restarts |

### `readiness-demo.yaml` + Service — non-destructive failure path

```yaml
readinessProbe:
  exec:
    command: ["cat", "/tmp/ready"]
  initialDelaySeconds: 5
  periodSeconds: 5
```

Paired with a Service (`readiness-svc`) specifically so the effect on
`Endpoints` — not just pod status — can be observed directly.

---

## 🔧 All Commands

```bash
# ── Apply ────────────────────────────────────────────────
kubectl apply -f fake-healthy.yaml
kubectl apply -f liveness-demo.yaml
kubectl apply -f readiness-demo.yaml

# ── Baseline gap (no probes) ─────────────────────────────
kubectl get pods -n dev-backend -l app=fake-healthy -w
kubectl exec -it fake-healthy -n dev-backend -- ls /tmp/healthy

# ── Liveness — destructive ───────────────────────────────
kubectl get pods -n dev-backend -l app=liveness-demo -w
kubectl describe pod liveness-demo -n dev-backend

# ── Readiness — non-destructive ──────────────────────────
kubectl get endpoints readiness-svc -n dev-backend
kubectl exec -it -n dev-backend $(kubectl get pod -n dev-backend -l app=readiness-demo -o jsonpath='{.items[0].metadata.name}') -- rm /tmp/ready
kubectl get pods -n dev-backend -l app=readiness-demo -w
kubectl get endpoints readiness-svc -n dev-backend

# ── Cleanup ──────────────────────────────────────────────
kubectl delete -f fake-healthy.yaml
kubectl delete -f liveness-demo.yaml
kubectl delete -f readiness-demo.yaml
```

---

## 🧪 What Actually Happened — Real Results

### A genuinely unusual bug — a non-breaking space broke `exec`

While attempting the bonus exercise (recreating `/tmp/ready` to test
readiness self-recovery), this error appeared:

```
exec: "touch\u00a0": executable file not found in $PATH
command terminated with exit code 127
```

The space between `touch` and `/tmp/ready` had been substituted with a
**non-breaking space** (`\u00a0`) somewhere during typing or pasting —
likely carried over from a chat interface or document that silently
converts regular spaces. Kubernetes attempted to execute a binary
literally named `touch<nbsp>` as one word, which doesn't exist, hence
the `not found in $PATH` error. This is a categorically different kind
of mistake from anything else in this learning journey so far — not a
logic error or missing resource, but an invisible character riding
along in otherwise-correct-looking text.

**Secondary finding from the same mistake:** the retried commands ran
successfully, but against the wrong pods — `liveness-demo` and
`fake-healthy` instead of `readiness-demo`. Neither of those pods has a
readiness probe checking `/tmp/ready` at all, so the commands succeeded
but had no meaningful effect. A command "working" (exit code 0) is not
the same as a command testing the right thing — worth checking which
pod a label selector actually resolved to, not just whether the command
itself errored.

### Readiness self-healing — confirmed, fully automatic

Re-run correctly against the actual `readiness-demo` pod (typed fresh,
avoiding the earlier invisible-character issue):

```bash
kubectl exec -it readiness-demo-fd999fdc6-gxl5s -n dev-backend -- touch /tmp/ready
```

```
NAME                             READY   STATUS    RESTARTS       AGE
readiness-demo-fd999fdc6-gxl5s   1/1     Running   1 (104m ago)   17h
```

```
NAME            ENDPOINTS        AGE
readiness-svc   10.244.0.70:80   17h
```

`READY` returned to `1/1` and the pod's real IP reappeared in
`Endpoints` — with zero manual intervention beyond recreating the one
file. The `RESTARTS: 1` shown is timestamped `104m ago`, well before
this test, confirming it's unrelated background history, not something
this test triggered. Kubernetes' own periodic readiness check (running
every `periodSeconds: 5`) detected the file's return on its own and
re-added the pod to the Service's routing table — no restart, no manual
re-registration, no intervention beyond the file itself reappearing.

This is the complete proof for the chapter: liveness failures cause
destructive restarts (Step 2); readiness failures cause non-destructive
removal from traffic routing, with fully automatic recovery the moment
the underlying condition resolves (this step).

---

## 💡 Key Takeaways

1. **"Running" only means the container process hasn't exited** — it says nothing about whether the application inside is actually functional; probes close that specific gap
2. **Liveness failure is destructive (restart); readiness failure is not** — conflating the two, or using one where the other belongs, is a real and common production mistake
3. **`failureThreshold` exists to absorb transient blips** — a single failed check shouldn't trigger a restart; consecutive failures should
4. **Invisible characters (like non-breaking spaces) can silently break shell commands** with confusing, hard-to-read error messages — worth re-typing a failing command from scratch rather than re-pasting it if the error mentions an "executable not found" for something that should obviously exist
5. **A command exiting successfully does not guarantee it targeted the right resource** — always confirm which pod a label selector actually resolved to, especially when multiple similarly-named demo pods exist in the same namespace

---

## 🏋️ Exercises Completed

- [x] Deployed `fake-healthy` — confirmed it stays `Running` even after internal breakage, with no probe watching
- [x] Deployed `liveness-demo` — confirmed the same breakage triggers an actual restart this time
- [x] Found the `Liveness probe failed` event in `kubectl describe pod`
- [x] Deployed `readiness-demo` + Service — confirmed Endpoints populated initially
- [x] Manually broke readiness — confirmed `READY` dropped while `RESTARTS` stayed at zero, and Endpoints emptied
- [x] Bonus: confirmed self-recovery of readiness without restart — `READY` returned to `1/1`, Endpoints reappeared, `RESTARTS` unaffected, entirely automatic

---

*Status: ✅ Complete*
*Next: 15 — Taints & Tolerations*
*Platform: macOS · minikube · Kubernetes*
*Namespace: `dev-backend`*