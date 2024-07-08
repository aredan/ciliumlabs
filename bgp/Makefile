KUBECONFIG := $(HOME)/.kube/config

default: kind clab cilium wait-for-cilium bgp status

kind:
	kind create cluster --config cluster.yaml

clab:
	sudo containerlab -d --log-level debug deploy -t topo.yaml

cilium:
	cilium install --version=1.14 \
	  --helm-set ipam.mode=kubernetes \
	  --helm-set tunnel=disabled \
	  --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
	  --helm-set bgpControlPlane.enabled=true \
	  --helm-set k8s.requireIPv4PodCIDR=true

wait-for-cilium:
	cilium status --wait

bgp:
	kubectl apply -f cilium-bgp-peering-policies.yaml

status:
	kubectl get pods -n kube-system

clean: clean-kind clean-clab

clean-kind:
	kind delete clusters clab-bgp-cplane-demo

clean-clab:
	sudo containerlab destroy -t topo.yaml
