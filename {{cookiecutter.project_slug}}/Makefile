KUBECTL_EXEC_BACKEND = kubectl exec -it $$(kubectl get pods -l app=backend -o jsonpath="{.items[0].metadata.name}") -- bash
{% if cookiecutter.create_nextjs_frontend == "y" %}
KUBECTL_EXEC_FRONTEND = kubectl exec -it $$(kubectl get pods -l app=frontend -o jsonpath="{.items[0].metadata.name}") -- bash
{% endif %}

# colors
BLUE:=$(shell echo "\033[0;36m")
GREEN:=$(shell echo "\033[0;32m")
YELLOW:=$(shell echo "\033[0;33m")
RED:=$(shell echo "\033[0;31m")
END:=$(shell echo "\033[0m")

PROJECT_SLUG:="{{ cookiecutter.project_slug }}"

## Release/Deployment Targets

.PHONY: ci
ci: test secure build  ## Execute the same checks used in CI/CD

check-lint-and-formatting:  ## Execute check of lint and formatting using existing pre-commit hooks
	pip install pre-commit; \
	cd backend/; \
	pre-commit install; \
	pre-commit run -a

{% if cookiecutter.create_nextjs_frontend == 'y' %}
check-lint-and-test-frontend:  ## Frontend Lint & Typecheck & Test
	cd frontend/; \
	npm install --legacy-peer-deps; \
	npm run lint-src; \
	npm run typecheck; \
	npm run test
{% endif %}

backend-test: ## Execute backend tests
ifeq ($(CI),true)
	cd backend; \
	uv pip install \
		-r requirements/production.txt \
		-r requirements/tests.txt; \
	cd {{ cookiecutter.project_slug }}; \
	pytest --cov=./ --cov-report html --ds=config.settings.test
else
	$(KUBECTL_EXEC_BACKEND) -c "cd {{ cookiecutter.project_slug }} && pytest --cov=./ --cov-report html --ds=config.settings.test"
endif

{% if cookiecutter.create_nextjs_frontend == 'y' %}
frontend-test: ## Execute frontend tests
ifeq ($(CI),true)
	cd frontend/; \
	npm ci; \
	npm run test
else
	$(KUBECTL_EXEC_FRONTEND) -c "npm run test"
endif

test: backend-test frontend-test  ## Run all tests
{% else %}
test: backend-test  ## Run all tests
{% endif %}

## Development Targets

backend/requirements/local.txt: compile
backend/requirements/production.txt: compile
backend/requirements/tests.txt: compile

setup:
	@echo " $(YELLOW)⛭$(END) Checking if the setup is correct and all prerequisites are installed..."
	@MISSING=""; \
	for exec in $(PREREQUISITE_COMMANDS); do \
		if ! which $$exec > /dev/null 2>&1; then \
			MISSING="$$MISSING $$exec"; \
		fi; \
	done; \
	if [ -n "$$MISSING" ]; then \
		echo " $(RED)❌$(END)Missing executables:$$MISSING. These must be installed by you to continue"; \
		false; \
	else \
		echo " $(GREEN)✔️$(END) All prerequisites are installed."; \
	fi
	@CURRENT_CONTEXT="$$(kubectl config current-context 2>&1)"; \
	if [ "$$CURRENT_CONTEXT" = "kind-$(PROJECT_SLUG)" ]; then \
		echo " $(GREEN)✔️$(END) The kubectl context is correctly set to kind-$(PROJECT_SLUG). Run 'tilt up' to start your cluster!"; \
	else \
		echo " $(YELLOW)⛭$(END) Current kubectl context is not 'kind-$(PROJECT_SLUG)'. Switching context now..."; \
		if [ -z "$$(kubectl config get-contexts -o name | grep -w 'kind-$(PROJECT_SLUG)$$')" ]; then \
			echo " $(YELLOW)⛭$(END) No context found for kind-$(PROJECT_SLUG). Creating one. Please wait, this may take a couple of minutes on a slower machine..."; \
			kind create cluster --name $(PROJECT_SLUG) 1>/dev/null 2>/tmp/scaf_error.log; \
			echo " $(GREEN)✔️$(END) kind-$(PROJECT_SLUG) cluster and context created."; \
			echo " $(BLUE)🗣️ Remember, you can safely run \"make setup\" any time to switch between Scaf projects.$(END)"; \
		fi; \
		kubectl config use-context kind-$(PROJECT_SLUG) 1>/dev/null 2>/tmp/scaf_error.log; \
		echo " $(GREEN)✔️$(END) Context switched to kind-$(PROJECT_SLUG). Run 'tilt up' to start your cluster! "; \
	fi

outdated: ## Show all the outdated packages with their latest versions in the container
	$(KUBECTL_EXEC_BACKEND) -c "pip list --outdated"

pipdeptree: ## Show the dependencies of installed packages as a tree structure
	$(KUBECTL_EXEC_BACKEND) -c "uv pip tree --no-cache"

compile: backend/requirements/base.in backend/requirements/tests.in ## compile latest requirements to be built into the docker image
	@docker run --rm -v $(shell pwd)/backend/requirements:/local python:3.12-slim /bin/bash -c \
		"apt-get update; apt-get install -y libpq-dev; \
		 pip install uv; \
		 touch /local/base.txt; touch /local/production.txt; touch /local/tests.txt; touch /local/local.txt; \
		 uv pip compile --upgrade --generate-hashes --output-file /local/base.txt /local/base.in; \
		 uv pip compile --upgrade --generate-hashes --output-file /local/production.txt /local/production.in; \
		 uv pip compile --upgrade --generate-hashes --output-file /local/tests.txt /local/tests.in; \
		 uv pip compile --upgrade --output-file /local/local.txt /local/local.in"

