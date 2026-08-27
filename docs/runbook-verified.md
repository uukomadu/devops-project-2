# Deploying Microservices on Amazon EKS — Verified Runbook

WordPress + MariaDB on EKS via Helm, with working horizontal pod autoscaling.

**This runbook has been executed end to end.** Every command below was run against a live
cluster on 2026-08-26; the observed output is recorded inline. Where the original project
document was wrong, the gap review doc explains why.

Verified on: EKS 1.36 (us-east-2), eksctl 0.229.0, Helm 3.21.3, kubectl 1.36.1,
AWS CLI 2.36.6, macOS arm64.

**Cost:** roughly $0.25/hour (control plane + 3 nodes + ELB). Set a billing alarm before
Step 3 and run the teardown in Step 11 when finished.

---

## A note on ordering

The original document runs 1→10 linearly, which parks you in front of CloudFormation for
15–20 minutes at Step 3 with nothing to do. Nothing in Step 6 needs a cluster. The order
used here:

**Step 2 → start Step 3 → build the chart (Step 6) while it provisions → Steps 4, 5 → 7.**

---

## Step 1 — Install the tools (macOS)

The original Step 1 is Windows-only, points at the retired `weaveworks/eksctl` repo, and
never installs kubectl despite every step from 4 onward needing it.

```bash
brew install awscli kubectl eksctl helm
```

Verify:

```bash
aws --version            # aws-cli/2.x  — v1 has different update-kubeconfig syntax
kubectl version --client
eksctl version
helm version --short
```

Observed: `aws-cli/2.36.6`, `kubectl v1.36.1`, `eksctl 0.229.0`, `helm v3.21.3`.

Note `aws --version` is a *verification*, not an install — the original document labels it
"Install AWS CLI".

## Step 2 — Configure credentials

```bash
aws configure            # output json
aws sts get-caller-identity
```

**Set the region deliberately and use the same one everywhere.** The original document
hardcodes `us-east-1` in four places; if your CLI default differs you get "cluster not
found" at Step 4 against a cluster that exists perfectly well elsewhere. This run used
`us-east-2` throughout.

`get-caller-identity` prints your account ID twice — in the `Account` field and inside the
`Arn`. Redact both if you paste it anywhere.

## Step 3 — Create the cluster (start it, then move to Step 6)

Three changes from the original: the region, an explicit `--version`, and `--with-oidc`.

```bash
eksctl create cluster \
  --name enterprise-cluster \
  --region us-east-2 \
  --version 1.36 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 6 \
  --with-oidc \
  --managed
```

**Why pin `--version`:** an unpinned runbook builds a different cluster depending on what
month you run it. Also keeps you inside Kubernetes' ±1 minor client/server skew window —
kubectl 1.36.1 against an eksctl-default cluster could otherwise be two minors apart.

**Why `--with-oidc`:** it creates the cluster's OIDC identity provider, which is what lets
service accounts assume IAM roles (IRSA). Nothing in this runbook strictly needs it, but
every add-on that touches an AWS API does — EBS CSI, EFS CSI, Load Balancer Controller,
External Secrets. One flag at create time; an annoying retrofit afterward.

15–20 minutes. **Leave it running and open a second terminal.**

---

## Step 6 — Build the chart (do this while the cluster provisions)

```bash
cd ~/Desktop/devops-project-2
helm create my-microservice
cd my-microservice
```

The scaffold ships four defaults that are wrong for this project, three of them silently:

| Scaffold default | Problem |
|---|---|
| `image.repository: nginx`, `image.tag: ""` | Empty tag falls back to `Chart.AppVersion` (`1.16.0`) → renders `wordpress:1.16.0` → `ImagePullBackOff` |
| `service.type: ClusterIP` | No ELB, no browser access |
| `resources: {}` | No CPU request → HPA reports `<unknown>` → never scales |
| no `env:` anywhere | No way to reach a database via values alone |

### 6a. `templates/mariadb.yaml` (new file)

