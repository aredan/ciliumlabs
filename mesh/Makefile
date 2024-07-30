KUBECONFIG := $(HOME)/.kube/config

default: ciliumlab node-status

ciliumlab:
	$(info Building Kind Cluster containerlab network routers....)
	sudo containerlab deploy -t topo.yaml

node-status:
	$(info Checking for nodes in the cluster....)
	kubectl get nodes

cilium-install:
	$(info Installing Cilium........)
	cilium install --version=1.16 \
	  --helm-set ipam.mode=kubernetes \
	  --helm-set tunnel-protocol=vxlan \
	  --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
	  --helm-set bgpControlPlane.enabled=true \
	  --helm-set k8s.requireIPv4PodCIDR=true