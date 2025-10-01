# Makefile — Cilium Mesh Lab
# Usage:
#   make                # shows help
#   make <target>       # run a task

# Cilium version
# always latest!
CILIUM_VERSION ?= $(shell curl -s https://raw.githubusercontent.com/cilium/cilium/main/stable.txt)

# ========= Settings (override with: make VAR=value) =========
KUBECONFIG            ?= $(HOME)/.kube/config
CLAB_TOPO             ?= topo.yaml
CTX1                  ?= kind-cluster1
CTX2                  ?= kind-cluster2
BR1                   ?= br-cluster1
BR2                   ?= br-cluster2

# Cilium install defaults
IPAM_MODE             ?= kubernetes
TUNNEL                ?= vxlan
NATIVE_ROUTING_CIDR   ?= 10.0.0.0/8
BGP_CP                ?= true
REQUIRE_POD_CIDR      ?= true

# Binaries (override if needed)
KUBECTL ?= kubectl
CILIUM  ?= cilium
CLAB    ?= containerlab
IP      ?= ip

# Use sudo only if not running as root
SUDO    := $(if $(filter 0,$(shell id -u)),,sudo)

# Safer shell (fail fast)
SHELL := bash
.SHELLFLAGS := -eu -o pipefail -c

# Default target + hygiene
.DEFAULT_GOAL := help
.ONESHELL:
.SILENT:                     # quiet recipes unless they echo/$(info)
.EXPORT_ALL_VARIABLES:
.PHONY: help default deps bridges ciliumlab cluster1-status cluster2-status \
        cilium-install-cluster1 cilium-install-cluster2 clean status all-status \
        uninstall-cilium-cluster1 uninstall-cilium-cluster2

## Show this help (default).
help:
	awk 'BEGIN {FS = ":.*##"; print "Usage: make <target>\nTargets:"} \
	/^[a-zA-Z0-9_.-]+:.*##/ { printf "  \033[36m%-28s\033[0m %s\n", $$1, $$2 }' $(MAKEFILE_LIST)

## Full lab bring-up: deps + bridges + containerlab + both cluster statuses.
init: deps bridges ciliumlab all-status ## Bring up lab and show statuses.
## Full lab bring-up: deps + bridges + containerlab + apply bgp + enable mesh.
build-all: deps bridges ciliumlab cilium-install-cluster1 cilium-install-cluster2 apply-bgp enable-mesh all-status ## Build the lab completely.

## Verify required tools are present.
deps: ## Check for required tools.
	@echo "Checking pre-required tools…"
	command -v $(KUBECTL) >/dev/null || { echo "kubectl not found"; exit 1; }
	command -v $(CILIUM)  >/dev/null || { echo "cilium CLI not found"; exit 1; }
	command -v $(CLAB)    >/dev/null || { echo "containerlab not found"; exit 1; }
	command -v $(IP)      >/dev/null || { echo "iproute2 'ip' not found"; exit 1; }
	@echo "OK"

## Create/up bridges (idempotent).
bridges: ## Create Linux bridges for cluster interconnects.
	@echo "Creating bridges for Cilium LAB for Mesh networking…"
	($(IP) link show $(BR1) >/dev/null 2>&1) || $(SUDO) $(IP) link add $(BR1) type bridge
	$(SUDO) $(IP) link set $(BR1) up
	($(IP) link show $(BR2) >/dev/null 2>&1) || $(SUDO) $(IP) link add $(BR2) type bridge
	$(SUDO) $(IP) link set $(BR2) up
	@echo "Bridges ready: $(BR1), $(BR2)"

## Deploy the containerlab topology.
ciliumlab: ## Deploy containerlab topology.
	@echo "Initializing Cilium LAB for Mesh networking…"
	$(SUDO) -E $(CLAB) deploy -t $(CLAB_TOPO)

## Show status for Cluster1 (context + nodes).
cluster1-status: ## Show node status for Cluster1.
	@echo "Status for Cluster1…"
	$(KUBECTL) config use-context $(CTX1)
	$(KUBECTL) get nodes -o wide

## Show status for Cluster2 (context + nodes).
cluster2-status: ## Show node status for Cluster2.
	@echo "Status for Cluster2…"
	$(KUBECTL) config use-context $(CTX2)
	$(KUBECTL) get nodes -o wide

## at one point we are not going to do this manually
apply-bgp: ## Apply BGP configuration to all clusters.
	@echo "Applying BGP configuration…"
	$(KUBECTL) config use-context kind-cluster1
	$(KUBECTL) apply -f cluster1-bgp-config.yaml
	$(KUBECTL) apply -f cluster1-public-pool.yaml
	$(KUBECTL) config use-context kind-cluster2
	$(KUBECTL) apply -f cluster2-bgp-config.yaml
	$(KUBECTL) apply -f cluster2-public-pool.yaml

## at one point we are not going to do this manually
enable-mesh: ## Enable Mesh on all clusters.
	@echo "Enabling Mesh on all clusters…"
	$(KUBECTL) config use-context kind-cluster1
	$(CILIUM) clustermesh enable --context kind-cluster1 --enable-kvstoremesh=false --service-type LoadBalancer
	$(KUBECTL) config use-context kind-cluster2
	$(CILIUM) clustermesh enable --context kind-cluster2 --enable-kvstoremesh=false --service-type LoadBalancer

rebuild: clean init ## Rebuild the lab.

## Show status for both clusters.
all-status: cluster1-status cluster2-status ## Show node status for both clusters.

# ---- Cilium install helpers (DRY) ----
define INSTALL_CILIUM
	@echo "Installing Cilium on $(1)…"
	$(KUBECTL) config use-context $(2)
	$(CILIUM) install --version=$(CILIUM_VERSION) \
	  --helm-set cluster.name=$(1) \
	  --helm-set cluster.id=$(3) \
	  --helm-set ipam.mode=$(IPAM_MODE) \
	  --helm-set routing-mode=native \
	  --helm-set tunnel-protocol=$(TUNNEL) \
	  --helm-set ipv4NativeRoutingCIDR="$(NATIVE_ROUTING_CIDR)" \
	  --helm-set bgpControlPlane.enabled=$(BGP_CP) \
	  --helm-set k8s.requireIPv4PodCIDR=$(REQUIRE_POD_CIDR) \
	  --helm-set kubeProxyReplacement=true 

	$(CILIUM) status --wait || true
endef

define UNINSTALL_CILIUM
	@echo "Uninstalling Cilium on $(1)…"
	$(KUBECTL) config use-context $(2)
	$(CILIUM) uninstall --wait || true
endef

## Install Cilium (cluster1).
cilium-install-cluster1: ## Install Cilium v$(CILIUM_VERSION) on Cluster1.
	$(call INSTALL_CILIUM,cluster1,$(CTX1),1)

## Install Cilium (cluster2).
cilium-install-cluster2: ## Install Cilium v$(CILIUM_VERSION) on Cluster2.
	$(call INSTALL_CILIUM,cluster2,$(CTX2),2)

## Uninstall Cilium (cluster1).
uninstall-cilium-cluster1: ## Uninstall Cilium from Cluster1.
	$(call UNINSTALL_CILIUM,cluster1,$(CTX1))

## Uninstall Cilium (cluster2).
uninstall-cilium-cluster2: ## Uninstall Cilium from Cluster2.
	$(call UNINSTALL_CILIUM,cluster2,$(CTX2))

## Destroy lab & bridges (idempotent).
clean: ## Destroy containerlab topology and bridges.
	@echo "Cleaning up Cilium LAB for Mesh networking…"
	$(SUDO) -E $(CLAB) destroy -t $(CLAB_TOPO) || true
	$(SUDO) rm -f .topo.yaml.bak || true
	$(SUDO) rm -rf clab-ciliumlab || true
	($(IP) link show $(BR1) >/dev/null 2>&1) && $(SUDO) $(IP) link delete $(BR1) || true
	($(IP) link show $(BR2) >/dev/null 2>&1) && $(SUDO) $(IP) link delete $(BR2) || true
	@echo "Cleanup complete"
