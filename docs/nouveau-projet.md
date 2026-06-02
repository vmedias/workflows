# Déployer un nouveau projet — procédure complète

Ce dépôt (`vmedias/vmedias-workflows`) fournit des **workflows réutilisables** :

- `build-nextjs.yml`, `build-laravel.yml`, `build-symfony.yml` — build + push de
  l'image Docker vers `ghcr.io`.
- `deploy-helm.yml` — `helm upgrade --install` sur le cluster k3s.

Les charts Helm vivent dans `vmedias/helm-charts` (`nextjs-app`, `laravel-app`,
`symfony-app`), publiés sur `https://vmedias.github.io/helm-charts`.

Pour un **nouveau projet**, l'objectif final est : *push sur la bonne branche →
build → deploy automatique*. Mais il faut d'abord un bootstrap one-time.

---

## Vue d'ensemble du flux

```
push (branche)
   └─► workflow appelant (.github/workflows/deploy.yml du PROJET)
          ├─► build-XXX.yml   → ghcr.io/vmedias/<app>:sha-<sha> + :staging
          └─► deploy-helm.yml → helm upgrade --install --create-namespace
                                   chart vmedias/<type>-app
                                   -f k8s/values-staging.yaml
                                   --set image.tag=sha-<sha>
```

`--create-namespace` crée le namespace **vide**. Le pull des images ghcr est
géré au niveau node k3s (`registries.yaml`, voir prérequis cluster) — donc
**aucun secret de pull par namespace**. Reste seulement, si l'app en a, ses
secrets/ConfigMap applicatifs (voir §2).

---

## Prérequis cluster (une seule fois pour tout)

- Le ServiceAccount `github-deployer` + ClusterRoleBinding existent.
  Comme le workflow utilise `--create-namespace` (write cluster-scoped), il faut
  le rôle **`cluster-admin`** (Option A dans [deploy-helm-setup.md](deploy-helm-setup.md)).
- Le kubeconfig est généré et accessible (voir [deploy-helm-setup.md](deploy-helm-setup.md)).
- **Creds ghcr au niveau node k3s** via `registries.yaml` — supprime le besoin
  d'un secret de pull dans chaque namespace. Sur **chaque node** :

  ```yaml
  # /etc/rancher/k3s/registries.yaml
  configs:
    ghcr.io:
      auth:
        username: <github-user>
        password: <PAT read:packages>
  ```

  ```bash
  sudo systemctl restart k3s        # ou k3s-agent sur les workers
  ```

  Le chart laisse `image.pullSecret` vide par défaut → pas d'`imagePullSecrets`
  dans les pods, containerd pulle avec les creds du node.
- L'`ingress-controller` et le `cert-manager` (`clusterIssuer: letsencrypt-prod`)
  tournent dans le cluster.

---

## Étapes pour un nouveau projet

### 1. Secret repo `KUBECONFIG`

Le workflow `deploy-helm.yml` déclare `KUBECONFIG` comme secret **requis**.

```bash
base64 -i deploy-kubeconfig.yaml          # macOS
# Settings → Secrets and variables → Actions → New repository secret
#   Nom : KUBECONFIG   Valeur : sortie base64
```

> Astuce : créer plutôt le secret au niveau **org** (scope = repos sélectionnés)
> pour ne pas le refaire à chaque projet.

### 2. Secrets applicatifs — uniquement si l'app utilise `envFrom`

Le namespace et le pull ghcr sont gérés automatiquement (`--create-namespace`
+ `registries.yaml`). **Rien à faire** ici si l'app n'a pas de secrets.

Si le values file référence `envFrom.secretName` (ou `configMapName`), créer
l'objet une fois dans le namespace :

```bash
NS=staging
kubectl -n "$NS" create secret generic my-app-secrets \
  --from-env-file=.env.staging
```

