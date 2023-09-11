# WL vs MGMT: Differences

WL cluster and mgmt cluster run different types of pods.  Heres a quick summary of the differences.  

## Things that ONLY run on management clusters

Management clusters manage all the auth and infra for workload clusters.  Thus some microservices not on WL clusters, are always explained in terms of "this isnt needed to run apps, but it is needed to manage clusters"

- CAPI Pods only run in management cluster, bc they are literally what manage the WL clusters.
- tanzu-auth / tanzu-auth-controller-manager: The management cluster does need to manage authentication to APIServers for ALL workload clusters... Thus this has to run on the management level.  `tanzu-auth-controller-manager` is the controller for **pinniped**, so it installs and manages our authentication wrappers in TKG.  Only mgmt cluster needs to manage this since it controls auth to many workload clusters.
- object-propagation-controller-manag: CAPI clusterclasses get copied to multiple namespaces.  This is something only done to support creation of new CAPI clusters, thus WL clusters dont run this . 
- tanzu-addons-controller-manager: Tanzu Addons manager will put carvel packages onto WL clusters. However, it is **kapp** on the wl cluster that installs them.  Thus, the WL cluster doesnt need to run addons bc management does this for it on bootstrap... the workload cluster **does though** notably, run **kapp controller** to install those packages ONCE addons manager puts them on there!
- cert-manager: Cert-manager pods are used to rotate CAPI containers so, they dont run on WL clusters.

Here's a table that shos all the common pods and where they run.

```
| Namespace               | Pod Type                            | tmgmt | wl | window|
|-------------------------|-------------------------------------|------------|-------|
| NAMESPACE               | NAME                                | ✓     | ✓  | ?     |
| avi-system              | ako                                 | ✓     | ✓  |       |
| caip-in-cluster-system  | caip-in-cluster-controller-manager  | ✓     |    |       |
| capi-kubeadm-btstrp-sys | capi-kubeadm-btstrp-ctrlmgr         | ✓     |    |       |
| capi-kubeadm-cp-system  | capi-kubeadm-cp-controller-manager  | ✓     |    |       |
| capv-system             | capv-controller-manager             | ✓     | ✓  |       |
| cert-manager            | cert-manager                        | ✓     |    |       |
| cert-manager            | cert-manager-cainjector             | ✓     |    |       |
| cert-manager            | cert-manager-webhook                | ✓     |    |       |       
| kube-system             | antrea-agent                        | ✓     | ✓  |  ✓    |
| kube-system             | antrea-controller                   | ✓     | ✓  |  ✓    |
| kube-system             | coredns                             | ✓     | ✓  |  ✓    |
| kube-system             | etcd                                | ✓     | ✓  | ✓     |   
| kube-system             | kube-apiserver                      | ✓     | ✓  | ✓     |
| kube-system             | kube-controller-manager             | ✓     | ✓  | ✓     |
| kube-system             | kube-proxy                          | ✓     | ✓  | ✓     |
| kube-system             | kube-scheduler                      | ✓     | ✓  | ✓     |
| kube-system             | metrics-server                      | ✓     | ✓  |  ✓    |
| kube-system             | vsphere-cloud-controller-manager    | ✓     | ✓  |  ✓    |
| secretgen-controller    | secretgen-controller                | ✓     | ✓  |  ✓    |
| tkg-system              | kapp-controller                     | ✓     | ✓  |   ✓   |
| tkg-system              | object-propagation-controller-manag | ✓     |    |       |
| tkg-system              | tanzu-addons-controller-manager     | ✓     |    |       |
| tkg-system              | tanzu-capabilities-controller-manag | ✓     | ✓  |  ✓    |
| tkg-system              | tanzu-featuregates-controller-manag | ✓     |    |       |  
| tkg-system              | tkr-conversion-webhook-manager      | ✓     |    |       |
| tkg-system              | tkr-resolver-cluster-webhook-manag  | ✓     |    |       |
| tkg-system              | tkr-source-controller-manager       | ✓     |    |       |
| tkg-system              | tkr-status-controller-manager       | ✓     |    |       |
| tkg-system              | tkr-vsphere-resolver-webhook-mana   | ✓     |    |       |
| vmware-system-antrea    | register-placeholder                | ✓     | ✓  |       |
| vmware-system-csi       | vsphere-csi-controller              | ✓     | ✓  |  ✓    | 
| vmware-system-csi       | vsphere-csi-node                    | ✓     | ✓  |  CP   |
```

# WL

Now lets see what happens when we bring up the workload cluster

```
[2023-01-10T20:09:55.303Z] INFO:root:    (\,*********'()'--o  [Verifying number of nodes and k8s [2023-01-10T20:09:55.585Z] INFO:root:    (\,*********'()'--o  [verify tkg-bom existing and not none]
[2023-01-10T20:09:55.585Z] INFO:root:    (\,*********'()'--o  [verify mgmt cluster's default k8s [2023-01-10T20:09:56.133Z] INFO:root:    (\,*********'()'--o  [Verifying anti affinity for management cluster]
[2023-01-10T20:09:56.133Z] INFO:root:    (\,*********'()'--o  [Verifying tkr/kapp configmap has correct configuration in http-proxy mode or normal iaas]
[2023-01-10T20:09:57.321Z] INFO:root:====== 254  CMD: tanzu cluster version
,,,
[2023-01-10T20:10:00.105Z] INFO:root:    (\,*********'()'--o  [CREATING A WORKLOAD CLUSTER tkg-vc-antrea with {'OS_VERSION': '20.04', 'OS_ARCH': 'amd64', 'OS_NAME': 'ubuntu', 'CNI': 'antrea', 'CLUSTER_PLAN': 'dev', 'KUBERNETES_VERSION': 'v1.24.9+vmware.1', 'INFRASTRUCTURE_PROVIDER': 'vsphere', 'AVI_CONTROL_PLANE_HA_PROVIDER': 'true'}]
```


We create our first workload cluster like this:

```
tanzu kubernetes cluster create \	
	tkg-vc-antrea -v 9 -f tkg-vc-antrea.yaml \
		--tkr v1.24.9---vmware.1-tkg.1-rc.2
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
`


