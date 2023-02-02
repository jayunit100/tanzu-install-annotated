# Webhooks



Looking at a TKG Cluster, we find a *topology* field....

```
  topology:
    class: tkg-vsphere-default-v1.0.0
    controlPlane:
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu
```

There are several key value pairs that are given to a ClusterClass cluster topology.  We can see that these are product specific: 

```
    - name: controlPlaneCertificateRotation
    - name: auditLogging
    - name: podSecurityStandard
    - name: apiServerEndpoint
    - name: aviAPIServerHAProvider
    - name: vcenter
    - name: user
    - name: controlPlane
    - name: etcdExtraArgs
    - name: worker
    - name: VSPHERE_WINDOWS_TEMPLATE
    - name: vipNetworkInterface
    - name: cni
    - name: network
    - name: kubeVipLoadBalancerProvider
    - name: nodePoolLabels
    - name: ntpServers
    - name: additionalFQDN
    - name: controlPlaneTaint
    - name: apiServerExtraArgs
    - name: kubeSchedulerExtraArgs
    - name: kubeControllerManagerExtraArgs
    - name: controlPlaneKubeletExtraArgs
    - name: workerKubeletExtraArgs
    - name: tlsCipherSuites
    - name: eventRateLimitConf
    - name: TKR_DATA
```

When a user creates a new TKG cluster, they dont memorize the path and the MOID of it the VMs they want CAPV to use.  Alternatively, they specify a few high level parameters (i.e. ubuntu, 2004, ...) and TKG figures out, for them... what exact OS Image (including the path  on vsphere) shoudl be sent to their underlying MachineDeployment and KubeAdmControlplane objects. 

Let's learn how the last field , `TKR_DATA` is populated with specific VSphere OVA paths.

## What is in TKR_DATA 

Inside of the TKR_DATA, we see version info for TKG, which can be used to know the specific version of antrea, etcd, kube-vip, and many other services
which a cluster might run.  Without this information, we cannot determine:
- What version of the CNI provider to run for this cluster.
- What version of the pause image this cluster is using.
- Wether this cluster is supported by a particular TKG version
And so on.  

So to ensure that ALL CLUSTERS always have this data, we have a webhook... The webhook PREVENTS a cluster object from being created until the TKR data has been
added to it. 

The TKR_DATA object in a clusters spec.topology fields looks like this: 
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
```
The above fields are relatively "easy" to resolve, because theyre just K8s objects... But the Vsphere information
with the specific OVA `<VERSION>` information is hidden under VSphere APIs.  Thus, we have a controller whose job is:
- To Query VSphere
- Find specific OVA Image template names
- Add information about those template names, and MOIDs, into the `osImageRef` object inside of the YAML definition for the `TKR_DATA` field:  
```
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
            moid: vm-51
            template: /dc0/vm/ubuntu-2004-kube-v1.24.9+vmware.1-tkg.1
            version: v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93
```
Looking just a few lines down in the same cluster object, we see that the `machineDeployment` definition, in fact, will reference the 
- image-type: ova
- os-name: ubuntu
fields:
```
            moid: vm-51
            template: /dc0/vm/ubuntu-2004-kube-v1.24.9+vmware.1-tkg.1
            version: v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93
    version: v1.24.9+vmware.1
    workers:
      machineDeployments:
      - class: tkg-worker
        metadata:
          annotations:
            # HERE !!!
            run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu
