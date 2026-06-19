# 06 - Jobs & CronJobs

## What problem it solves

Everything before this chapter (Deployment, ReplicaSet, DaemonSet) assumes
the pod should run forever - if it exits, that's treated as a failure and
it gets restarted. But plenty of real work isn't a long-running service:
a database migration, a nightly backup, generating a report, sending a
batch of emails. These need to run to completion and then stop - exiting
with status 0 is success, not a crash.

If you ran one of these as a Deployment, the controller would keep
restarting your "finished" task forever, because it has no concept of
"done."

- **Job** = run this to completion, retry on failure up to a limit, track
  success.
- **CronJob** = do that, on a schedule - it's a Job factory that fires on
  cron syntax.

## How to work with it

The one rule that trips people up: a Job's pod template **cannot** use
`restartPolicy: Always` - that's rejected by validation, because "restart
forever" contradicts "this should finish." Use `Never` or `OnFailure`.

```yaml
spec:
  backoffLimit: 3       # retry attempts before marking the Job Failed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: report
        image: busybox:1.36
        command: ["sh", "-c", "echo done"]
```

A CronJob is the same Job spec wrapped with a `schedule`:

```yaml
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid   # don't start a new run if the last one is still going
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
```

`concurrencyPolicy` matters more than it looks:

- `Allow` (default) - runs can pile up if one is still going.
- `Forbid` - skip the new run if the previous one isn't finished yet.
- `Replace` - kill the old run and start the new one.

For anything touching shared state (a DB migration, a billing job),
`Forbid` is usually the safe default.

## Files in this folder

| File                 | Purpose                                          |
|----------------------|---------------------------------------------------|
| `06-namespace.yaml`  | Creates the `06-jobs-cronjobs` namespace         |
| `06-job.yaml`        | One-off Job that generates a "report" and exits  |
| `06-cronjob.yaml`    | Same idea, fired automatically every minute      |

## How to run the Job

```bash
kubectl apply -f 06-namespace.yaml
kubectl apply -f 06-job.yaml
kubectl get jobs -n 06-jobs-cronjobs
kubectl get pods -n 06-jobs-cronjobs
```

Watch the pod go `Running` -> `Completed` - note it's `Completed`, not
deleted. Jobs keep finished pods around by default so you can inspect
them:

```bash
kubectl logs -n 06-jobs-cronjobs -l job-name=report-generator
```

## Then the CronJob

```bash
kubectl apply -f 06-cronjob.yaml
kubectl get cronjob -n 06-jobs-cronjobs
kubectl get jobs -n 06-jobs-cronjobs -w
```

Leave it running for 2-3 minutes - a brand new Job object spawns every
minute, each creating its own pod.

## Worth breaking on purpose

Change the command in `06-job.yaml` to something that fails:

```yaml
command: ["sh", "-c", "exit 1"]
```

Reapply and watch:

```bash
kubectl apply -f 06-job.yaml
kubectl get pods -n 06-jobs-cronjobs -w
```

It retries up to `backoffLimit` times before the Job is marked `Failed` -
that retry behavior is the entire reason `backoffLimit` exists.

## Cleanup

```bash
kubectl delete namespace 06-jobs-cronjobs
```

Removes the namespace, the Job, the CronJob, and all their spawned pods -
including any leftover Completed/Failed pods from your testing.

## Check your understanding

- Why does a Job's pod template reject `restartPolicy: Always`, when a
  Deployment's pod template uses exactly that?
- What's the practical difference between setting `concurrencyPolicy` to
  `Forbid` vs `Replace` for a CronJob that's still mid-run when the next
  scheduled time arrives?
- If `backoffLimit` is exceeded and a Job is marked `Failed`, does
  Kubernetes delete the failed pods automatically? Why might that default
  behavior actually be useful?