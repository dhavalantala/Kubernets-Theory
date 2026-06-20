# LinkedIn Post — Ingress

---

🚀 Day 9 of my Kubernetes journey — this one took hours, not minutes. And it was worth every second.

I set out to learn Ingress: one entry point routing to multiple backend Services by path. Simple concept on paper.

Then reality happened.

Bug #1: DNS didn't resolve. Fixed /etc/hosts.

Bug #2: Connection refused on port 80. Tried `minikube tunnel`. It looked like it ran, then silently exited. I only caught this by checking `ps aux` — the terminal gave zero indication anything had failed.

Bug #3 (hiding inside bug #2): while debugging, I discovered my cluster had quietly become a 2-node setup, with the second node's kubelet stopped. Had nothing to do with Ingress at all. Wiped the cluster, rebuilt clean.

Bug #4: Still couldn't connect — until I ran a plain `ping` against the minikube IP and got "Destination Host Unreachable." That one command proved the real issue: on macOS with the Docker driver, the cluster's node IP simply isn't reachable from the Mac directly. Not a misconfiguration — a structural fact about how Docker Desktop networks on macOS.

The fix: `minikube service --url`, which forwards a real local port. First success: curling "/" returned "Hello from FRONTEND."

Then bug #5, a completely different kind of failure: "/api" returned a clean 503. Not a networking problem this time. I went straight to the ingress controller's own logs and found the actual line:

"Error obtaining Endpoints for Service dev-frontend/backend-svc: no object matching key"

That error rewrote something I'd assumed going in: an Ingress's backend reference resolves inside the INGRESS'S OWN namespace, not wherever the target Service actually lives. There's no built-in cross-namespace backend field.

The real fix — and the one senior platform teams actually use — is an ExternalName Service: a lightweight pointer Service living in the same namespace as the Ingress, that does nothing but forward via DNS to the real Service elsewhere.

One YAML file later: "Hello from BACKEND." Both paths finally routing correctly through one entry point.

Five separate root causes, stacked on top of each other, each one only visible once the previous layer was peeled back. Logs over guessing, every single time.

This is the chapter I'll remember most — not because Ingress is hard conceptually, but because debugging it end-to-end taught me more about how Kubernetes networking actually behaves than any tutorial could have.

What's the longest you've debugged something that turned out to be five unrelated problems stacked together?

#Kubernetes #DevOps #K8s #Ingress #CloudNative #Debugging #PlatformEngineering #DevOpsJourney