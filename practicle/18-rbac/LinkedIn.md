# LinkedIn Post — RBAC

---

🚀 Day 18 of my Kubernetes journey — today I watched a pod try, and fail, to delete itself. On purpose.

This chapter closes a loop that started way back on Day 1. Early in this journey, I ran `kubectl delete namespace staging` with zero confirmation and watched everything inside vanish instantly. The lesson then was: this is exactly why real companies use RBAC to stop people from having more access than they need. Today I built that exact protection and tested it against a real, running pod.

The setup: a ServiceAccount with zero permissions. A Role granting only `get/list/watch` on pods — explicitly nothing else. A RoleBinding connecting the two. Then a real pod, running `kubectl` itself, authenticated as that ServiceAccount — not a simulated check, the actual in-cluster credential path.

From inside that live pod:

```bash
kubectl get pods
```
Worked. Returned real data.

```bash
kubectl delete pod rbac-test-pod
```
```
Error from server (Forbidden): pods "rbac-test-pod" is forbidden:
User "system:serviceaccount:dev-backend:readonly-sa" cannot delete
resource "pods" in API group "" in the namespace "dev-backend"
```

Blocked. The pod had valid, working credentials — it just wasn't authorized to do that specific thing, even to itself.

What stood out most was the error message itself. It's not a vague "access denied." It names the exact ServiceAccount, the exact verb, the exact resource, the exact namespace. That's genuinely useful when you're the one debugging a permissions issue at 2am, not just a security feature working in the background.

The bigger realization: most of what I've built across this entire learning journey has been running as cluster admin, with effectively unlimited power. Today was the first time I deliberately built something with LESS power than me, and proved the boundary held even when tested from the inside.

What's the first time you deliberately built a system to be less powerful than yourself, on purpose?

#Kubernetes #DevOps #K8s #RBAC #Security #CloudNative #Learning #PlatformEngineering #DevOpsJourney