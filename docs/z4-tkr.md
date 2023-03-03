# TKR

The TKR has a close relationship with the management cluster version.  So, before we look at the TKR, lets
look at our management cluster.  Note, we're going to start by looking at the TKR_DATA field, which you'll understand
much better if you read about the [WEBHOOKS](https://tanzu-install-annotated.readthedocs.io/en/latest/z3-webhooks/) article on this same site...

## Management Cluster version

```
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  annotations:
```

If we look at the `spec.topology` section, then we will see a `name` field corresponding the the TKR_DATA value.
```
    - name: TKR_DATA
      value:
        v1.24.9+vmware.1:
          kubernetesSpec:
            coredns:
              imageTag: v1.8.6_vmware.15
            etcd:
              imageTag: v3.5.6_vmware.3
            imageRepository: projects.registry.vmware.com/tkg
            kube-vip:
              imageTag: v0.5.7_vmware.1
            pause:
              imageTag: "3.7"
            version: v1.24.9+vmware.1
          labels:
            image-type: ova
            os-arch: amd64
            os-name: ubuntu
            os-type: linux
            os-version: "2004"
            ova-version: v1.24.9---vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93
            run.tanzu.vmware.com/os-image: v1.24.9---vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93
            run.tanzu.vmware.com/tkr: v1.24.9---vmware.1-tkg.1
          osImageRef:
            moid: vm-48
            template: /dc0/vm/ubuntu-2004-kube-v1.24.9+vmware.1-tkg.1
            version: v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93
    version: v1.24.9+vmware.1 <-- this version defines TKR compatibility
```
The field at the bottom, `version`, is the "left hand side" of our compatibility equation. 
A particular management cluster is compatible with many TKRs.

Now, 

## Lets look at a TKR:

Your current version of TKG will have 3 default TKRs you can see... 

```yaml
kubo@Yk9ph4RbYmJO7:~$ kubectl get TanzuKubernetesRelease
NAME                        VERSION                   READY   COMPATIBLE   CREATED
v1.22.17---vmware.1-tkg.1   v1.22.17+vmware.1-tkg.1   True    True         8d
v1.23.15---vmware.1-tkg.1   v1.23.15+vmware.1-tkg.1   True    True         8d
v1.24.9---vmware.1-tkg.1    v1.24.9+vmware.1-tkg.1    True    True         8d
```

Now, picking 1.24.9... we can look at its contents `kubectl edit tkr v1.24.9---vmware.1-tkg.1`... 


### TKR metadata 

The metadata section has multiple labels...

```yaml
apiVersion: run.tanzu.vmware.com/v1alpha3
kind: TanzuKubernetesRelease
metadata:
  creationTimestamp: "2023-02-22T20:16:43Z"
  generation: 1
  labels:
    v1: ""
    v1.24: ""
    v1.24.9: ""
    v1.24.9---vmware: ""
    v1.24.9---vmware.1: ""
    v1.24.9---vmware.1-tkg: ""
    v1.24.9---vmware.1-tkg.1: ""
  name: v1.24.9---vmware.1-tkg.1
spec:
  ...
```
or in a BYO TKR, maybe youll have something like 

```yaml
kind: TanzuKubernetesRelease
apiVersion: run.tanzu.vmware.com/v1alpha3
metadata:
  name: v1.24.9---vmware.1-gpu-efi
  labels:
    tkr.tanzu.vmware.com/gpu-efi: ""
spec:
  ...
```

You'll notice these labels are redundant.  That gives you the ability to quickly find
TKRs related to a particular version, for example:
```
k get tkr -l 'v1.24'
```

Will give you all TKRs that have any relation to the Kubernetes 1.24 Release minor.

### TKR Packages

These are the package versions which we'll use for bootstrapping our TKR.  For example, we know from this that
antrea version 1.7.2 is the CNI provider that TKG certifies for running Kubernetes 1.24 on TKG, from looking at this list.
```
spec:
  bootstrapPackages:
  - name: antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced
  - name: vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1
  - name: vsphere-cpi.tanzu.vmware.com.1.24.3+vmware.1-tkg.1
  - name: kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1
  - name: metrics-server.tanzu.vmware.com.0.6.2+vmware.1-tkg.1
  - name: secretgen-controller.tanzu.vmware.com.0.11.2+vmware.1-tkg.1
  - name: tkg-storageclass.tanzu.vmware.com.0.28.0+vmware.1
  - name: pinniped.tanzu.vmware.com.0.12.1+vmware.2-tkg.3
  - name: capabilities.tanzu.vmware.com.0.28.0+vmware.1
  - name: calico.tanzu.vmware.com.3.24.1+vmware.1-tkg.1
  - name: kube-vip-cloud-provider.tanzu.vmware.com.0.0.4+vmware.2-tkg.1
  - name: load-balancer-and-ingress-service.tanzu.vmware.com.1.8.2+vmware.1-tkg.1
```

### TKR Kubernetes Core

Every version of Kubernetes also comes along with a "core" release , i.e. a well defined coredns, etcd, and pause image.  We define
these separately because they are not packages, but rather, parts of our Kubernetes OVA (virtual machine image) that is 
booted up.  They are not carvel packages which are installed at a later date, but rather, essential bits without which, 
bootstrapping is not possible.

```
  kubernetes:
    coredns:
      imageTag: v1.8.6_vmware.15
    etcd:
      imageTag: v3.5.6_vmware.3
    imageRepository: projects.registry.vmware.com/tkg
    kube-vip:
      imageTag: v0.5.7_vmware.1
    pause:
      imageTag: "3.7"
    version: v1.24.9+vmware.1
  osImages:
  - name: v1.24.9---vmware.1-tkg.1-f5e94dab9dfbb9988aeb94f1ffdc6e5e
  - name: v1.24.9---vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93
  - name: v1.24.9---vmware.1-tkg.1-226b7a84930e5368c38aa867f998ce33
  version: v1.24.9+vmware.1-tkg.1
```

### TKR Status 

Finally, it has a status of **compatible**.  That is calculated by our tanzu framework controllers, which run in the management
cluster.  They make this calculation my looking at the version of the management cluster:

```
status:
  conditions:
  - lastTransitionTime: "2023-02-23T01:24:01Z"
    status: "True"
    type: Ready
  - lastTransitionTime: "2023-02-23T01:24:01Z"
    status: "True"
    type: Compatible
```

The compatibility of our TKR is defined based on the *version* of the management cluster
we showed at the beggining of this article. 


# BYO TKRs

You can build your own TKR.  You might want to do this if your making custom Virtual Machine images for TKG (for example
using a different OS then the standard Ubuntu or Photon ones we ship), or if you wanted a special set of packages to be installed (i.e. a
different antrea version).  

To do this, you 

- get a existing TKR, as a carvel package
- change some of the peices of it
- create it as a Kubernetes API object

In a case where you wanted to create a custom image, for example, you would 
edit the *OSImage*, *ClusterBootstrapTemplate*, and *TanzuKubernetesRelease* objects , like so...

_NOTE: 

This is a contrived example.
The below example has an image built for **gpu workloads**, however, please don't build a custom
image for GPUs.  
GPUs will be fully supported without the need for custom images in 2.1.1 !!!
_

## OSImage

The OSImage object defines a specific Operating System that will be referenced in your TKR.  When you
tell TKG to make a new cluster, it will query all `osImages` that are compatible for your TKR, and then
pick one which matches your desired:
- name: (i.e. ubuntu)
- version:  (i.e. 2004)
- architecture: (i.e. amd64)

Now, lets look at the 3 objects.

### QUERY ???

Yup, we literally query all of the CRDs to match up your OSImage in a declarative way.  There is also a more imperative
way to select your OS, which is, to use the VSPHERE_TEMPLATE field, but that is an older, and less well documented approach for
selecting OS Images.

### OSImage

An OSImage object thus defines these fields.  

```
kind: OSImage
apiVersion: run.tanzu.vmware.com/v1alpha3
metadata:
  name: v1.24.9---vmware.1-gpu-efi ### We will reference this name in our TKR.
  labels:
    tkr.tanzu.vmware.com/gpu-efi: "" ### This label is important, it allows us to ENSURE that we DONT use this TKR on accident!
spec:
  kubernetesVersion: v1.24.9+vmware.1
  os:
    type: linux
    name: ubuntu
    version: "2004"
    arch: amd64
  image:
    type: ova
    ref:
      version: v1.24.9+vmware.1-gpu-efi <-- this is important: It must match the OVA <VERSION> field !!!
```


*AN OVA FILE IS A OS IMAGE + some XML METADATA THAT VSPHERE KNOWS HOW TO READ !!!*

- DISTRO_NAME: ubuntu
- VERSION: 1.24.9+vmware.1-gpu-efi
- DISTRO_ARCH: amd64
- DISTRO_VERSION: 20.04

I mean, you'll literally see this in the XML files, i.e. 

```
 <Property ovf:key="VERSION" 
           ovf:type="string" 
           ovf:userConfigurable="false" 
           ovf:value="1.24.9+vmware.1-gpu-efi" <--  this is what TKG matches your version to !!!
 />
```

So ultimately, the value of the TKR name, which we called `1.24.9+vmware.1-gpu-efi` , will have to be
placed inside of our OSImage object, and it must be equal:

```
spec
  image
     ref
       version
```
It is defined when we run `image-builder` inside of a `metadata.json` file...

### ClusterBootstrapTemplate

Cluster Bootstrap Templates exist for every TKR.  When you make a new cluster, a clusterbootstrap template is
converted to a clusterbootstrap object...

For example, if i have 3 clusters, then each one will have a unique **clusterbootstrap** object for 1.24... 

```
kubo@Yk9ph4RbYmJO7:~$ kubectl get clusterbootstrap -A
NAMESPACE    NAME          CNI                                                     CSI                                                 CPI                                                  KAPP                                                     RESOLVED_TKR
default      tkg-mgmt-vc   antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced   vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1   vsphere-cpi.tanzu.vmware.com.1.24.3+vmware.1-tkg.1   kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1   v1.24.9---vmware.1-tkg.1
omg          wl            antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced   vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1   vsphere-cpi.tanzu.vmware.com.1.24.3+vmware.1-tkg.1   kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1   v1.24.9---vmware.1-tkg.1
tkg-system   tkg-mgmt-vc   antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced   vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1   vsphere-cpi.tanzu.vmware.com.1.24.3+vmware.1-tkg.1   kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1   v1.24.9---vmware.1-tkg.1
```

This bootstrap object was made, in all 3 cases from a pre-existing **clusterbootstraptemplate** object... 

```
kubo@Yk9ph4RbYmJO7:~$ kubectl get clusterbootstraptemplate -A
NAMESPACE    NAME                        CNI                                                     CSI                                                 CPI                                                  KAPP
tkg-system   v1.24.9---vmware.1-tkg.1    antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced   vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1   vsphere-cpi.tanzu.vmware.com.1.24.3+vmware.1-tkg.1   kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1
tkg-system   v1.22.17---vmware.1-tkg.1   antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced   vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1   vsphere-cpi.tanzu.vmware.com.1.22.7+vmware.1-tkg.1   kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1
tkg-system   v1.23.15---vmware.1-tkg.1   antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced   vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1   vsphere-cpi.tanzu.vmware.com.1.23.3+vmware.1-tkg.1   kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1
```

Note that our clusterbootstraptemplate simplifies what we must define for a cluster - it gives us a default CNI, CPI, CSI, and so on... Thus making
it easier for us to define a minimal amount of inputs when we run `tanzu cluster create ...`

Now, lets look at a ClusterbootstrapTemplate in a custom TKR.

The cni, cpi, csi, and kapp objects are all generated in the namespace of your cluster as soon as you run `tanzu cluster create...`,
from reasonable defaults... UNLESS you generate them yourself.

You can, however `kubectl create` an `AntreaConfig` object which:
- Has the same name as your cluster you are making and
- Lives in the same namespace as the cluster you are making

In which case, TKG will IGNORE the ClusterBootstrapTemplate's CNI configuration, and instead just use the one you precreated.

```
kind: ClusterBootstrapTemplate
apiVersion: run.tanzu.vmware.com/v1alpha3
metadata:
  name: v1.24.9---vmware.1-gpu-efi
  labels:
    tkr.tanzu.vmware.com/gpu-efi: ""
...
spec:
  additionalPackages:
  - refName: metrics-server.tanzu.vmware.com.0.6.2+vmware.1-tkg.1
  - refName: secretgen-controller.tanzu.vmware.com.0.11.2+vmware.1-tkg.1
  - refName: tkg-storageclass.tanzu.vmware.com.0.28.0+vmware.1
    valuesFrom:
      inline:
        infraProvider: vsphere
  - refName: pinniped.tanzu.vmware.com.0.12.1+vmware.2-tkg.3
    valuesFrom:
      secretRef: default-pinniped-config-v1.24.9---vmware.1-tkg.1
  - refName: capabilities.tanzu.vmware.com.0.28.0+vmware.1
    valuesFrom:
      secretRef: default-capabilities-package-config-v1.24.9---vmware.1-tkg.1
  cni:
    refName: antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced
    valuesFrom:
      providerRef:
        apiGroup: cni.tanzu.vmware.com
        kind: AntreaConfig              <--- customizing Antrea is done by making an AntreaConfig object
        name: v1.24.9---vmware.1-tkg.1
  cpi:
    refName: vsphere-cpi.tanzu.vmware.com.1.24.3+vmware.1-tkg.1
    valuesFrom:
      providerRef:
        apiGroup: cpi.tanzu.vmware.com
        kind: VSphereCPIConfig          <----- customizing VsphereCPIConfig is done by making one of these
        name: v1.24.9---vmware.1-tkg.1
  csi:
    refName: vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1
    valuesFrom:
      providerRef:
        apiGroup: csi.tanzu.vmware.com
        kind: VSphereCSIConfig           <------ and so on ....
        name: v1.24.9---vmware.1-tkg.1
  kapp:
    refName: kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1
    valuesFrom:
      providerRef:
        apiGroup: run.tanzu.vmware.com
        kind: KappControllerConfig         <------ and so on ... 
        name: v1.24.9---vmware.1-tkg.1
  paused: false

```

### TanzuKubernetesRelease

Finally we have our TanzuKubernetesRelease object.  The important thing to note here
is that it references the OSImage that we made above... `name: v1.24.9---vmware.1-gpu-efi`.

```
kind: TanzuKubernetesRelease
apiVersion: run.tanzu.vmware.com/v1alpha3
metadata:
  name: v1.24.9---vmware.1-gpu-efi
  labels:
    tkr.tanzu.vmware.com/gpu-efi: ""
spec:
  version: v1.24.9+vmware.1-gpu-efi
  kubernetes:
    version: v1.24.9+vmware.1
    imageRepository: localhost:5000/vmware.io
    etcd:
      imageTag: v3.5.6_vmware.3
    pause:
      imageTag: "3.7"
    coredns:
      imageTag: v1.8.6_vmware.15
  osImages:
  - name: v1.24.9---vmware.1-gpu-efi
```

# Putting it all together

So, when you make a cluster with a custom TKR, or any TKR for that matter:

- A ClusterBootstrapTemplate is cloned into your clusters namespace, w/ the name of your cluster
- AntreaConfig, CSIConfig, and other bootstrap configurations are generated from that bootstrap's defaults
- The custom OS Image which we chose for our GPU node will now come to life once we
  - add an annotation of `resolve-tkr`  to the `Cluster` object
  - add a `resolve-os-image` to the `Cluster.Spec.Topology.ControlPlane.metadata.Annotations`
  - add a `resolve-os-image` to the `Cluster.Spec.Topology.Workers.MachineDeployments.metadata.Annotations`

In other words, we reference our image by:
- forcing our new cluster to use a TKR that references the OSImage object we created
- ensuring that the image-type and os-name fields in our machineDeployment and controlPlane VMs are that of the OSImage

```
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  annotations:
    run.tanzu.vmware.com/resolve-tkr: 'tkr.tanzu.vmware.com/gpu-efi' ### Here
spec:
  clusterNetwork:
  ...
  topology:
    class: tkg-vsphere-default-v1.0.0 
    controlPlane:
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu ### here
      replicas: 1
    variables:
    - name: controlPlaneCertificateRotation
    ....
    - name: worker
    version: v1.24.9+vmware.1 
    workers:
      machineDeployments:
      - class: tkg-worker
        metadata:
          annotations:
            run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu ### And here 
        name: md-0
```
 