WordPress is not self-contained. Without a database it returns HTTP 500, the readiness
probe never passes, and the LoadBalancer registers zero healthy targets.

```yaml
{{- $fullName := include "my-microservice.fullname" . -}}
{{- $dbName := printf "%s-db" (include "my-microservice.name" .) -}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ $fullName }}-db
  labels:
    {{- include "my-microservice.labels" . | nindent 4 }}
type: Opaque
stringData:
  mariadb-root-password: {{ required "database.rootPassword must be set" .Values.database.rootPassword | quote }}
  mariadb-password: {{ required "database.password must be set" .Values.database.password | quote }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ $fullName }}-db
  labels:
    {{- include "my-microservice.labels" . | nindent 4 }}
spec:
  type: ClusterIP
  ports:
    - port: 3306
      targetPort: mysql
      protocol: TCP
      name: mysql
  selector:
    app.kubernetes.io/name: {{ $dbName }}
    app.kubernetes.io/instance: {{ .Release.Name }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $fullName }}-db
  labels:
    {{- include "my-microservice.labels" . | nindent 4 }}
spec:
  replicas: 1
  strategy:
    type: Recreate          # two MariaDB pods must never share storage
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ $dbName }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ $dbName }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: mariadb
          image: {{ .Values.database.image | quote }}
          imagePullPolicy: IfNotPresent
          ports:
            - name: mysql
              containerPort: 3306
              protocol: TCP
          env:
            - name: MARIADB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ $fullName }}-db
                  key: mariadb-root-password
            - name: MARIADB_DATABASE
              value: {{ .Values.database.name | quote }}
            - name: MARIADB_USER
              value: {{ .Values.database.user | quote }}
            - name: MARIADB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ $fullName }}-db
                  key: mariadb-password
          readinessProbe:
            exec:
              command: ["healthcheck.sh", "--connect", "--innodb_initialized"]
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 6
          livenessProbe:
            exec:
              command: ["healthcheck.sh", "--connect", "--innodb_initialized"]
            initialDelaySeconds: 45
            periodSeconds: 20
            failureThreshold: 6
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

**The subtle part is the labels.** The database pods deliberately carry a `-db` suffix so
they do *not* match the chart's `selectorLabels`. If they matched, the WordPress
LoadBalancer Service would select the database pods too and route web traffic at port 3306.

### 6b. Wire the env vars into `templates/deployment.yaml`

The scaffold generates no `env:` block and values.yaml has no `env:` key — there is no
values-only path. Insert this in the container spec, between `ports:` and the probes:

```yaml
          env:
            - name: WORDPRESS_DB_HOST
              value: {{ include "my-microservice.fullname" . }}-db:3306
            - name: WORDPRESS_DB_NAME
              value: {{ .Values.database.name | quote }}
            - name: WORDPRESS_DB_USER
              value: {{ .Values.database.user | quote }}
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-microservice.fullname" . }}-db
                  key: mariadb-password
            {{- with .Values.env }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
```

The password comes from the Secret via `secretKeyRef`, never as a literal in the pod spec.

### 6c. `values.yaml`

```yaml
replicaCount: 1

image:
  repository: wordpress
  pullPolicy: IfNotPresent
  tag: "php8.3-apache"      # NEVER leave empty — see the table above

database:
  image: "mariadb:11.4"
  name: wordpress
  user: wordpress
  password: ""              # supply with --set at install time
  rootPassword: ""

env: []

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}
podSecurityContext: {}
securityContext: {}

service:
  type: LoadBalancer        # scaffold ships ClusterIP
  port: 80                  # also sets containerPort — leave it

ingress:
  enabled: false
httpRoute:
  enabled: false

# Load-bearing. HPA utilization = actual CPU / REQUESTED CPU.
# Delete requests.cpu and autoscaling silently stops working.
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 300m
    memory: 256Mi

livenessProbe:
  httpGet: { path: /, port: http }
  initialDelaySeconds: 30
  periodSeconds: 15
  failureThreshold: 6
readinessProbe:
  httpGet: { path: /, port: http }
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 6