> Le namespace peut ne pas exister encore au moment où tu crées le secret.
> Soit tu lances un premier deploy (qui le crée), soit :
> `kubectl create namespace "$NS" --dry-run=client -o yaml | kubectl apply -f -`.

### 3. Values file dans le repo projet

Le chart prend ses valeurs d'un fichier versionné dans le projet
(par défaut `k8s/values-staging.yaml`). `image.repository` doit **matcher**
l'image buildée — le deploy ne surcharge que `image.tag`.

```yaml
# k8s/values-staging.yaml
name: my-app
image:
  repository: ghcr.io/vmedias/my-app   # == inputs.image du build
  # tag : surchargé par le deploy (--set image.tag=sha-<sha>)
replicas: 2
ingress:
  host: my-app.dev.vmedias.fr
# Laravel/Symfony : décommenter si secrets/worker/scheduler
# envFrom:
#   secretName: my-app-secrets
# worker:
#   enabled: true
# scheduler:
#   enabled: true
```

Valeurs par défaut complètes : `charts/<type>-app/values.yaml` dans
`vmedias/helm-charts`.

### 4. Workflow appelant dans le repo projet

`.github/workflows/deploy.yml` :

```yaml
name: Deploy
on:
  push:
    branches: [staging]          # branche = environnement cible

jobs:
  build:
    uses: vmedias/vmedias-workflows/.github/workflows/build-nextjs.yml@main
    with:
      image: ghcr.io/vmedias/my-app
      tag: staging
    # build-laravel.yml / build-symfony.yml selon le projet

  deploy:
    needs: build
    uses: vmedias/vmedias-workflows/.github/workflows/deploy-helm.yml@main
    with:
      app_name: my-app
      namespace: staging
      chart: vmedias/nextjs-app          # ou laravel-app / symfony-app
      values_file: k8s/values-staging.yaml
      image_tag: sha-${{ github.sha }}
    secrets:
      KUBECONFIG: ${{ secrets.KUBECONFIG }}
      # ou : secrets: inherit
```

`image_tag: sha-${{ github.sha }}` correspond au tag `:sha-<sha>` poussé par le
build — garantit qu'on déploie exactement l'image qu'on vient de builder.

### 5. Push

```bash
git push origin staging
```

Le workflow build l'image, la pousse sur ghcr, puis déploie. Suivi du rollout
inclus dans `deploy-helm.yml` (`kubectl rollout status`, timeout 120s).

---

## Récap : one-time vs récurrent

| Action | Fréquence |
|--------|-----------|
| SA `github-deployer` + ClusterRoleBinding `cluster-admin` | 1× cluster |
| `registries.yaml` (creds ghcr) sur les nodes | 1× cluster |
| Secret repo/org `KUBECONFIG` | 1× (org : jamais si scope étendu) |
| Secrets app dans le namespace | 1× par app, **et seulement si** `envFrom` |
| Values file + workflow appelant | 1× par projet |
| **Push sur la branche** | **à chaque déploiement** |

Bootstrap par-projet réduit au strict minimum : **values file + workflow
appelant**, puis push. Le namespace et le pull ghcr ne demandent plus rien.
Une app sans secrets se déploie donc par simple push, sans toucher au cluster.

---

## Dépannage

| Symptôme | Cause | Fix |
|----------|-------|-----|
| `ImagePullBackOff` | `registries.yaml` absent/erroné sur le node, ou PAT expiré | corriger `/etc/rancher/k3s/registries.yaml` + `systemctl restart k3s` |
| `cannot create resource "namespaces"` | rôle = `admin` (Option B) | passer en `cluster-admin` (Option A) |
| Pod up mais mauvaise version | tag déployé ≠ tag buildé | vérifier `image_tag` == `sha-${{ github.sha }}` |
| `secret "my-app-secrets" not found` | `envFrom.secretName` sans secret | créer le secret (§2) |
| Cert TLS absent | `clusterIssuer`/ingress KO | vérifier cert-manager + `ingress.host` |