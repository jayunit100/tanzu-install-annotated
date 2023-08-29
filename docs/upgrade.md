When thinking of TKG packages, break them down into 4 categories, like so

<img width="1261" alt="image" src="https://github.com/jayunit100/tanzu-install-annotated/assets/826111/f2217325-aa83-44cf-8fb1-867d242be665">

# How do upgrades work

Theres alot of boiler plate around Upgrades that you probably already can make up in your head, or chatGPT could just tell you
if you asked it.  But somethings need careful explanation.

## Upgrading TKG is all about...  Packages !

- First we upgrade packages
- Then we upgrade nodes

## How packages are structured in TKG

TKG has two types of packages that you should understand, management and core packages. The lifecycle of these
packages is different - 

*Management Packages*

Examples of management packages are things like ****

- Required only on management cluster.
- Contents may change after releases.
- No customizability

*Core Packages*

Examples of core packages are things like **antrea**, which will go on both WL and MGMT clusters.

- Required on both management and workload clusters.
- Networking section available for CNI options.


## Package breakdown

Since carvel and package management is so key to TKG, heres the breakdown:

MANAGEMENT PACKAGES
- tkg (TKG Meta Package)
- framework (Framework Meta Package)
- addons-manager
- tkr-service
- featuregates
- tanzu-auth
- cliplugins
- tkg-clusterclass (TKG ClusterClass Meta Package)
- tkg-clusterclass-aws
- tkg-clusterclass-azure
- tkg-clusterclass-vsphere
- core-management-plugins
- tkr-source-controller

**?Unused TBD why?**
- cluster-api
- cluster-api-bootstrap-kubeadm
- cluster-api-control-plane-kubeadm
- cluster-api-provider-aws
- cluster-api-provider-azure
- cluster-api-provider-vsphere

STANDALONE PACKAGES no dependencies:
- standalone-plugins
- capabilities
- tkg-autoscaler
- tkg-storageclass


## Bootstrap Controller

The TKG Bootstrap controller is responsible for coordinating a complex dance of "pausing" and "unpausing" clusers.  
First the bootstrap controller *notices* that a cluster is in a state where it is ABOUT to be upgraded.  

It notices this by recognizing that the desired TKR version doesnt match the ACTUAL TKR version.

<img width="1143" alt="image" src="https://github.com/jayunit100/tanzu-install-annotated/assets/826111/b11368b8-31bd-413f-bef0-d26732e8effc">

As an example in the image above... We uprade the CNI package which is compatible w/ 1.25 BEFORE we upgrade the K8s version on the node to 1.25. This happens by:

- bootstrap controller pausing the cluster, so CAPI wont do anything
- upgrading the packages for the CNI provider on all nodes
- unpausing the cluster, so that CAPI can finish upgrading nodes
- running a CAPI upgrade

This means that *upgrades* are safe, bc the packages of a node are gauranteed to work with its TARGET version of K8s when it is rebuilt.

- theres a theoretical possibility that the CNI might not work perfectly on the node before it is upgraded, b/c, for example, the calico version have a k8s client that is N+1 of the kube apiserver.
- theres NO possibility that the NEW node (post upgrade) will have a CNI that isnt working - thus the upgrade will not fail due to CNI packages being made for k8s n-1 not running on the new K8s version. 

(note these ideas are still under construciton..but something like...) 
## Runtime Extensions

The fact that a controller has to watch a TKG node, to "catch it in the act" of an upgrade is a little unelegant.  Soon there will be runtime extensions, which will not have
to annotate the state of a node or pause/unpause its reconcilation, and instead work at runtime to trigger events just exactly when needed.  For example:

- Upgrade requested.
- Runtime extension triggers a package upgrade.
- PAckages upgraded on node.
- Node is upgraded.
