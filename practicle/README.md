# Kubernetes Object Model — Why Each Object Exists

A working mental model for the core Kubernetes objects, built around one question for each layer: *what real-world problem does this object solve that the simpler one below it couldn't?*

> **Note on the diagram below:** this is a *learning progression*, not a literal dependency tree. Deployment, StatefulSet, and DaemonSet are siblings — each is a controller solving a different problem on top of the same "watch and reconcile" pattern that ReplicaSet introduced. They don't depend on each other.

```
Pod            → needs auto-healing
ReplicaSet     → needs rolling updates + rollback
Deployment     → needs stable identity + per-pod storage
StatefulSet    → (sibling) needs exactly one copy per node
DaemonSet      → (sibling) needs a run-once task instead of "stay alive forever"
Job / CronJob  → (sibling) needs that run-once task on a schedule
```

## Table of Contents

- [Pod](#pod--the-smallest-unit)
- [ReplicaSet](#replicaset--keep-n-copies-running-always)
- [Deployment](#deployment--rolling-updates-and-rollback)
- [Service](#service-tbd)
- [StatefulSet](#statefulset--stable-identity--storage)
- [DaemonSet](#daemonset--run-exactly-one-pod-per-node)
- [Job / CronJob](#job--cronjob--run-once-vs-run-on-a-schedule)
- [PersistentVolume / PersistentVolumeClaim](#persistentvolume--persistentvolumeclaim)
- [Quick Reference](#quick-reference)

---

## Pod — the smallest unit

A Pod wraps one or more containers (usually one). Kubernetes never runs a container directly — it always wraps it in a Pod, because the scheduler and kubelet need extra metadata around the container: networking, restart policy, labels, resource limits.

```bash
kubectl run mypod --image=nginx
```

The problem with a bare Pod:

- If the container crashes, the Pod is gone for good.
- If the node dies, the Pod is gone for good.
- No automatic restart, no scaling.

This is exactly the gap ReplicaSet fills.

## ReplicaSet — keep N copies running, always

A ReplicaSet watches a set of Pods and ensures a fixed number are always running.

- Pod dies → ReplicaSet creates a replacement.
- Node dies → ReplicaSet recreates the Pod elsewhere.
- Need 3 copies for traffic? → ReplicaSet keeps exactly 3.

**Why isn't ReplicaSet enough on its own?** Consider shipping v2 of an app with *only* a ReplicaSet:

1. Manually create a new ReplicaSet pointing at the v2 image.
2. Manually scale the old ReplicaSet down to 0.
3. Manually scale the new one up.
4. If something breaks, manually reverse all of the above.

There's no history and no rollback — this is how teams end up debugging `ImagePullBackOff` at 2am with no way back to the last known-good state.

## Deployment — rolling updates and rollback

A Deployment manages a ReplicaSet for you and adds an update strategy and revision history on top of it.

```bash
kubectl set image deployment/myapp app=myapp:v2
```

What happens automatically:

1. Deployment creates a new ReplicaSet running v2.
2. Traffic shifts gradually from the old ReplicaSet to the new one (rolling update).
3. The old ReplicaSet is kept around in case a rollback is needed.
4. `kubectl rollout undo` reverts instantly.

| | ReplicaSet | Deployment |
|---|---|---|
| Keeps N pods alive | ✅ | ✅ |
| Update strategy | ❌ | ✅ Rolling update |
| Rollback | ❌ | ✅ `kubectl rollout undo` |
| Revision history | ❌ | ✅ |

In practice, you almost never create a ReplicaSet directly. You create a Deployment, and it creates and manages the ReplicaSet for you.

## Service (TBD)

Without a Service, Pods have no stable network identity — Pod IPs change every time a Pod is recreated, the same problem you'd hit with raw Docker networking. A Service gives a stable DNS name / virtual IP that load-balances across whichever Pods currently match its label selector.

*(This section is a placeholder — fill in once Services are covered in depth: ClusterIP, NodePort, LoadBalancer, and how the selector mechanism ties back to Pod labels.)*

## StatefulSet — stable identity + storage

Deployment Pods are interchangeable — random names (`pod-a7x9k`, `pod-b3m1p`), no guaranteed storage, fine for stateless apps like web servers and APIs.

Databases need more:

- Stable, predictable names (`mongodb-0`, `mongodb-1`, `mongodb-2`), not random ones.
- Stable storage — when a pod restarts, it reattaches to *its own* disk.
- Ordered startup — `mongodb-0` comes up before `mongodb-1`.

| | Deployment | StatefulSet |
|---|---|---|
| Pod naming | Random | Predictable: `app-0`, `app-1`, ... |
| Storage | Shared or none | Each pod gets its own PVC |
| Start order | Any order | Ordered (0, 1, 2, ...) |
| Typical use | APIs, web servers | MongoDB, Postgres, Kafka |

If a database Pod restarted under a plain Deployment, it could come back attached to a *different* empty disk and lose its data. StatefulSet guarantees `pod-0` always reconnects to the same disk it had before.

## DaemonSet — run exactly one Pod per node

A Deployment with `replicas: 5` on a 5-node cluster gives no guarantee of distribution — the scheduler might place 3 Pods on node-1 and 2 on node-2.

A DaemonSet guarantees one Pod on every node, and automatically schedules a Pod onto any new node that joins the cluster.

**Typical use:** log collectors and monitoring agents (Fluentd, Datadog agent) that need to run everywhere, no exceptions.

## Job / CronJob — run-once vs. run on a schedule

A Deployment's entire purpose is to keep a Pod alive indefinitely. Running a one-time script (say, a data migration) as a Deployment backfires:

1. The script finishes and the container exits.
2. The Deployment sees the Pod as "died" and restarts it.
3. The script runs again — and again.

A **Job** runs a Pod to completion exactly once and does not restart it afterward. Typical uses: database migrations, batch processing, a single model training run.

A **CronJob** runs a Job on a schedule (hourly, daily, weekly) — the same relationship CronJob has to Job that Deployment has to ReplicaSet: it's the scheduling/automation layer on top. Typical uses: nightly backups, daily report generation, retraining a model every Sunday.

## PersistentVolume / PersistentVolumeClaim

A Pod's local filesystem dies with the Pod — the same limitation as an unmounted Docker container's writable layer.

- **PV (PersistentVolume):** the actual storage resource — a disk, a cloud volume, an NFS share.
- **PVC (PersistentVolumeClaim):** a Pod's request for storage ("I need 5GB").

A PVC binds to a PV, the Pod mounts the PVC, and the data survives the Pod being deleted and recreated. This maps directly onto Docker's named-volume model: just as a Docker volume survives a container's removal, a PV survives a Pod's removal.

## Quick Reference

| Object | Solves | Use it when |
|---|---|---|
| Pod | Runs containers | Never used alone in practice |
| ReplicaSet | Keeps N Pods alive | Rarely created directly — Deployment manages this for you |
| Deployment | Rolling updates + rollback | Default choice for stateless workloads |
| Service | Stable networking for Pods | Any time Pods need to be reachable reliably |
| StatefulSet | Stable identity + per-pod storage | Databases, message queues (MongoDB, Postgres, Kafka) |
| DaemonSet | One Pod per node | Log collectors, monitoring agents, node-level daemons |
| Job | Run-to-completion task | Migrations, batch jobs, one-off scripts |
| CronJob | Scheduled run-to-completion task | Nightly backups, scheduled reports |
| PV / PVC | Storage that outlives a Pod | Anything that needs to persist data across restarts |