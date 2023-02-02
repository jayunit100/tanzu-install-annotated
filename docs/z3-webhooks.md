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

Let's learn how the last field , TKR_DATA is added to a cluster... 

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
            ...
```

Webhooks are clearly a critical part of how TKG adds product specific logic into Cluster API.  So lets see how they work and how they are installed.

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

## How does this workin again ? 

At a high level, lets imagine what happens in TKG when you make a `Cluster` object at the API level (the tanzu cli normally makes this object for you, but in the end
it ultimately makes a `Cluster` object).  That said, in TKG you *can* make clusters using raw `kubectl`... so this example isnt totally contrived.

- You make a `Cluster`
- Webhook sees what you tried to do.
- Webhook adds the TKR data to your `Cluster`
- Api server adds the `Cluster` object to `etcd`

These are generic kubernetes constructs that allow people to dynamically validate
API calls, for example, by: 
- enforcing simple things like "all new Deployments have a similar naming convention", or 
- enforcing complex things, like "The TKR_BOM" field must be filled in when TKG Cluster installations happen.

### BUT There is another NEW kind of hook being proposed upstream: A LIFECYCLE hook.  
- This is essentially a fine grained, RPC like interaction
- cluster API can "call out" to other components when certain "things" are happening.

This hasnt been implemented yet, but it's worth knowing that, someday, there may be finer grained pluggability that will be built
into cluster API over time.

## A Mutating Webhook example

Lets take a mutating webhook.  In this case, we'll look at the `TKR-vsphere-resolver`.

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
This means that the APIServer can access "thing that mutates, and resolves vsphere tkr info" at 
`https://my-k8s-api-server:443/tkg-system/tkr-vsphere-resolver-webhook-service/resolve-template`. 

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
 
 ```
 
 ```
 
 
