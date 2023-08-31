# CNI changes

TKG ships w/ antrea. you can use other CNIs instead..  First set it up w `value: none`.
```
70 ---
 71 apiVersion: cluster.x-k8s.io/v1beta1
 72 kind: Cluster
 73 metadata:
 74   annotations:
 75     osInfo: ubuntu,20.04,amd64
 76     tkg.tanzu.vmware.com/cluster-controlplane-endpoint: 10.221.159.242
 77     tkg/plan: dev
 78   labels:
 79     tkg.tanzu.vmware.com/cluster-name: windows-cluster
 80   name: windows-cluster
 81   namespace: default
 82 spec:
 83   clusterNetwork:
 84     pods:
 85       cidrBlocks:
 86       - 100.96.0.0/11
 87     services:
 88       cidrBlocks:
 89       - 100.64.0.0/13
 90   topology:
 91     class: tkg-vsphere-default-v1.1.0
 92     controlPlane:
 93       metadata:
 94         annotations:
 95           run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu
 96       replicas: 1
 97     variables:
 98     - name: cni
 99       value: none # <----------------------------------  antrea by default
100     - name: controlPlaneCertificateRotation
101       value:
102         activate: true
103         daysBefore: 90
104     - name: imageRepository
```

## Example Tigera operator

Calico nowadays has a fancy operator to install things for you.  You can get it at 

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```
Which will work since tigera-operator is hosted outside docker, so theres no pull limit.

You then can customize it via https://docs.tigera.io/calico/latest/reference/installation/api 

## Pulling node and other images

Youll want to 
- modify the **registry** you pull calico from b/c docker.io is unreliable.  
- modify the podCidr default for calico to match your podCIDR.

First make sure you see the operator running happily.  Then you'll use CRDs to install calico.
The operator will do the heavy lifting for you , you just need to make an Installation CRD.

## Now install calico 
```
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  registry: harbor-repo.vmware.com/dockerhub-proxy-cache ###### <--- use this or another similar docker hub proxy 
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 100.96.1.0/16 #192.168.0.0/16 #### <<<- make this the same as TKG pod_cidr
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}

```

## What about windows nodes

You can setup windows on calico as a **host process container** which is super easy:

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico-windows-vxlan.yaml -o calico-windows.yaml
```
Then from there, you can just create it via `kubectl create ...`
Note though that NOW you need to install a kube-proxy implementation .... there are instructions to do this here

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/windows-kube-proxy.yaml -o windows-kube-proxy.yaml
kubectl apply -f windows-kube-proxy.yaml
kubectl describe ds -n kube-system kube-proxy-windows
```











