# LinkedIn Post — ConfigMaps

---

🚀 Day 11 of my Kubernetes journey — today I proved a claim was true by deliberately trying to break it.

I'd been told: environment variables sourced from a ConfigMap are frozen at pod startup. Change the ConfigMap later, and a running pod's env vars won't update — only a restart would pick up the new value.

Easy to accept. Better to verify.

So I set up a pod with an env var wired from a ConfigMap:
```
APP_ENV=development
```

Then, WITHOUT touching the pod at all, I edited the ConfigMap directly:
```bash
kubectl edit configmap app-config -n dev-backend
```
Changed the value to `PRODUCTION-CHANGED`. Confirmed the ConfigMap itself updated:
```
APP_ENV: PRODUCTION-CHANGED
```

Then went back into the SAME still-running pod and checked its environment again:
```bash
kubectl exec -it <pod> -- env | grep APP_ENV
```
```
APP_ENV=development
```

Unchanged. Exactly as claimed — confirmed empirically, not just trusted.

Then I tested the other side of the same coin: mounting that same kind of config as a FILE instead of an env var, editing it the same way, and checking the file inside the running pod. That one updated live, no restart needed. Same ConfigMap concept, two completely different update behaviors depending on how you consume it.

Also hit a real infrastructure snag mid-chapter that had nothing to do with ConfigMaps at all — a routine image pull suddenly failed with a DNS resolution error inside the cluster:
```
lookup registry-1.docker.io on 192.168.65.254:53: no such host
```

Same image that had pulled fine in five earlier chapters. Diagnosed it as the minikube node itself losing outbound DNS, fixed with a clean stop/start cycle. Good reminder: when something fails that's never failed before, check the infrastructure layer before assuming you broke your own YAML.

The pattern across this whole learning journey keeps repeating: don't just learn what an object does — verify it directly, on a real cluster, and you'll remember it permanently instead of just reciting it.

What's something in your stack you accepted as true until you actually tested it?

#Kubernetes #DevOps #K8s #ConfigMap #CloudNative #Learning #PlatformEngineering #DevOpsJourney