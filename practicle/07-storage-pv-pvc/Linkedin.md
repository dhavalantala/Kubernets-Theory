# LinkedIn Post — PersistentVolume & PersistentVolumeClaim

---

🚀 Day 7 of my Kubernetes journey — and today's lesson hit different.

I deliberately tried to delete a PersistentVolumeClaim while a Pod was still using it.

The command just... hung. No error, no response.

Turns out that's not a bug — it's Kubernetes refusing to let me rip storage out from under a running workload. It only completed after I deleted the Pod first.

That one stuck moment taught me more about PV/PVC than any diagram could.

Here's the setup I practiced:

PersistentVolume (PV) → the actual storage resource, managed like infrastructure
PersistentVolumeClaim (PVC) → an app's request for storage, "I need 1Gi, don't care how"

Why split into two objects?
App teams write PVCs without knowing if storage is AWS EBS, GCP disk, or local.
Infra teams manage PVs. Same separation of concerns as the rest of the platform.

The real proof of why this matters:

I wrote a file inside a Pod backed by a PVC, then FULLY deleted the Pod and recreated it from scratch — brand new container, brand new filesystem.

The old data was still there. Pods are disposable. The data isn't.

Then I deleted the PVC (after freeing the Pod) and checked the PV:
```
STATUS: Released
```

Not destroyed. Not available again automatically. Released — meaning a human has to consciously decide whether that data gets reused or wiped.

That's the entire philosophy of the `Retain` reclaim policy in one word: no silent data loss, ever, by default.

Building this with real namespace structure too now — `dev-frontend`, `dev-backend`, `dev-database` — applied once via an `_infra/` folder, kept separate from chapter workloads, so cleanup of one chapter's resources can never accidentally take down the namespace itself.

Small architectural decisions like that are exactly the kind of thinking I'm trying to build, not just memorize YAML.

What's the most "aha" Kubernetes moment you've had?

#Kubernetes #DevOps #K8s #PersistentVolume #CloudNative #Learning #PlatformEngineering #DevOpsJourney