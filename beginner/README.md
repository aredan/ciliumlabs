
> Before starting with this step, validate the pre-reqs.

Here you will find steps to create a basic K8s with Kind and deploy Cilium CNI to play with  polices and get your first experience with this awesome tool!


#### Create kind cluster.
Using this yaml code, createa file and save it as cluster1.yml

```yaml
kind: Cluster
name: cilium1
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
networking:
  podSubnet: 10.241.0.0/16
  serviceSubnet: 10.112.0.0/16
  disableDefaultCNI: true
```

Create kind cluster with cluster1.yml 

```shell
kind create cluster --config cluster1.yml
```

#### Validating cluster.


```shell
kubectl get nodes
kubectl get all 
kubectl get pods -A
kubectl config current-context
```

![cluster status](pictures/kgetnodes.jpg)

> node status is NotReady, this is because there is not CNI installed yet.


#### Install and setup Cilium with Hubble
Cilium and hubble will be installed on this local Kubernetes cluster.

```shell
cilium install  --set cluster.name=cilium1 --set cluster.id=1 --context kind-cilium1
cilium status --wait
cilium hubble enable --ui
cilium hubble port-forward &
cilium hubble ui &
cilium connectivity test
```

Cilium will be installed with name "cilium1" and ID 1, this way we can create a second cluster and just set name and id to a different value. 

![cilium install](pictures/cinstall.jpg)

![cilium status](pictures/cstatus.jpg)


> If you are using kind in a remote computer and want to port fwd the Hubble port, you need to use 0.0.0.0 as a bind IP address.

```shell
kubectl port-forward -n kube-system svc/hubble-ui --address 0.0.0.0 --address :: 12000:80 &
```

#### Verify that outgoing requests work

Test the connection from the first pod in namespace cilium-test to https://cilium.io and get a positive return code.

```shell
BACKEND=$(kubectl get pods -n cilium-test -o jsonpath='{.items[0].metadata.name}')
kubectl -n cilium-test exec -ti ${BACKEND} -- curl -Ik --connect-timeout 5 https://cilium.io | head -1
HTTP/2 200
```

#### Apply a first policy for zero trust

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: cilium-test
  namespace: cilium-test
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*"
```

or you can go to https://editor.networkpolicy.io/ and do it manually

![create policy](pictures/editor-cilium-io-1.png)

* "Create new policy" (empty page bottom left)
* "Edit" icon in the middle of the page => enter a namespace and a policy name

![deny egress](pictures/editor-cilium-io-2.png)

* "Egress Default Deny"
* "Allow Kubernetes DNS"
* "Download"

Apply locally

```shell
kubectl apply -f https://raw.githubusercontent.com/chornberger-c2c/isovalent-cilium-lab/main/cilium-network-policies/egress-default-deny.yml
```

Observe the change

```shell
kubectl -n cilium-test exec -ti ${BACKEND} -- curl -Ik --connect-timeout 5 https://cilium.io | head -1
curl: (28) Connection timeout after 5001 ms
command terminated with exit code 28
hubble observe --output jsonpb --last 1000  > backend-cilium-io.json
```

The connection to https://cilium.io won't work, as we configured "Egress Default Deny" in our first policy.

#### Apply new rule that allows access to cilium.io

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: cilium-test
  namespace: cilium-test
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*"
    - toFQDNs:
        - matchName: cilium.io
      toPorts:
        - ports:
            - port: "443"
```

or go to https://editor.cilium.io and do it manually

![upload flows and add rule](pictures/editor-cilium-io-3.png)

* "Flows upload"
* "Upload flows"
* "Add rule"
* "Download"

![rule added](pictures/editor-cilium-io-4.png)

Apply locally

```shell
kubectl apply -f https://raw.githubusercontent.com/chornberger-c2c/isovalent-cilium-lab/main/cilium-network-policies/allow-cilium-io.yml
```

### Verify

```shell
kubectl -n cilium-test exec -ti ${BACKEND} -- curl -Ik --connect-timeout 5 https://kubernetes.io | head -1
curl: (28) Connection timeout after 5001 ms
command terminated with exit code 28
```

Timeout indicates that the connection to https://kubernetes.io doesn't work, as of "Egress Default Deny".

```shell
kubectl -n cilium-test exec -ti ${BACKEND} -- curl -Ik --connect-timeout 5 https://cilium.io | head -1
HTTP/2 200
```

Positive return code shows that the connection to https://cilium.io works, as of our applied policy.
