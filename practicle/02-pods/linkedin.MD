# LinkedIn Post — Pods

---

🚀 Day 2 of my Kubernetes journey — I watched my app crash 7 times and smiled 😄

Here's why that's actually a good thing.

Today I learned about Pods — the smallest deployable unit in Kubernetes.

Coming from Docker, my first question was:
"Why can't K8s just run containers directly?"

The answer changed how I think about container orchestration.

A Pod gives your container things Docker alone can't guarantee at scale:
→ A unique IP address inside the cluster
→ Shared network between containers (they talk via localhost)
→ Shared storage volumes
→ Lifecycle management

Think of it like this:
Container = a person
Pod = that person's office (has its own address, shared resources)

The most eye-opening moment today?

I created a pod that intentionally crashes after 5 seconds:
```yaml
command: ["sh", "-c", "echo Pod started && sleep 5 && exit 1"]
```

Then watched it with:
```bash
kubectl get pods -w
```

Output:
```
crash-pod   Running            0
crash-pod   Error              0
crash-pod   CrashLoopBackOff   1
crash-pod   CrashLoopBackOff   2
...
crash-pod   CrashLoopBackOff   7  ← still going!
```

K8s never gives up. It keeps restarting with exponential backoff.
But here's the problem — it's restarting ONE pod.
What if you need 10 pods always running?
What if you need zero-downtime updates?

That's exactly why Deployment exists. (Coming tomorrow 👀)

Also shelled into a running pod today:
```bash
kubectl exec -it nginx-pod -- sh
wget -O- localhost  # nginx response from INSIDE the cluster ✅
```

Same as docker exec — but for Kubernetes.

The journey from ML Engineer → DevOps is real. One concept at a time. 💪

What's the first K8s concept that clicked for you?

#Kubernetes #DevOps #K8s #Pods #CloudNative #Learning #MLOps #DevOpsJourney