# 03 — Deployments

> **Video timestamp:** `01:44:00`  
> **Cluster:** Minikube (local)

---

## What is a Deployment?

A **Deployment** is a higher-level controller that manages Pods for you. It gives you:

- **Self-healing** — if a pod crashes, the deployment recreates it
- **Scaling** — run multiple replicas with one command
- **Rolling updates** — update your app with zero downtime
- **Rollback** — instantly revert to a previous version

You almost never create pods directly in production. You always create a Deployment.

---

## Deployment → ReplicaSet → Pod (Hierarchy)

```
Deployment
    └── ReplicaSet (manages desired state)
            └── Pod  ←  Pod  ←  Pod
```

When you create a Deployment, it automatically creates a ReplicaSet, which automatically creates the Pods.

---

## Key Commands

```bash
# Create a deployment (imperative)
kubectl create deployment nginx-deploy --image=nginx --replicas=3

# Apply from YAML (declarative — preferred)
kubectl apply -f deployment.yaml

# List deployments
kubectl get deployments
kubectl get deploy
kubectl get deploy -o wide

# Get detailed info (great for debugging rollouts)
kubectl describe deployment nginx-deploy

# Scale up/down
kubectl scale deployment nginx-deploy --replicas=5
kubectl scale deployment nginx-deploy --replicas=1

# Update image (triggers a rolling update)
kubectl set image deployment/nginx-deploy nginx=nginx:1.21

# Check rollout status (watch it happen live)
kubectl rollout status deployment/nginx-deploy

# View rollout history
kubectl rollout history deployment/nginx-deploy
kubectl rollout history deployment/nginx-deploy --revision=2

# Rollback to previous version
kubectl rollout undo deployment/nginx-deploy

# Rollback to a specific revision
kubectl rollout undo deployment/nginx-deploy --to-revision=1

# Pause a rollout (useful to canary test)
kubectl rollout pause deployment/nginx-deploy
kubectl rollout resume deployment/nginx-deploy

# Delete deployment (also deletes its ReplicaSets and Pods)
kubectl delete deployment nginx-deploy
kubectl delete -f deployment.yaml
```

---

## YAML — Basic Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 3                     # how many pods to run
  selector:
    matchLabels:
      app: nginx                  # must match pod template labels
  template:                       # pod template starts here
    metadata:
      labels:
        app: nginx                # these labels are matched by the selector above
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

---

## YAML — Deployment with Rolling Update Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-rolling
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # max extra pods during update (above desired count)
      maxUnavailable: 1    # max pods that can be down during update
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
```

---

## YAML — Deployment with Recreate Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-recreate
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: Recreate    # Kill ALL old pods first, then create new ones (causes downtime)
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

---

## Rolling Update vs Recreate

| | RollingUpdate | Recreate |
|---|---|---|
| Downtime | Zero downtime | Brief downtime |
| How it works | Replace pods one by one | Kill all, then start all |
| Use case | Most web apps | DB migrations, breaking changes |
| Default? | ✅ Yes | No |

---

## Deployment Rollout Flow (Rolling Update)

```
Old:  [v1] [v1] [v1] [v1]
          ↓  update image
Step 1: [v1] [v1] [v1] [v2]   ← +1 new, 1 old stays
Step 2: [v1] [v1] [v2] [v2]
Step 3: [v1] [v2] [v2] [v2]
Step 4: [v2] [v2] [v2] [v2]   ← complete
```

---

## Hands-on Practice Checklist

- [ ] Create a deployment with 3 replicas of nginx
- [ ] Run `kubectl get pods` — see 3 pods running
- [ ] Delete one pod manually — watch Kubernetes recreate it automatically
- [ ] Scale the deployment to 5 replicas
- [ ] Update the image from `nginx:1.20` to `nginx:1.21`
  - Watch the rolling update: `kubectl rollout status deployment/nginx-deploy`
- [ ] Check rollout history: `kubectl rollout history deployment/nginx-deploy`
- [ ] Rollback to the previous version: `kubectl rollout undo deployment/nginx-deploy`
- [ ] Try the `Recreate` strategy and observe the downtime
- [ ] Pause a rollout mid-way and resume it
- [ ] Delete the deployment and confirm all pods are also deleted

---

## Interview Questions

**Q: What is the difference between a Deployment and a ReplicaSet?**  
A ReplicaSet ensures N pods are always running. A Deployment wraps the ReplicaSet and adds update/rollback capabilities. You should always use Deployments, not ReplicaSets directly.

**Q: How does a rolling update achieve zero downtime?**  
It replaces pods incrementally — `maxUnavailable` ensures some old pods stay running while `maxSurge` allows temporarily running extra pods during the transition.

**Q: What happens to the old ReplicaSet after a rolling update?**  
Kubernetes keeps old ReplicaSets (scaled down to 0 replicas) so you can rollback. The number kept is controlled by `revisionHistoryLimit` (default: 10).

**Q: How does Kubernetes know which pods belong to a Deployment?**  
Via **label selectors**. The `.spec.selector.matchLabels` must match `.spec.template.metadata.labels`. This is the link between the Deployment and its pods.

---

## Files in This Directory

```
03-deployments/
├── README.md                      ← this file
├── deployment.yaml                ← basic deployment
├── deployment-rolling.yaml        ← rolling update strategy
└── deployment-recreate.yaml       ← recreate strategy
```