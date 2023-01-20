# (Kind) Addons

These logs show us what the (kind) addons controller manager does when we run initialization of a management cluster is coming up.

```
0120 14:41:09.302300       1 logr.go:261] setup "msg"="error retrieving GroupVersion" "GroupVersion"="run.tanzu.vmware.com/v1alpha1"                                                                                                    [31/447]
I0120 14:41:14.303926       1 logr.go:261] setup "msg"="error retrieving GroupVersion" "GroupVersion"="run.tanzu.vmware.com/v1alpha1"
I0120 14:41:19.300236       1 logr.go:261] setup "msg"="error retrieving GroupVersion" "GroupVersion"="run.tanzu.vmware.com/v1alpha1"
I0120 14:41:24.300668       1 logr.go:261] setup "msg"="error retrieving GroupVersion" "GroupVersion"="run.tanzu.vmware.com/v1alpha1"
I0120 14:41:30.431939       1 request.go:682] Waited for 1.037310558s due to client-side throttling, not priority and fairness, request: GET:https://100.64.0.1:443/apis/addons.cluster.x-k8s.io/v1beta1?timeout=32s
I0120 14:41:31.135629       1 logr.go:261] controller-runtime/metrics "msg"="Metrics server is starting to listen" "addr"="localhost:18317"
I0120 14:41:31.141085       1 azurediskcsiconfig_controller.go:162]  "msg"="SetupWithManager azureDiskcsicontroller start"
I0120 14:41:36.737729       1 logr.go:261] controller-runtime/builder "msg"="skip registering a mutating webhook, object does not implement admission.Defaulter or WithDefaulter wasn't called" "GVK"={"Group":"cni.tanzu.vmware.com","Version":"
v1alpha1","Kind":"AntreaConfig"}
I0120 14:41:36.737787       1 logr.go:261] controller-runtime/builder "msg"="Registering a validating webhook" "GVK"={"Group":"cni.tanzu.vmware.com","Version":"v1alpha1","Kind":"AntreaConfig"} "path"="/validate-cni-tanzu-vmware-com-v1alpha1-
antreaconfig"
I0120 14:41:36.739366       1 server.go:148] controller-runtime/webhook "msg"="Registering webhook" "path"="/validate-cni-tanzu-vmware-com-v1alpha1-antreaconfig"
I0120 14:41:36.739867       1 logr.go:261] controller-runtime/builder "msg"="skip registering a mutating webhook, object does not implement admission.Defaulter or WithDefaulter wasn't called" "GVK"={"Group":"cni.tanzu.vmware.com","Version":"
v1alpha1","Kind":"CalicoConfig"}
I0120 14:41:36.740028       1 logr.go:261] controller-runtime/builder "msg"="Registering a validating webhook" "GVK"={"Group":"cni.tanzu.vmware.com","Version":"v1alpha1","Kind":"CalicoConfig"} "path"="/validate-cni-tanzu-vmware-com-v1alpha1-
calicoconfig"
I0120 14:41:36.741582       1 server.go:148] controller-runtime/webhook "msg"="Registering webhook" "path"="/validate-cni-tanzu-vmware-com-v1alpha1-calicoconfig"
I0120 14:41:38.566622       1 logr.go:261] controller-runtime/builder "msg"="Registering a mutating webhook" "GVK"={"Group":"run.tanzu.vmware.com","Version":"v1alpha3","Kind":"ClusterBootstrap"} "path"="/mutate-run-tanzu-vmware-com-v1alpha3-
clusterbootstrap"
I0120 14:41:38.567283       1 server.go:148] controller-runtime/webhook "msg"="Registering webhook" "path"="/mutate-run-tanzu-vmware-com-v1alpha3-clusterbootstrap"
I0120 14:41:38.569904       1 logr.go:261] controller-runtime/builder "msg"="Registering a validating webhook" "GVK"={"Group":"run.tanzu.vmware.com","Version":"v1alpha3","Kind":"ClusterBootstrap"} "path"="/validate-run-tanzu-vmware-com-v1alp
ha3-clusterbootstrap"
I0120 14:41:38.572994       1 server.go:148] controller-runtime/webhook "msg"="Registering webhook" "path"="/validate-run-tanzu-vmware-com-v1alpha3-clusterbootstrap"
I0120 14:41:38.573944       1 logr.go:261] controller-runtime/builder "msg"="skip registering a mutating webhook, object does not implement admission.Defaulter or WithDefaulter wasn't called" "GVK"={"Group":"run.tanzu.vmware.com","Version":"
v1alpha3","Kind":"ClusterBootstrapTemplate"}
I0120 14:41:38.574115       1 logr.go:261] controller-runtime/builder "msg"="Registering a validating webhook" "GVK"={"Group":"run.tanzu.vmware.com","Version":"v1alpha3","Kind":"ClusterBootstrapTemplate"} "path"="/validate-run-tanzu-vmware-c
om-v1alpha3-clusterbootstraptemplate"
I0120 14:41:38.575324       1 server.go:148] controller-runtime/webhook "msg"="Registering webhook" "path"="/validate-run-tanzu-vmware-com-v1alpha3-clusterbootstraptemplate"
I0120 14:41:38.575620       1 logr.go:261] controller-runtime/builder "msg"="Registering a mutating webhook" "GVK"={"Group":"cluster.x-k8s.io","Version":"v1beta1","Kind":"Cluster"} "path"="/mutate-cluster-x-k8s-io-v1beta1-cluster"
I0120 14:41:38.577014       1 server.go:148] controller-runtime/webhook "msg"="Registering webhook" "path"="/mutate-cluster-x-k8s-io-v1beta1-cluster"
I0120 14:41:38.577737       1 logr.go:261] controller-runtime/builder "msg"="skip registering a validating webhook, object dtroller"="cluster" "controllerGroup"="cluster.x-k8s.io" "controllerKind"="Cluster" "source"="kind source: *v1.Secret"
I0120 14:41:40.391445       1 controller.go:185]  "msg"="Starting EventSource" "controller"="cluster" "controllerGroup"="cluster.x-k8s.io" "controllerKind"="Cluster" "source"="kind source: *v1alpha1.TanzuKubernetesRelease"
I0120 14:41:40.391641       1 controller.go:185]  "msg"="Starting EventSource" "controller"="cluster" "controllerGroup"="cluster.x-k8s.io" "controllerKind"="Cluster" "source"="kind source: *v1.ConfigMap"
I0120 14:41:40.391669       1 controller.go:185]  "msg"="Starting EventSource" "controller"="cluster" "controllerGroup"="cluster.x-k8s.io" "controllerKind"="Cluster" "source"="kind source: *v1beta1.KubeadmControlPlane"
I0120 14:41:40.391683       1 controller.go:193]  "msg"="Starting Controller" "controller"="cluster" "controllerGroup"="cluster.x-k8s.io" "controllerKind"="Cluster"
I0120 14:41:40.395407       1 controller.go:185]  "msg"="Starting EventSource" "controller"="antreaconfig" "controllerGroup"="cni.tanzu.vmware.com" "controllerKind"="AntreaConfig" "source"="kind source: *vrollerKind"="CalicoConfig"
I0120 14:41:40.400516       1 controller.go:185]  "msg"="Starting EventSource" "controller"="vspherecpiconfig" "controllerGroup"="cpi.tanzu.vmware.com" "controlle.go:193]  "msg"="Starting Controller" "controller"="kappcontrollerconfig" "controllerGroup"="run.tanzu.vmware.com" "controllerKind"="KappControllerConfig"
I0120 14:41:40.405646       1 controller.go:185]  "msg"="Starting EventSource" "controller"="vsph120 14:41:40.409699       1 controller.go:185]  "msg"="Starting EventSource" "controller"="oraclecpiconfig" "controllerGroup"="cpi.tanzu.vmware.com" "controllerKind"="OracleCPIConfig" "source"="kind source: *v1beta1.Cluster"
I0120 14:41:40.409818       1 controller.go:193]  "msg"="Starting Controller" "controller"="oraclecpiconfig" "controllerGroup"="cpi.tanzu.vmware.com" "controllerKind"="OracleCPIConfig"
I0120 14:41:40.410043       1 controller.go:185]  "msg"="Starting EventSource" "controller"="azurefilecsiconfig" "controllerGroup"="csi.tanzu.vmware.com" "controllerKind"="AzureFileCSIConfig" "source"="kind source: *v1alpha1.AzureFileCSIConf
ig"
I0120 14:41:40.410384       1 controlr"="azurediskcsiconfig" "controllerGroup"="csi.tanzu.vmware.com" "controllerKind"="AzureDiskCSIConfig"
I0120 14:41:40.419582       1 logr.go:261] controller-runtime/certwatcher "msg"="Updat41:40.424565       1 controller.go:185]  "msg"="Starting EventSource" "controller"="cluster" "controllerGroup"="cluster.x-k8s.io" "controllerKind"="Cluster" "source"="kind source: *v1alpha3.TanzuKubernetesRelease"
```

