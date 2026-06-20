# 09 — Ingress 🚪

> A single entry point that routes external traffic to many internal Services.
> This chapter took far longer than expected — and that's exactly why it's the most valuable one so far.

---

## 📁 Folder Structure

```
09-ingress/
├── backend-deployment.yaml      # backend pods, in dev-backend
├── backend-service.yaml         # backend-svc, in dev-backend
├── frontend-deployment.yaml     # frontend pods, in dev-frontend
├── frontend-service.yaml        # frontend-svc, in dev-frontend
├── backend-svc-pointer.yaml     # ExternalName Service in dev-frontend — the actual fix
├── ingress.yaml                 # routing rules
└── README.md
```

Namespaces used (from `_infra/namespaces.yaml`):
```
dev-backend     ← real backend Deployment + Service live here
dev-frontend    ← real frontend Deployment + Service, AND the Ingress object itself
```

---

## 🧠 Why Ingress Exists — The Problem First

Chapter 08 proved Services give stable internal addresses. But:

```
5 different backend Services, each needing external access
With only Service type=LoadBalancer:
  → 5 separate cloud load balancers
  → 5 separate external IPs
  → 5 separate monthly bills 💸
  → no shared TLS strategy, no shared routing logic
```

Ingress solves this with one entry point, routed by path or hostname,
to many internal Services behind it.

```
Ingress object       = the routing RULES (a config)
Ingress Controller    = the actual software that READS those rules
                        and performs the routing

An Ingress with no controller running does nothing — silently.
```

On minikube the controller must be explicitly enabled:
```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

---

## ⚠️ The Real Lesson of This Chapter — Ingress Backend Scoping

**This corrects something stated incorrectly earlier in this learning
journey, discovered only through hands-on debugging:**

```
WRONG assumption going in:
  "An Ingress can route to a Service in any namespace,
   just by naming it in the backend."

ACTUAL Kubernetes rule:
  A plain Ingress's backend.service.name ALWAYS resolves within
  the Ingress object's OWN namespace. There is no native
  cross-namespace backend reference for a standard Ingress.
```

This was proven directly via the ingress controller's own logs:

```
Error obtaining Endpoints for Service "dev-frontend/backend-svc":
no object matching key "dev-frontend/backend-svc" in local store
```

and confirmed via `kubectl describe ingress`:

```
/api   backend-svc:80 (<error: services "backend-svc" not found>)
```

The Ingress lived in `dev-frontend`. It looked for `backend-svc` inside
`dev-frontend`. The real `backend-svc` lived in `dev-backend`. Two
different namespaces, same Service name — Kubernetes does not bridge
that gap automatically.

### The fix — `ExternalName` Service as a cross-namespace pointer

This is the actual pattern platform teams use when an edge-routing layer
legitimately needs to reach a Service owned by a different namespace:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: dev-frontend
spec:
  type: ExternalName
  externalName: backend-svc.dev-backend.svc.cluster.local
  ports:
    - port: 80
```

This creates a Service named `backend-svc` *inside* `dev-frontend` that
does nothing but forward via DNS to the real Service in `dev-backend`.
The Ingress's reference now resolves locally, satisfying Kubernetes'
namespace-scoping rule, while transparently redirecting to the actual
backend elsewhere.

```bash
kubectl apply -f backend-svc-pointer.yaml
```

This is the single change that took `/api` from a `503` to
`Hello from BACKEND`.

---

## 📄 Other Files

### `ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: dev-frontend
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

Note: the `rewrite-target` annotation seen in many tutorials was tried
and removed during debugging — it was **not** the cause of the 503 in
this case. Worth knowing it's a real, separate feature (rewrites the
request path before forwarding) but it was a red herring here.

---

## 🍎 macOS-Specific Networking Reality (Docker Driver)

This is the part that consumed most of this chapter's debugging time,
and it had nothing to do with the YAML at all.

```
minikube ip  →  192.168.49.2

ping -c 3 192.168.49.2
→ Destination Host Unreachable

This proves: on macOS with the Docker driver, the minikube "node"
is a Docker container whose IP is NOT directly reachable from the
Mac's network stack — unlike Linux hosts or VM-based drivers.
```

**Things that did NOT fix this, tried in order:**
1. Fixing `/etc/hosts` — necessary, but not sufficient alone
2. `minikube tunnel` — works in theory, proved genuinely unreliable
   for privileged ports (80/443) on this specific driver/OS combo
3. Curling the NodePort directly via the minikube IP — failed for the
   same root reason as the ping test (`Destination Host Unreachable`)

**What actually worked:**

```bash
minikube service ingress-nginx-controller -n ingress-nginx --url
```

This prints local `127.0.0.1:<random-port>` URLs that minikube
specifically sets up to be reachable from the Mac, bypassing the
Docker-container IP entirely. **This command must stay running in its
own terminal** — closing it (even accidentally with Ctrl+C) kills the
forwarded connection instantly.

```bash
curl -H "Host: myapp.local" http://127.0.0.1:<port>/
curl -H "Host: myapp.local" http://127.0.0.1:<port>/api
```

