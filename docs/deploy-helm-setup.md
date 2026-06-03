# Deploy with Helm — self-hosted runner setup (ARC, scale-to-zero)

The `deploy-helm.yml` workflow runs on a **self-hosted** runner labelled
`k3s-vmedias`, provided by **Actions Runner Controller (ARC)** inside the k3s
cluster. Runners are **ephemeral and scale to zero**: no pod runs when idle, a
pod spins up per queued job and is destroyed after. Everything is `kubectl` /
`helm` — no host install, no kubeconfig file.

## Why self-hosted (not GitHub-hosted)

A GitHub-hosted runner lives outside the network, so it needs a `KUBECONFIG`
secret + network reachability (Tailscale / tunnel) to the API server. To avoid
re-adding that secret on every project, the natural move is an **org-level**
secret — but on the current GitHub plan **org secrets are only available for
public repos**. Our project repos are private, so an org `KUBECONFIG` can't be
shared to them.

Running runners **in the cluster** sidesteps all of it:

- They reach the k3s API over the in-cluster endpoint — no exposed endpoint,
  no Tailscale.
- `kubectl` / `helm` use the runner pod's **in-cluster ServiceAccount** token
  automatically — no `KUBECONFIG` secret, no `k3s.yaml` to copy.
- Bind that SA to `cluster-admin` and `--create-namespace` works.

## Why ARC (not a plain Deployment)

A plain `Deployment` runner idles 24/7 for nothing. ARC is GitHub's official
k8s-native autoscaler: it watches the job queue and creates an **ephemeral**
runner pod only when a job is queued, then deletes it. `minRunners: 0` ⇒ zero
pods at rest. Cost: ~10-30s cold start (image pull + register) on the first job
after idle.

---

## 1. Install the ARC controller

The controller chart ships the CRDs (`AutoscalingRunnerSet`, …). It **must**
install before the scale-set, and at the **same chart version**.

```bash
ARC_VERSION=0.14.2

helm install arc \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version "$ARC_VERSION" \
  -n arc-systems --create-namespace
```

One controller per cluster; it manages every scale-set. Confirm the CRDs landed
before continuing:

```bash
kubectl get crd | grep actions.github.com
# autoscalingrunnersets.actions.github.com
# ephemeralrunners.actions.github.com  ...
```

---

## 2. ServiceAccount + RBAC for runner pods

Runner pods need cluster-admin so `kubectl`/`helm` can deploy (incl.
`--create-namespace`). Create the SA in the runners namespace and bind it:

```yaml
# runner-rbac.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: arc-runners
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gha-runner
  namespace: arc-runners
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gha-runner-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: gha-runner
  namespace: arc-runners
```

```bash
kubectl apply -f runner-rbac.yaml
```

---

## 3. Install the runner scale-set

The Helm **release name becomes the runner label** — name it `k3s-vmedias` so
`runs-on: k3s-vmedias` matches. Auth via a PAT (classic, scopes `repo` +
`admin:org`); a GitHub App is the cleaner alternative for production.

```bash
helm install k3s-vmedias \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version "$ARC_VERSION" \
  -n arc-runners \
  --set githubConfigUrl="https://github.com/vmedias" \
  --set githubConfigSecret.github_token="<github-pat>" \
  --set minRunners=0 \
  --set maxRunners=3 \
  --set template.spec.serviceAccountName=gha-runner
```

- `githubConfigUrl` at the **org** (`/vmedias`) → every repo in the org can use
  it. The org-runner equivalent of the org secret we couldn't have. Scope it to
  selected repos under **Org → Actions → Runners → Runner groups**.
- `minRunners=0` → scale to zero when idle.
- `template.spec.serviceAccountName=gha-runner` → runner pods get the
  cluster-admin SA, so `kubectl`/`helm` authenticate in-cluster with no
  kubeconfig (token + CA mounted at
  `/var/run/secrets/kubernetes.io/serviceaccount`).

---

## 4. Verify

```bash
kubectl -n arc-runners get pods                 # idle: only the listener pod
kubectl -n arc-systems logs deploy/arc-gha-rs-controller | tail
```

The runner set appears under **Org → Settings → Actions → Runners** as
`k3s-vmedias`. Trigger a deploy job — a runner pod is created for it, then
disappears.

---

## 5. Call the workflow

No secrets required — the runner pod's ServiceAccount already has cluster access:

```yaml
jobs:
  deploy:
    uses: vmedias/vmedias-workflows/.github/workflows/deploy-helm.yml@main
    with:
      app_name: my-app
      namespace: staging
      image_tag: sha-${{ github.sha }}
      values_file: k8s/values-staging.yaml
```

---

## Maintenance

```bash
# Change autoscaling bounds
helm upgrade k3s-vmedias \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  -n arc-runners --reuse-values --set maxRunners=5

# Rotate the PAT
helm upgrade k3s-vmedias \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  -n arc-runners --reuse-values --set githubConfigSecret.github_token=<new-pat>

# Remove the runner entirely
helm uninstall k3s-vmedias -n arc-runners
```

Cluster creds never leave the cluster — they're the runner pod's ServiceAccount
token, auto-rotated by k8s and bound only to `gha-runner`.