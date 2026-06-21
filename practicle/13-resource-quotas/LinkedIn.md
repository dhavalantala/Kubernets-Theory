# LinkedIn Post — Resource Quotas & Limits

---

🚀 Day 13 of my Kubernetes journey — and today I got blocked twice before I even got to run my actual test.

The plan: trigger a deliberate OOMKill to see firsthand why Kubernetes handles memory limits differently from CPU limits (CPU gets throttled, memory gets the container killed outright).

I never got that far. Here's what happened instead, and why I think it's a better lesson than the one I originally planned.

Attempt 1:
```
pods "memory-hog" is forbidden: minimum memory usage per Container
is 64Mi, but request is 50Mi
```

My own LimitRange — set earlier in this same chapter — rejected my pod before it was even created. Not after it started. Not after it ran. At admission time, instantly.

Fixed the value, tried again.

Attempt 2:
```
pods "memory-hog" is forbidden: exceeded quota: backend-quota,
requested: pods=1, used: pods=10, limited: pods=10
```

A completely different limit this time — my namespace had quietly accumulated exactly 10 pods from earlier steps, hitting a hard pod-COUNT cap with zero headroom left, regardless of how reasonable my memory request was.

Two separate enforcement layers, two separate root causes, neither one being the test I actually set out to run.

Here's the real takeaway: Kubernetes enforces resource rules at the API server, before a pod ever gets a chance to misbehave. That's a genuinely good design — you find out a request is unreasonable immediately, in a clear error message, not three hours later when something mysteriously won't schedule.

I didn't get my OOMKill today. I got something arguably more useful: direct, hands-on proof that LimitRange and ResourceQuota are two independent gates, and that namespace hygiene from earlier chapters directly determines whether later experiments can even run at all.

Documenting that honestly, unfinished test and all, because that's closer to what real debugging actually looks like than a clean walkthrough would be.

What's a test you set out to run that taught you something completely different instead?

#Kubernetes #DevOps #K8s #ResourceQuota #LimitRange #CloudNative #Learning #PlatformEngineering #DevOpsJourney