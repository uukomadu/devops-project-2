# Gap Review — "Deploying Microservices on Amazon EKS"

Review of the original runbook, **verified by running it end to end** on a live EKS 1.36
cluster in us-east-2 on 2026-08-26. Every finding below is either confirmed in practice or
was corrected by what the cluster actually did.

**Verdict:** Steps 1–5 (tooling, cluster, Helm) are sound. Steps 6–10 (the chart, the
deployment, the HPA, the load test) will not produce a working, autoscaling WordPress. Four
blockers stack up, and each one alone is enough to stop it.

**Outcome after fixes:** WordPress + MariaDB deployed clean on first boot (0 restarts),
HPA scaled 1 → 2 → 3 → 5 under load and back to 1, and teardown left nothing behind.

---

## Blockers — this will not work as written

### 1. WordPress has no database — CONFIRMED

`helm create my-microservice` scaffolds a generic chart. Pointing `image.repository` at
`wordpress` gives you the WordPress container and nothing else. That image needs
`WORDPRESS_DB_HOST`, `WORDPRESS_DB_USER`, `WORDPRESS_DB_PASSWORD`, `WORDPRESS_DB_NAME` —
none are set, and no MySQL/MariaDB is deployed anywhere in the runbook.

The container starts, Apache serves a 500, the readiness probe on `/` fails, the pod never
reaches Ready, the LoadBalancer registers zero healthy targets.

**Fix applied:** added `templates/mariadb.yaml` (Secret + ClusterIP Service + single-replica
Deployment) and an `env:` block in `templates/deployment.yaml`. Verified: both pods reached
`1/1 Running` with **0 restarts** on first install.

### 2. The HPA targets a Deployment that doesn't exist — CONFIRMED

```yaml
scaleTargetRef:
  name: wordpress     # <- nothing is named this
```

`helm install my-microservice ./my-microservice` creates a Deployment named
`my-microservice`. The HPA looks for `wordpress`, finds nothing, emits `FailedGetScale`.

**Fix applied:** deleted the standalone `hpa.yaml` entirely and set `autoscaling.enabled:
true` to use the chart's own template, which derives the scale target from the same
`fullname` helper as the Deployment — so the name cannot drift. Verified: `kubectl get hpa`
showed `REFERENCE: Deployment/my-microservice`.

### 3. metrics-server — **THIS FINDING WAS WRONG, AND THE TRUTH IS THE OPPOSITE**

The original review said: "EKS ships no metrics source, so add an install step:
`kubectl apply -f .../metrics-server/releases/latest/download/components.yaml`."

**That instruction broke a working cluster.** On this EKS 1.36 cluster created by
eksctl 0.229.0, `aws eks list-addons` returned:

```
coredns, kube-proxy, metrics-server, vpc-cni
```

metrics-server was **already installed as an EKS managed add-on**. Applying the upstream
manifest over it did the following:

- The Deployment apply failed outright — `spec.selector: field is immutable`. The managed
  add-on uses Helm-chart labels (`app.kubernetes.io/name`, `app.kubernetes.io/instance`);
  the upstream manifest uses `k8s-app`. A Deployment's selector cannot be changed after
  creation.
- But `service/metrics-server configured` **succeeded first**. `kubectl apply` is a merge,
  not a replace: it unioned `k8s-app: metrics-server` into the Service's selector. Selector
  labels are ANDed, and no pod carried that label — so the Service went from matching two
  pods to matching zero.
- Result: `ENDPOINTS <none>`, `v1beta1.metrics.k8s.io … False (MissingEndpoints)`,
  `kubectl top nodes` → `error: Metrics API not available`. The pods were healthy the whole
  time; they had simply been orphaned from their Service.

**Repair:** `aws eks update-addon --addon-name metrics-server --resolve-conflicts OVERWRITE`
reapplied the add-on's own manifest and restored the selector. Endpoints returned within
about a minute.

**Corrected Step 5:**

