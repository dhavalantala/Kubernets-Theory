# LinkedIn Post — Node Affinity

---

🚀 Day 17 of my Kubernetes journey — today's chapter was the mirror image of yesterday's, and that contrast is exactly what made it click.

Chapter 15 was taints: a NODE repelling pods by default. Today was Node Affinity: a POD actively pulling toward a specific kind of node. Opposite direction, same underlying idea — scheduling control, just from the other side of the relationship.

Ran four pods, same labeled node, four different outcomes:

A pod asking for `disktype=hdd` via plain `nodeSelector`, when the only node available has `disktype=ssd`:
```
0/1 nodes are available: 1 node(s) didn't match
Pod's node affinity/selector.
```
Stuck Pending. Forever. Zero flexibility — nodeSelector is just an exact match, nothing more.

A pod using the newer `nodeAffinity` with `operator: In` and TWO acceptable values, `[ssd, nvme]`:
```
Successfully assigned dev-backend/affinity-required to minikube
```
Scheduled instantly — `ssd` was one of the listed options. Still a hard rule, just more expressive than a plain selector.

Then the one that actually taught me something: a pod requesting `disktype=nvme` — a value that exists on ZERO nodes in the entire cluster — using `preferredDuringScheduling` instead of `required`:
```
NAME                 READY   STATUS    NODE
affinity-preferred   1/1     Running   minikube
```
Scheduled anyway. Completely ignored its own unmet preference and ran fine.

Same kind of "nothing matches" situation as the very first failure — but a completely different outcome, purely because of one word: `preferred` instead of `required`.

That's the real lesson. It's not about how close a match is. It's about whether the rule is a hard requirement or a soft suggestion the scheduler is allowed to drop entirely if it has to. Two completely different failure philosophies, expressed in nearly identical YAML.

What's a "soft vs hard rule" distinction you've had to learn the hard way in your own systems?

#Kubernetes #DevOps #K8s #NodeAffinity #Scheduling #CloudNative #Learning #PlatformEngineering #DevOpsJourney