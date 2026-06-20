# LinkedIn Post — Services

---

🚀 Day 8 of my Kubernetes journey — today a hung terminal taught me more than a working one would have.

I created a Service, set its port to 8080, then ran a simple wget against it.

It just... hung. No error. No response. Nothing.

My first instinct was "something's broken." It wasn't. I had broken it — by assuming the default port (80) would work, when I'd already configured the Service to listen on 8080.

That hang was actually proof of something I was trying to learn: that a Service's `port` field is a real, independent setting — not cosmetic.

Here's the concept underneath it:

Every Pod gets its own IP. But Pods are disposable — a Deployment can kill and replace one anytime, and the new Pod gets a completely different IP.

If anything called that Pod's IP directly, it would be calling a dead address the moment that Pod got replaced.

A Service solves this by giving a stable address that sits in front of a CHANGING set of Pods. Underneath, it's not magic — it's a `selector` that continuously watches for pods matching a label, and an `Endpoints` object that tracks their current IPs live.

I proved this directly:
→ Deleted one of 3 backend pods
→ Service's ClusterIP: unchanged
→ Endpoints list: automatically updated to drop the dead pod, add the new one

Then I checked the Service's actual YAML and saw this:
```yaml
ports:
- port: 8080
  targetPort: 5678
```

Two different numbers. `port` is what callers dial. `targetPort` is what the container actually listens on. The Service translates between them in real time — which is exactly why my wrong-port wget hung instead of erroring cleanly.

Small debugging moment, big concept locked in.

What's a bug that taught you more than the working version would have?

#Kubernetes #DevOps #K8s #Services #CloudNative #Learning #PlatformEngineering #DevOpsJourney