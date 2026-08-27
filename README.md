# DevOps Project 2 — WordPress on Amazon EKS with Helm and HPA

A Helm chart that deploys WordPress plus a MariaDB backing store to Amazon EKS, with a
Horizontal Pod Autoscaler that scales on CPU. Built, deployed, load-tested and torn down
against a live cluster.

The chart is the deliverable. The `docs/` folder is the more useful part: it records what
was originally broken, what actually happened when it was run, and why each fix is the fix.

---

## What's here

```
my-microservice/          the Helm chart
  templates/
    mariadb.yaml          Secret + Service + Deployment for the database (added)
    deployment.yaml       scaffold + WORDPRESS_DB_* env wiring (modified)
    hpa.yaml              scaffold, unmodified (autoscaling/v2)
    ...
  values.yaml             rewritten; every change marked [CHANGED] with its reason
  _scaffold-backups/      the untouched `helm create` originals, for diffing

docs/
  runbook-verified.md     step-by-step, with observed output at each step
  gap-review.md           what was wrong with the original doc, and what running it revealed
  runbook-original.md     the original document, archived unmodified
```

## Verified against

EKS 1.36 (us-east-2) · eksctl 0.229.0 · Helm 3.21.3 · kubectl 1.36.1 · AWS CLI 2.36.6

## Quickstart

Assumes an EKS cluster with a metrics source. Full walkthrough in `docs/runbook-verified.md`.

```bash
read -s -p "db password: " DBPW; echo
helm install my-microservice ./my-microservice \
  --set database.password="$DBPW" \
  --set database.rootPassword="$DBPW"

kubectl get pods -w                    # both pods 1/1, ~45s
kubectl get svc my-microservice        # EXTERNAL-IP resolves in 2-5 min
kubectl get hpa                        # expect cpu: N%/50%, never <unknown>
```

Teardown:

```bash
helm uninstall my-microservice
kubectl get svc                        # wait until no LoadBalancer remains
eksctl delete cluster --name enterprise-cluster --region us-east-2
```

Deleting the cluster before the ELB is released orphans it and fails on a VPC dependency
violation.

---

## What the original chart got wrong

Four independent blockers, each sufficient on its own to stop the deployment.

**1. No database.** `helm create` scaffolds a generic chart. Setting
`image.repository: wordpress` gives you the container and nothing else — no MySQL, no
`WORDPRESS_DB_*` env. WordPress returns HTTP 500, the readiness probe never passes, and the
LoadBalancer registers zero healthy targets. Fixed by `templates/mariadb.yaml` and an `env:`
block in the Deployment.

**2. The HPA pointed at a Deployment that doesn't exist.** `scaleTargetRef.name: wordpress`,
but the release creates `my-microservice` → `FailedGetScale`. Fixed by deleting the
standalone `hpa.yaml` and enabling the chart's own template, which derives the target from
the same `fullname` helper as the Deployment, so the name cannot drift.

**3. A metrics-server install step that broke the cluster.** The original review said EKS
ships no metrics source and to apply the upstream `components.yaml`. eksctl had already
installed metrics-server as a **managed add-on**. `kubectl apply` is a strategic merge, not
a replace: it unioned the upstream `k8s-app` label into the managed Service's selector,
selector labels are ANDed, no pod carried it, and the Service dropped to zero endpoints —
taking the Metrics API down while the pods stayed healthy. Check `aws eks list-addons`
before touching `kube-system`.

**4. `persistence:` that did nothing, and would have broken scaling if it had.** No PVC
template consumed those keys and Helm doesn't warn on unused values. Worse, `ReadWriteOnce`
EBS single-attaches — the moment HPA scales past one replica the extra pods sit `Pending` on
`FailedAttachVolume`. Removed. Durable `wp-content` needs EFS with `ReadWriteMany`.

Full list, including the `tag: ""` → `Chart.AppVersion` → `ImagePullBackOff` trap and the
`kubectl run ... --command --` fix, is in `docs/gap-review.md`.

## Observed autoscaling

```
cpu:  14%/50%   1 replica    idle
cpu:  83%/50%   1 → 2        first scale-up
cpu:  70%/50%   3
cpu:  91%/50%   5            ceiling
cpu: 105%/50%   5            saturated
```

Then back to 1 after roughly five minutes — the HPA's scale-down stabilization window.
Scale-up is aggressive; scale-down is patient, so brief spikes don't cause thrash.

### The interesting failure mode

At max replicas, established pods flipped to `0/1 Running`, and one pod recorded a restart.

Those are two different things. `0/1` is a **readiness** failure: the pod is throttled at its
500m CPU limit and can't answer `GET /` inside the probe timeout, so it leaves the Service
endpoints — and its traffic redistributes onto the remaining pods, pushing them closer to
their own limit. The restart is a **liveness** failure: the same throttling eventually blew
the liveness threshold and the kubelet killed a container that was never broken, removing
capacity at the worst possible moment.

Three levers, and the tradeoff is the point:

| Lever | Effect | Cost |
|---|---|---|
| Lower HPA target (50% → 30%) | Scales before saturation | Pays for idle headroom |
| Raise or remove the CPU limit | Ends throttling | Noisy-neighbor risk |
| Loosen probe thresholds | Stops the flapping | Treats the symptom; hides real failures |

For a web tier, lowering the target is usually right — and give liveness a looser threshold
than readiness, so degradation removes a pod from rotation before it kills it.

## Not production-ready

Deliberately. This is a lab that demonstrates the deployment and scaling mechanics:

- MariaDB has **no persistent volume** — the database is lost if the pod dies.
- Credentials come from `--set`, not a secrets manager. Real deployments should use AWS
  Secrets Manager with the External Secrets Operator, or IRSA.
- No TLS. The LoadBalancer serves plain HTTP.
- `wordpress:php8.3-apache` pins the runtime variant but not the WordPress version.
- Single-AZ database, no backups, no replication.

## Cost

Control plane + 3× t3.medium + ELB ≈ $0.25/hour. Set a billing alarm before creating the
cluster and run the teardown when finished.
