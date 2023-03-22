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

The input for a node pool file is documented here: https://docs-staging.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/using-tkg-21/workload-clusters-pool.html#sample-config.
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








