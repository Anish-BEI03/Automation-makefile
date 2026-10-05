# Automation-makefile

# DevOps-Style Makefile Template

```make
# ---------- Config ----------
SHELL       := bash
.SHELLFLAGS := -eu -o pipefail -c
.DEFAULT_GOAL := help
MAKEFLAGS   += --no-print-directory

-include .env
export

APP       ?= myapp
ENV       ?= dev
REGISTRY  ?= ghcr.io/youruser
TAG       ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo dev)
IMG       := $(REGISTRY)/$(APP):$(TAG)
K8S_CTX   ?= $(ENV)-cluster
NS        ?= $(ENV)
TF_DIR    := infra/$(ENV)
INVENTORY := ansible/inventory/$(ENV)

VALID_ENVS := dev staging prod
ifeq ($(filter $(ENV),$(VALID_ENVS)),)
$(error ENV must be one of: $(VALID_ENVS))
endif

# ---------- Help ----------
.PHONY: help
help: ## Show targets
	@grep -E '^[a-zA-Z_-]+:.*?## ' $(MAKEFILE_LIST) | \
	awk 'BEGIN{FS=":.*?## "}{printf "  \033[36m%-14s\033[0m %s\n",$$1,$$2}'

# ---------- Guards ----------
.PHONY: confirm check-ctx
confirm:
	@[ "$(ENV)" != "prod" ] || { read -p "PROD action. Continue? [y/N] " a; [ "$$a" = y ]; }
check-ctx:
	@[ "$$(kubectl config current-context)" = "$(K8S_CTX)" ] || \
	{ echo "wrong kube context (want $(K8S_CTX))"; exit 1; }

# ---------- Lint & Security ----------
.PHONY: lint scan
lint: ## yamllint, hadolint, tflint, ansible-lint
	yamllint .
	hadolint Dockerfile
	terraform fmt -check -recursive infra/
	tflint --chdir=$(TF_DIR)
	ansible-lint ansible/
scan: build ## Trivy image + config scan
	trivy image --severity HIGH,CRITICAL --exit-code 1 $(IMG)
	trivy config .

# ---------- Docker ----------
.PHONY: build push
build: ## Build image
	docker build -t $(IMG) .
push: build ## Push image
	docker push $(IMG)

# ---------- Terraform ----------
.PHONY: tf-init plan apply destroy
tf-init:
	terraform -chdir=$(TF_DIR) init -input=false
plan: tf-init ## Terraform plan
	terraform -chdir=$(TF_DIR) plan -out=tfplan
apply: confirm ## Apply saved plan
	terraform -chdir=$(TF_DIR) apply tfplan
destroy: ## Destroy (always asks)
	@read -p "DESTROY $(ENV)? type '$(ENV)': " a; [ "$$a" = "$(ENV)" ]
	terraform -chdir=$(TF_DIR) destroy

# ---------- Ansible ----------
.PHONY: ping playbook
ping: ## Ansible connectivity check
	ansible all -i $(INVENTORY) -m ping
playbook: confirm ## make playbook TAGS=nginx
	ansible-playbook -i $(INVENTORY) ansible/site.yml --tags "$(TAGS)"

# ---------- Kubernetes ----------
.PHONY: deploy status logs rollback port-forward
deploy: check-ctx confirm push ## Deploy to cluster
	kubectl -n $(NS) set image deploy/$(APP) $(APP)=$(IMG)
	kubectl -n $(NS) rollout status deploy/$(APP) --timeout=120s
status: check-ctx ## Pods and rollout state
	kubectl -n $(NS) get pods,svc -l app=$(APP)
logs: check-ctx ## Tail logs
	kubectl -n $(NS) logs -f deploy/$(APP)
rollback: check-ctx confirm ## Undo last rollout
	kubectl -n $(NS) rollout undo deploy/$(APP)
port-forward: check-ctx ## Local access on :8080
	kubectl -n $(NS) port-forward deploy/$(APP) 8080:80

# ---------- CI ----------
.PHONY: ci
ci: lint build scan ## What CI runs
```

## Sample runs

```
$ make plan ENV=staging
$ make deploy ENV=prod
PROD action. Continue? [y/N]
$ make deploy ENV=qa
Makefile:17: *** ENV must be one of: dev staging prod.  Stop.
```

## Why it's DevOps-style

- **Env validation** blocks typos like `ENV=prd`.
- **Kube context check** stops wrong-cluster deploys.
- **Confirm guard** on prod apply, deploy, rollback.
- **Scan and lint** gate CI before any push.
- **Destroy** makes you type the env name.

Delete the sections you don't use (e.g. Ansible) and keep the rest.