```

When new CAPI MachineDeployments are created.    


# Webhook

If you look in the TKG payload you'll see alot of webhooks.

These are installed as YAML files, and part of the management cluster setup payload.
To find these, look inside:
- .config/tanzu/tkg/providers
  - infrastructure-oci/...
  - infrastructure-azure/...
  - infrastructure-aws/...
  - cluster-api/...
  - cert-manager/...
  - bootstrap-kubeadm/... 

Also, you'll see other controllers with webhooks like antrea. 

```
./.config/tanzu/tkg/providers/infrastructure-oci/v0.4.0/infrastructure-components.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/cluster-api/v1.2.8/core-components.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/infrastructure-aws/v2.0.2/infrastructure-components.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/infrastructure-docker/v1.2.8/infrastructure-components.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/cert-manager/v1.7.2/cert-manager.yaml:  - mutatingwebhookconfigurations
./.config/tanzu/tkg/providers/cert-manager/v1.7.2/cert-manager.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/cert-manager/v1.9.1/cert-manager.yaml:    resources: ["validatingwebhookconfigurations", "mutatingwebhookconfigurations"]
./.config/tanzu/tkg/providers/cert-manager/v1.9.1/cert-manager.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/bootstrap-kubeadm/v1.2.8/bootstrap-components.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/infrastructure-vsphere/v1.5.1/infrastructure-components-supervisor.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/infrastructure-vsphere/v1.5.1/infrastructure-components.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/ytt/vendir/cni/_ytt_lib/addons/packages/antrea/1.7.2/bundle/config/upstream/antrea.yaml:      - mutatingwebhookconfigurations
./.config/tanzu/tkg/providers/ytt/vendir/cni/_ytt_lib/addons/packages/antrea/1.7.2/bundle/config/upstream/antrea.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/ytt/vendir/kapp-controller/_ytt_lib/addons/packages/kapp-controller/0.41.5/bundle/config/upstream/kapp-controller.yaml:  - mutatingwebhookconfigurations
./.config/tanzu/tkg/providers/infrastructure-ipam-in-cluster/v0.1.0/ipam-components.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/infrastructure-azure/v1.6.1/infrastructure-components.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/yttcc/vendir/cni/_ytt_lib/addons/packages/antrea/1.2.3/bundle/config/upstream/antrea.yaml:  - mutatingwebhookconfigurations
./.config/tanzu/tkg/providers/yttcc/vendir/cni/_ytt_lib/addons/packages/antrea/1.2.3/bundle/config/upstream/antrea.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/yttcc/vendir/cni/_ytt_lib/addons/packages/antrea/1.2.3/bundle/config/upstream/antrea.yaml:kind: MutatingWebhookConfiguration
./.config/tanzu/tkg/providers/yttcc/vendir/kapp-controller/_ytt_lib/addons/packages/kapp-controller/0.30.1/bundle/config/upstream/kapp-controller.yaml:  - mutatingwebhookconfigurations
./.config/tanzu/tkg/providers/control-plane-kubeadm/v1.2.8/control-plane-components.yaml:kind: MutatingWebhookConfiguration
```

## How do Webhooks get installed ? 

There are two componments to the typical webhook.
- The creation of the kubernetes webhook object for the APIServer to know about.
- The golang code (and yes, i know there are ways to write web servers w/o go but were trying to be concrete here) which binds to a REST endpoint.

We can see these two components in the tanzu-framework code base... 

So, who tells the APIServer to call your webhook, and when to call it?  You do.  When you make the MutatingWebhook YAML.  In other words:

- First you make a webhook description, with a dns name that points to a place your webhook code will run
- Then you write code that does something to an incoming object


```
packages/tkr-vsphere-resolver/bundle/config/upstream/webhook.yaml:      
   path: /resolve-template

