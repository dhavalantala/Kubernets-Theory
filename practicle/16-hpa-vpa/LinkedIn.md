# LinkedIn Post — HPA (Horizontal Pod Autoscaler)

---

🚀 Day 16 of my Kubernetes journey — watched a single pod become five, with zero manual commands, purely because I made it work hard.

The chapter: Horizontal Pod Autoscaler. Every scaling action I've done before this was manual — `kubectl scale --replicas=5`, a human typing a number. Today's goal: let Kubernetes decide that number on its own, based on real CPU pressure.

Got there through two real configuration mistakes first, which honestly taught me more than a clean setup would have.

Mistake one: I had the CPU request commented out in my YAML. Result:
```
TARGETS: cpu: <unknown>/50%
```
Not 0%. <unknown>. Forever. Turns out HPA's whole calculation is current usage divided by what you requested — with no request defined, there's no denominator. It's not being cautious, it's mathematically unable to compute anything.

Mistake two, stacked on top: my image was pulling from a deprecated registry path (`k8s.gcr.io` → should be `registry.k8s.io` now). Fixed both, redeployed.

Still stuck on `<unknown>` for nearly 3 more minutes — even though `kubectl top pods` was already showing real numbers. Had to dig into `kubectl describe hpa` to find out why, and the Events section explained it precisely: a stale ReplicaSet from my earlier broken version was still being briefly evaluated, plus the HPA controller's own internal sync loop runs on a separate, slower polling interval than direct metrics queries. Not broken — just genuinely still warming up.

Then the actual test. Spun up a load generator hammering the pod nonstop:

```
cpu: 0%/50%      REPLICAS: 1
cpu: 35%/50%     REPLICAS: 1
cpu: 249%/50%    REPLICAS: 1
...
REPLICAS: 4
REPLICAS: 5   ← hit the configured max
```

249% against a 50% target — nearly 5x over. Kubernetes scaled from 1 pod to 5 entirely on its own, reacting to real, sustained pressure, with the only human input being "start the load and watch."

The lesson underneath the lesson: two stacked, unrelated config mistakes can produce a single confusing symptom (`<unknown>`/`REPLICAS: 0`), and the fix is checking each layer independently rather than assuming one fix solves everything. Same instinct that's carried through every chapter so far — verify, don't assume.

What's the most satisfying "it just scaled itself" moment you've had with your infrastructure?

#Kubernetes #DevOps #K8s #HPA #Autoscaling #CloudNative #Learning #PlatformEngineering #DevOpsJourney