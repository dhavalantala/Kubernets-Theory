# LinkedIn Post — Sidecar / Init Containers

---

🚀 Day 20 of my Kubernetes journey — today's most interesting finding came from a command that returned nothing.

The chapter: Init Containers and Sidecar Containers. Two different patterns for running multiple containers in one pod, solving two completely different problems.

Init containers run BEFORE the main app, must succeed before the main app is allowed to start. Sidecar containers run ALONGSIDE the main app for its entire lifetime.

I proved the init container behavior by deliberately breaking one:

```
NAME        READY   STATUS
init-fail   0/1     Init:CrashLoopBackOff
init-fail   0/1     Init:Error
init-fail   0/1     Init:CrashLoopBackOff
```

The main container's column never appeared. Not `0/1`, not `Pending`. Just absent. That's the whole point — a broken init container actively prevents the app from starting in a broken state, rather than letting it fail later in a harder-to-diagnose way.

Then the sidecar. A main-app container writing timestamped log entries to a file every 3 seconds. A log-shipper container tailing that same file. Both in the same pod, sharing a volume, running concurrently.

```
kubectl logs sidecar-demo -c main-app
(empty)

kubectl logs sidecar-demo -c log-shipper
Fri Jul  3 12:21:07 UTC 2026: app log entry
Fri Jul  3 12:21:10 UTC 2026: app log entry
Fri Jul  3 12:21:13 UTC 2026: app log entry
```

The empty output from `main-app` looks like a failure. It isn't. `kubectl logs` reads stdout — `main-app` writes to a FILE, not to stdout. The sidecar's entire job is to bridge that: it reads the file, sends the content to its own stdout, making it visible to the logging system without the main app ever changing how it works.

The main app doesn't know the sidecar exists. They share purely through a volume. That's the pattern — and it's exactly how real log shipping works in production clusters with tools like Fluentd and Datadog.

One thing clicked that connected back to Chapter 07: the volume I used was `emptyDir`, which lives with the pod. Delete the pod, lose the logs. A PVC would survive pod deletion — same principle, different scope. Every chapter in this series has been building toward the same handful of underlying ideas, just expressed in different objects.

What's a "that output looks wrong but it's actually correct" moment you've had debugging your own systems?

#Kubernetes #DevOps #K8s #Sidecar #InitContainers #CloudNative #Learning #PlatformEngineering #DevOpsJourney