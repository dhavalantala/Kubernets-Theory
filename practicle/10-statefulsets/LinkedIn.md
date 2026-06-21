# LinkedIn Post — StatefulSets

---

🚀 Day 10 of my Kubernetes journey — today I deliberately broke a database to prove a point.

StatefulSets exist to solve a problem Deployments can't: stable identity and stable storage for things like databases. I'd already proven this conceptually with a fake echo container. Today I redid it with real PostgreSQL, and the difference in what it revealed was significant.

The setup: 3 Postgres pods, each meant to be independent — postgres-0, postgres-1, postgres-2 — each with its own dedicated storage, automatically provisioned per pod.

Step one: connect to postgres-0, create a real table, insert a real row.

Step two: fully delete that pod. Not restart — delete. Brand new container, in theory a blank slate.

```bash
kubectl delete pod postgres-0 -n dev-database
```

It came back. Same name: postgres-0. Same data:
```
 id |               note
----+----------------------------------
  1 | data written before pod deletion
```

That's the headline feature working as advertised — pod identity and its disk both survived complete destruction.

But the more interesting result came from the test I did NOT expect to feel as clarifying as it did. I queried postgres-1 for that same row:

```
ERROR:  relation "proof" does not exist
```

That error is correct. That's not a bug — that's the proof that each pod in this setup has its own completely independent database, with zero automatic replication between them. A StatefulSet guarantees stable identity and storage. It does NOT give you a clustered, replicated database for free. That's a separate, much bigger topic (streaming replication, Patroni, etc.) that I now know NOT to assume is included.

The lesson that's sticking with me: knowing what an object does is only half the job. Knowing exactly what it does NOT do is the other half — and it's usually the half that causes production incidents when assumed incorrectly.

What's a Kubernetes (or any infra) assumption you had to unlearn the hard way?

#Kubernetes #DevOps #K8s #StatefulSet #PostgreSQL #CloudNative #Learning #PlatformEngineering #DevOpsJourney