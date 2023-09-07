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

For windows nodes, you can run a host-process container you need to make sure that 
- IPIP is disabled, VXlan:Always
- bird checks are disabled (delete the liveness and readiness related calico probes).  You can use 
- HNS is created (this can be done by manually running this script https://gist.githubusercontent.com/jayunit100/c7b2e69110bc16af69048be1f065e555/raw/f61133f70dcfbbf02defbe8635ca4c4eaa72e6e6/gistfile1.txt which works on TKG nodes to create an HNS Network -- it will fail but thats ok) .... 

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
But this kube-proxy will WAIT until an HNS Network exists.  So how do you create the HNS NEtwork? 

## One hack

- install kube proxy as shown
- install calico as a host process container
- SSH into each node and run the calico install script to setup some of the hns stuff that the kube proxy needs .

Im not sure why, but it appears in some cases calico running in a host process container doesnt make the HNS NEtwork.  So you can 
ssh into nodes and manually run https://github.com/jayunit100/k8sprototypes/blob/master/windows/calico/calico-hack-fixer.ps1 one at a time
to unblock the kube proxy.  Once this happens:
- the kube proxy will have an HNS netwwork and start
- it will make the internal service IP that calico node agent needs to access the APISERver
- the calico agent will then come up

  And you can then run pods.










