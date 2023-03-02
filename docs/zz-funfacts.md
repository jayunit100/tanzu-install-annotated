# FUN_FACTS



## 2.1 (Glasgow)


- There is a mega function, called `InitRegion` in tanzu-framework that has all the logic for bootstrapping kind, making a mgmt cluster, and migrating to mgmt cluster.
- Reading Tanzu CLI logs means being able to interpolate back and forth between the `clusterctl` logs, and the `tanzu cli` logs
- The TKG Addons controller lives in `kind` and installs `antrea` is the thing that bootstraps mgmt clusters.
- We install `tkg-pkg` on both the kind cluster and the mgmt cluster.  It is a super-package - it has other packages in it.
	- Tanzu CLI after installing tkg-pkg, makes sure there's a PackageInstall object for everything in `tkg-pkg`.
	- This makes Glasgow different from Fuji: In Fuji, the entire kind cluster wasnt able to bootstrap all of its own packages, and neither was the ManagementCluster.
	- Thus, in Glasgow, ClusterResourceSets arent needed anymore
- Question: When Tanzu CLI installs `tkg-pkg` are there any other customizations needed ? 
- You'll see alot of `succesfully reconciled package` statements in both kind and mgmt bootstrapping.  Thats because both phases have the same logical flow:
	- Wait for CAPI to come up
	- Create PackageInstalls on the cluster
	- Wait for all packages to come up
- GPU on ClusterClass not yet supported, stay tuned for 2.1.1....

### Features

- Theres an IPAM controller which CAPV relies on via a provider interface.  It can give your nodes static IPs.  Its part of the CAPI api and CAPV implements it.
- Theres a kube-vip loadbalancer for workload clusters.
- Theres an `isolated-cluster` plugin which will bootstrap a harbor registry for you w/ containers you need.


## PRs welcome

If you know of anything interesting worth noting in the build logs for a tanzu installation, please let us know !
