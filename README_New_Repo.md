 # New Repo First-Time Dev Setup (Devcontainer)

 This guide defines the minimal files a new repository must include so the devcontainer works on day one. In this step, we focus on two files only:

 - **.envrc**
 - **Makefile**

 The **.devcontainer** and **.github/workflows** folders will be added next.


 ## Prerequisites

 - **direnv** (for loading environment automatically)
 - **Docker** (Docker Desktop) and **Docker Compose**
 - **make**
 - **zsh** or **bash**

 Quick setup for direnv:

 - Install direnv and add the shell hook (per direnv docs).
 - After adding `.envrc`, run: `direnv allow`


 ## .envrc

 - **Purpose**: Load shared environment for local dev, especially inside the devcontainer.
 - **Pattern** taken from `bcit-ltc/qcon-api`.

 Put this in `.envrc` at the repo root:

 ```sh
 # Always load shared script env first
 source ./.devcontainer/scripts/env.sh

 # Optional interactive-only overrides
 dotenv_if_exists ./.devcontainer/skaffold/skaffold.env
 ```

 Notes:

 - **source ./.devcontainer/scripts/env.sh** loads variables and helper paths used by Make targets and scripts.
 - **dotenv_if_exists** pulls in local, optional overrides for interactive workflows (e.g., Skaffold). It won’t fail if the file is missing.
 - Commit `.envrc`, but do not commit secret values. Keep secrets in files that are gitignored (e.g., a local `.env` or `skaffold.env`) or in your secret manager.
 - After creating/updating `.envrc`, run `direnv allow` to let your shell load it.


 ## Makefile

 - **Purpose**: Provide a consistent command surface for common local-dev tasks (cluster setup, dashboard, chart operations, cleanup), and ensure scripts run with the right shell/env.
 - **Pattern** aligned with `bcit-ltc/qcon-api`.

 Use the following `Makefile` at the repo root:

 ```make
 # Prefer zsh, fall back to bash
 SHELL := $(shell command -v zsh 2>/dev/null || command -v bash)
 
 # Only set ZDOTDIR if the Make shell is zsh
 ifeq ($(notdir $(SHELL)),zsh)
   export ZDOTDIR := $(CURDIR)/.devcontainer/scripts
 endif
 
 # Tool discovery
 K3D      := $(shell command -v k3d)
 KUBECTL  := $(shell command -v kubectl)
 HELM     := $(shell command -v helm)
 SKAFFOLD := $(shell command -v skaffold)
 DOCKER   := $(shell command -v docker)
 
 # Files/paths
 ENVSH   := $(CURDIR)/.devcontainer/scripts/env.sh
 LIBSH   := $(CURDIR)/.devcontainer/scripts/lib.sh
 K3D_CFG := $(CURDIR)/.devcontainer/k3d/k3d.yaml
 
 # Targets
 .PHONY: help cluster dashboard chart token delete
 
 help:
 	@echo ""
 	@echo "Targets:"
 	@echo ""
 	@echo "  cluster     → create k3d cluster using config \"$(K3D_CFG)\""
 	@echo "  dashboard   → install Kubernetes Dashboard and print login token"
 	@echo "  token       → re-print Kubernetes Dashboard login token"
 	@echo "  chart       → pull/unpack app chart (clobbers existing files)"
 	@echo "                  - set APP_CHART_URL to override default \"oci://ghcr.io/$${ORG_NAME}/oci/$${APP_NAME}\""
 	@echo "  delete      → delete all k3d clusters (local dev cleanup)"
 	@echo ""
 	@echo "Other devcontainer commands:"
 	@echo ""
 	@echo "  docker compose up                   → local dev"
 	@echo "  skaffold dev                        → build + deploy to local cluster to verify deployment/helm release"
 	@echo "  nix-shell -p {nixPackage}           → enter nix shell with specific package"
 	@echo "  helm repo add {repoName} {repoURL}  → add a helm repository"
 	@echo "  kubeconform|kubeval {file}          → validate Kubernetes YAML files"
 	@echo ""
 
 cluster:
 	@. "$(ENVSH)"; . "$(LIBSH)"; \
 	"$(CURDIR)/.devcontainer/scripts/cluster.sh"
 	@. "$(ENVSH)"; . "$(LIBSH)"; \
 	"$(CURDIR)/.devcontainer/scripts/app-chart.sh"
 
 dashboard:
 	@. "$(ENVSH)"; . "$(LIBSH)"; \
 	"$(CURDIR)/.devcontainer/scripts/kubernetes-dashboard.sh"
 
 chart:
 	@. "$(ENVSH)"; . "$(LIBSH)"; \
 	"$(CURDIR)/.devcontainer/scripts/app-chart.sh"
 
 token:
 	@. "$(ENVSH)" && \
 	if [ -s "$$TOKEN_PATH" ]; then \
 	  echo "Token file: $$TOKEN_PATH"; \
 	  echo "---- TOKEN ----"; \
 	  cat "$$TOKEN_PATH"; echo; \
 	else \
 	  echo "No token found. Run 'make dashboard' first."; \
 	  exit 1; \
 	fi
 
 delete:
 	@echo "❌ Deleting all k3d clusters..."
 	@. "$(ENVSH)"; \
 	if [ -z "$(K3D)" ]; then echo "k3d not found"; exit 127; fi; \
 	$(K3D) cluster delete -a || true; \
 	rm -f "$$TOKEN_PATH"
 ```
 
 Notes:
 
 - The Makefile defers to scripts located under `.devcontainer/scripts`. We will provide these in the next step. Until then, the targets may not run.
 - `SHELL` is set to `zsh` when available for better compatibility with the provided scripts; otherwise it falls back to `bash`.
 - `ZDOTDIR` is pointed at `.devcontainer/scripts` so zsh loads repo-local zsh config for scripts.
 - Tool discovery lines make targets fail fast if a required tool is missing.
 - `ENVSH` and `LIBSH` centralize env and helper functions. `TOKEN_PATH` and other variables are populated by `env.sh`.
 
 
 ## First-time setup flow (now)
 
 - Add the above `.envrc` and `Makefile` to your new repo.
 - Install direnv and enable its shell hook.
 - In the repo root, run: `direnv allow`
 - Run: `make help` to see available targets.
 
 Once the `.devcontainer` folder is added, you can:
 
 - `docker compose up` for local dev (service-dependent).
 - `make cluster` to create a local k3d cluster.
 - `make dashboard` then `make token` to view Kubernetes Dashboard.
 - `skaffold dev` to build and deploy to the local cluster.


