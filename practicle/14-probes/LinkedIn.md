# LinkedIn Post — Probes

---

🚀 Day 14 of my Kubernetes journey — today's bug was invisible. Literally.

The chapter: Probes. Specifically, the difference between a container being "alive" (process running) and an application actually being healthy (able to serve real traffic).

I proved the core gap directly: deployed a pod that silently breaks internally after 30 seconds, with zero probes watching it. Kubernetes kept showing it as "Running" forever, completely unaware anything was wrong. That's the entire reason probes exist — a process not crashing tells you nothing about whether the app inside it actually works.

Then I added a livenessProbe to the same broken scenario. This time Kubernetes caught it and restarted the container — RESTARTS counter incrementing, right in front of me.

Then readiness — the non-destructive sibling. Broke a pod's readiness check manually. RESTARTS stayed at zero. But the pod vanished from its Service's routing table instantly. Alive, never restarted, just quietly stopped receiving traffic. That contrast — same kind of failure, two completely different consequences depending on which probe type you use — is the real lesson of this chapter.

Then I tried to prove readiness could recover on its own. And hit this:

```
exec: "touch\u00a0": executable file not found in $PATH
```

A non-breaking space. Not a regular space — an invisible Unicode character that had snuck into a copy-pasted command, turning "touch /tmp/ready" into one broken word Kubernetes couldn't execute. No visual difference in the terminal. Just silent failure with a confusing error.

Worse — my retry attempts technically succeeded, but I'd run them against the wrong pods entirely, ones with no readiness probe checking that file at all. A command exiting cleanly told me nothing about whether it tested the right thing.

Retyped it fresh on the correct pod. And this time:

```
READY   STATUS    RESTARTS
1/1     Running   1 (104m ago)   ← old, unrelated restart

ENDPOINTS
10.244.0.70:80   ← back, automatically
```

No restart triggered. No manual re-registration. Kubernetes' own periodic check noticed the file had returned and quietly put the pod back into rotation, entirely on its own. That's readiness working exactly as designed, start to finish, once I actually tested the right thing.

The real takeaway isn't just "liveness restarts, readiness doesn't." It's that an invisible character cost me two wasted attempts before the lesson actually landed — and re-typing instead of re-pasting fixed it completely.

What's the smallest, most invisible thing that's ever cost you the most debugging time?

#Kubernetes #DevOps #K8s #Probes #CloudNative #Debugging #Learning #PlatformEngineering #DevOpsJourney