shell-backend:  ## Shell into the running Django container
	$(KUBECTL_EXEC_BACKEND)

{% if cookiecutter.create_nextjs_frontend == "y" %}
shell-frontend:
	$(KUBECTL_EXEC_FRONTEND)
{% endif %}

.PHONY: secure
secure:  ## Analyze dependencies for security issues
	$(KUBECTL_EXEC_BACKEND) -c "safety check"

DEBUG ?= false
secrets:
	@$(call check_var,ENVIRONMENT)
	@echo "Creating sealed secrets for $$(kubectl config current-context) cluster"
	kubectl config get-contexts --no-headers | \
		grep {{ cookiecutter.project_slug }}-$(ENVIRONMENT) && \
		kubectl config use-context admin@{{ cookiecutter.project_slug }}-$(ENVIRONMENT)
	ENVIRONMENT=$(ENVIRONMENT) \
	envsubst < k8s/templates/secrets.yaml.template | op inject | if [ "$(DEBUG)" = "true" ]; then cat; else kubeseal -n {{ cookiecutter.project_slug }}-$(ENVIRONMENT) --format yaml > k8s/$(ENVIRONMENT)/secrets.yaml; fi

sandbox-secrets: ## Create sealed secrets for sandbox
	$(MAKE) secrets ENVIRONMENT=sandbox

debug-sandbox-secrets: ## Debug sandbox secrets
	$(MAKE) secrets ENVIRONMENT=sandbox DEBUG=true

staging-secrets: ## Create sealed secrets for sandbox
	$(MAKE) secrets ENVIRONMENT=sandbox

debug-staging-secrets: ## Debug sandbox secrets
	$(MAKE) secrets ENVIRONMENT=sandbox DEBUG=true

prod-secrets: ## Create sealed secrets for prod
	$(MAKE) secrets ENVIRONMENT=prod

debug-prod-secrets: ## Create sealed secrets for prod
	$(MAKE) secrets ENVIRONMENT=prod DEBUG=true


argocd-app:
	envsubst < k8s/templates/argocd.application.yaml.template | kubeseal --format yaml > k8s/argocd/application.yaml
	envsubst < k8s/templates/repocreds.yaml.template | kubeseal --format yaml > k8s/argocd/repocreds.yaml
	sed -i '/spec:/ {N; s/spec:\n/  annotations:\n    sealedsecrets.bitnami.com\/managed: "true"\nspec:\n/}' k8s/argocd/repocreds.yaml

## Monitoring
check_var = $(strip $(if $(filter undefined,$(origin $1)),$(error $1 is not set)))

kube-prometheus-up:  ## Install kube-prometheus
	@$(call check_var,GRAFANA_ADMIN_PASSWORD)
	@helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
	@helm repo update
	@helm install kube-prometheus prometheus-community/kube-prometheus-stack --set grafana.sidecar.datasources.isDefaultDatasource=false,grafana.persistence.enabled=true,grafana.adminPassword=$(GRAFANA_ADMIN_PASSWORD) --namespace monitoring --create-namespace

loki-up:  ## Install loki
	@helm repo add grafana https://grafana.github.io/helm-charts
	@helm repo update
	@helm install loki grafana/loki-stack --values k8s/_monitoring/loki-stack-values.yaml --namespace monitoring --create-namespace

monitoring-up: kube-prometheus-up loki-up  ## Install the monitoring stack

loki-down:  ## Uninstall loki
	helm uninstall loki --namespace monitoring

kube-prometheus-down:  ## Uninstall kube-prometheus
	helm uninstall kube-prometheus --namespace monitoring

monitoring-down: loki-down kube-prometheus-down  ## Uninstall the monitoring stack

monitoring-login:  ## Get the login credentials for the monitoring stack
	@kubectl get secret --namespace monitoring kube-prometheus-grafana -o jsonpath='{.data.admin-user}' | base64 -d; echo
	@kubectl get secret --namespace monitoring kube-prometheus-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo

monitoring-port-forward:  ## Forward the port for the monitoring stack
	kubectl --namespace monitoring port-forward svc/kube-prometheus-grafana 8080:80

monitoring-dashboard:  ## Create the dashboard for the monitoring stack
	kubectl apply -f k8s/_monitoring/django-logs-table.yaml -n monitoring

monitoring:  ## Show the status of the monitoring stack
	kubectl get all -n monitoring

## Help

help: ## Show the list of all the commands and their help text
	@awk 'BEGIN 	{ FS = ":.*##"; target="";printf "\nUsage:\n  make \033[36m<target>\033[33m\n\nTargets:\033[0m\n" } \
		/^[a-zA-Z_-]+:.*?##/ { if(target=="")print ""; target=$$1; printf " \033[36m%-10s\033[0m %s\n\n", $$1, $$2 } \
		/^([a-zA-Z_-]+):/ {if(target=="")print "";match($$0, "(.*):"); target=substr($$0,RSTART,RLENGTH) } \
		/^\t## (.*)/ { match($$0, "[^\t#:\\\\]+"); txt=substr($$0,RSTART,RLENGTH);printf " \033[36m%-10s\033[0m", target; printf " %s\n", txt ; target=""} \
		/^## .*/ {match($$0, "## (.+)$$"); txt=substr($$0,4,RLENGTH);printf "\n\033[33m%s\033[0m\n", txt ; target=""} \
	' $(MAKEFILE_LIST)

.PHONY: help build build-dev outdated pipdeptree compile destroy-data load-dump clean clean-dev squeaky_clean update check-sandbox-release check-staging-release check-prod-release

.DEFAULT_GOAL := help
