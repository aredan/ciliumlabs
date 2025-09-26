KUBECONFIG := $(HOME)/.kube/config

.PHONY: default ciliumlab cluster1-status cluster2-status cilium-install-cluster1 cilium-install-cluster2 clean

default: ciliumlab cluster1-status cluster2-status

ciliumlab:
	$(info Initializing Cilium LAB for Mesh networking....)
	sudo -E containerlab deploy -t topo.yaml

cluster1-status:
	$(info Status for Cluster1....)
	kubectl config use-context kind-cluster1
	kubectl get nodes -owide

cluster2-status:
	$(info Status for Cluster2....)
	kubectl config use-context kind-cluster2
	kubectl get nodes -owide


cilium-install-cluster1:
	$(info Installing Cilium on Cluster1........)
	kubectl config use-context kind-cluster1
	cilium install --version=1.18 \
	  --helm-set cluster.name=cluster1 \
	  --helm-set cluster.id=1 \
	  --helm-set ipam.mode=kubernetes \
	  --helm-set tunnel-protocol=vxlan \
	  --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
	  --helm-set bgpControlPlane.enabled=true \
	  --helm-set k8s.requireIPv4PodCIDR=true

cilium-install-cluster2:
	$(info Installing Cilium on Cluster2........)
	kubectl config use-context kind-cluster2
	cilium install --version=1.18 \
	  --helm-set cluster.name=cluster2 \
	  --helm-set cluster.id=2 \
	  --helm-set ipam.mode=kubernetes \
	  --helm-set tunnel-protocol=vxlan \
	  --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
	  --helm-set bgpControlPlane.enabled=true \
	  --helm-set k8s.requireIPv4PodCIDR=true

clean:
	sudo -E containerlab destroy -t topo.yaml