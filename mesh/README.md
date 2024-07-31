### Lab Cilium Mesh

A lab for Cilium Mesh networking, with this code you will be able to spin-up two cluster connected to two different routers that simulated different networks where you will build a Mesh an from client0 test that everything works (or not?).

### Pre-requisites

You should have the following tools installed (make sure to also check their pre-requisites):
- [docker](https://docs.docker.com/engine/install/)
- [containerlab](https://containerlab.dev/install/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [cilium-cli](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli)

You maybe already have all the pre-reqs, execpt for containerlab. This one is easy to install if you are in de Debian derivative.

```shell
echo "deb [trusted=yes] https://netdevops.fury.site/apt/ /" | \
sudo tee -a /etc/apt/sources.list.d/netdevops.list

sudo apt update && sudo apt install containerlab
```

For more installation steps on other platforms [containerlab/install](https://containerlab.dev/install/)


### How to start the clusters and routers.

All the steps are described in the `Makefile`, just type `make` to initialize the lab.

When you're done, use `make clean` to remove the lab.
> This lab can run totally separate from the other labs, if your machine have the resources, the labs can be running at the same time.

### What is being tested.

 In this lab you will be able to get familiar with the Cilium configurations needed to connect multiple Kubernetes clusters together in a Mesh.

 > Cilium installation can be made using make or you can do it manually if this is the first time.
 > using make cilium-install-cluster1 and make cilium-install-cluster2

or

```shell
	kubectl config use-context kind-cluster1
	cilium install --version=1.16 \
	  --helm-set cluster.name=cluster1 \
	  --helm-set cluster.id=1 \
	  --helm-set ipam.mode=kubernetes \
	  --helm-set tunnel-protocol=vxlan \
	  --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
	  --helm-set bgpControlPlane.enabled=true \
	  --helm-set k8s.requireIPv4PodCIDR=true
```
 > change the context to cluster2, the name and id, now you can run the same command targeted to cluster2.
 > if you look closelly to the cilium install, there is a name and also an ID, since this cluster will talk to each other, there need to be able to identify each other.

Before enabling Mesh, lets apply some BGP configuration to be able to use LoadBalancer service for the Mesh endpoints, if you played with the BGP Labs, this is the same.
```shell
kubectl config use-context kind-cluster1
kubectl apply -f peering-policy.yaml
kubectl apply -f cluster1-public-pool.yaml

kubectl config use-context kind-cluster2
kubectl apply -f peering-policy.yaml
kubectl apply -f cluster1-public-pool.yaml
```



Now we need to enable the actual mesh functionality.

```shell
cilium clustermesh enable --context kind-cluster1 --enable-kvstoremesh=false --service-type LoadBalancer
cilium clustermesh enable --context kind-cluster2 --enable-kvstoremesh=false --service-type LoadBalancer
```
Check for Mesh status in Cilium
```shell
cilium clustermesh status --context kind-cluster1 --wait 
cilium clustermesh status --context kind-cluster2 --wait 
```
> In some cases, the service type cannot be automatically detected and you need to specify it manually. This can be done with the option --service-type
> LoadBalancer:
> A Kubernetes service of type LoadBalancer is used to expose the control plane. This uses a stable LoadBalancer IP and is typically the best option.
> note: Remenber that LoadBalancer is not available unless configure, you can use part of the bgp lab.

Finally, connect the clusters. This step only needs to be done in one direction. The connection will automatically be established in both directions:
```shell
cilium clustermesh connect --context kind-cluster1 --destination-context kind-cluster2
```
It may take a bit for the clusters to be connected. You can run cilium clustermesh status --wait to wait for the connection to be successful:
```shell
cilium clustermesh status --context kind-cluster1 --wait
```

Testing pod connectivity between clusters.
```shell
cilium connectivity test --context kind-cluster1 --multi-cluster kind-cluster2
```

> All this was taken from Cilium Docs [Great documentation!](https://docs.cilium.io/en/stable/network/clustermesh/clustermesh/)


### Installing an application and making it available from both clusters.
WIP....