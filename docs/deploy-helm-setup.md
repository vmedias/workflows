# Deploy with Helm — GitHub-hosted runner setup

The `deploy-helm.yml` workflow now runs on GitHub-hosted `ubuntu-latest`
runners instead of the self-hosted `k3s-home` runner. A GitHub-hosted runner
lives outside your network, so it needs two things:

1. A **kubeconfig** to authenticate against the cluster.
2. **Network reachability** to the k3s API server (default port `6443`).

---

## 1. Make the k3s API reachable

The GitHub runner must reach the API server. Pick one:

| Option | Effort | Exposure | Notes |
|--------|--------|----------|-------|
| **Public endpoint + firewall allowlist** | Low | API on internet | GitHub runner IPs are dynamic — allowlisting is impractical. Not recommended. |
| **Tailscale / WireGuard** | Medium | None | Add a step to join the runner to your tailnet, then use the cluster's tailnet IP in the kubeconfig. Recommended. |
| **Cloudflare Tunnel** | Medium | None | Expose API server through a tunnel, lock with mTLS / access policy. |

### Tailscale (recommended)

Add before the kubeconfig step in `deploy-helm.yml`:

```yaml
      - name: Connect Tailscale
        uses: tailscale/github-action@v3
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
          oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
          tags: tag:ci
```

The kubeconfig `server:` then points at the k3s node's tailnet IP
(e.g. `https://100.x.y.z:6443`).

---

## 2. Create a cluster-wide ServiceAccount

The default k3s kubeconfig (`/etc/rancher/k3s/k3s.yaml`) is cluster-admin.
Don't ship that file to CI — it carries a non-rotatable client cert. Create a
dedicated ServiceAccount whose token you can revoke. The SA lives in one
namespace (`kube-system` here) but a **ClusterRoleBinding** gives it
cluster-wide reach.

```bash
# On a cluster node / with admin access
kubectl -n kube-system create serviceaccount github-deployer
```

Pick the role for the binding:

> **The workflow now runs `helm upgrade --install --create-namespace`.**
> Creating a namespace is a cluster-scoped write, which the built-in `admin`
> ClusterRole (Option B) **cannot** do. So unless you pre-create every target
> namespace by hand, you need **Option A (`cluster-admin`)**.

```bash
# Option A — full cluster-admin. REQUIRED for --create-namespace.
# Also needed if charts install CRDs or touch RBAC / cluster-scoped objects.
kubectl create clusterrolebinding github-deployer-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:github-deployer

# Option B — built-in `admin` across all namespaces, but cannot mutate
# cluster-scoped resources (namespaces, nodes, PVs, CRDs, RBAC). Tighter
# blast radius — only viable if you create the namespace manually first:
#   kubectl create namespace <namespace>
kubectl create clusterrolebinding github-deployer-admin \
  --clusterrole=admin \
  --serviceaccount=kube-system:github-deployer
```

Generate a long-lived token (k8s 1.24+ no longer auto-creates secrets):

```bash
kubectl -n kube-system create token github-deployer --duration=8760h
```

> Use Option A if `helm upgrade` ever fails with `cannot create resource
> "customresourcedefinitions"` or similar cluster-scoped denials.

---

## 3. Build the kubeconfig

```bash
SERVER="https://100.x.y.z:6443"     # tailnet IP or reachable endpoint
TOKEN="<token from step 2>"

# CA cert from the cluster (on a node):
#   sudo cat /var/lib/rancher/k3s/server/tls/server-ca.crt | base64 -w0
CA_DATA="<base64 CA cert>"

cat > deploy-kubeconfig.yaml <<EOF
apiVersion: v1
kind: Config
clusters:
- name: k3s
  cluster:
    server: ${SERVER}
    certificate-authority-data: ${CA_DATA}
contexts:
- name: deploy
  context:
    cluster: k3s
    user: github-deployer
current-context: deploy
users:
- name: github-deployer
  user:
    token: ${TOKEN}
EOF
```

Test locally:

```bash
KUBECONFIG=./deploy-kubeconfig.yaml kubectl get pods -n staging
```

---

## 4. Store the kubeconfig as a secret

The workflow reads `secrets.KUBECONFIG` (base64-encoded).

```bash
base64 -w0 deploy-kubeconfig.yaml      # Linux
base64 -i deploy-kubeconfig.yaml       # macOS (no -w flag)
```

Add the output as a repository (or org) secret named `KUBECONFIG`:

- Repo: **Settings → Secrets and variables → Actions → New repository secret**
- Org-wide: **Org Settings → Secrets and variables → Actions**, scope to selected repos.

Delete the local `deploy-kubeconfig.yaml` after.

---

## 5. Call the workflow

The reusable workflow now declares a required secret, so callers must pass it:

```yaml
jobs:
  deploy:
    uses: vmedias/vmedias-workflows/.github/workflows/deploy-helm.yml@main
    with:
      app_name: my-app
      namespace: staging
      image_tag: sha-${{ github.sha }}
      values_file: k8s/values-staging.yaml
    secrets:
      KUBECONFIG: ${{ secrets.KUBECONFIG }}
```

Or `secrets: inherit` to forward all caller secrets.

---

## Rotation

`create token --duration` tokens expire. Re-run step 2 + step 4 before expiry,
or set up a shorter-lived token with a scheduled refresh. To revoke
immediately, delete the ServiceAccount (invalidates all its tokens):

```bash
kubectl -n kube-system delete serviceaccount github-deployer
```