## Kind is bootstrapping stuff for us

One thing to note is that... after a Management Cluster is up. It appears it installs antrea for us.  This means that,  the addons controller manager
in our kind cluster is in fact responsible for bootstrapping the management clusters CNI... 

```

I0120 14:45:19.118970       1 cluster_metadata_controller.go:107] ClusterMetadataReconciler "msg"="Reconciling cluster" "cluster-name"="tkg-mgmt-vc" "cluster-ns"="tkg-system"                                                                   E0120 14:45:19.173384       1 controller.go:326]  "msg"="Reconciler error" "error"="cannot get the bom configuration: ConfigMap \"tkg-bom-v2.1.0-rc.3\" not found" "cluster"={"name":"tkg-mgmt-vc","namespace":"tkg-system"} "controller"="cluste
rMetadata-controller" "controllerGroup"="cluster.x-k8s.io" "controllerKind"="Cluster" "name"="tkg-mgmt-vc" "namespace"="tkg-system" "reconcileID"="c11fe81e-29f9-4f61-afec-52c19831a087"

I0120 14:45:23.644028       1 clusterbootstrap_controller.go:170] ClusterBootstrapController "msg"="Reconciling cluster" "cluster-name"="tkg-mgmt-vc" "cluster-ns"="tkg-system"
```

### Here is where antrea is reconciled... 

```
I0120 14:45:23.675437       1 logr.go:261] antreaconfig-resource "msg"="validate update" "name"="tkg-mgmt-vc-antrea-package"
I0120 14:45:23.709431       1 clusterbootstrap_controller.go:1377] ClusterBootstrapController "msg"="setting proxy and network configurations in Cluster annotation" "cluster-name"="tkg-mgmt-vc" "cluster-ns"="tkg-system" "tkg.tanzu.vmware.com

/skip-tls-verify"="" "tkg.tanzu.vmware.com/tkg-http-proxy"="" "tkg.tanzu.vmware.com/tkg-https-proxy"="" "tkg.tanzu.vmware.com/tkg-ip-family"="" "tkg.tanzu.vmware.com/tkg-no-proxy"=null "tkg.tanzu.vmware.com/tkg-proxy-ca-cert"=""
```


