# h2.3

Some notes on "differences" in TKG 2.3 from the original notes on TKG 2.1...

## Plugins

In the old TKG , you got...
```
 ℹ  Installing plugin 'pinniped-auth:v0.28.0-dev'
  ℹ  Installing plugin 'secret:v0.28.0-dev' with target 'kubernetes'
```
Now, when initializing plugins you have targets: *Global* and *Kubernetes*...

```
09:07:02  [i] Installing plugin 'isolated-cluster:v0.30.0-dev' with target 'global'
09:07:09  [i] Installing plugin 'package:v0.30.0-dev' with target 'kubernetes'
09:07:17  [ok] successfully installed all plugins from group 'vmware-tkg/tkg:v2.3.0'
```

Additionally, in older TKGs there was a login plugin
```
    login               Login to the platform                                                          default    v0.28.0-dev  installe
```
Which has now gone away. 

## Bootstrapping

- CEIP defaults to false (it used to be true?)
- Warning: unable to find component 'kube_rbac_proxy' under BoM <-- this error needs to go away , some day

## Cluster API installation on the Kind Cluster

- Theres a new "ipam-in-cluster" component that is bootstrapped.  
- The original "ipam-components.yaml" for the "in-cluster" provider is also installed.  

Note the old versions were
```
 Fetching File="control-plane-components.yaml" Provider="kubeadm" Type="ControlPlaneProvider" Version="v1.2.8  
 ...
 Fetching File="infrastructure-components.yaml" Provider="vsphere" Type="InfrastructureProvider" Version="v1.5.1"
```
And the new ones are ...
```
 Fetching File="ipam-components.yaml" Provider="in-cluster" Type="IPAMProvider" Version="v0.1.0"
 Fetching File="metadata.yaml" Provider="cluster-api" Type="CoreProvider" Version="v1.4.2"
 Fetching File="metadata.yaml" Provider="ipam-in-cluster" Type="InfrastructureProvider" Version="v0.1.0"
 Fetching File="control-plane-components.yaml" Provider="kubeadm" Type="ControlPlaneProvider" Version="v1.4.2"

```

Cluster API is now 1.4.2 instead of 1.2.8.

## MGMT cluster

Now the TKR that we check for is ` Checking Tkr v1.26.5---vmware.1-tkg.1-rc.1 package is installed successfully...` instead
of the older one `1.24.9`

Theres a new tkg-clusterclass package.  the `tkg-clusterclass-vsphere` is gone....?
```
    [2023-01-10T20:09:27.780Z] successfully reconciled package: tkg-clusterclass-vsphere
```

Will add more over time, but , here are some new features worth noting

## New Field: Custom VSphere.Template

```
   workers:
     machineDeployments:
     - class: tkg-worker
       metadata:
         annotations:
           run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu
       name: md-1
       replicas: 2
       variables:
         overrides:
         - name: vcenter
           Value:
		cloneMode: fullClone
             datacenter: /dc0
             datastore: /dc0/datastore/sharedVmfs-0
             folder: /dc0/vm/folder0
             network: /dc0/network/VM Network
             resourcePool: /dc0/host/cluster0/Resources/rp0
             server: <server-ip>
             storagePolicyID: ""
             tlsThumbprint: <xxx>
             datacenter: /dco
             template: /dc0/vm/ubuntu-2004-kube-v1.24.10+vmware.1-tkg.1 <-- we now support this new template, no mor guessing the vsphere template resolvers logic 
```

