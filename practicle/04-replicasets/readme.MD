# 04 - ReplicaSets

## What problem it solves

A Pod, on its own, has no self-healing. If it crashes, gets evicted, or
the node dies, nothing brings it back - you'd have to notice and recreate
it by hand. That doesn't work once real traffic depends on it. You need
something constantly watching and asking "I should have N of these
running, do I actually have N?" and fixing it the instant that's false.

A ReplicaSet is exactly that - a controller whose only responsibility is
keeping a fixed number of matching pods alive, always.

## How to work with it

**Declare what you want, not what to do.** You don't tell it "create a
pod" - you tell it the end state, and it figures out the steps:

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-demo
  template:        # the pod blueprint it stamps out when short
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**Apply it, then stop micromanaging pods:**
```bash
kubectl apply -f  replicaset.yaml -n  replicasets
kubectl get pods -n  replicasets
```

**Let it prove itself** by killing a pod manually - the only way to
actually see the mechanism work rather than just read about it:
```bash
kubectl delete pod <name> -n  replicasets
kubectl get pods -n  replicasets -w
```
A replacement appears on its own within seconds.

**Scale it without touching pods directly:**
```bash
kubectl scale rs nginx-rs --replicas=5 -n  replicasets
```
It adds pods from the same template - you never pick which ones, it
decides.

**The one rule that explains its behavior:** it matches pods by **label**,
never by name. Manually relabel a running pod and the ReplicaSet stops
counting it - it sees a gap and creates a new one, while your relabeled
pod keeps running, unmanaged. That single fact explains almost every
"weird ReplicaSet behavior" you'll hit later.

In daily work you'll almost never write one of these by hand - a
Deployment generates and manages it for you, adding rolling
updates/rollbacks on top. This chapter isolates the mechanism so you can
see it without that extra layer - so when something behaves strangely
inside a Deployment later, you'll know exactly which layer to look at.

## Files in this folder

| File                  | Purpose                                              |
|------------------------|-------------------------------------------------------|
| ` namespace.yaml`    | Creates the ` replicasets` namespace for isolation |
| ` replicaset.yaml`   | 3-replica nginx ReplicaSet, scoped to that namespace |

## How to run

```bash
kubectl apply -f  namespace.yaml
kubectl apply -f  replicaset.yaml
kubectl get pods -n  replicasets
```

Optional - stop typing `-n  replicasets` on every command:

```bash
kubectl config set-context --current --namespace= replicasets
```

Revert later with:

```bash
kubectl config set-context --current --namespace=default
```

## Test self-healing

```bash
kubectl get pods -n  replicasets -w
```

In a second terminal:

```bash
kubectl delete pod <one-pod-name> -n  replicasets
```

A replacement pod should appear within seconds in the first terminal.

## Inspect ownership

```bash
kubectl get pod <pod-name> -n  replicasets -o yaml | grep -A6 ownerReferences
```

Confirms the ReplicaSet as the controlling owner (`controller: true`).

## Try the orphan trap

```bash
kubectl label pod <pod-name> -n  replicasets app=detached --overwrite
kubectl get pods -n  replicasets --show-labels
```

The relabeled pod keeps running but is no longer managed - a fresh
replacement gets created in its place.

## Cleanup

```bash
kubectl delete namespace  replicasets
```

Deletes the namespace and everything inside it (ReplicaSet + pods) without
touching any other chapter.

## Check your understanding

- Why does the ReplicaSet match pods by label instead of by name?
- What's the actual difference in responsibility between a Deployment and
  the ReplicaSet it creates?
- If you `kubectl scale rs --replicas=5` directly on a ReplicaSet that a
  Deployment owns, what happens on the next reconciliation pass, and why?