The `Host` header is required here because we're bypassing DNS/`/etc/hosts`
entirely — we're hitting a raw forwarded port, so nginx-ingress needs to
be told explicitly which virtual host this request is meant to match.

---

## 🔧 All Commands — Full Working Sequence

```bash
# ── Namespaces (applied once, from _infra) ──────────────
kubectl apply -f ~/Desktop/k8s-practice/_infra/namespaces.yaml

# ── Workloads ────────────────────────────────────────────
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml

# ── Ingress controller ───────────────────────────────────
minikube addons enable ingress
kubectl get pods -n ingress-nginx     # wait for Running, 1/1

# ── Cross-namespace pointer (the critical fix) ──────────
kubectl apply -f backend-svc-pointer.yaml

# ── Ingress rules ────────────────────────────────────────
kubectl apply -f ingress.yaml
kubectl get ingress -n dev-frontend
kubectl describe ingress app-ingress -n dev-frontend

# ── macOS-reachable access (separate terminal, leave running) ─
minikube service ingress-nginx-controller -n ingress-nginx --url

# ── Test (from a DIFFERENT terminal) ─────────────────────
curl -H "Host: myapp.local" http://127.0.0.1:<port>/
curl -H "Host: myapp.local" http://127.0.0.1:<port>/api

# ── Debugging tools used along the way ──────────────────
kubectl get pods -n dev-backend -o wide
kubectl get endpoints backend-svc -n dev-backend
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=50
ps aux | grep "minikube service"
```

---

## 🧪 Real Debugging Timeline — What Actually Happened

```
1. DNS failure          → myapp.local not in /etc/hosts
   Fix: added the line  → resolved, but...

2. Connection refused on port 80
   → minikube tunnel attempted, appeared to exit silently
   → confirmed via `ps aux` it genuinely wasn't running
   → re-ran, confirmed alive via ps aux this time
   → STILL failed (Docker driver limitation, not a tunnel mistake)

3. Discovered a SECOND, unrelated problem mid-debug:
   minikube had silently become a 2-node cluster
   (minikube-m02 worker, kubelet: Stopped)
   → minikube delete + minikube start → clean single-node cluster
   → had to reapply namespaces + all workloads from scratch

4. Direct NodePort IP curl → "Destination Host Unreachable"
   → proved via ping that 192.168.49.2 is unreachable from
     the Mac directly — root cause of every port-80 failure so far

5. minikube service --url → finally a reachable local port
   → first real success: "/" returned "Hello from FRONTEND" ✅

6. "/api" still failing → 503 Service Temporarily Unavailable
   → NEW, separate bug — not a networking issue this time
   → tried removing rewrite-target annotation — did not fix it
   → checked controller logs directly:
     "Error obtaining Endpoints for Service dev-frontend/backend-svc"
   → root cause: Ingress backends resolve within the INGRESS'S
     OWN namespace, not the target Service's namespace
   → fix: ExternalName Service as a same-namespace pointer
   → "/api" finally returned "Hello from BACKEND" ✅
```

---

## 💡 Key Takeaways

1. **Ingress backends resolve in the Ingress's own namespace** — there is no native cross-namespace `backend.service` reference; use an `ExternalName` Service as a pointer when you genuinely need this
2. **`minikube tunnel` and `minikube service --url` must stay running** in their own terminal — closing them (even by accident) kills the connection instantly with no warning
3. **On macOS with the Docker driver, the minikube node's IP is not directly reachable** from the Mac — this is a structural networking fact, not a misconfiguration, and `minikube service --url` is the reliable workaround
4. **A `503` from nginx specifically means it found the routing rule but couldn't reach a healthy backend** — check `kubectl get endpoints` and the controller's own logs before suspecting the Ingress YAML itself
5. **Always verify a background process is genuinely alive** with `ps aux`, not just by trusting a clean-looking terminal — a silently exited process produces confusing, inconsistent symptoms
6. **A two-node cluster can appear silently** (e.g. from an earlier `--nodes` flag) and cause failures that look identical to networking misconfiguration — always sanity check `minikube status` early when debugging connectivity

---

## 🏋️ Exercises Completed

- [x] Enabled ingress controller — confirmed `Running`, `1/1`
- [x] Deployed backend + frontend Deployments and Services in separate namespaces
- [x] Created Ingress with path-based routing rules
- [x] Hit a real cross-namespace backend resolution failure — diagnosed via controller logs
- [x] Fixed it with an `ExternalName` Service pointer
- [x] Worked around macOS Docker-driver networking limitations via `minikube service --url`
- [x] Verified `/` → frontend, `/api` → backend, both correctly routed through one entry point

---

*Status: ✅ Complete — the hard way, which is the way that actually teaches it*
*Next: 10 — StatefulSets*
*Platform: macOS · minikube (Docker driver) · Kubernetes*
*Namespaces: `dev-backend` (real backend), `dev-frontend` (real frontend + Ingress + cross-namespace pointer)*