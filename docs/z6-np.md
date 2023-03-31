# NP

nodepools !

## Getting Node Pools

A node pool, as we can see from get_node_pools.go (in https://github.com/vmware-tanzu/tanzu-framework/blob/main/cmd/cli/plugin/cluster/get_node_pools.go), is just a view
over existing CAPI MachineDeployments .
```
func listNodePoolsInternal(cmd *cobra.Command, server *configapi.Server, clusterName string) error {
	mdOptions := tkgclient.GetMachineDeploymentOptions{
		ClusterName: clusterName,
		Namespace:   lnp.namespace,
	}
	machineDeployments, err := tkgctlClient.GetMachineDeployments(mdOptions)
  ...
  for _, md := range machineDeployments
			t.AddRow(md.Name, md.Namespace, md.Status.Phase, md.Status.Replicas, md.Status.ReadyReplicas, md.Status.UpdatedReplicas, md.Status.UnavailableReplicas)
```

## Setting Node Pools 

The Tanzu Framework codebase defines 3 inputs to a node pool.
```
type clusterSetNodePoolCmdOptions struct {
	FilePath              string
	Namespace             string
	BaseMachineDeployment string
}
```

You thus create a node pool by:
- defining its namespace
- defining its parent MachineDeployment
- defining a FilePath with its customizations

## How Node pool files are interpretted

The node pool input value is a golang struct.  There are two layers to it:

The first layer is infrastructure independent:
```
// NodePool a struct describing a node pool
type NodePool struct {
	Name                  string                    `yaml:"name"`
	Replicas              *int32                    `yaml:"replicas,omitempty"`
	AZ                    string                    `yaml:"az,omitempty"`
	NodeMachineType       string                    `yaml:"nodeMachineType,omitempty"`
	WorkerClass           string                    `yaml:"workerClass,omitempty"`
	Labels                *map[string]string        `yaml:"labels,omitempty"`
	VSphere               VSphereNodePool           `yaml:"vsphere,omitempty"`
	Taints                *[]corev1.Taint           `yaml:"taints,omitempty"`
	VMClass               string                    `yaml:"vmClass,omitempty"`
	StorageClass          string                    `yaml:"storageClass,omitempty"`
	TKRResolver           string                    `yaml:"tkrResolver,omitempty"`
	Volumes               *[]tkgsv1alpha2.Volume    `yaml:"volumes,omitempty"`
	TKR                   tkgsv1alpha2.TKRReference `yaml:"tkr,omitempty"`
	NodeDrainTimeout      *metav1.Duration          `yaml:"nodeDrainTimeout,omitempty"`
	BaseMachineDeployment string                    `yaml:"baseMachineDeployment,omitempty"`
}
```
The second layer is specific to your cloud: Vsphere, AWS, Azure, etc...

```
// VSphereNodePool a struct describing properties necessary for a node pool on vSphere
type VSphereNodePool struct {
	CloneMode         string   `yaml:"cloneMode,omitempty"`
	Datacenter        string   `yaml:"datacenter,omitempty"`
	Datastore         string   `yaml:"datastore,omitempty"`
	StoragePolicyName string   `yaml:"storagePolicyName,omitempty"`
	Folder            string   `yaml:"folder,omitempty"`
	Network           string   `yaml:"network,omitempty"`
	Nameservers       []string `yaml:"nameservers,omitempty"`
	TKGIPFamily       string   `yaml:"tkgIPFamily,omitempty"`
	ResourcePool      string   `yaml:"resourcePool,omitempty"`
	VCIP              string   `yaml:"vcIP,omitempty"`
	Template          string   `yaml:"template,omitempty"`
	MemoryMiB         int64    `yaml:"memoryMiB,omitempty"`
	DiskGiB           int32    `yaml:"diskGiB,omitempty"`
	NumCPUs           int32    `yaml:"numCPUs,omitempty"`
	       []string `yaml:"nameservers,omitempty"`

}
```

## The input for a node pool file

The input for a node pool file is documented here: 

**https://docs-staging.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/using-tkg-21/workload-clusters-pool.html#sample-config**

You can see in this file that the schema is something like

```
name: <-- first layer 
replica:  <-- first layer
labels:
  ...
vsphere:
  ... <-- second layer of infra specific parameters
```

# Example: Evolving VsphereNodePool to support GPUs

Every time new data is added to a VsphereMachineTemplate, we need to update the code in tanzu framework to have coverage of the new types of inputs we provide to
the underlying machine tempaltes.  A good example of this is PCI Passthrough, wherein these parameters.  Specifically, lets look at how GPUs work.

## What changes in a PCI/GPU enabled VSphereMachineTemplate ? 

The following fields are modified when making new VSphere machines that allow GPU workloads:

```
1) /spec/template/spec/hardwareVersion
vmx-17

2) /spec/template/spec/customVMXKeys:
- pciPassthru.allowP2P:true
  pciPassthru.RelaxACSforP2P:true
  pciPassthru.use64bitMMIO:true
  pciPassthru.64bitMMIOSizeGB:512

3) /spec/template/spec/pciDevices
- deviceId: 0x10DE  <-- this defines "nvidia"
  vendorId: 0x1EB8 <-- this identifies that it is a "T4 GPU"
- deviceId: ...
  vendorId: ... 
```

In otherwords, the three fields: hardwareVersion, customVMXKeys, and pciDevices, all must be modified
when making a GPU compatible VSphereVM.  And thus, we will add these keys in future versions of TKG
to support the creation of node pools, on the fly, which are able to inject vGPUs.

0) CAPV ensures that tanzu cli boots your VM in mode vmx 17, by setting the 
```
 spec.template.spec.hardwareVersion=vmx-17.
```
1) CAPV then launches a VM with VMX Keys which are sent to your device (you send this in as a string) to tanzu CLI.
```
/spec/template/spec/customVMXKeys

- pciPassthru.allowP2P:true
  pciPassthru.RelaxACSforP2P:true
  pciPassthru.use64bitMMIO:true
  pciPassthru.64bitMMIOSizeGB:512
```
2) VSphere then boots VM which has a PCI card attached to it, with PCI settings that allow it to work in a GPU context.
3) You then install the nvidia gpu driver pods
4) The PC card is mounted into the nvidia driver pod as a file
5) NVIDIA driver pod then runs a C program that scans PCI devices .  This C Driver has the ability
to read and parse PCI devices (using nvml.h)
6) It then tells the kubelet via GRPC
```
service Registration {
	rpc Register(RegisterRequest) returns (Empty) {}
}
```
about this new device
7) The Kubelet then adds metadata to the nodes APIServer object.
8) The scheduler can now allocate GPUs, by querying the node.status "nvidia.com/gpu" field:

