# LinkedIn Post — Secrets

---

🚀 Day 12 of my Kubernetes journey — and this is the chapter where I got corrected by my own terminal.

The topic was Kubernetes Secrets. The core lesson going in: Secret data is base64 ENCODED, not encrypted. Encoding is reversible by anyone. I proved that directly:

```bash
kubectl get secret app-secret -o yaml
# DB_PASSWORD: YXBwcGFzc3dvcmQxMjM=

echo "YXBwcGFzc3dvcmQxMjM=" | base64 -d
# apppassword123
```

One command. No special tools. The "secret" was just sitting there in plain base64. That part of the lesson held up exactly as expected.

Then I went to check something else — whether Secret-mounted files get stricter permissions than ConfigMap-mounted files. I was confident going in that they would.

They didn't:
```
drwxrwxrwt    3 root     root   ...   .
lrwxrwxrwx    1 root     root   ...   DB_PASSWORD -> ..data/DB_PASSWORD
```

Wide open permission bits. Not what I expected at all. Rather than ignore the mismatch, I had to go back and correct the claim — the actual interesting thing in that output wasn't permissions at all, it was a double-symlink structure that lets Kubernetes swap an entire mounted file atomically, so a running pod never sees a half-written config mid-update.

That's a better, truer lesson than the wrong one I started with.

This is becoming a real pattern in how I'm learning this: state a claim, test it against a real cluster, and be willing to be wrong out loud when the evidence says otherwise. Today the evidence corrected me twice in one chapter — once confirming what I expected, once overturning it.

The bigger takeaway about Secrets themselves: they're not a vault by default. No encryption-at-rest unless your cluster operator turns it on. No protection beyond RBAC. Real secret management at scale needs something more (Vault, Sealed Secrets, AWS Secrets Manager) — K8s Secrets alone are a convention, not a guarantee.

What's something you were confidently wrong about until your own tools proved otherwise?

#Kubernetes #DevOps #K8s #Secrets #Security #CloudNative #Learning #PlatformEngineering #DevOpsJourney