## .devcontainer explained

Structure (as used in qcon-api):

- **.devcontainer/scripts/**
  - **env.sh**: Central env defaults. Defines `APP_NAME`, `ORG_NAME`, `CLUSTER_NAME`, `REGISTRY_HOST`, chart ref precedence (`APP_CHART_URL` → `CHART_REF` → default `oci://{REGISTRY_HOST}/{ORG_NAME}/oci/{APP_NAME}`), k3d config path, Skaffold defaults, and PATH augmentation.
  - **lib.sh**: Common helpers (`need`, `die`, `log`) and strict shell flags.
  - **cluster.sh**: Creates a local k3d cluster using `K3D_CFG_PATH` (defaults to `.devcontainer/k3d/k3d.yaml`). Waits for nodes.
  - **app-chart.sh**: Pulls and unpacks the application Helm chart into `./app-chart` using `CHART_REF` or `APP_CHART_URL` (optional `APP_CHART_VERSION`).
  - **kubernetes-dashboard.sh**: Installs Dashboard via Helm and writes a login token to `TOKEN_PATH`. Prints a `kubectl port-forward` command and the token.
- **.devcontainer/k3d/k3d.yaml**
  - k3d cluster config. References `SKAFFOLD_DEFAULT_REPO` for the local registry mirror.
- **.devcontainer/skaffold/skaffold.env**
  - Optional overrides for Skaffold defaults (`SKAFFOLD_DEFAULT_REPO`, `SKAFFOLD_PORT_FORWARD`, `SKAFFOLD_FILENAME`). Loaded by `.envrc` via `dotenv_if_exists`.

Key environment variables (from `scripts/env.sh`):

- **APP_NAME**, **ORG_NAME**, **CLUSTER_NAME**: Identity defaults (repo folder name used for `APP_NAME` if unset).
- **APP_STATE_DIR**, **TOKEN_PATH**: Local state directory and dashboard token file path.
- **K3D_CFG_PATH**: Path to the k3d config file (defaults to `.devcontainer/k3d/k3d.yaml`).
- **REGISTRY_HOST**: Registry host (default `ghcr.io`).
- **CHART_REF** and **APP_CHART_URL**: Chart reference resolution; you can override with `APP_CHART_URL` or set `CHART_REF` directly.
- **SKAFFOLD_DEFAULT_REPO**, **SKAFFOLD_PORT_FORWARD**, **SKAFFOLD_FILENAME**, **SKAFFOLD_ENV_FILE**: Skaffold defaults used by local dev tooling.

Usage notes:

- After copying `.devcontainer`, ensure `.envrc` sources `./.devcontainer/scripts/env.sh` and then run `direnv allow`.
- Run `make help`, then use `make cluster`, `make dashboard`, `make token`. `skaffold dev` works if you have a valid `skaffold.yaml` (path defaults to `.devcontainer/skaffold/skaffold.yaml` per env).


## Copy .devcontainer from qcon-api (temporary bootstrap)

Until we create a shared template repo, copy the working devcontainer from `bcit-ltc/qcon-api`:

- Option A (simple, manual):
  - Open https://github.com/bcit-ltc/qcon-api
  - Click Code → Download ZIP
  - Extract, then copy its `.devcontainer` folder into your new repo root.

- Option B (advanced, sparse-checkout):
  ```sh
  git init tmp-qcon && cd tmp-qcon
  git remote add origin https://github.com/bcit-ltc/qcon-api.git
  git sparse-checkout init --cone
  git sparse-checkout set .devcontainer
  git pull origin HEAD
  cp -R .devcontainer ../
  cd .. && rm -rf tmp-qcon
  ```

After copying:

- Review `.devcontainer/scripts/env.sh` and update any repo-specific values (e.g., org/app names).
- Run `direnv allow` (if `.envrc` changed), then `make help`.
- You can now run the Make targets listed above.


