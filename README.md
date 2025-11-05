# Course Production dev environment setup instructions

This repository collects concise how-to's and setup instructions for common tasks such as pipeline setup, application deployment setup, and GitHub Actions configuration.

## Table of contents

- [Run in GitHub Codespaces](#run-in-github-codespaces)
- [Kubernetes testing (k3d, Skaffold, Helm)](#kubernetes-testing-k3d-skaffold-helm)
  - [Create a local cluster (k3d)](#create-a-local-cluster-k3d)
  - [Continuous dev with Skaffold](#continuous-dev-with-skaffold)
  - [Test deployment with Helm](#test-deployment-with-helm)
 - [GitHub Actions (CI/CD)](#github-actions-cicd)

## Run in GitHub Codespaces

This repo can be developed and tested in GitHub Codespaces using a dev container similar to `bcit-ltc/qcon-api` (Docker-in-Docker, nix tools, and post-create/start scripts). Replace placeholders like `<image>`, `<port>`, `<namespace>`, `<service>` with your project's values. `qcon-api` is referenced as an example only.

1. Create a Codespace on this repo (GitHub → Code → Create codespace).
2. Wait for the container to build and post-create scripts to finish.
3. If prompted by the direnv extension, run `direnv allow` in the project root.
4. Verify Docker works:
    ```bash
    docker version
    docker info
    ```
5. Start the API/service:
    - Option A (published image):
      ```bash
      docker run -p <host_port>:<container_port> <image>[:<tag>]
      # Example (from qcon-api): docker run -p 8000:8000 ghcr.io/bcit-ltc/qcon-api
      ```
    - Option B (compose from source):
      ```bash
      docker compose up --build
      ```
6. Open the forwarded port in your browser (e.g., `<host_port>`; commonly 8000). If needed, set the port’s visibility to Public in the Ports panel.
7. Stop with Ctrl+C, or run `docker compose down`.

Troubleshooting:
- If Docker errors, rebuild the container (Dev Containers: Rebuild Container) or restart the Codespace.
- Ensure the app binds to `0.0.0.0:<container_port>` so Codespaces can expose it.
- Open a new terminal after initialization to pick up group membership/env changes.
- For env vars, use a `.env` file (auto-read by `docker compose`) or export them before running.

## Kubernetes testing (k3d, Skaffold, Helm)

The dev container includes Kubernetes tools (`k3d`, `kubectl`, `skaffold`, `helm`). Use the steps below to test a k8s deployment inside Codespaces.

### Create a local cluster (k3d)
1. Create a cluster and set context:
    ```bash
    k3d cluster create dev
    kubectl config use-context k3d-dev
    kubectl get nodes
    ```
2. (Optional) If you build images locally and deploy via Helm directly, import the image into k3d:
    ```bash
    # example
    docker build -t <image>:<tag> .
    k3d image import <image>:<tag> -c dev
    ```

### Continuous dev with Skaffold
Requires a `skaffold.yaml` in the repo.
```bash
skaffold dev --kube-context k3d-dev --port-forward
```
- Rebuilds on code changes and redeploys automatically.
- With `--port-forward`, Skaffold creates local forwards for services/pods.
- Stop with Ctrl+C.

Common tweaks:
- Use `-n <namespace>` or define namespace in your manifests.
- Use profiles: `skaffold dev -p <profile> --kube-context k3d-dev`.

### Test deployment with Helm
If you have a Helm chart (e.g., in `./chart`):
```bash
kubectl create namespace <namespace> || true
helm upgrade --install <release_name> <chart_path> \
  --namespace <namespace> \
  --set image.repository=<image_repo> \
  --set image.tag=<tag>

kubectl -n <namespace> get pods
```

Port-forward to access the service in Codespaces (adjust names/ports to match your chart):
```bash
kubectl -n <namespace> port-forward svc/<service_name> <host_port>:<service_port>
# Example (from qcon-api): kubectl -n qcon port-forward svc/qcon-api 8000:80
# Open the forwarded port in the Ports panel
```

Cleanup:
```bash
helm -n <namespace> uninstall <release_name>
k3d cluster delete dev

## GitHub Actions (CI/CD)

Use a reusable workflow from [`bcit-ltc/.github`](https://github.com/bcit-ltc/.github) to build and push application and chart images to your container registry (e.g., GHCR). Call the shared workflow from your repository to standardize CI/CD.

### Overview
- Trigger: on push to `main` (adjust as needed).
- Job: delegates to a reusable workflow in [`bcit-ltc/.github`](https://github.com/bcit-ltc/.github).
- Permissions: writes to contents, packages, OIDC tokens, attestations; and optionally PRs/issues.
- Secrets: inherited from the caller repository/environment.

### Example: reuse the shared workflow
Create `.github/workflows/build-and-push-app.yaml` in your repo:

```yaml
name: Build and push app/chart images

on:
  push:
    branches: [ main ]

jobs:
  build-and-ship:
    uses: bcit-ltc/.github/.github/workflows/build-and-push-app.yaml@main
    permissions:
      contents: write
      packages: write
      id-token: write
      attestations: write
      pull-requests: write
      issues: write
    secrets: inherit
```

Notes:
- Ensure your repo has permissions to push to your container registry (e.g., `GHCR` via `packages: write`).
- If your default branch is not `main`, adjust the trigger.
- Add any required inputs/vars per your project’s conventions if the reusable workflow expects them.
