# LinkedIn Post — Helm

---

🚀 Day 19 of my Kubernetes journey — today I watched a rollback happen in mid-flight, and it was more convincing than a clean "done" state would have been.

The chapter: Helm. If you've been following this series, you know I've been writing raw YAML by hand for 18 chapters — hardcoded namespaces, image tags, replica counts. One environment is manageable. Three environments (dev, staging, prod) with the same app means three near-duplicate YAML sets, changed in multiple places every time anything shifts. Helm solves that by treating Kubernetes manifests like templates, with values injected per deployment.

The core concept clicked pretty fast. The execution had the usual surprises.

`helm create` scaffolded more default files than I expected. When I stripped down `values.yaml` to just the settings I actually needed, `helm template` immediately told me exactly what broke and where:

```
Error: myapp/templates/httproute.yaml:1:14
  executing "myapp/templates/httproute.yaml" at <.Values.httpRoute.enabled>:
    nil pointer evaluating interface {}.enabled
```

One deleted file, then:
```
Error: myapp/templates/NOTES.txt:2:14
  ...same error, different file
```

Two separate leftovers, each named explicitly in the error. `helm template` is genuinely the most useful command in this whole chapter — it renders the final YAML without touching the cluster at all, which means you find out what's broken before it's already deployed.

After cleaning it up, the full cycle worked: install (replicas: 2), upgrade (replicas: 4 via `--set`), rollback. The rollback was the interesting one:

```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Rollback to 1
```

The rollback doesn't rewrite history — it creates a new revision. Same audit-trail philosophy as `kubectl rollout history` from Chapter 03, but at the whole-release level.

And the pod output I caught right at that moment:
```
myapp-dev-myapp-...   Terminating   ← excess from revision 2
myapp-dev-myapp-...   Terminating
myapp-dev-myapp-...   Running       ← keeping for revision 1
myapp-dev-myapp-...   Running
```

2 terminating, 2 running, exactly halfway through. The transition in mid-flight, timestamped by the command output itself.

Also worth noting: Helm stores its own release metadata separately from where the actual resources live. `helm history myapp-dev -n dev-backend` returned "release not found" — because the release is tracked in `default`, even though the pods run in `dev-backend`. That distinction isn't documented prominently anywhere obvious, and it's the kind of thing that costs real time when you first hit it in a real cluster.

What's the first time a package manager for infrastructure made something genuinely simpler for you, rather than adding one more thing to learn?

#Kubernetes #DevOps #K8s #Helm #CloudNative #Learning #PlatformEngineering #DevOpsJourney