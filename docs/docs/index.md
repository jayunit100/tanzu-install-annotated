# TKG Client and Kind Bootstrapping

```
  [2023-01-10T19:36:11.674Z] + curl https://build-artifactory.eng.vmware.com/kscom-generic-local/TKG/channels/442519250544895703/_boltArtifacts/tkg-v2.1.0-rc.2.buildinfo.yaml
```

```
  sudo -S -p '[sudo] password: ' ln -sf $(realpath /home/kubo/tanzu_tools/cli/core/v0.28.0-dev/ INFO:root:====== 99   CMD: tanzu config get | yq eval '.clientOptions.features.global.context-aware-cli-for-plugins' -
  true
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


- INFO:root:Downloading OVA from http://build-squid.eng.vmware.com/build/mts/release/bora-21093855/publish/lin64/tkg_release/node/ova-ubuntu-2004-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93/ubuntu-2004-kube-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93.ova to 
temp_ova_dir-NVZOBCZG

-  INFO:root:Importing OVA from http://build-squid.eng.vmware.com/build/mts/release/bora-21093855/publish/lin64/tkg_release/node/ova-ubuntu-2004-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93/ubuntu-2004-kube-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93.ova to /dc0/vm/.  This might take a while...

# TKG Bootstrap, Kind

The `InitRegion` function lives inside Tanzu CLI.  IT has ALL the logic for kind and mgmt cluster bootstrapping.  

- You can't have a workload cluster w/o a persistent management cluster.
- You can't have a persistent management cluster without a kind bootstrap cluster.
- You can't have a kind bootstrap cluster that works w/ TKG without installing tkg-pkg and other pre-requisites

Keep in mind that MANY OF THE THINGS the kind cluster does has to happen AGAIN when we make the persistent mangaement cluster.  So, the purpose of the `kind` cluster is to

- *define* the management cluster as a set of CRDs that run a particular K8s version, CNI, and so on.
- *run* a few `capv-controller` objects that can make VMs in the vsphere cloud, where those contorllers read the CRDs, and act on them (i.e. by making VMs on vsphere that run k8s )
- *move* those `capv-controller` and other `capi` objects into the vsphere cluster, once it is up
- *self-destruct* once the persistant management cluster is up and running.

This page ONLY defines the creation of the `kind` cluster.  IT doesn't ACTUALLY create any CAPI clusters, that is on the *next* markdown page, where we define the `Management Cluster` creation.

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

## Kind Cluster: TKG And ClusterClass customizations

Now that the kind cluster is up, tanzu cli will start customizing it. 
```
 Warning: unable to find component 'kube_rbac_proxy' under BoM
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

We now see in the logs, something like this.  At this point, tanzu cli is now `clusterctl`, internally, and clusterctl is doing a bunch
of magic to turn our kind cluster into a cluster API Management cluster... this means... just adding a bunch of containers, provider CRDs, and so on...

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

# Kind Cert Manager setup 

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
      Creating Issuer="test-selfsigned" Namespace="cert-manager-test"
      ...
      Deleting Certificate="selfsigned-cert" Namespace="cert-manager-test"
```

# Installing CAPI objects onto the Kind cluster

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

# MAGIC PART: tkg-pkg and "Waiting for package" hot loop 


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

## Create the CAPI object

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


We now have some code from cluster API that we call out to : cluster-api/cmd/clusterctl/client/init.go: 
```
	// checks if the cluster already contains a Core provider.
	// if not we consider this the first time init is executed, and thus we enforce the installation of a core provider,
	// a bootstrap provider and a control-plane provider (if not already explicitly requested by the user)
	log.Info("Fetching providers")
```
We can see the result of this command below: 
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

# MAGIC PART: tkg-pkg and "Waiting for package" hot loop 

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

## Create the CAPI object

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


We now have some code from cluster API that we call out to : cluster-api/cmd/clusterctl/client/init.go: 
```
	// checks if the cluster already contains a Core provider.
	// if not we consider this the first time init is executed, and thus we enforce the installation of a core provider,
	// a bootstrap provider and a control-plane provider (if not already explicitly requested by the user)
	log.Info("Fetching providers")