> Run `kubectl top nodes` FIRST. If it returns numbers, you are done — recent EKS provides
> metrics-server as a managed add-on installed by eksctl. Only if it errors do you install
> anything, and prefer `aws eks create-addon --addon-name metrics-server` over the upstream
> manifest.

**The general lesson, which matters more than the specific one:** `kubectl apply` against a
resource another controller owns is not idempotent — it is a strategic merge. It will
silently union your fields into theirs and can break the resource without any error at the
point of damage. Check `aws eks list-addons` before hand-rolling anything into `kube-system`.

### 4. The `persistence:` block in values.yaml does nothing — CONFIRMED

```yaml
persistence:
  enabled: true
  size: 10Gi
```

Inert. `helm create` scaffolds no PVC template that reads those keys, and Helm does not warn
on unused values — it renders happily and persists nothing.

**Fix applied:** removed the block (see #5 for why removal, not repair).

---

## Design conflicts

### 5. ReadWriteOnce + HPA is a contradiction — CONFIRMED BY COUNTERFACTUAL

`accessModes: [ReadWriteOnce]` on an EBS volume means one node can mount it. The HPA scales
to 5 replicas. Replicas 2–5 land on other nodes, cannot attach, and sit `Pending` with
`FailedAttachVolume`.

In the live run with persistence removed, all five replicas scheduled and reached `Running`
with nothing stuck `Pending` — which is exactly what would NOT have happened with the
original values.yaml.

The framing worth keeping: **EBS is block storage, single-attach; horizontal scale needs
shared filesystem semantics.** For durable `wp-content` the answer is EFS with
`ReadWriteMany`, not EBS.

### 6. The EBS CSI driver isn't installed by default either

Since EKS 1.23 the in-tree EBS provisioner is gone. Even a correct PVC stays `Pending` until
the `aws-ebs-csi-driver` add-on is installed with an IRSA service account — which requires
the cluster to have an OIDC provider. Not exercised in this run (persistence removed), but
it stands.

### 7. Two competing autoscaling mechanisms — CONFIRMED

values.yaml sets `autoscaling.enabled: false` while Step 9 applies a standalone `hpa.yaml`.
The chart's own HPA template sits there unused — and on Helm 3.21 it already emits
`autoscaling/v2`. Using it fixes findings #2 and #12 together.

---

## New finding from the live run: readiness flapping under saturation

Not in the original review — only visible by actually running the load test.

At maximum replicas the HPA read `cpu: 105%/50%`, and the pod watch showed **established,
healthy pods flipping to `0/1 Running`**:

```
my-microservice-c845cf5bd-nldb7   0/1   Running   0   22m
my-microservice-c845cf5bd-d7jrk   0/1   Running   0   3m5s
```

Those are not pods starting up. They are pods pinned at the 500m CPU limit, throttled, and
unable to answer `GET /` inside the probe timeout. A pod that goes NotReady is removed from
the Service endpoints, so its traffic redistributes onto the remaining pods and pushes them
closer to their own limit — the shape of a cascading failure.

`105%/50%` at `maxReplicas: 5` is the same fact from the HPA's side: it wants more pods, the
ceiling says no, and the deficit surfaces as CPU pressure instead.

Three levers, with real tradeoffs:

| Lever | Effect | Cost |
|---|---|---|
| Lower HPA target (50% → 30%) | Scales before saturation | Pays for idle headroom |
| Raise or remove the CPU limit | Ends throttling | Noisy-neighbor risk |
| Loosen `timeoutSeconds` / `failureThreshold` | Stops the flapping | Treats the symptom; hides real failures |

Lowering the target is usually right for a web tier. Removing limits (keeping requests) is
defensible on a dedicated node group.

**It went further than flapping.** At teardown, one of the pods created during scale-up
showed `RESTARTS 1 (4h11m ago)` — timed to the load test. That is not readiness; readiness
only removes a pod from the Service. A restart means the **liveness** probe failed its
threshold and the kubelet killed the container. Under saturation the same CPU throttling
that trips readiness will eventually trip liveness, and Kubernetes responds by restarting a
pod that was never broken — removing capacity at exactly the moment the system needs it
most. This is the strongest argument for setting the HPA target low enough that pods scale
before they saturate, and for giving liveness a looser threshold than readiness.

---

## Correctness and freshness

| # | Issue | Fix | Status |
|---|---|---|---|
| 8 | eksctl download points at `weaveworks/eksctl` | Canonical repo is `eksctl-io/eksctl` | Confirmed |
| 9 | `kubectl` never installed but needed from Step 4 | Add an install step | Confirmed |
| 10 | Step 1 titles `aws --version` as "Install AWS CLI" | That verifies; add the installer | Confirmed |
| 11 | `curl <url> \| bash` for Helm | Use `curl -fsSL` | Confirmed |
| 12 | `autoscaling/v1` + `targetCPUUtilizationPercentage` | Helm 3.21's own template already uses `autoscaling/v2` | Confirmed |
| 13 | "Note the EXTERNAL-IP address" | It is a DNS hostname, `<pending>` for ~4 min first. Live value was `a31a...us-east-2.elb.amazonaws.com` | Confirmed |
| 14 | `kubectl run siege ... -- /bin/sh -c` | Those are args, not a command override — needs `--command --`. Also target in-cluster Service DNS, not the public ELB | Confirmed |
| 15 | Cluster create omits `--with-oidc` | No OIDC → no IRSA → blocks every add-on needing AWS permissions | Confirmed |
| 16 | `resources.requests.cpu: 300m` present but unexplained | Load-bearing: HPA utilization is computed against the request | Confirmed |
| 17 | **Nothing is pinned** — not the Kubernetes version, not the image tag | Worse than it looks: the scaffold ships `tag: ""`, which falls back to `Chart.AppVersion` (`1.16.0`) and renders `wordpress:1.16.0` → `ImagePullBackOff`. The original's `tag: latest` dodges this by luck | New |
| 18 | No kubeconfig context check | `kubectl config current-context` before any cluster command — a machine with minikube installed will happily accept the deploy | New |
| 19 | Region hardcoded to `us-east-1` in four places | Any mismatch with the CLI default surfaces as "cluster not found" three steps later. This run used `us-east-2` throughout | New |

---

## Cleanup and cost

### 20. `kubectl delete all --all` doesn't delete what costs money

`all` covers Pods, Services, Deployments, ReplicaSets, StatefulSets, Jobs, CronJobs. It does
**not** include PVCs, Secrets, ConfigMaps, Ingresses, or the Helm release record. Orphaned
PVCs leave orphaned EBS volumes that keep billing. It is also default-namespace only.

### 21. Delete order matters

Tearing down the cluster while a `type: LoadBalancer` Service still exists can orphan the
ELB and its security group, and `eksctl delete cluster` then fails on a VPC dependency
violation.

```bash
helm uninstall my-microservice
kubectl get svc            # wait until no LoadBalancer remains
eksctl delete cluster --name enterprise-cluster --region us-east-2
```

### 22. No cost warning

Control plane + 3× t3.medium + an ELB runs roughly $0.25/hour. This run sat up ~4.5 hours
for about $1–1.50. Check aws.amazon.com/eks/pricing for current rates and set a billing
alarm before Step 3.

---

## What the runbook gets right

- Managed node group with min/max set, rather than an unmanaged ASG.
- CPU **requests** on the pod — the actual prerequisite for CPU-based HPA.
- Resource limits set at all, so one pod cannot starve a node.
- A real load-generation step to prove autoscaling, instead of asserting it works.
- An explicit teardown section.

---

*Verified against a live EKS 1.36 cluster, eksctl 0.229.0, Helm 3.21.3, kubectl 1.36.1,
AWS CLI 2.36.6, macOS arm64. Sources: AWS EKS documentation on HPA and metrics-server;
eksctl-io/eksctl; Helm v3.21.3 chart scaffold source.*