tkg/vsphere-template-resolver/main.go:  
   hookServer.Register("/resolve-template", 
      &webhook.Admission{
```

## How does this relate to a Cluster object?


- You install a tanzu management cluster
- The tanzu cli, during installation, creates a K8s APIServer.
- Then the tanzu cli adds CAPI controllers to the APIServer, along with webhook definitions which point to those controllers, so that any `Cluster` objects being created can be processed by CAPI before they are stored in the Kubernetes APIServer.
  - one of these webhooks is the `tkr-vsphere-resolver` webhook definition.
- You run `tanzu cluster create... --tkr ..."`, making a new workload cluster. 
- A `Cluster` object is created for you by tanzu and sent to the APIServer as YAML.
- The APIServer sees that theres a Cluster object, and so immediately calls out to the `tkr-vsphere-resolver` Mutating Cluster Webhook to finish creating it.
- Webhook goes off and talkes to Vsphere, to query the `<VERSION>` values for all OS Templates
- It then writes this data into the TKR_DATA field 
- Api server recieves the "mutated" `Cluster` struct, now having fine grained OVA path info, and stores the `Cluster` object to `etcd`
- Now, the MachindDeployment and KubeadmControlPlane objects (which will get created by CAPI), will have well defined images, and your nodes will come up.

### What about other types of hooks ? 

If this feels a little too asnychronous for you, then never fear, for finer grained configuration of CAPI, there are new mechanisms being researched to "plug out" to other controllers for precise and immediate feedback while doing a complex infrasttructure installation. These are currently called *lifecycle hooks*.  But theyre not a discussion for today.

## A Mutating Webhook example

Anyways, back to our mutating webhook.  Let's look at it...  

```
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  annotations:
    ...
  generation: 2
  labels:
  name: tkr-vsphere-resolver-mutating-webhook-configuration
webhooks:
- admissionReviewVersions:
  - v1
  - v1beta1
  clientConfig:
```

The first thing we see, is a `service`.  This service corresponds to a namespace and a path: 

```
   service:
      name: tkr-vsphere-resolver-webhook-service
      namespace: tkg-system
      path: /resolve-template
      port: 443
```

This means that the APIServer can access "thing that mutates, and resolves vsphere tkr info" at  `https://my-k8s-api-server:443/tkg-system/tkr-vsphere-resolver-webhook-service/resolve-template`. 

Now, we ask *WHEN* will the APIServer actually call this API? Whenever a *cluster* is *CREATED* or *UPDATED*.  We can see this in the
*rules* section below:

```
  ...
  rules:
  - apiGroups:
    - cluster.x-k8s.io
    apiVersions:
    ...
    operations:
    - CREATE
    - UPDATE
    resources:
    - clusters
    scope: '*'
```

Finally a couple of minor details: The Kubernetes API Server will give up after 10 seconds.  That means that if the TKR resolving webhook service
takes more then 10 seconds to add the TKR_BOM field into an incoming cluster definition... The cluster creation will fail. 

```
  sideEffects: None
  timeoutSeconds: 10
```

### Wheres the code (TODO) 
 
Ok so weve really looked at the high level definition.  But what is the webhook DOING once this web service is called by the APIServer when you made your `Cluster`? 
 
The code for all of this is in tkg/vsphere-template-resolver/template/resolver.go, which lives in tanzu-framework. 
 
The function that is triggered, ultimately, which joins data from vsphere into the `Cluster` object, is shown here... 
```
 func (cw *Webhook) resolve(ctx context.Context, cluster *clusterv1.Cluster) (string, error) {
        topology := cluster.Spec.Topology
        ... 
        mdDatas, err := getMDDatas(cluster)
        if err != nil {
                return "", err
        }
        ovaTemplateQueries := collectOVATemplateQueries(append(mdDatas, cpData))
        ...
        // Find the OVAs template paths in vsphere that will be used to make new CAPV VMs
        result := cw.Resolver.Resolve(ctx, vSphereContext, query, vc)
        ... 
        // Write the data out to our OSImage structures 
        return "", cw.processAndSetResult(result, cluster, cpData, mdDatas)
}
```

## Conclusion
 
The MutatingWebhook pattern is used by TKG in many places.  One example, is how we mutating incoming Cluster definitions to reference specific VSphere OVA templates, including their precise MOID and path values, before a new CAPI/CAPV cluster is created.  This ensures that the capv-controller-manager always has a usable and correct VsphereMachineTemplate, which points to a well defined OS Template.
 
