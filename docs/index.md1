# MC_INIT

A Fully annotated walk through of TKG bootstrap and management cluster bring up for the curious.

This markdown takes all of your favorite TKG logging messages and explains them in plain english, and associates them
with code snippets inside of tanzu-framework and cluster-api.

[Contributors Welcome](https://github.com/jayunit100/tanzu-install-annotated) !!!

## TKG Client and Kind Bootstrapping

Curling down the TKG bom.  This only happens on prem.  But its a starting point that is worth noting.
```
 curl https://build-artifactory.eng.vmware.com/kscom-generic-local/TKG/channels/442519250544895703/_boltArtifacts/tkg-v2.1.0-rc.2.buildinfo.yaml
```

In the real world, you'll get BOM components (like the OVAs you upload, or the TKG client) from https://customerconnect.vmware.com/downloads/details?downloadGroup=TKG-160&productId=988&rPId=93384, or a similar url.

We now will check tanzu's and update the plugins: 

```
tanzu config get | yq eval '.clientOptions.features.global.context-aware-cli-for-plugins' - true
  INFO:root:context-aware-cli-for-plugins is ON
  tanzu plugin clean
  ✔  successfully cleaned up all plugins 
  tanzu plugin repo update -b tanzu-cli-framework core
  ✔  successfully updated repository configuration for core
```

##  Making sure the client plugins are there... 
```
  ℹ  Installing plugin 'pinniped-auth:v0.28.0-dev'
  ℹ  Installing plugin 'secret:v0.28.0-dev' with target 'kubernetes'
  ℹ  Installing plugin 'telemetry:v0.28.0-dev' with target 'kubernetes'
  ℹ  Installing plugin 'isolated-cluster:v0.28.0-dev'
  ℹ  Installing plugin 'login:v0.28.0-dev'
  ℹ  Installing plugin 'management-cluster:v0.28.0-dev' with target 'kubernetes'
  ℹ  Installing plugin 'package:v0.28.0-dev' with target 'kubernetes'
  ℹ  Successfully installed all required plugins 
  ✔  Done 
  INFO:root:====== 103  CMD: tanzu plugin list
  Standalone Plugins
    NAME                DESCRIPTION                                                        TARGET      DISCOVERY  VERSION      STATUS  
    isolated-cluster    isolated-cluster operations                                                    default    v0.28.0-dev  installe
    login               Login to the platform                                                          default    v0.28.0-dev  installe
    pinniped-auth       Pinniped authentication operations (usually not directly invoked)              default    v0.28.0-dev  installe
    management-cluster  Kubernetes management-cluster operations                           kubernetes  default    v0.28.0-dev  installe
    package             Tanzu package management                                           kubernetes  default    v0.28.0-dev  installe
    secret              Tanzu secret management                                            kubernetes  default    v0.28.0-dev  installe
    telemetry           Configure cluster-wide telemetry settings                          kubernetes  default    v0.28.0-dev  installe
  INFO:root:====== 104  CMD: tanzu version
  version: v0.28.0-dev
  buildDate: 2023-01-09
  sha: d293a0881-dirty
```

# Uploading the OVAs to Vsphere


On VSphere, without the supervisor, users typically will need to download and then import OVAs into their data center.     This is often done
with a tool like `govc`.   For example:

```
govc import.ova ./ubuntu-2004-kube-v1.23.8+vmware.2-tkg.1-85a434f93857371fccb566a414462981.ova 
```

These logs look something like... 

```
INFO:root:Downloading OVA from http://build-squid.eng.vmware.com/build/mts/release/bora-21093855/publish/lin64/tkg_release/node/ova-ubuntu-2004-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93/ubuntu-2004-kube-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93.ova to 
temp_ova_dir-NVZOBCZG

INFO:root:Importing OVA from http://build-squid.eng.vmware.com/build/mts/release/bora-21093855/publish/lin64/tkg_release/node/ova-ubuntu-2004-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93/ubuntu-2004-kube-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93.ova to /dc0/vm/.

This might take a while...
```


# Bootstrapping TKG with Kind

The `InitRegion` function lives inside Tanzu CLI.  IT has ALL the logic for kind and mgmt cluster bootstrapping.  
`tanzu management-cluster create...` will start by making kind cluster... then mgmt cluster in vsphere...  i.e. 

```
- Make Kind cluster
  - Install Certmanager
  - Install CAPI
  - Install CAPV
  - Install TF and Addons
- CAPV pods in Kind now makes a CAPV Cluster in Vsphere
  - Install CAPI
  - Install CAPV
  - Install TF
  - TF and Addons
- Delete original Kind Cluster
```

Ok, lets look in detail at this: 

### To bootstrap TKG , we run 

```
tanzu management-cluster create  --yes -v 9 --deploy-tkg-on-vSphere7 -f tkg-mgmt-vc.yaml
```

### Result: 

```
 compatibility file (/home/kubo/.config/tanzu/tkg/compatibility/tkg-compatibility.yaml) already exists, skipping download
 BOM files inside /home/kubo/.config/tanzu/tkg/bom already exists, skipping download
 CEIP Opt-in status: true
 cluster log directory does not exist. Creating new one at "/home/kubo/.config/tanzu/tkg/logs"
 
 Validating the pre-requisites...
 
 vSphere 7.0 Environment Detected.
 
 You have connected to a vSphere 7.0 environment which does not have vSphere with Tanzu enabled. vSphere with Tanzu includes
 an integrated Tanzu Kubernetes Grid Service which turns a vSphere cluster into a platform for running Kubernetes workloads in dedicated
 resource pools. Configuring Tanzu Kubernetes Grid Service is done through vSphere HTML5 client.
 
 Tanzu Kubernetes Grid Service is the preferred way to consume Tanzu Kubernetes Grid in vSphere 7.0 environments. Alternatively you may
 deploy a non-integrated Tanzu Kubernetes Grid instance on vSphere 7.0.
 Deploying TKG management cluster on vSphere 7.0 ...
 Identity Provider not configured. Some authentication features won't work.
 Using default value for CONTROL_PLANE_MACHINE_COUNT = 1. Reason: CONTROL_PLANE_MACHINE_COUNT variable is not set
 Using default value for WORKER_MACHINE_COUNT = 1. Reason: WORKER_MACHINE_COUNT variable is not set
 Setting config variable "VSPHERE_DATACENTER" to value "/dc0"
 Setting config variable "VSPHERE_NETWORK" to value "/dc0/network/VM Network"
 Setting config variable "VSPHERE_RESOURCE_POOL" to value "/dc0/host/cluster0/Resources/rp0"
 Setting config variable "VSPHERE_DATASTORE" to value "/dc0/datastore/sharedVmfs-0"
 Setting config variable "VSPHERE_FOLDER" to value "/dc0/vm/folder0"
 Checking if VSPHERE_CONTROL_PLANE_ENDPOINT  is already in use
```


### Now the (big) InitRegion() method kicks off

```
func (c *TkgClient) InitRegion(options *InitRegionOptions) error { //nolint:funlen,gocyclo
```

This function lives in tkg/client/init.go.    Once it takes over, and it does quite a bit... 

- Setting up kind
- Setting up a management cluster on kind
- Calling the `InstallOrUpgradeManagementComponents` method, which will wait for all the management components and packages to come up.

```
 Setting up management cluster...
 Validating configuration...
 Setting CLUSTER_TOPOLOGY to "true"
 Using infrastructure provider vsphere:v1.5.1
 Generating cluster configuration...
 Setting up bootstrapper...
 Fetching configuration for kind node image...
 kindConfig: 
  &{{Cluster kind.x-k8s.io/v1alpha4}  [{  map[] [{/var/run/docker.sock /var/run/docker.sock false false }] [] [] []}] { 0  100.96.0.0/11 100.64.0.0/13 false } map[] map[] [apiVersion: kubeadm.k8s.io/v1beta3
```

#### InitRegion: This is the kind configuration that we'll use when bootstrapping TKG. .... 

Here's how TKG configures the Kind cluster... 

```
    kind: ClusterConfiguration
    imageRepository: projects.registry.vmware.com/tkg
    etcd:
    local:
        imageRepository: projects.registry.vmware.com/tkg
        imageTag: v3.5.6_vmware.3
    dns:
    type: CoreDNS
    imageRepository: projects.registry.vmware.com/tkg
    imageTag: v1.8.6_vmware.15] [] [[plugins]
    [plugins.'io.containerd.grpc.v1.cri']
    [plugins.'io.containerd.grpc.v1.cri'.registry]
    [plugins.'io.containerd.grpc.v1.cri'.registry.configs]
    [plugins.'io.containerd.grpc.v1.cri'.registry.configs.'projects-stg.registry.vmware.com']
    [plugins.'io.containerd.grpc.v1.cri'.registry.configs.'projects-stg.registry.vmware.com'.tls] 
    insecure_skip_verify = false
    ca_file = ''
    ] []}

```

Now, the `InitRegion` function  will begin running kind (it vendors `kind`) and, in a few moments, `clusterctl` operations against that kind cluster... 

```
 Creating kind cluster: tkg-kind-ceus13jb4t5tab9k5jk0
 Creating cluster "tkg-kind-ceus13jb4t5tab9k5jk0" ...
 Ensuring node image (projects-stg.registry.vmware.com/tkg/kind/node:v1.24.9_vmware.1-tkg.1_v0.17.0) ...
 Pulling image: projects-stg.registry.vmware.com/tkg/kind/node:v1.24.9_vmware.1-tkg.1_v0.17.0 ...
 Preparing nodes ...
 Writing configuration ...
 U 
[2023-01-10T19:46:42.469Z] Starting control-plane ...

 GET https://tkg-kind-ceus13jb4t5tab9k5jk0-control-plane:6443/healthz?timeout=10s in 0 millisecondsI0110 19:46:46.954014 280 round_trippers.go:553] GET https://tkg-kind-ceus13jb4t5tab9k5jk0-control-plane:6443/healthz?timeout=10s in 0 millisecondsI0110 19:46:47.454402 280 round_trippers.go:553] GET https://
 ...
 tkg-kind-ceus13jb4t5tab9k5jk0-control-plane:6443/healthz?timeout=10s in 0 millisecondsI0110 19:46:54.201890 280 round_trippers.go:553] GET https://
 ...
 tkg-kind-ceus13jb4t5tab9k5jk0-control-plane:6443/healthz?timeout=10s 500 Internal Server Error in 5 millisecondsI0110 19:46:55.956712 280 round_trippers.go:553] GET https://
 ...
 tkg-kind-ceus13jb4t5tab9k5jk0-control-plane:6443/healthz?timeout=10s 200 OK in 2 milliseconds[apiclient] All control plane components are healthy after 10.006544 secondsI0110 
```

InitRegion: Next, the CNI Installation (kind-net, and so on happens).  Note were still just making the bootstrap cluster.  Nothing will go wrong here, ever... 

```
 Installing CNI ...
 Installing StorageClass ...
 Waiting 2m0s for control-plane = Ready ...
 Ready after 28s 
 Bootstrapper created. Kubeconfig: /home/kubo/.kube-tkg/tmp/config_Z1ics3TW
```

## Customizing the Kind Cluster with TKG's payload

There are two major differences between a `kind` cluster and a TKG bootstrap cluster. 

- CAPI: The ability to install and create cluster API objects, and infrastructure.
- Tanzu: The installation of packages for things such as kapp-controller, which orchestrate installation of packages onto clusters.

So, we'll now "mature" or `kind` cluster into a true `TKG bootstrap` cluster.

Now that the kind cluster is up, tanzu cli will start customizing it. 

### Take special note of kapp

The logs below show kapp-controller being installed.  This must be one of the first steps in TKG setup, you'll see why later!
```
 Warning: unable to find component 'kube_rbac_proxy' under BoM

 #### Note this !!!! One of the first things we do is install kapp-controller.  We'll rely on it heavily later.

 Installing kapp-controller on bootstrap cluster...
 
 User ConfigValues File: /tmp/2991662634.yaml
 Kapp-controller values-file: /tmp/330923605.yaml
 Kapp-controller configuration file: /tmp/3117802898
 
 waiting for resource kapp-controller of type *v1.Deployment to be up and running
 pods are not yet running for deployment 'kapp-controller' in namespace 'tkg-system', retrying
 ...
 pods are not yet running for deployment 'kapp-controller' in namespace 'tkg-system', retrying
 Installing providers on bootstrapper...
 Installing the clusterctl inventory CRD
```

Now youll get ALOT of logs for cluster CTL inventory setup.  Close your eyes, we'll wake you up when its over.

## InitRegion: Cluster API installation onto the Kind cluster

At this point, on our kind cluster, we'll start adding basic primitives necessary to make our bootstrap cluster.
That means, teaching it CAPI.
- Teach the kind cluster about CAPI  <-- you are here
- Tell the kind cluster to make a workload cluster on vsphere (later)
- Tell the kind cluster to nominate the workload cluster to a management cluster (later)
- Tell the kind cluster to go away... forever (later)

### Tell kind about CAPI 

At this point, 
- tanzu cli is now `clusterctl`, internally, 
- clusterctl is doing a bunch of magic to turn our kind cluster into a cluster API Management cluster... 
  - adding containers, provider CRDs, and so on...

```
 Creating CustomResourceDefinition="providers.clusterctl.cluster.x-k8s.io"
```

Now we "Fetch" providers.   This happens in clusterctl https://github.com/kubernetes-sigs/cluster-api/blob/main/cmd/clusterctl/client/init.go
- ipam provider
- bootstrap provider
- control plane provider
- infrastructure provider

```
    Fetching providers
    Fetching File="core-components.yaml" Provider="cluster-api" Type="CoreProvider" Version="v1.2.8"
    Fetching File="bootstrap-components.yaml" Provider="kubeadm" Type="BootstrapProvider" Version="v1.2.8"
    Fetching File="control-plane-components.yaml" Provider="kubeadm" Type="ControlPlaneProvider" Version="v1.2.8"
    Fetching File="infrastructure-components.yaml" Provider="vsphere" Type="InfrastructureProvider" Version="v1.5.1"
    Fetching File="ipam-components.yaml" Provider="ipam-in-cluster" Type="InfrastructureProvider" Version="v0.1.0"
    Fetching File="metadata.yaml" Provider="cluster-api" Type="CoreProvider" Version="v1.2.8"
    Fetching File="metadata.yaml" Provider="kubeadm" Type="BootstrapProvider" Version="v1.2.8"
    Fetching File="metadata.yaml" Provider="kubeadm" Type="ControlPlaneProvider" Version="v1.2.8"
    Fetching File="metadata.yaml" Provider="vsphere" Type="InfrastructureProvider" Version="v1.5.1"
    Fetching File="metadata.yaml" Provider="ipam-in-cluster" Type="InfrastructureProvider" Version="v0.1.0"
```

At this poing we see the cert-manager components being created, https://github.com/kubernetes-sigs/cluster-api/blob/main/cmd/clusterctl/client/cluster/cert_manager.go#L495. 
Again, remember, this function is being called by tanzu cli, but under the hood tanzu cli is calling `clusterctl`, which knows natively how to set up certmanager.

## Adding CAPI to Kind: Installing Cert Manager 

When you install CAPI, one of the first things that happens, is `clusterctl` will make sure you set up cert manager.

Cert manager is how CAPI webhooks authenticate to each other.  It's an implementation detail of how CAPI communicates within itself. 

```
      Creating Namespace="cert-manager-test"
      I0110 19:47:49.143398    5793 request.go:601] Waited for 1.047538531s due to client-side throttling, not priority and fairness, request: GET:https://127...
      Installing cert-manager Version="v1.9.1"
      Fetching File="cert-manager.yaml" Provider="cert-manager" Type="" Version="v1.9.1"
      Creating Namespace="cert-manager"
      Creating CustomResourceDefinition="certificaterequests.cert-manager.io"
      Creating CustomResourceDefinition="certificates.cert-manager.io"
      ...
      Creating Deployment="cert-manager-webhook" Namespace="cert-manager"
      Creating MutatingWebhookConfiguration="cert-manager-webhook"
      Creating ValidatingWebhookConfiguration="cert-manager-webhook"
      Waiting for cert-manager to be available...
      Updating Namespace="cert-manager-test"
      Creating Issuer="test-selfsigned" Namespace="cert-manager-test" <-- note this happens again on the mgmt cluster, sometimes takes a while though?
      ...
      Deleting Certificate="selfsigned-cert" Namespace="cert-manager-test"
```

# Adding CAPI to kind: Other CAPI objects

Once cert-manager is up, you'll see CAPI providers and CRDs being added into the Kind cluster.

Now, still seting up CAPI on kind , we are now installing CAPI objects: These "Creating" logs come from 
https://github.com/kubernetes-sigs/cluster-api/blob/main/cmd/clusterctl/client/cluster/components.go the internal `createObj` tools in cluster api, which know how to install arbitrary kubernetes objects.... 

```
      Installing Provider="cluster-api" Version="v1.2.8" TargetNamespace="capi-system"
      Creating objects Provider="cluster-api" Version="v1.2.8" TargetNamespace="capi-system"
      Creating Namespace="capi-system"
      Creating CustomResourceDefinition="clusterclasses.cluster.x-k8s.io"
        
        ... (you'll see hundreds of other "Creating" logs here... )
      
      pods are not yet running for deployment 'caip-in-cluster-controller-manager' in namespace 'caip-in-cluster-system', retrying
```

InitRegion.... still going ! We're still in the bootstrap cluster: Next, we'll wait for "packages" and "providers" to come online.  
Still in the init.go function of tkg/client... 

```
      func (c *TkgClient) WaitForProviders(clusterClient clusterclient.Client, options waitForProvidersOptions) error {
```
# Wait for CAPI PRoviders on the kind cluster
```

      Passed waiting on provider bootstrap-kubeadm after 15.51390451s
      Passed waiting on provider control-plane-kubeadm after 15.566739105s
      Passed waiting on provider infrastructure-vsphere after 15.602801637s
      Passed waiting on provider cluster-api after 15.694037955s
      Passed waiting on provider infrastructure-ipam-in-cluster after 20.223833161s
      Success waiting on all providers.
      [ ℹ  Updated package repository 'tanzu-management' in namespace 'tkg-system'
```

Still in the bootstrap cluster: Next, we'll wait for packages. 

# TKG'ifying our Kind cluster with the "tkg-pkg" metapackage

The tkg-pkg metapackage is a new TKG feature.  It puts all of the management cluster's packages into one, easy to grok location.

- Earlier, we installed `kapp` on the kind cluster.  You'll notice it was one of the first things we did.
- This single package will then be introspected by tanzu CLI to make a bunch of `PackageInstall`s.  
- Those `PackageInstall` objects will be reconciled by `kapp` controller
- `kapp` will then ensure that all the packages for TKG are running on your kind cluster, so that you can use it to eventually bootstrap a new permanant management cluster.

Below we'll "wait for package", i.e. we'll wait for individual packages to come online:
```
  [2023-01-10T19:50:48.603Z]  Added installed package 'tkg-pkg'waiting for package: tkg-pkg
  [2023-01-10T19:50:48.604Z] waiting for package: tanzu-addons-manager

  [2023-01-10T19:50:48.604Z] waiting for package: tanzu-auth <--- not surey we need THIS PACKAEG on the bootstrap cluster though???

  waiting for package: tanzu-cliplugins
  waiting for package: tanzu-core-management-plugins
  waiting for resource tanzu-addons-manager of type *v1alpha1.PackageInstall to be up and running
  waiting for package: tanzu-featuregates
  waiting for package: tanzu-framework
  waiting for package: tkg-clusterclass
  waiting for resource tanzu-cliplugins of type *v1alpha1.PackageInstall to be up and running
  waiting for package: tkg-clusterclass-vsphere

  waiting for package: tkr-service
  waiting for resource tanzu-framework of type *v1alpha1.PackageInstall to be up and running
  waiting for package: tkr-source-controller
  waiting for package: tkr-vsphere-resolver

```

## Now, we wait for PackageInstall's to complete on the kind cluster...

Were now inside of `WaitForManagementPackages` in tanzu cli.  We're going to wait for all the children of `tkg-pkg` to be
installed.... once these are installed, `kind` can do everything it needs to do, in order to bootstrap a management cluster.

Why does our "bootstrap" cluster need: tanzu-auth ?  (which comes with `tkg-pkg`) to be installed as a `PackageInstall` ? 


```
 waiting for resource tkr-service of type *v1alpha1.PackageInstall to be up and running
 successfully reconciled package: tkr-source-controller
 successfully reconciled package: tanzu-auth
 successfully reconciled package: tanzu-featuregates
 successfully reconciled package: tkg-clusterclass-vsphere
 successfully reconciled package: tanzu-framework
 successfully reconciled package: tkg-pkg
 successfully reconciled package: tanzu-core-management-plugins
 successfully reconciled package: tanzu-cliplugins
 successfully reconciled package: ako-operator
 successfully reconciled package: tkr-service
 successfully reconciled package: tkg-clusterclass
 successfully reconciled package: tanzu-addons-manager
 successfully reconciled package: tkr-vsphere-resolver
```
Carrying on 
```
[2023-01-10T19:50:48.605Z] Get AVI_CONTROL_PLANE_HA_PROVIDER from user config 
[2023-01-10T19:50:48.605Z] Installing AKO on bootstrapper...
[2023-01-10T19:50:48.605Z] Using default value for CONTROL_PLANE_MACHINE_COUNT = 1. Reason: CONTROL_PLANE_MACHINE_COUNT variable is not set
[2023-01-10T19:50:48.605Z] Fetching File="cluster-template-definition-devcc.yaml" Provider="vsphere" Type="InfrastructureProvider" Version="v1.5.1"
[2023-01-10T19:50:48.605Z] Management cluster config file has been generated and stored at: '/home/kubo/.config/tanzu/tkg/clusterconfigs/tkg-mgmt-vc.yaml'
[2023-01-10T19:50:48.605Z] Checking Tkr v1.24.9---vmware.1-tkg.1-rc.2 in bootstrap cluster...
[2023-01-10T19:50:48.605Z] waiting for resource v1.24.9---vmware.1-tkg.1-rc.2 of type *v1alpha3.TanzuKubernetesRelease to be up and running
```

All in all, the payload on our Kind cluster is below.  Note that instead of *antrea* we actually used `kindnet`... 

```
root@tkg-kind-cf5acj16tfr0jpgkuofg-control-plane:/# kubectl get pods -A --sort-by=.metadata.creationTimestamp
NAMESPACE                           NAME                                                                  READY   STATUS    RESTARTS   AGE
kube-system                         kube-controller-manager-tkg-kind-cf5acj16tfr0jpgkuofg-control-plane   1/1     Running   0          3m20s
kube-system                         etcd-tkg-kind-cf5acj16tfr0jpgkuofg-control-plane                      1/1     Running   0          3m18s
kube-system                         kube-scheduler-tkg-kind-cf5acj16tfr0jpgkuofg-control-plane            1/1     Running   0          3m18s
kube-system                         kube-apiserver-tkg-kind-cf5acj16tfr0jpgkuofg-control-plane            1/1     Running   0          3m18s
kube-system                         kube-proxy-fmb2s                                                      1/1     Running   0          3m5s
kube-system                         kindnet-wp255                                                         1/1     Running   0          3m5s
kube-system                         coredns-77d74f6759-hs8xw                                              1/1     Running   0          3m5s
kube-system                         coredns-77d74f6759-rmxrj                                              1/1     Running   0          3m5s
local-path-storage                  local-path-provisioner-6b84c5c67f-pjzwx                               1/1     Running   0          3m5s
tkg-system                          kapp-controller-64c8bdfc86-4gcqc                                      2/2     Running   0          2m55s
cert-manager                        cert-manager-webhook-5b85b9b58d-7wd84                                 1/1     Running   0          2m26s
cert-manager                        cert-manager-84b664ffb4-gkvgn                                         1/1     Running   0          2m26s
cert-manager                        cert-manager-cainjector-6cc99f4f85-fjn8k                              1/1     Running   0          2m26s
capi-system                         capi-controller-manager-8778fdfcb-22lx4                               1/1     Running   0          2m8s
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-75dc58b698-nnrnj            1/1     Running   0          2m6s
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-558bd599b4-jx5bk        1/1     Running   0          2m3s
capv-system                         capv-controller-manager-bbc859947-qlb59                               1/1     Running   0          2m
caip-in-cluster-system              caip-in-cluster-controller-manager-7ff5d7dd44-rlxrh                   1/1     Running   0          118s
tkg-system                          object-propagation-controller-manager-76df74cd4d-bswvm                1/1     Running   0          73s
tkg-system-networking               ako-operator-controller-manager-b5f9c68b9-t7g9c                       1/1     Running   0          55s
tkg-system                          tanzu-featuregates-controller-manager-688985774b-7tf9x                1/1     Running   0          11s
```

# TKG Management Cluster Creation (post-kind bootstrapping)

Now, we have some CAPI controllers, running in a `kind` cluster.  Let's create our first real CAPI cluster.  This cluster will eventually become our persistant Mangement Cluster for our TKG distribution, but it has a long way to go before it grows into that.   We will

- Create a CAPI object representing the mgmt cluster
- Wait for the capv controllers to reconcile this, and make VMs in vsphere running k8s
- Install the `tkg-pkg` package onto that cluster
    - Create `PackageInstall`s for all of the packages in `tkg-pkg`
    - Wait for `kapp controller` to install all of the `PackageInstall`
    - Wait for TKG Packages to come alive on the cluster and start running
- Then we'll graduate the cluster: We'll clusterctl move the KIND objects INTO THE management cluster

At that point, our ephemeral kind cluster, which we used as a bootloader for the mgmt cluster, is no longer needed. 

Ok, lets see how this all works:

## MGMT: Order of components

Before we show you the installation logs, lets look at the order of components as they come up:

### MGMT ORDER 1: Kubernetes Core components and KAPP
The first wave of components to come up is of course the core components of TKG.... kcm, etcd, apiserver, coredns, kube proxy, and kapp

```
kubectl get pods -A --context=tkg-mgmt-vc-admin@tkg-mgmt-vc --sort-by=.metadata.creationTimestamp
NAMESPACE                           NAME                                                             READY   STATUS    RESTARTS      AGE
kube-system                         kube-controller-manager-tkg-mgmt-vc-9tzbw-btchv                  1/1     Running   0             22m
kube-system                         etcd-tkg-mgmt-vc-9tzbw-btchv                                     1/1     Running   0             22m
kube-system                         kube-scheduler-tkg-mgmt-vc-9tzbw-btchv                           1/1     Running   0             22m
kube-system                         kube-apiserver-tkg-mgmt-vc-9tzbw-btchv                           1/1     Running   0             22m
kube-system                         kube-proxy-n8xnx                                                 1/1     Running   0             22m
tkg-system                          kapp-controller-64c8bdfc86-lcq2p                                 2/2     Running   0             22m
kube-system                         kube-proxy-76mkk                                                 1/1     Running   0             20m

### note: Some of these dont actually come up until AFTER antrea is up...
cert-manager                        cert-manager-84b664ffb4-lrqxw                                    1/1     Running   0             21m
cert-manager                        cert-manager-cainjector-6cc99f4f85-tqfc9                         1/1     Running   0             21m
cert-manager                        cert-manager-webhook-5b85b9b58d-46ckw                            1/1     Running   0             21m
kube-system                         vsphere-cloud-controller-manager-z2r5b                           1/1     Running   0             19m
kube-system                         vsphere-csi-node-gld59                                           3/3     Running   2 (17m ago)   19m
kube-system                         vsphere-csi-controller-658b6b6557-zvdn2                          7/7     Running   0             19m
kube-system                         vsphere-csi-node-v2qbx                                           3/3     Running   4 (17m ago)   19m
kube-system                         metrics-server-6ff66f4589-v2lrl                                  1/1     Running   0             19m
kube-system                         coredns-5776487ffb-47llq                                         1/1     Running   0             22m
kube-system                         coredns-5776487ffb-vgsjh                                         1/1     Running   0             22m

```
#### MTMG ORDER 2: Antrea !? 
 Would assume it would come on earlier (i.e. right after kapp and kube proxy). Otherwise, how else do things like cert-manager get IP addresses ? 
```
kube-system                         antrea-controller-74d4bb88ff-4vgk6                               1/1     Running   0             19m
kube-system                         antrea-agent-6r8jq                                               2/2     Running   0             19m
kube-system                         antrea-agent-pgzh5                                               2/2     Running   0             19m
```
#### MGMT ORDER 3: TKG Stuff and other CAPI stuff
```
secretgen-controller                secretgen-controller-7669575fdc-jbb82                            1/1     Running   0             19m
tkg-system                          tanzu-capabilities-controller-manager-764dc744b4-lc527           1/1     Running   0             18m
capi-system                         capi-controller-manager-8778fdfcb-srm6n                          1/1     Running   0             17m
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-75dc58b698-mb992       1/1     Running   0             17m
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-558bd599b4-d8454   1/1     Running   0             17m
capv-system                         capv-controller-manager-bbc859947-2lxnd                          1/1     Running   0             17m
caip-in-cluster-system              caip-in-cluster-controller-manager-7ff5d7dd44-vw5mn              1/1     Running   0             17m
tkg-system                          object-propagation-controller-manager-5dbdcd6447-7znnd           1/1     Running   0             16m
tkg-system-networking               ako-operator-controller-manager-557cdc564-qr5qf                  1/1     Running   0             16m
tkg-system                          tanzu-addons-controller-manager-754b967d8c-sv2zx                 1/1     Running   0             15m
tanzu-auth                          tanzu-auth-controller-manager-854798b9c-n7bj7                    1/1     Running   0             11m
tkg-system                          tkr-status-controller-manager-684d5c7968-hm74k                   1/1     Running   0             11m
tkg-system                          tkr-conversion-webhook-manager-fb45f8874-hmfh8                   1/1     Running   0             10m
```

We will add more dteails to the rationale above, soon.  For now, lets watch it happen in real time.

## MGMT 1: Create the ControlPlane Nodes

Now, we're going to start creating what will be the *real* Management Cluster..... 

```
    [2023-01-10T19:50:48.605Z] Start creating management cluster...
    [2023-01-10T19:50:48.605Z] patch cluster object with operation status: 
    [2023-01-10T19:50:48.606Z] 	{
    [2023-01-10T19:50:48.606Z] 		"metadata": {
    [2023-01-10T19:50:48.606Z] 			"annotations": {
    [2023-01-10T19:50:48.606Z] 				"TKGOperationInfo" : "{\"Operation\":\"Create\",\"OperationStartTimestamp\":\"2023-01-10 19:50:42.118795146 +0000 UTC\",\"OperationTimeout\":1800}",
    [2023-01-10T19:50:48.606Z] 				"TKGOperationLastObservedTimestamp" : "2023-01-10 19:50:42.118795146 +0000 UTC"
    [2023-01-10T19:50:48.606Z] 			}
    [2023-01-10T19:50:48.606Z] 		}
    [2023-01-10T19:50:48.606Z] 	}
    [2023-01-10T19:50:48.606Z] Applying patch to resource tkg-mgmt-vc of type *v1beta1.Cluster ...
    [2023-01-10T19:50:48.606Z] zero or multiple KCP objects found for the given cluster, 0 tkg-mgmt-vc tkg-system, retrying
    [2023-01-10T19:50:52.925Z] zero or multiple KCP objects found for the given cluster, 0 tkg-mgmt-vc tkg-system, retrying
    [2023-01-10T19:51:03.091Z] zero or multiple KCP objects found for the given cluster, 0 tkg-mgmt-vc tkg-system, retrying
    [2023-01-10T19:51:13.277Z] control plane is not available yet, retrying
...

[2023-01-10T19:54:23.036Z] control plane is not available yet, retrying
...

    [2023-01-10T19:54:33.277Z] Management cluster control plane is available, means API server is ready to receive requests
    [2023-01-10T19:54:33.277Z] getting secret for cluster
    [2023-01-10T19:54:33.277Z] waiting for resource tkg-mgmt-vc-kubeconfig of type *v1.Secret to be up and running
    [2023-01-10T19:54:33.277Z] Saving management cluster kubeconfig into /home/kubo/.kube/config
```

## MGMT 2: Install KAPP

We don't put Kapp in a ClusterResourceSet (although we used to at one time).  Instead, we manually add it to
a cluster as an inception component. 
```
    [2023-01-10T19:54:33.277Z] Installing kapp-controller on management cluster...
    [2023-01-10T19:54:33.277Z] User ConfigValues File: /tmp/2323403141.yaml
    [2023-01-10T19:54:33.277Z] Kapp-controller values-file: /tmp/4151289121.yaml
    [2023-01-10T19:54:33.554Z] Kapp-controller configuration file: /tmp/543269407
    [2023-01-10T19:54:35.004Z] waiting for resource kapp-controller of type *v1.Deployment to be up and running
    [2023-01-10T19:54:36.455Z] pods are not yet running for deployment 'kapp-controller' in namespace 'tkg-system', retrying
    ...
    [2023-01-10T19:55:07.427Z] pods are not yet running for deployment 'kapp-controller' in namespace 'tkg-system', retrying
    [2023-01-10T19:55:12.878Z] Installing providers on management cluster...
    [2023-01-10T19:55:12.878Z] Installing the clusterctl inventory CRD
    ...
    [2023-01-10T19:55:12.878Z] Creating CustomResourceDefinition="providers.clusterctl.cluster.x-k8s.io"
```

## MGMT 3: Install CAPI

We now have some code from cluster API that we call out to : cluster-api/cmd/clusterctl/client/init.go: 
```
	// checks if the cluster already contains a Core provider.
	// if not we consider this the first time init is executed, and thus we enforce the installation of a core provider,
	// a bootstrap provider and a control-plane provider (if not already explicitly requested by the user)
	log.Info("Fetching providers")
```

We can see the result of this command below.  Note that when we "fetch providers", Cluster API can get providers:
- from the internet (github repo for CAPI has a canonical set of default providers, and those are used for upstream e2e testing of CAPI).
- from a local directory (in TKG, the "providers" for CAPI are pre-packaged and unzipped from the management cluster plugin).
  
So, when we "fetch the providers", we are reading from a local directory of "provider" yamls.  Each provider implements the core
functionality of CAPI required for a given cloud.  For example:
- the "kubeadm bootstrap provider" writes a secret with cloud init bootstrapping data for VMs, which allows them to join the cluster.
- the "infrastructure-provider" (a list of them is here https://cluster-api.sigs.k8s.io/reference/providers.html) installs (in this case) the
capv-controller-manager, which reads "Machine" resources, and then makes "VsphereMachine" resource objects (capv then makes API calls
to vsphere to create these objects).


```
    [2023-01-10T19:55:14.880Z] Fetching providers
```


Followed by https://github.com/kubernetes-sigs/cluster-api/blob/main/cmd/clusterctl/client/repository/metadata_client.go#L76, which goes and fetches many things
```
    [2023-01-10T19:55:14.880Z] Fetching File="core-components.yaml" Provider="cluster-api" Type="CoreProvider" Version="v1.2.8"
    [2023-01-10T19:55:14.880Z] Fetching File="bootstrap-components.yaml" Provider="kubeadm" Type="BootstrapProvider" Version="v1.2.8"
    [2023-01-10T19:55:14.880Z] Fetching File="control-plane-components.yaml" Provider="kubeadm" Type="ControlPlaneProvider" Version="v1.2.8"
    ...
    [2023-01-10T19:55:29.851Z] Creating Deployment="cert-manager-webhook" Namespace="cert-manager"
    [2023-01-10T19:55:30.129Z] Creating MutatingWebhookConfiguration="cert-manager-webhook"
    [2023-01-10T19:55:30.129Z] Creating ValidatingWebhookConfiguration="cert-manager-webhook"
    [2023-01-10T19:55:30.403Z] Waiting for cert-manager to be available...
    [2023-01-10T19:55:30.403Z] Updating Namespace="cert-manager-test"
    [2023-01-10T19:55:30.403Z] Creating Issuer="test-selfsigned" Namespace="cert-manager-test"\
    ...
    23-01-10T20:06:32.719Z] Creating Issuer="test-selfsigned" Namespace="cert-manager-test"
    [2023-01-10T20:06:32.994Z] Creating Certificate="selfsigned-cert" Namespace="cert-manager-test"
    [2023-01-10T20:06:32.994Z] Deleting Namespace="cert-manager-test"
    [2023-01-10T20:06:33.269Z] Deleting Issuer="test-selfsigned" Namespace="cert-manager-test"
    ...
    [2023-01-10T20:07:00.515Z] Creating Issuer="caip-in-cluster-selfsigned-issuer" Namespace="caip-in-cluster-system"
    [2023-01-10T20:07:00.515Z] Creating MutatingWebhookConfiguration="caip-in-cluster-mutating-webhook-configuration"
    [2023-01-10T20:07:00.515Z] Creating ValidatingWebhookConfiguration="caip-in-cluster-validating-webhook-configuration"
```

Still in clusterctl, now https://github.com/kubernetes-sigs/cluster-api/blob/main/cmd/clusterctl/client/cluster/installer.go 
```    
    [2023-01-10T20:07:00.515Z] Creating inventory entry Provider="infrastructure-ipam-in-cluster" Version="v0.1.0" 
    TargetNamespace="caip-in-cluster-system"
```

Now, back in TKG Client.  We have tkg/client/init.go -> We are in the `WaitForProviders` method..... 

```
func (c *TkgClient) WaitForProviders(clusterClient clusterclient.Client, options waitForProvidersOptions) error {
```

##### These logs from from tanzu cli, 

Again , like in the kind cluster, we're going to iterate through all the "installed" components that clusterctl setup....

```
    [2023-01-10T20:07:00.515Z] installed  Component=="cluster-api"  Type=="CoreProvider"  Version=="v1.2.8"
    [2023-01-10T20:07:00.515Z] installed  Component=="kubeadm"  Type=="BootstrapProvider"  Version=="v1.2.8"
    [2023-01-10T20:07:00.515Z] installed  Component=="kubeadm"  Type=="ControlPlaneProvider"  Version=="v1.2.8"
    [2023-01-10T20:07:00.515Z] installed  Component=="vsphere"  Type=="InfrastructureProvider"  Version=="v1.5.1"
    [2023-01-10T20:07:00.515Z] installed  Component=="ipam-in-cluster"  Type=="InfrastructureProvider"  Version=="v0.1.0"
    [2023-01-10T20:07:00.515Z] Waiting for provider bootstrap-kubeadm
    [2023-01-10T20:07:00.515Z] Waiting for provider infrastructure-ipam-in-cluster
    [2023-01-10T20:07:00.515Z] Fetching File="ipam-components.yaml" Provider="ipam-in-cluster" Type="InfrastructureProvider" Version="v0.1.0"
    [2023-01-10T20:07:00.515Z] Fetching File="bootstrap-components.yaml" Provider="kubeadm" Type="BootstrapProvider" Version="v1.2.8"
    [2023-01-10T20:07:00.515Z] Waiting for provider infrastructure-vsphere
    [2023-01-10T20:07:00.515Z] Fetching File="infrastructure-components.yaml" Provider="vsphere" Type="InfrastructureProvider" Version="v1.5.1"
    [2023-01-10T20:07:00.515Z] Waiting for provider cluster-api
    [2023-01-10T20:07:00.515Z] Fetching File="core-components.yaml" Provider="cluster-api" Type="CoreProvider" Version="v1.2.8"
    [2023-01-10T20:07:00.515Z] Waiting for provider control-plane-kubeadm
    [2023-01-10T20:07:00.515Z] Fetching File="control-plane-components.yaml" Provider="kubeadm" Type="ControlPlaneProvider" Version="v1.2.8"
    [2023-01-10T20:07:00.787Z] waiting for resource caip-in-cluster-controller-manager of type *v1.Deployment to be up and running
    [2023-01-10T20:07:00.787Z] pods are not yet running for deployment 'caip-in-cluster-controller-manager' in namespace 'caip-in-cluster-system', retrying
    [2023-01-10T20:07:01.058Z] waiting for resource capi-kubeadm-bootstrap-controller-manager of type *v1.Deployment to be up and running
    [2023-01-10T20:07:01.058Z] waiting for resource capi-kubeadm-control-plane-controller-manager of type *v1.Deployment to be up and running
    [2023-01-10T20:07:01.058Z] pods are not yet running for deployment 'capi-kubeadm-control-plane-controller-manager' in namespace 'capi-kubeadm-control-plane-system', retrying
    [2023-01-10T20:07:01.058Z] pods are not yet running for deployment 'capi-kubeadm-bootstrap-controller-manager' in namespace 'capi-kubeadm-bootstrap-system', retrying
    [2023-01-10T20:07:01.058Z] waiting for resource capv-controller-manager of type *v1.Deployment to be up and running
    [2023-01-10T20:07:01.058Z] pods are not yet running for deployment 'capv-controller-manager' in namespace 'capv-system', retrying
    [2023-01-10T20:07:01.058Z] waiting for resource capi-controller-manager of type *v1.Deployment to be up and running
```

 # Wait for CAPI Providers on the management cluster 
 Remember this section in the bootstrap cluster?  Same thing here.  Just that it takes a little longer... 
```
    Passed waiting on provider cluster-api after 424.811557ms

    [2023-01-10T20:07:06.446Z] pods are not yet running for deployment 'caip-in-cluster-controller-manager' in namespace 'caip-in-cluster-system', retrying
    [2023-01-10T20:07:06.446Z] pods are not yet running for deployment 'capi-kubeadm-control-plane-controller-manager' in namespace 'capi-kubeadm-control-plane-system', retrying
    
    Passed waiting on provider bootstrap-kubeadm after 5.342992042s
    
    [2023-01-10T20:07:06.446Z] pods are not yet running for deployment 'capv-controller-manager' in namespace 'capv-system', retrying
    [2023-01-10T20:07:10.765Z] pods are not yet running for deployment 'caip-in-cluster-controller-manager' in namespace 'caip-in-cluster-system', retrying
    
    Passed waiting on provider control-plane-kubeadm after 10.343552056s
    Passed waiting on provider infrastructure-vsphere after 10.386474481s
    Passed waiting on provider infrastructure-ipam-in-cluster after 15.119210611s
```

And eventually, we get the same success message" 

```
    [2023-01-10T20:07:16.515Z] Success waiting on all providers.
    [2023-01-10T20:07:20.828Z] ℹ  Updated package repository 'tanzu-management' in namespace 'tkg-system'
    [2023-01-10T20:09:27.780Z] ℹ  
    [2023-01-10T20:09:27.780Z]  Added installed package 'tkg-pkg'waiting for package: tkg-pkg
```

# MGMT 4: MAGIC PART: tkg-pkg and "Waiting for package" hot loop 

Just like the kind cluster.  We let `tkg-pkg` pollute our cluster with all of our required management packages.   Interesting to note here that
packages we install are specific to the infrastructure provider.  So, we must either have `ako-operator` on all clouds (probably not), or, we have a different version of `tkg-pkg` meta-package repo that is used depending on if your azure or AWS.

```
    [2023-01-10T20:09:27.780Z] waiting for package: ako-operator
    [2023-01-10T20:09:27.780Z] waiting for package: tanzu-addons-manager
    [2023-01-10T20:09:27.780Z] waiting for package: tanzu-auth
    [2023-01-10T20:09:27.780Z] waiting for package: tanzu-cliplugins
    [2023-01-10T20:09:27.780Z] waiting for package: tanzu-core-management-plugins
    [2023-01-10T20:09:27.780Z] waiting for package: tanzu-featuregates
    [2023-01-10T20:09:27.780Z] waiting for package: tanzu-framework
    [2023-01-10T20:09:27.780Z] waiting for package: tkg-clusterclass
    [2023-01-10T20:09:27.780Z] waiting for package: tkg-clusterclass-vsphere
    [2023-01-10T20:09:27.780Z] waiting for package: tkr-service
    [2023-01-10T20:09:27.780Z] waiting for package: tkr-source-controller
    [2023-01-10T20:09:27.780Z] waiting for package: tkr-vsphere-resolver
    [2023-01-10T20:09:27.780Z] waiting for resource tkr-vsphere-resolver of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tkg-pkg of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource ako-operator of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tanzu-addons-manager of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tanzu-auth of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tanzu-cliplugins of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tanzu-core-management-plugins of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tkg-clusterclass of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tkg-clusterclass-vsphere of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tanzu-featuregates of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tkr-service of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tanzu-framework of type *v1alpha1.PackageInstall to be up and running
    [2023-01-10T20:09:27.780Z] waiting for resource tkr-source-controller of type *v1alpha1.PackageInstall to be up and running
```

And again, we now have the main management cluster packages, this time running in the VSphere cluster.

```
    [2023-01-10T20:09:27.780Z] successfully reconciled package: tkr-service
    [2023-01-10T20:09:27.780Z] successfully reconciled package: tkg-clusterclass-vsphere
    [2023-01-10T20:09:27.780Z] successfully reconciled package: tkr-vsphere-resolver
    [2023-01-10T20:09:27.780Z] successfully reconciled package: tanzu-addons-manager
    [2023-01-10T20:09:27.780Z] successfully reconciled package: tkg-pkg
    [2023-01-10T20:09:27.780Z] successfully reconciled package: tanzu-cliplugins
    [2023-01-10T20:09:27.781Z] successfully reconciled package: tanzu-framework
    [2023-01-10T20:09:27.781Z] successfully reconciled package: ako-operator
    [2023-01-10T20:09:27.781Z] successfully reconciled package: tkr-source-controller
    [2023-01-10T20:09:27.781Z] successfully reconciled package: tkg-clusterclass
    [2023-01-10T20:09:27.781Z] successfully reconciled package: tanzu-auth
    [2023-01-10T20:09:27.781Z] successfully reconciled package: tanzu-core-management-plugins
    [2023-01-10T20:09:27.781Z] successfully reconciled package: tanzu-featuregates
```

Now, we get ready to migrate the management cluster to VSphere.

```
    [2023-01-10T20:09:27.781Z] Waiting for the management cluster to get ready for move...
    [2023-01-10T20:09:27.781Z] waiting for resource tkg-mgmt-vc of type *v1beta1.Cluster to be up and running
    [2023-01-10T20:09:27.781Z] waiting for resources type *v1beta1.KubeadmControlPlaneList to be up and running
    [2023-01-10T20:09:27.781Z] waiting for resources type *v1beta1.MachineDeploymentList to be up and running
    [2023-01-10T20:09:27.781Z] waiting for resources type *v1beta1.MachineList to be up and running
    [2023-01-10T20:09:27.781Z] Waiting for addons installation...
    [2023-01-10T20:09:27.781Z] waiting for resources type *v1beta1.ClusterResourceSetList to be up and running
    [2023-01-10T20:09:27.781Z] waiting for resource antrea-controller of type *v1.Deployment to be up and running
    [2023-01-10T20:09:27.781Z] Applying ClusterBootstrap and its associated resources on management cluster
    [2023-01-10T20:09:27.781Z] User ConfigValues File: /tmp/1826823029.yaml
    [2023-01-10T20:09:27.781Z] Checking if TKr v1.24.9---vmware.1-tkg.1-rc.2 is created on management cluster
    [2023-01-10T20:09:27.781Z] waiting for resource v1.24.9---vmware.1-tkg.1-rc.2 of type *v1alpha3.TanzuKubernetesRelease to be up and running
    [2023-01-10T20:09:27.781Z] Applying ClusterBootstrap: apiVersion: v1
    [2023-01-10T20:09:27.781Z] kind: Secret
    [2023-01-10T20:09:27.781Z] metadata:
    [2023-01-10T20:09:27.781Z]   name: tkg-mgmt-vc-pinniped-package
    [2023-01-10T20:09:27.781Z]   namespace: tkg-system
    [2023-01-10T20:09:27.781Z]   labels:
    [2023-01-10T20:09:27.781Z]     tkg.tanzu.vmware.com/addon-name: pinniped
    [2023-01-10T20:09:27.781Z]     tkg.tanzu.vmware.com/cluster-name: tkg-mgmt-vc
    [2023-01-10T20:09:27.781Z]     clusterctl.cluster.x-k8s.io/move: ""
    [2023-01-10T20:09:27.781Z]   annotations:
    [2023-01-10T20:09:27.781Z]     tkg.tanzu.vmware.com/addon-type: authentication/pinniped
    [2023-01-10T20:09:27.781Z] type: clusterbootstrap-secret
    [2023-01-10T20:09:27.781Z] stringData:
    [2023-01-10T20:09:27.781Z]   values.yaml: |
    [2023-01-10T20:09:27.781Z]     infrastructure_provider: vsphere
    [2023-01-10T20:09:27.781Z]     tkg_cluster_role: workload
    [2023-01-10T20:09:27.781Z]     identity_management_type: none
    [2023-01-10T20:09:27.781Z] ---
```
The remainder of the YAML has "ClusterBootstrap" information, including the TKR used to create your cluster.  The ClusterBootstrap resource
defines all the packages that need to be installed on your management cluster once it starts up.  This is a TKG specific construct, it tellsTKG things like
- `spec.cni.refName`: what CNI you are using, (i.e. Antrea or Calico)
- CalicoConfig, with `spec.calico.config` and how youll configure it, for example fields like `vethMTU:`, or `vxlan` vs `bgp` mode.
- what additional packages you want to install (i.e. `pinniped`, `metrics-server`, ...)
- KappControllerConfig (where you can specify what namespace kapp runs in and so on).
This bootstrap is made for you when you setup TKG the first time.

Note that, if you're a TKG 1.6 user, this is a new object for you: It represents the fact that you're cluster configuration no longer is defaulted
via YTT, but rather, as a Kubernetes object stored inside the Kubernetes API Server.  Thus, you can mix and match objects referenced by your
cluster bootstrap template without having to do a large synchronous "YTT creation" step, as was once done in early versions of TKG.
```
    [2023-01-10T20:09:27.781Z] apiVersion: run.tanzu.vmware.com/v1alpha3
    [2023-01-10T20:09:27.781Z] kind: ClusterBootstrap
    [2023-01-10T20:09:27.781Z] metadata:
    [2023-01-10T20:09:27.781Z]   name: tkg-mgmt-vc
    [2023-01-10T20:09:27.781Z]   namespace: tkg-system
    [2023-01-10T20:09:27.781Z]   annotations:
    [2023-01-10T20:09:27.781Z]     tkg.tanzu.vmware.com/add-missing-fields-from-tkr: v1.24.9---vmware.1-tkg.1-rc.2
    [2023-01-10T20:09:27.781Z] spec:
    [2023-01-10T20:09:27.781Z]   kapp:
    [2023-01-10T20:09:27.781Z]     refName: kapp-controller*
    [2023-01-10T20:09:27.781Z]   additionalPackages:
    [2023-01-10T20:09:27.781Z]   - refName: metrics-server*
    [2023-01-10T20:09:27.781Z]   - refName: secretgen-controller*
    [2023-01-10T20:09:27.781Z]   - refName: pinniped*
    [2023-01-10T20:09:27.781Z]   - refName: tkg-storageclass*
    [2023-01-10T20:09:27.781Z]     valuesFrom:
    [2023-01-10T20:09:27.781Z]       inline:
    [2023-01-10T20:09:27.781Z]         metadata:
    [2023-01-10T20:09:27.781Z]           infraProvider: vsphere
```

## Performing the move

At this point we have:
- kind running
  - Cluster API object for a VSPhere cluster
  - Vsphere VMs that are a K8s cluster
    - They are running Cluster API controllers
    - They are running Cluster API VSphere controllers
    - They are running AVI/AKO/Antrea/etc

### Sounds like a good Management Cluster....so why do we need to MOVE anything ? 

- The VSphere K8s cluster is not managing itself...
- The Kind cluster is still capable of manageming the management cluster !
- That means kind needs to go away... otherwise it will be a security issue, and it will be confusing for people.

```
    [2023-01-10T20:09:27.781Z] Moving all Cluster API objects from bootstrap cluster to management cluster...
    [2023-01-10T20:09:27.781Z] Performing move...
```
### Move: part 0,  Discovery 

Moving CAPI resources requires listing them all, first...  Theres lots of stuff (Even cloud provider credentials, for example), that need
to get migrated over.  For details on this, check https://cluster-api.sigs.k8s.io/clusterctl/provider-contract.html#move.  This describes yhe
discovery process...  Let's look at the logs from TKG for the discovery.  Remember here, 

- we're reading Cluster API objects that are living in our Kind cluster  
- with the intent of migrating them to our PERMANANT managemnt cluster.

```
    # introspect the CAPI objects on the kind cluster so that the mgmt cluster can become self-aware and we dont lose any info after kind dies.
    [2023-01-10T20:09:27.781Z] Discovering Cluster API objects
    [2023-01-10T20:09:27.781Z] Certificate Count=4
    [2023-01-10T20:09:27.781Z] KubeadmControlPlane Count=1
    [2023-01-10T20:09:27.781Z] KubeadmControlPlaneTemplate Count=1
    [2023-01-10T20:09:27.781Z] VSphereClusterTemplate Count=1
    [2023-01-10T20:09:27.781Z] CertificateRequest Count=4
    [2023-01-10T20:09:27.781Z] ClusterClass Count=1
    [2023-01-10T20:09:27.781Z] KubeadmConfigTemplate Count=2
    [2023-01-10T20:09:27.781Z] MachineSet Count=1
    [2023-01-10T20:09:27.781Z] VSphereMachineTemplate Count=4
    [2023-01-10T20:09:27.781Z] Issuer Count=3
    [2023-01-10T20:09:27.781Z] Machine Count=2
    [2023-01-10T20:09:27.781Z] Secret Count=51
    [2023-01-10T20:09:27.781Z] ConfigMap Count=42
    [2023-01-10T20:09:27.781Z] KubeadmConfig Count=2
    [2023-01-10T20:09:27.781Z] MachineDeployment Count=1
    [2023-01-10T20:09:27.781Z] VSphereVM Count=2
    [2023-01-10T20:09:27.781Z] Cluster Count=1
    [2023-01-10T20:09:27.781Z] VSphereCluster Count=1
    [2023-01-10T20:09:27.781Z] MachineHealthCheck Count=2
    [2023-01-10T20:09:27.781Z] VSphereMachine Count=2
    [2023-01-10T20:09:27.781Z] Total objects Count=145

    [2023-01-10T20:09:27.781Z] Excluding secret from move (not linked with any Cluster) name="ako-operator-v2-values"
    [2023-01-10T20:09:27.782Z] Excluding secret from move (not linked with any Cluster) name="tanzu-framework-values"
    ...
    [2023-01-10T20:09:27.782Z] Excluding secret from move (not linked with any Cluster) name="tkr-source-controller-values"
    [2023-01-10T20:09:27.782Z] Excluding secret from move (not linked with any Cluster) name="tkr-vsphere-resolver-values"

    ...
    [2023-01-10T20:09:27.782Z] Object won't be moved because it's not included in GVK considered for move kind="PackageRepository" 
    [2023-01-10T20:09:27.782Z] Object won't be moved because it's not included in GVK considered for move kind="PackageInstall" name="tanzu-addons-manager"

    [2023-01-10T20:09:27.782Z] Object won't be moved because it's not included in GVK considered for move kind="ClusterBootstrap" name="tkg-mgmt-vc"
```

### Move: part 1, Now we know WHAT to move to the mgmt cluster

Now finally, we start the "move" .  FIRST WE HAVE TO PAUSE the existing `Cluster` object!.

```
    [2023-01-10T20:09:27.782Z] Moving Cluster API objects Clusters=1
    [2023-01-10T20:09:27.782Z] Moving Cluster API objects ClusterClasses=1

    #### The PAUSE generally is IMPORTANT !!!!!!!!  (in TKG, its not a huge deal bc nobody is using bootstrap cluster right now...) But
    #### In the real world, (i.e. in a backup restore situation) when using cluster API, this is a non-trivial operation.
    #### Hence we ALWAYS pause the source cluster before migraion as a matter of how Clusterctl works.

    [2023-01-10T20:09:27.782Z] Pausing the source cluster

    [2023-01-10T20:09:27.782Z] Set Cluster.Spec.Paused Paused=true Cluster="tkg-mgmt-vc" Namespace="tkg-system"
    [2023-01-10T20:09:27.782Z] Pausing the source cluster classes
    [2023-01-10T20:09:27.782Z] Set Paused annotation ClusterClass="tkg-vsphere-default-v1.0.0" Namespace="tkg-system"
    [2023-01-10T20:09:27.782Z] Creating target namespaces, if missing
    [2023-01-10T20:09:27.782Z] Creating objects in the target cluster
    [2023-01-10T20:09:27.782Z] Creating ClusterClass="tkg-vsphere-default-v1.0.0" Namespace="tkg-system"
    ...
    [2023-01-10T20:09:32.471Z] Deleting VSphereVM="tkg-mgmt-vc-md-0-69pnq-858fcd866b-4h59q" Namespace="tkg-system"
    [2023-01-10T20:09:32.471Z] Deleting Secret="tkg-mgmt-vc-md-0-bootstrap-mzn9v-cgbx2" Namespace="tkg-system"
    [2023-01-10T20:09:32.747Z] Deleting KubeadmConfig="tkg-mgmt-vc-md-0-bootstrap-mzn9v-cgbx2" Namespace="tkg-system"
    [2023-01-10T20:09:32.747Z] Deleting VSphereVM="tkg-mgmt-vc-4b4wz-6vhgl" Namespace="tkg-system"
    
    [2023-01-10T20:09:34.409Z] Deleting Secret="tkg-mgmt-vc-antrea-data-values" Namespace="tkg-system"
    [2023-01-10T20:09:34.409Z] Deleting VSphereMachineTemplate="tkg-mgmt-vc-control-plane-6g4h4" Namespace="tkg-system"
    ...
    [2023-01-10T20:09:36.122Z] Deleting VSphereMachineTemplate="tkg-vsphere-default-v1.0.0-control-plane" Namespace="tkg-system"
    [2023-01-10T20:09:36.396Z] Deleting VSphereClusterTemplate="tkg-vsphere-default-v1.0.0-cluster" Namespace="tkg-system"

    [2023-01-10T20:09:37.271Z] Remove Paused annotation ClusterClass="tkg-vsphere-default-v1.0.0" Namespace="tkg-system"
```

... WAIT FOR IT ...

```
    #### This is the last step of the clusterctl move !!! 
    [2023-01-10T20:09:37.271Z] Set Cluster.Spec.Paused Paused=false Cluster="tkg-mgmt-vc" Namespace="tkg-system"
    [2023-01-10T20:09:37.271Z] Resuming the target cluster

```

How is this different then Velero migrations ? Clusterctl looks at `ownerRef` fields.  
- CAPI: Preserves exact identities of each object (ownerRef, finalizer, managed fields)
  - This forces CAPI to order `Creation` and `Deletion`
- Velero: DROPS all ownerRefs, finalizers, managed fields.... 

Now WE start back up the management cluster.   We can see the logic for how we patch things in tkg/client/init.go in the `PatchClusterInitOperations` function...

Concretely, one reason to patch mgmt cluster is so that tanzu cli can statelessly determine what verion of TKG is associated w/ the MGMT cluster.
That is super important, for example, when a user wants to upgrade a WL cluster to a New TKR, bc only CERTAIN Mgmt clusters support CERTAIN TKRs.
i.e. you cant run create a WL cluster w k8s 1.25 if you are on a 1.2 TKG management cluster, bc that mgmt cluster only runs k8s 1.20(or something).

### Move: part 2,Patch the management cluster after the fact

```
    [2023-01-10T20:09:37.872Z] Applying patch to resource tkg-mgmt-vc of type *v1beta1.Cluster ...
    [2023-01-10T20:09:37.872Z] Applying patch to resource tkg-mgmt-vc of type *v1beta1.Cluster ...
    [2023-01-10T20:09:38.472Z] Applying patch to resource tkg-vsphere-default-v1.0.0-kcp of type *unstructured.Unstructured ...
    [2023-01-10T20:09:38.743Z] Applying patch to resource tkg-vsphere-default-v1.0.0-cluster of type *unstructured.Unstructured ...
    [2023-01-10T20:09:39.016Z] Applying patch to resource tkg-mgmt-vc-control-plane-6g4h4 of type *unstructured.Unstructured ...
    [2023-01-10T20:09:39.016Z] Applying patch to resource tkg-mgmt-vc-md-0-infra-hjcsr of type *unstructured.Unstructured ...
    [2023-01-10T20:09:39.016Z] Applying patch to resource tkg-vsphere-default-v1.0.0-control-plane of type *unstructured.Unstructured ...
    [2023-01-10T20:09:39.289Z] Applying patch to resource tkg-vsphere-default-v1.0.0-worker of type *unstructured.Unstructured ...
    [2023-01-10T20:09:39.289Z] Applying patch to resource tkg-vsphere-default-v1.0.0 of type *unstructured.Unstructured ...
    [2023-01-10T20:09:39.918Z] Applying patch to resource tkg-mgmt-vc-md-0-bootstrap-mzn9v of type *unstructured.Unstructured ...
    [2023-01-10T20:09:39.918Z] Applying patch to resource tkg-vsphere-default-v1.0.0-md-config of type *unstructured.Unstructured ...
    [2023-01-10T20:09:40.899Z] IsProd: 
    [2023-01-10T20:09:40.899Z] IsOfficialBuild: False
    [2023-01-10T20:09:40.899Z] ---
    [2023-01-10T20:09:40.899Z] apiVersion: v1
    [2023-01-10T20:09:40.899Z] kind: Namespace
    [2023-01-10T20:09:40.899Z] metadata:
    [2023-01-10T20:09:40.899Z]   name: tkg-system-telemetry
    [2023-01-10T20:09:40.899Z] 
    [2023-01-10T20:09:40.899Z] ---
    [2023-01-10T20:09:40.899Z] apiVersion: v1
    [2023-01-10T20:09:40.899Z] kind: ServiceAccount
    [2023-01-10T20:09:40.899Z] metadata:
    [2023-01-10T20:09:40.899Z]   name: tkg-telemetry-sa
    [2023-01-10T20:09:40.899Z]   namespace: tkg-system-telemetry
    [2023-01-10T20:09:40.899Z] 
    [2023-01-10T20:09:40.899Z] ---
    [2023-01-10T20:09:40.899Z] kind: ClusterRole
    [2023-01-10T20:09:40.899Z] apiVersion: rbac.authorization.k8s.io/v1
    [2023-01-10T20:09:40.899Z] metadata:
    [2023-01-10T20:09:40.899Z]   name: tkg-telemetry-cluster-role
    [2023-01-10T20:09:40.899Z] rules:
    [2023-01-10T20:09:40.899Z]   - apiGroups: [""]
    [2023-01-10T20:09:40.900Z]     resources: ["secrets", "namespaces", "configmaps"]
    ...
    [2023-01-10T20:09:40.901Z]           restartPolicy: Never
    [2023-01-10T20:09:41.497Z] Creating tkg-bom versioned ConfigMaps...
```

And finally we have a management cluster........
```
    [2023-01-10T20:09:41.497Z] You can now access the management cluster tkg-mgmt-vc by running 'kubectl config use-context tkg-mgmt-vc-admin@tkg-mgmt-vc'
    [2023-01-10T20:09:41.497Z] Deleting kind cluster: tkg-kind-ceus13jb4t5tab9k5jk0
    [2023-01-10T20:09:45.829Z] 
    [2023-01-10T20:09:45.829Z] Management cluster created!
    [2023-01-10T20:09:45.829Z] 
    [2023-01-10T20:09:45.829Z] 
    [2023-01-10T20:09:45.829Z] You can now create your first workload cluster by running the following:
    [2023-01-10T20:09:45.829Z] 
    [2023-01-10T20:09:45.829Z]   tanzu cluster create [name] -f [file]
    [2023-01-10T20:09:45.829Z] 
    [2023-01-10T20:09:45.829Z] 
    [2023-01-10T20:09:45.829Z] Some addons might be getting installed! Check their status by running the following:
    [2023-01-10T20:09:45.829Z] 
    [2023-01-10T20:09:45.829Z]   kubectl get apps -A
    [2023-01-10T20:09:45.829Z] 
```

Now the managementcluster/create.go checks one last time, after MGMT cluster creation, to confirm that
the plugins  it cares about, are installed.  See cmd/cli/plugin/managementcluster/create.go for details. 

```
// Sync plugins if management-cluster creation is successful and --dry-run was not set
	if config.IsFeatureActivated(cliconfig.FeatureContextAwareCLIForPlugins) && !iro.dryRun {
		err = pluginmanager.SyncPlugins()
```
And of course, this works ....

```
    [2023-01-10T20:09:45.829Z] ℹ  Checking for required plugins... 
    [2023-01-10T20:09:46.826Z] ℹ  Installing plugin 'kubernetes-release:v0.28.0-dev' with target 'kubernetes'
    [2023-01-10T20:09:50.250Z] ℹ  Installing plugin 'cluster:v0.28.0-dev' with target 'kubernetes'
    [2023-01-10T20:09:52.879Z] ℹ  Installing plugin 'feature:v0.28.0-dev' with target 'kubernetes'
    [2023-01-10T20:09:53.865Z] ℹ  Successfully installed all required plugins 
```

Now we list available clusters, and we see the mgmt cluster.  Note the need to use -A... 

```
    [2023-01-10T20:09:54.698Z] INFO:root:====== 198  CMD: tanzu cluster list --include-management-cluster -A --output=json
    [2023-01-10T20:09:55.303Z] [
    [2023-01-10T20:09:55.303Z]   {
    [2023-01-10T20:09:55.303Z]     "name": "tkg-mgmt-vc",
    [2023-01-10T20:09:55.303Z]     "namespace": "tkg-system",
    [2023-01-10T20:09:55.303Z]     "status": "running",
    [2023-01-10T20:09:55.303Z]     "plan": "dev",
    [2023-01-10T20:09:55.303Z]     "controlplane": "1/1",
    [2023-01-10T20:09:55.303Z]     "workers": "1/1",
    [2023-01-10T20:09:55.303Z]     "kubernetes": "v1.24.9+vmware.1",
    [2023-01-10T20:09:55.303Z]     "roles": [
    [2023-01-10T20:09:55.303Z]       "management"
    [2023-01-10T20:09:55.303Z]     ],
    [2023-01-10T20:09:55.303Z]     "tkr": "v1.24.9---vmware.1-tkg.1-rc.2",
    [2023-01-10T20:09:55.303Z]     "labels": {
    [2023-01-10T20:09:55.303Z]       "cluster-role.tkg.tanzu.vmware.com/management": "",
    [2023-01-10T20:09:55.303Z]       "cluster.x-k8s.io/cluster-name": "tkg-mgmt-vc",
    [2023-01-10T20:09:55.303Z]       "networking.tkg.tanzu.vmware.com/avi": "install-ako-for-management-cluster",
    [2023-01-10T20:09:55.303Z]       "run.tanzu.vmware.com/tkr": "v1.24.9---vmware.1-tkg.1-rc.2",
    [2023-01-10T20:09:55.303Z]       "tkg.tanzu.vmware.com/cluster-name": "tkg-mgmt-vc",
    [2023-01-10T20:09:55.303Z]       "topology.cluster.x-k8s.io/owned": ""
    [2023-01-10T20:09:55.303Z]     }
    [2023-01-10T20:09:55.303Z]   }
```

Ok thats it.  We now have a fully functional TKG MAnagement cluster !!!


