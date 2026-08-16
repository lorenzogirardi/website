# Skill: gitops

Trigger: user asks to "set up CI/CD", "automatic deploy", "GitHub Actions", "push and deploy", or scaffold creates initial manifests.

## What this skill does

Creates GitHub Actions workflows for a app:
- **CI**: build image → security scan (Trivy) → push to registry
- **CD**: on merge to main → deploy via Helm or kubectl

---

## Prerequisites (ask user once if not already known)

```
To set up automatic deploy, I need:
1. Where is your code? (GitHub repo URL)
2. Where does the cluster registry live? (from PLATFORM.md → registry value)
3. Do you have a GitHub Secret named KUBECONFIG with the cluster credentials?
   If not, I'll tell you how to add it.
```

---

## GitHub Secrets setup

User must add these secrets in GitHub repo → Settings → Secrets → Actions:

| Secret | Value | How to get it |
|--------|-------|---------------|
| `KUBECONFIG_B64` | base64 of kubeconfig file | `base64 -i /path/to/kubeconfig` |
| `REGISTRY_USERNAME` | registry username | from PLATFORM.md / cluster admin |
| `REGISTRY_PASSWORD` | registry password | from PLATFORM.md / cluster admin |

Tell user:
```
To add these:
1. Go to your GitHub repo → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. For KUBECONFIG_B64: run this command and paste the output:
   base64 -i <path-to-your-kubeconfig>
```

---

## CI workflow - `.github/workflows/ci.yml`

```yaml
name: CI - Build, Scan, Push

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: <APP_SLUG>-app
  REGISTRY: <REGISTRY_URL>    # from PLATFORM.md

jobs:
  build-scan-push:
    name: Build → Scan → Push
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write  # for SARIF upload

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set image tag (commit SHA)
        id: meta
        run: |
          SHA="${{ github.sha }}"
          SHORT_SHA="${SHA:0:7}"
          echo "tag=${SHORT_SHA}" >> $GITHUB_OUTPUT
          echo "full_image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${SHORT_SHA}" >> $GITHUB_OUTPUT

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64
          push: false           # build only for scan
          tags: ${{ steps.meta.outputs.full_image }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          outputs: type=docker,dest=/tmp/image.tar

      - name: Load image for scanning
        run: docker load --input /tmp/image.tar

      - name: Scan image with Trivy (vulnerabilities)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.meta.outputs.full_image }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1           # fails CI on CRITICAL/HIGH
          ignore-unfixed: true

      - name: Upload Trivy scan results
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif

      - name: Scan secrets with Trufflehog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --only-verified

      - name: Push image (main branch only)
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64
          push: true
          tags: |
            ${{ steps.meta.outputs.full_image }}
          cache-from: type=gha

      - name: Output image tag
        if: github.ref == 'refs/heads/main'
        run: |
          echo "Image pushed: ${{ steps.meta.outputs.full_image }}"
          echo "Tag: ${{ steps.meta.outputs.tag }}"
```

---

## CD workflow - low-env uses ArgoCD (preferred)

Low-env uses **ArgoCD GitOps** as primary CD method.
Workflow: CI pushes image → CD commits updated tag to git → ArgoCD detects change → syncs to cluster.

See `github-workflows/cd-argocd.yml` and `github-workflows/argocd-app.yaml`.

ArgoCD install check:
```bash
kubectl -n argocd get pods
kubectl -n argocd get applications
```

---

## CD workflow - `.github/workflows/cd.yml`

### Option A - kubectl apply (fallback, for clusters without ArgoCD)