autoscaling:
  enabled: true             # use the chart's own hpa.yaml, not a standalone file
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50

volumes: []
volumeMounts: []
nodeSelector: {}
tolerations: []
affinity: {}
```

**Two deliberate removals.** There is no standalone `hpa.yaml` — Helm 3.21's scaffold
already emits `autoscaling/v2` and derives the scale target from the same `fullname` helper
as the Deployment, so the name cannot drift out of sync the way the original's did. And
there is no `persistence:` block: it was inert in the original (no PVC template consumed
it), and a `ReadWriteOnce` EBS volume cannot back a Deployment that scales horizontally.
For durable `wp-content`, use EFS with `ReadWriteMany`.

### 6d. Render before you install

```bash
helm lint . --set database.password=x --set database.rootPassword=y
helm template my-microservice . --set database.password=x --set database.rootPassword=y \
  | grep -E "^kind:|^  name:|  image:|WORDPRESS_DB_HOST" -A1
```

Confirm: two Deployments, `image: wordpress:php8.3-apache`, and the HPA's `scaleTargetRef`
reading `my-microservice`.

---

## Step 4 — Kubeconfig and nodes

```bash
aws eks --region us-east-2 update-kubeconfig --name enterprise-cluster
kubectl config current-context
kubectl get nodes
```

The context check is an addition. `update-kubeconfig` does switch context, but on a machine
with minikube installed, "why isn't my Deployment in the console" is almost always this.

Expect three nodes `Ready`.

## Step 5 — Ensure a metrics source exists (do NOT blindly install one)

**Check first:**

```bash
kubectl top nodes
```

**If that returns numbers, you are done.** Recent EKS provides metrics-server as a managed
add-on, installed by eksctl alongside coredns, kube-proxy, and vpc-cni. Confirm with:

```bash
aws eks list-addons --cluster-name enterprise-cluster --region us-east-2
```

Observed on this cluster: `coredns, kube-proxy, metrics-server, vpc-cni`.

**Only if `kubectl top nodes` errors** do you install anything — and prefer the managed
add-on:

```bash
aws eks create-addon --cluster-name enterprise-cluster --region us-east-2 \
  --addon-name metrics-server
```

> **Do not `kubectl apply` the upstream `components.yaml` onto a cluster that already has
> the add-on.** `kubectl apply` is a strategic merge, not a replace. It unions the upstream
> `k8s-app` label into the managed Service's selector; selector labels are ANDed; no pod
> carries that label; the Service drops to zero endpoints and the Metrics API goes down —
> while the pods themselves stay perfectly healthy, which makes it hard to diagnose. This
> happened during this run. Repair is
> `aws eks update-addon --addon-name metrics-server --resolve-conflicts OVERWRITE`.

Observed baseline once healthy: 3 nodes at ~1% CPU, 12–13% memory.

## Step 7 — Deploy

```bash
cd ~/Desktop/devops-project-2/my-microservice
read -s -p "db password: " DBPW; echo
helm install my-microservice . \
  --set database.password="$DBPW" \
  --set database.rootPassword="$DBPW"
```

`read -s` keeps the password out of shell history; `--set` values otherwise land in
`~/.zsh_history` in plaintext.

## Step 8 — Verify

```bash
kubectl get pods -w
```

Observed: both pods `1/1 Running` in about 45 seconds, **0 restarts**. The WordPress pod may
restart once or twice while MariaDB initializes — the probe delays above are sized to avoid
it, but a restart there is not a failure.

```bash
kubectl get svc my-microservice
```

`EXTERNAL-IP` sits at `<pending>` for 2–5 minutes, then resolves to a **DNS hostname**, not
an IP:

```
NAME              TYPE           EXTERNAL-IP                                              PORT(S)
my-microservice   LoadBalancer   a31a0d6...-1832947987.us-east-2.elb.amazonaws.com        80:32487/TCP
```

Open `http://<hostname>` — plain HTTP, no certificate on this listener. Expect the WordPress
installation wizard.

