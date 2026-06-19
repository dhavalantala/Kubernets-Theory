#  05 - DaemonSets

## What problem it solves

A Deployment/ReplicaSet answers "give me N pods somewhere in the cluster" -
it doesn't care which nodes, and it doesn't auto-adjust when nodes are
added or removed. Some workloads need the opposite guarantee: exactly one
pod on every node, automatically, no matter how the cluster grows or
shrinks.

Think log collectors (Fluentd, Filebeat), node monitoring agents
(node-exporter), or networking components - anything that needs to watch
*that specific node*, not "some pod somewhere." A ReplicaSet can't express
that. If you set `replicas: 5` and the cluster scales to 8 nodes, 3 nodes
go silently unmonitored.

## How to work with it

A DaemonSet has no `replicas` field - that's the tell. Instead:

- It watches the node list.
- A new node joins -> it schedules a pod there immediately.
- A node leaves -> that pod goes with it.
- One pod per matching node, period - you don't control the count, you
  control *which nodes qualify* (via `nodeSelector`, affinity, or
  tolerations).

```yaml
spec:
  selector:
    matchLabels:
      app: node-logger
  template:
    metadata:
      labels:
        app: node-logger
    spec:
      containers:
      - name: logger
        image: busybox:1.36
```

By default, DaemonSet pods automatically tolerate the
`node-not-ready`/`unreachable` taints so they keep running even if a node
briefly has trouble - that's intentional, a log collector should stay put
through transient issues. They do **not** automatically tolerate every
taint though (see Troubleshooting below) - covered properly in chapter 15.

## Files in this folder

| File                 | Purpose                                              |
|----------------------|-------------------------------------------------------|
| ` -namespace.yaml`  | Creates the ` -daemonsets` namespace for isolation  |
| ` -daemonset.yaml`  | DaemonSet running a simple logger on every node      |

## How to run

```bash
kubectl apply -f  -namespace.yaml
kubectl apply -f  -daemonset.yaml
kubectl get pods -n  -daemonsets -o wide
```

The `-o wide` matters - check the `NODE` column.

## Test the "one pod per node" behavior

```bash
minikube node add
kubectl get pods -n  -daemonsets -o wide -w
```

A second pod should appear automatically on the new node.

```bash
kubectl logs -n  -daemonsets -l app=node-logger
```

## Troubleshooting: new node, no new pod

If the DaemonSet doesn't create a pod on a newly added node, check:

```bash
kubectl get nodes
kubectl describe node <new-node-name>
```

Look at `Taints:` and `Conditions:`. The common cause on minikube: the
cluster was created without a CNI plugin, so the new node gets a
`network-unavailable` taint (NoSchedule) until pod networking is wired up.
DaemonSet pods are scheduled by the regular scheduler and respect this
taint - they don't get a free pass on it.

Fix options:

- Recreate the cluster with multi-node + CNI specified up front:
  ```bash
  minikube delete --all
  minikube start --nodes=2 --cni=true
  ```
- Or drop the extra node and treat multi-node CNI behavior as something
  you'll get for free later (the minikube 3-tier project and EKS project
  both have working networking by default):
  ```bash
  minikube node delete m02
  ```

## Cleanup

```bash
kubectl delete namespace  -daemonsets
```

## Check your understanding

- Why can't a ReplicaSet/Deployment give you the same guarantee as a
  DaemonSet, even if you set `replicas` equal to your node count?
- If you wanted a DaemonSet to skip certain nodes entirely (e.g. GPU
  nodes), what would you add to the spec?
- Why does a tainted/not-ready node block new DaemonSet pods from being
  scheduled, even though already-running DaemonSet pods tolerate that
  node briefly going unhealthy?