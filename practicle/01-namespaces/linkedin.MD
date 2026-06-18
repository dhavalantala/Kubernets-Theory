# LinkedIn Post — Namespaces

---

🚀 Day 1 of my Kubernetes journey — and I already hit my first real error!

I was trying to deploy my first Namespace on Kubernetes using a YAML file.

Got this error:
```
yaml: line 4: found character that cannot start any token
```

Spent time debugging... turns out YAML hates TABs. Only spaces allowed. 😅

One setting change in VS Code fixed it permanently.

Here's what I learned today about Kubernetes Namespaces:

Without Namespaces → two teams both name their pod "my-app" → chaos 💥
With Namespaces → team-a/my-app and team-b/my-app coexist peacefully ✅

Think of it like folders on your computer. Same filename, different folder = no conflict.

The 4 default namespaces K8s creates for you:
→ default (where your pods go if you don't specify)
→ kube-system (K8s internal brain — DNS, scheduler, etcd)
→ kube-public (readable by everyone)
→ kube-node-lease (node health tracking)

Commands I ran today:
```bash
kubectl get namespaces          # see all namespaces
kubectl apply -f namespace.yaml # create via YAML
kubectl get pods -A             # see pods across ALL namespaces
kubectl config set-context --current --namespace=dev  # switch namespace
```

The coolest thing I learned?
When you delete a namespace → EVERYTHING inside it gets deleted too.
That's why real companies never deploy to the "default" namespace in production.

Coming up next: Pods — the smallest deployable unit in K8s 🔜

What's your biggest K8s mistake you learned from? Drop it below 👇

#Kubernetes #DevOps #K8s #Learning #CloudNative #MLOps #DevOpsJourney #100DaysOfCode