## Step 9 — Confirm the HPA

```bash
kubectl get hpa
```

```
NAME              REFERENCE                    TARGETS       MINPODS   MAXPODS   REPLICAS
my-microservice   Deployment/my-microservice   cpu: 3%/50%   1         5         1
```

Two things to check: `REFERENCE` reads `Deployment/my-microservice` (the original's
`wordpress` would dangle), and `TARGETS` is a real percentage rather than `<unknown>`.

`<unknown>/50%` means one of three things: no metrics source (Step 5), no CPU request
(Step 6c), or an unresolvable scale target. `kubectl describe hpa my-microservice` names it.

## Step 10 — Load test

Two changes from the original: `--command --` so the shell actually overrides siege's
entrypoint, and the in-cluster Service DNS instead of the public ELB hostname — no reason
to route load out to the internet and back to reach a pod on the same cluster.

```bash
kubectl run siege --image=jstarcher/siege --restart=Never \
  --command -- /bin/sh -c "siege -c 100 -t 5m http://my-microservice"
```

Equivalent with no special image:

```bash
kubectl run load-gen --image=busybox:1.36 --restart=Never \
  --command -- /bin/sh -c "while true; do wget -q -O- http://my-microservice >/dev/null; done"
```

Watch, in two more terminals:

```bash
kubectl get hpa -w
kubectl get pods -w
```

Observed progression:

```
cpu: 14%/50%    1 replica     idle
cpu: 83%/50%    1 → 2         first scale-up
cpu: 70%/50%    3
cpu: 91%/50%    5             ceiling reached
cpu: 105%/50%   5             saturated
```

All five pods scheduled and reached `Running` — nothing stuck `Pending`, which is exactly
what the original values.yaml's `ReadWriteOnce` persistence block would have caused.

**Two things worth studying in that output.**

*Scale-down is slow on purpose.* After the load stops, replicas stay elevated for about five
minutes. That is the HPA's stabilization window preventing thrash on brief spikes. Scale-up
is aggressive; scale-down is patient. Not a bug.

*Readiness probes flap under saturation.* At max replicas, established pods flip to
`0/1 Running` — throttled at the 500m CPU limit and unable to answer `GET /` inside the
probe timeout. A NotReady pod leaves the Service endpoints, so its traffic redistributes
onto the rest and pushes them closer to their own limit. See the gap review for the three
levers and their tradeoffs.

Clean up the generator — **both, if you ran both**:

```bash
kubectl delete pod siege
kubectl delete pod load-gen
```

A generator left running holds you at max replicas indefinitely.

## Step 11 — Teardown

Order matters.

```bash
helm uninstall my-microservice
kubectl get pods                 # confirm no load generators remain
kubectl delete pvc --all
kubectl get svc                  # WAIT until no LoadBalancer remains
eksctl delete cluster --name enterprise-cluster --region us-east-2
```

Deleting the cluster while a `type: LoadBalancer` Service still exists can orphan the ELB
and its security group, and `eksctl delete cluster` then fails on a VPC dependency
violation you clean up by hand.

`kubectl delete all --all` — the original's teardown — is **not** sufficient: `all` excludes
PVCs, Secrets, ConfigMaps, and the Helm release record, and orphaned PVCs leave orphaned EBS
volumes that keep billing. It is also default-namespace only.

Afterward, confirm in the console that the CloudFormation stacks are gone and no stray EBS
volumes or load balancers remain.

---

## Optional: adding real persistence

For durable `wp-content` across replicas the path is EFS, not EBS:

1. Install the EFS CSI driver add-on (needs the OIDC provider from Step 3).
2. Create an EFS filesystem with mount targets in the cluster's subnets.
3. Create a StorageClass with `provisioner: efs.csi.aws.com`.
4. Add a PVC template with `accessModes: [ReadWriteMany]`, mounted at
   `/var/www/html/wp-content`.

EBS is single-attach `ReadWriteOnce` — it can back a StatefulSet with per-pod volumes, but
never a horizontally scaled Deployment sharing state.