```
apiVersion: v1
kind: Node
metadata:
  name: gpu-node-1
  labels:
    kubernetes.io/hostname: gpu-node-1
spec:
  taints:
  - effect: NoSchedule
    key: nvidia.com/gpu
    value: "true"
status:
  capacity:
    cpu: "8"
    memory: 32100Mi
    nvidia.com/gpu: "1" <-- this is what is scheduled to the pods
```

Note that above, we have exactly 1 gpu, because with PCI Passthrough, you only get one
schedulable GPU card per device.


# An example of how to make a cluster with two, node pools


When making a new TKG Cluster, you can define a 2nd machinedeployment, as shown below... 

```
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  annotations:
    osInfo: ubuntu,20.04,amd64
    tkg/plan: dev
  labels:
    tkg.tanzu.vmware.com/cluster-name: l02-tkg-wld-gpu
  name: l02-tkg-wld-gpu
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 100.96.0.0/11
    services:
      cidrBlocks:
      - 100.64.0.0/13
  topology:
    class: tkg-vsphere-default-v1.0.0
    controlPlane:
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu
      replicas: 1
    variables:
    - name: cni
      value: antrea
    - name: controlPlaneCertificateRotation
      value:
        activate: true
        daysBefore: 90
    - name: auditLogging
      value:
        enabled: false
    - name: trust
      value:
        additionalTrustedCAs:
        - data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURkekNDQWwrZ0F3SUJBZ0lRSDR3TmRUaEtDSnBPdjhLMUx6UDdzREFOQmdrcWhraUc5dzBCQVFzRkFEQk8KTVJVd0V3WUtDWkltaVpQeUxHUUJHUllGYkc5allXd3hGekFWQmdvSmtpYUprL0lzWkFFWkZnZDBaWEpoYzJ0NQpNUnd3R2dZRFZRUURFeE4wWlhKaGMydDVMVXhCUWkxQlJEQXhMVU5CTUI0WERURTRNVEl5TURFME5UVTBORm9YCkRUTXpNVEl5TURFMU1EVTBNMW93VGpFVk1CTUdDZ21TSm9tVDhpeGtBUmtXQld4dlkyRnNNUmN3RlFZS0NaSW0KaVpQeUxHUUJHUllIZEdWeVlYTnJlVEVjTUJvR0ExVUVBeE1UZEdWeVlYTnJlUzFNUVVJdFFVUXdNUzFEUVRDQwpBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUxWWHAwUlhyT09DZmRVZElUNmF1aDU0CmFTNXN2STNPVml0VGVmUFFiRTQxd0U0Y1FRNll6SDB2cnQ3QjZscnlMSFF0L0VROGxVVTNQTEdEOU4rT25rWWwKa2tKcmZTZ2FTMHlLU3htaXJkaFlRNHZ6Z3psL2hyRXMxZkFQWVo2NkUra3lBc29aQTRsQnZrR0wxNFZ3MVNBMAo5TkV3eTlqOTZsOU9WdFlQcDV4R1c0SWJGZHdLMk96bW9SWFFGRmdBd3JlQkdOS2l0M3BJZkRPby82bWZxZTVXCnFYNUNZbGNVOXJjR3VzWnBoc0U0WTVkS1FRelF2dFMwOUxnSEZlNjA2Wm5QMXUyc1FUdFp2VzdUSzBDYW81ZnMKUkF4T0NmS1ZzZ1Z1SVVNbUhKd0lRK2x4MEdQS3ppVWx6K2F0Qm05Z09UWXBjZmQzcFlQMUJPQnBOTzROUTk4QwpBd0VBQWFOUk1FOHdDd1lEVlIwUEJBUURBZ0dHTUE4R0ExVWRFd0VCL3dRRk1BTUJBZjh3SFFZRFZSME9CQllFCkZISE4rVlp6L1hGR0JJZEZmOEV6UnYyNnd1WFFNQkFHQ1NzR0FRUUJnamNWQVFRREFnRUFNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFCS2pFQlJRSERzY01aZnBhVEJmRDhLcjBMU2dpbnJzU0JNWmtiUHdMMmt1bTlPdVM5RgpzVVZQOEd4OVc3dDBJdzMveGFHN01qL1ZOUDVHSDVVNFAzUW56dXRRMUo5L1Vsb1hrRThlZHlqYlEvUzVLeFU4CjBERFBZSXk2aTdBR09nR3RuYm4zbDZabEtXd0R5cGQzVFcwVFEwVVFObTlqNFhuTm9vM0xwbTFpV3NKVFRCMGwKbFdNZVNxMzZOeHAyTzlZcnh6amhxdnRLTmZKNjAzY0gycVlMK1UvZ0V4OEp1VGVLdEpQOTI3bnNqSlUzbGltegpYSzFjdndxQlpSbEFXcGd0RGxRSWw4U3B1Nnc2TDJBaWdVdHg0ZjhZMFFWV29EWEw3dmxXN1JxSlQ0THpCUlRzCkl5MUdGVjlxdnF3NFdlTWs3TXl2QWRrM3d3Vy9nQ01RY1BzSAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
          name: proxy
    - name: podSecurityStandard
      value:
        audit: baseline
        deactivated: false
        warn: baseline
    - name: apiServerEndpoint
      value: ""
    - name: aviAPIServerHAProvider
      value: true
    - name: vcenter
      value:
        cloneMode: fullClone
        datacenter: /Main
        datastore: /Main/datastore/LAB-V3-vSANDatastore
        folder: /Main/vm/LABS/itay/l02/tkg/workload-clusters
        network: /Main/network/itay-k8s-nodes
        resourcePool: /Main/host/LAB-V3/Resources/US
        server: ts-vc-01.terasky.local
        storagePolicyID: k8s-storage-policy-vsan
        template: /Main/vm/LABS/itay/l01/tkg/templates/ubuntu-2004-efi-kube-v1.24.10+vmware.1
        tlsThumbprint: ""
    - name: user
      value:
        sshAuthorizedKeys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC6TPWbdTQQGQzypvuhYCdUK0ZEnjgaCi0ilQbgHlgiicAFG6Nlw5NqBi7UtYm5fFurzQ4sNHl5ysgQM9lIODHt/RsdL0hZFjxpnQGRpHZU856s8DKbGuv7Sm/7M8E7oQxqWqzlhwauddFCI+wy6jVxAZhFpFraM5kcP7bPNUnRwb70hatxgeDvHrDjwO/qPIu6i9E5bGwCMJ8dvOS3Ujv//YuLh2IRecPZoFqLF98nxXOHCX38lPikEtzd7sLf3t+rRYOMlJVgkBz5URbM0VWvHgeXSvBiGogxVrDD8PLDfUPJ01v4WoC+hxkq0F8YwfQcCNi4vYzZKEDgauzs+TWbG0jnZ2SSw3vIgUPsgn+W+8PBrQne5YaNRUzpGNhk9VkhTPsPhSuis6B1sxpJ+m5jZdNw1LxkEKclslN5jbFfBviSuLMNi7jbzfFJeGJxAqyMxlblGyxnt/FlqSkmS5Owjsqz1UNtJFfoqhz73VApLzaG379CjX3Z2TX2zMgWjec=
    - name: controlPlane
      value:
        machine:
          diskGiB: 50
          memoryMiB: 8192
          numCPUs: 2
    - name: worker
      value:
        count: 1
        machine:
          diskGiB: 300
          memoryMiB: 16384
          numCPUs: 4
    - name: pci #### <--- variables.overrides will override this value below... 
      value: {}
    version: v1.24.10+vmware.1-tkg.2
    workers:
      machineDeployments:
      - class: tkg-worker ### <-- note this tkg-worker is reused in the 2nd node pool 
        metadata:
          annotations:
            run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu
        name: md-0
        replicas: 1
      - class: tkg-worker ### <-- reusing the tkg-worker , we will pretend to customize these nodes to support GPUs
        metadata:
          annotations:
            run.tanzu.vmware.com/resolve-os-image: image-type=ova,os-name=ubuntu
        name: md-1-gpu
        replicas: 1
	### Note that the below in tkg 2.1.1 must be created via kubectl create -f , rather then tanzu create -f...
        variables:
          overrides:
          - name: worker
            value:
              count: 1
              machine:
                customVMXKeys:
                  pciPassthru.64bitMMIOSizeGB: "16" 
                  pciPassthru.RelaxACSforP2P: "true"
                  pciPassthru.allowP2P: "true"
                  pciPassthru.use64bitMMIO: "true"
                diskGiB: 300
                memoryMiB: 16384
                numCPUs: 4
          - name: pci ########## <--- override this parameter w/ your PCI settings for this node pool 
	    value:
	      worker:
	        devices:
		  deviceId: 7864
		  workerId: 4318
              hardwareVersion: vmx-17
```

Note that in the below machineDeployments, we created 2 node pools, and in the second we override:
- worker
- pci
Inside of the vspheremachinetemplate for the GPU node pool.
This correlated to the pci variable we added above in our cluster variables
```
    - name: pci #### <--- variables.overrides will override this value below... 
      value: {}
```
You don't necessarily need to add EVERYTHING you want to override to the cluster variables, but
pci parameters (for some reason) are one such variable that DO need to be explicitly given a placeholder.



# What about TKGS ? 

In TKGS, the TanzuKubernetesCluster has a nodepool object that is explicitly defined with regard to a vmClass.

```
    nodePools:
    - name: string 
      labels: map[string]string
      taints:
        -  key: string
           value: string
           effect: string
           timeAdded: time
      replicas: int32
      vmClass: string
      storageClass: string
      volumes:
        - name: string
          mountPath: string
          capacity:
            storage: size in GiB
      tkr:  
        reference:
          name: string
      nodeDrainTimeout: string
```