```
We can see the result of this command below: 
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

# MAGIC PART: tkg-pkg and "Waiting for package" hot loop 

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

HOWEVER:

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

# TKG Workload Cluster Creation

```
[2023-01-10T20:09:55.303Z] INFO:root:    (\,*********'()'--o  [Verifying number of nodes and k8s [2023-01-10T20:09:55.585Z] INFO:root:    (\,*********'()'--o  [verify tkg-bom existing and not none]
[2023-01-10T20:09:55.585Z] INFO:root:    (\,*********'()'--o  [verify mgmt cluster's default k8s [2023-01-10T20:09:56.133Z] INFO:root:    (\,*********'()'--o  [Verifying anti affinity for management cluster]
[2023-01-10T20:09:56.133Z] INFO:root:    (\,*********'()'--o  [Verifying tkr/kapp configmap has correct configuration in http-proxy mode or normal iaas]
[2023-01-10T20:09:57.321Z] INFO:root:====== 254  CMD: tanzu cluster version
,,,
[2023-01-10T20:10:00.105Z] INFO:root:    (\,*********'()'--o  [CREATING A WORKLOAD CLUSTER tkg-vc-antrea with {'OS_VERSION': '20.04', 'OS_ARCH': 'amd64', 'OS_NAME': 'ubuntu', 'CNI': 'antrea', 'CLUSTER_PLAN': 'dev', 'KUBERNETES_VERSION': 'v1.24.9+vmware.1', 'INFRASTRUCTURE_PROVIDER': 'vsphere', 'AVI_CONTROL_PLANE_HA_PROVIDER': 'true'}]
```


Now we create our first workload cluster: 

```
tanzu kubernetes cluster create tkg-vc-antrea -v 9 -f tkg-vc-antrea.yaml --tkr v1.24.9---vmware.1-tkg.1-rc.2
```

This is normal, b/c we dont usually install pinniped in testbeds.
```
    [2023-01-10T20:10:00.672Z] configmaps "pinniped-info" not found, retrying
    ...
    [2023-01-10T20:10:25.534Z] configmaps "pinniped-info" not found, retrying
    [2023-01-10T20:10:25.534Z] Warning: Pinniped configuration not found; Authentication via Pinniped will not be set up in this cluster. If you wish to set up Pinniped after the cluster is created, please refer to the documentation.
```

We dont make old clusters, instead we use cluster class for everything... 
```
[2023-01-10T20:10:26.403Z] Setting ALLOW_LEGACY_CLUSTER to "false"
```

## Telling Tanzu cli to create a WL cluster

```
    [2023-01-10T20:10:26.403Z] Fetching File="cluster-template-definition-devcc.yaml" Provider="vsphere" Type="InfrastructureProvider" Version="v1.5.1"
    [2023-01-10T20:10:27.389Z] 
    [2023-01-10T20:10:27.390Z] Legacy configuration file detected. The inputs from said file have been converted into the new Cluster configuration as '/home/kubo/.config/tanzu/tkg/clusterconfigs/tkg-vc-antrea.yaml'
    [2023-01-10T20:10:27.390Z] 
    [2023-01-10T20:10:27.390Z] Using this new Cluster configuration '/home/kubo/.config/tanzu/tkg/clusterconfigs/tkg-vc-antrea.yaml' to create the cluster.

    ### Heres where we start
    [2023-01-10T20:10:27.390Z] creating workload cluster 'tkg-vc-antrea'...
```

Why do we patch the workload cluster object ? 

```
[2023-01-10T20:10:30.789Z] patch cluster object with operation status: 
[2023-01-10T20:10:30.789Z] 	{
[2023-01-10T20:10:30.789Z] 		"metadata": {
[2023-01-10T20:10:30.789Z] 			"annotations": {
[2023-01-10T20:10:30.789Z] 				"TKGOperationInfo" : "{\"Operation\":\"Create\",\"OperationStartTimestamp\":\"2023-01-10 20:10:30.295742703 +0000 UTC\",\"OperationTimeout\":1800}",
[2023-01-10T20:10:30.789Z] 				"TKGOperationLastObservedTimestamp" : "2023-01-10 20:10:30.295742703 +0000 UTC"
[2023-01-10T20:10:30.789Z] 			}
[2023-01-10T20:10:30.789Z] 		}
[2023-01-10T20:10:30.789Z] 	}
```

Wait a while... 
```
[2023-01-10T20:10:30.789Z] Applying patch to resource tkg-vc-antrea of type *v1beta1.Cluster ...
[2023-01-10T20:10:30.790Z] waiting for cluster to be initialized...
[2023-01-10T20:10:30.790Z] cluster state is unchanged 1
[2023-01-10T20:10:30.790Z] [zero or multiple KCP objects found for the given cluster, 0 tkg-vc-antrea default, no MachineDeployment objects found for the given cluster]
[2023-01-10T20:10:30.790Z] [zero or multiple KCP objects found for the given cluster, 0 tkg-vc-antrea default, no MachineDeployment objects found for the given cluster], retrying
```

Wait for controlplane... 

```
[2023-01-10T20:10:45.875Z] Applying patch to resource tkg-vc-antrea of type *v1beta1.Cluster ...
[2023-01-10T20:10:45.875Z] cluster control plane is still being initialized: ScalingUp
[2023-01-10T20:10:45.875Z] cluster control plane is still being initialized: ScalingUp, retrying
[2023-01-10T20:11:00.953Z] cluster state is unchanged 1
[2023-01-10T20:11:00.954Z] cluster control plane is still being initialized: ScalingUp, retrying
[2023-01-10T20:11:16.075Z] cluster state is unchanged 2
[2023-01-10T20:11:16.075Z] cluster control plane is still being initialized: ScalingUp, retrying
[2023-01-10T20:11:31.194Z] cluster state is unchanged 3
[2023-01-10T20:11:31.194Z] cluster control plane is still being initialized: ScalingUp, retrying
[2023-01-10T20:11:46.325Z] cluster state is unchanged 4
[2023-01-10T20:11:46.325Z] cluster control plane is still being initialized: ScalingUp, retrying
[2023-01-10T20:12:01.434Z] cluster state is unchanged 5
[2023-01-10T20:12:01.434Z] cluster control plane is still being initialized: ScalingUp, retrying
[2023-01-10T20:12:16.537Z] cluster state is unchanged 6
[2023-01-10T20:12:16.537Z] cluster control plane is still being initialized: ScalingUp, retrying
[2023-01-10T20:12:31.619Z] cluster state is unchanged 7
[2023-01-10T20:12:31.620Z] cluster control plane is still being initialized: ScalingUp, retrying
```

Control plane up now.... now we can poll the workload cluster till its up: 
```
[2023-01-10T20:12:46.750Z] getting secret for cluster
[2023-01-10T20:12:46.750Z] waiting for resource tkg-vc-antrea-kubeconfig of type *v1.Secret to be up and running
[2023-01-10T20:12:46.750Z] waiting for cluster nodes to be available...
[2023-01-10T20:12:46.750Z] waiting for resource tkg-vc-antrea of type *v1beta1.Cluster to be up and running
[2023-01-10T20:12:46.750Z] waiting for resources type *v1beta1.KubeadmControlPlaneList to be up and running
[2023-01-10T20:12:46.750Z] control-plane is still creating replicas, DesiredReplicas=1 Replicas=1 ReadyReplicas=0 UpdatedReplicas=1, retrying
...
[2023-01-10T20:14:16.330Z] waiting for resources type *v1beta1.MachineDeploymentList to be up and running
[2023-01-10T20:14:16.330Z] worker nodes are still being created for MachineDeployment 
...
[2023-01-10T20:14:47.042Z] waiting for resources type *v1beta1.MachineList to be up and running
[2023-01-10T20:14:47.042Z] unable to get the autoscaler deployment, maybe it is not exist
```

Waiting for addons packages (like antrea) to come online: 

```
[2023-01-10T20:14:47.042Z] waiting for addons core packages installation...
[2023-01-10T20:14:47.042Z] getting ClusterBootstrap object for cluster: tkg-vc-antrea
[2023-01-10T20:14:47.042Z] waiting for resource tkg-vc-antrea of type *v1alpha3.ClusterBootstrap to be up and running
```

Heres where we loop through all the packages:
```
[2023-01-10T20:14:47.042Z] getting package:kapp-controller.tanzu.vmware.com.0.41.5+vmware.1-tkg.1-rc.2 in namespace:default
[2023-01-10T20:14:47.042Z] getting package:antrea.tanzu.vmware.com.1.7.2+vmware.1-tkg.1-advanced-rc.2 in namespace:tkg-system
[2023-01-10T20:14:47.042Z] getting package:vsphere-csi.tanzu.vmware.com.2.6.2+vmware.2-tkg.1-rc.2 in namespace:tkg-system
[2023-01-10T20:14:47.042Z] getting package:vsphere-cpi.tanzu.vmware.com.1.24.3+vmware.1-tkg.1-rc.2 in namespace:tkg-system
[2023-01-10T20:14:47.042Z] waiting for package: 'tkg-vc-antrea-kapp-controller'
[2023-01-10T20:14:47.042Z] waiting for resource tkg-vc-antrea-kapp-controller of type *v1alpha1.PackageInstall to be up and running
[2023-01-10T20:14:47.042Z] successfully reconciled package: 'tkg-vc-antrea-kapp-controller' in namespace: 'default'
[2023-01-10T20:14:47.042Z] waiting for package: 'tkg-vc-antrea-antrea'
[2023-01-10T20:14:47.042Z] waiting for resource tkg-vc-antrea-antrea of type *v1alpha1.PackageInstall to be up and running
[2023-01-10T20:14:47.042Z] waiting for package: 'tkg-vc-antrea-vsphere-csi'
[2023-01-10T20:14:47.042Z] waiting for package: 'tkg-vc-antrea-vsphere-cpi'
[2023-01-10T20:14:47.042Z] waiting for resource tkg-vc-antrea-vsphere-cpi of type *v1alpha1.PackageInstall to be up and running
[2023-01-10T20:14:47.042Z] waiting for resource tkg-vc-antrea-vsphere-csi of type *v1alpha1.PackageInstall to be up and running
```

Now some of the packages start appearing as installed.  

```
[2023-01-10T20:14:47.042Z] successfully reconciled package: 'tkg-vc-antrea-vsphere-cpi' in namespace: 'tkg-system'
[2023-01-10T20:14:47.043Z] successfully reconciled package: 'tkg-vc-antrea-antrea' in namespace: 'tkg-system'
[2023-01-10T20:14:47.043Z] waiting for 'tkg-vc-antrea-vsphere-csi' Package to be installed, retrying
[2023-01-10T20:14:57.211Z] successfully reconciled package: 'tkg-vc-antrea-vsphere-csi' in namespace: 'tkg-system'

############## And finally, the cluster is up !!!!!!!!!!!!!!!!!!!!!!!

[2023-01-10T20:14:57.211Z] Workload cluster 'tkg-vc-antrea' created
```