```yaml
name: CD - Deploy to cluster

on:
  workflow_run:
    workflows: [CI - Build, Scan, Push]
    branches: [main]
    types: [completed]

jobs:
  deploy:
    name: Deploy to low-env
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    env:
      NAMESPACE: <NAMESPACE>
      APP_SLUG: <APP_SLUG>
      REGISTRY: <REGISTRY_URL>

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set image tag
        id: meta
        run: |
          SHA="${{ github.event.workflow_run.head_sha }}"
          echo "tag=${SHA:0:7}" >> $GITHUB_OUTPUT

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG_B64 }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      - name: Update image tag in manifests
        run: |
          # Replace image tag in app deployment manifest
          sed -i "s|image: ${{ env.REGISTRY }}/${{ env.APP_SLUG }}-app:.*|image: ${{ env.REGISTRY }}/${{ env.APP_SLUG }}-app:${{ steps.meta.outputs.tag }}|g" \
            k8s/08-app.yaml

      - name: Apply namespace + policies (idempotent)
        run: |
          kubectl apply -f k8s/00-namespace.yaml
          kubectl apply -f k8s/01-network-policies.yaml
          kubectl apply -f k8s/03-configmap.yaml

      - name: Apply storage (only if not exists)
        run: |
          kubectl apply -f k8s/04-storage.yaml || true

      - name: Apply DB + stateful services
        run: |
          [ -f k8s/05-db.yaml ] && kubectl apply -f k8s/05-db.yaml || true
          [ -f k8s/06-redis.yaml ] && kubectl apply -f k8s/06-redis.yaml || true
          [ -f k8s/07-rabbitmq.yaml ] && kubectl apply -f k8s/07-rabbitmq.yaml || true

      - name: Deploy app
        run: |
          kubectl apply -f k8s/08-app.yaml
          [ -f k8s/09-nginx.yaml ] && kubectl apply -f k8s/09-nginx.yaml || true
          [ -f k8s/10-ingress.yaml ] && kubectl apply -f k8s/10-ingress.yaml || true
          [ -f k8s/11-metrics-exporters.yaml ] && kubectl apply -f k8s/11-metrics-exporters.yaml || true

      - name: Wait for rollout
        run: |
          kubectl -n ${{ env.NAMESPACE }} rollout status deployment/${{ env.APP_SLUG }}-app \
            --timeout=120s

      - name: Verify health
        run: |
          INGRESS_URL="<INGRESS_URL_PATTERN>"  # from PLATFORM.md
          sleep 5
          curl -sf "${INGRESS_URL}/readiness" || echo "Health check failed - check pod logs"
```

### Option B - Helm upgrade (for Helm-based deploys)

```yaml
name: CD - Helm Deploy to cluster

on:
  workflow_run:
    workflows: [CI - Build, Scan, Push]
    branches: [main]
    types: [completed]

jobs:
  helm-deploy:
    name: Helm upgrade
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set image tag
        id: meta
        run: |
          SHA="${{ github.event.workflow_run.head_sha }}"
          echo "tag=${SHA:0:7}" >> $GITHUB_OUTPUT

      - name: Set up kubectl + helm
        run: |
          curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG_B64 }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      - name: Helm upgrade / install
        run: |
          helm upgrade --install <APP_SLUG> ./helm/app-stack \
            --namespace <NAMESPACE> \
            --create-namespace \
            --set app.image.tag=${{ steps.meta.outputs.tag }} \
            --set cluster.registry=<REGISTRY_URL> \
            --atomic \
            --timeout 2m \
            --wait
```

---

## Branch strategy

| Branch | Triggers | Action |
|--------|----------|--------|
| `main` | push | CI build + push → CD deploy |
| `develop` | push | CI build only (no push, no deploy) |
| PR | open/update | CI build + scan only |

---

## Git workflow for developers (plain language)

Tell the user:

```
How deploys work:
1. You make changes to your app
2. You push to GitHub (git push)
3. GitHub automatically:
   - Builds your app into a container
   - Scans it for security issues
   - Deploys it to the cluster
4. If there are security issues, the deploy stops and you get a notification

To trigger a deploy:
  git add .
  git commit -m "describe your change"
  git push origin main

To check deploy status:
  Go to your repo → Actions tab → latest workflow run
```

---

## Troubleshooting

### CI fails on Trivy scan

```
Error: Trivy found CRITICAL vulnerabilities
```
Fix: update the base image in Dockerfile to a patched version, or add to `.trivyignore`:
```
CVE-XXXX-XXXX  # if confirmed false positive
```

### CD fails: ErrImagePull

Cluster can't pull image.
- Check registry URL in workflow matches PLATFORM.md
- Check `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` secrets are set
- Verify image was pushed: GitHub Actions → CI job → Push step logs

### CD fails: rollout timeout

App doesn't become ready in 120s.
```bash
kubectl -n <NAMESPACE> get pods
kubectl -n <NAMESPACE> logs <pod-name> --previous
kubectl -n <NAMESPACE> describe pod <pod-name>
```
Common causes: missing secret values (CHANGE_ME not filled), DB not ready, OOMKilled.

### Workflow doesn't trigger

Check: branch name in `on.push.branches` matches your branch.
Check: secrets are set in the correct repo (not org-level if not org repo).
