# WIN-imgbld

What is a TKG windows cluster?

It looks something like this.... 

<img width="1094" alt="image" src="https://github.com/jayunit100/tanzu-install-annotated/assets/826111/b4b21f89-13db-4f70-9e2e-4718932270b1">

- C:/Program Files/ .... is where containerd will live
- C:/k/... is where antrea and the kubernetes binaries live
- Image builder sets up the initial nodes disk topology for these items, via the **windows resource kit** or **burrito**

<img width="1386" alt="image" src="https://github.com/jayunit100/tanzu-install-annotated/assets/826111/68219ea8-39b6-42b4-aed7-c7648c78a0ca">

To understand image builder, and how it related to TKG, you have to understand TKRs.

- TKRs reference OSImages
- OSImages reference OVA details, including <Version> field and template (sometimes)
- The value in VSphereMachineTemplates of the underlying OVAs location that will be created from OsImage metadata

Now, lets see how we build images that we plugin into the `OsImage -> TKR` workflow. See the *TKR* page if you want to learn
more about the TKR stuff.

# Lets make windows clusters

This article was written on TKG 2.3.0... It looks at what image-builder does when making windows nodes.
Note you can also use image builder to customize Linux nodes, and TKG itself uses image-builder
to generate its golden image that you get from customer connect.

However the most common requirement for image-builder usage from an end user perspective, is when making Windows images, 
bc its illegal to distribute windows OVAs - thus we don't ship them in customer connect, but rather, we 
give users directions on how to make their own Kubnernetes OVAs which can be used to create windows worker nodes for
running legacy containerized windows workloads.

# Windows Clusters 
  
Installing a TKG Windows cluster is a good way to learn about TKRs, image-builder, and so on.
Let's go.

## Links

- 2.2: https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-byoi-windows.html 
- 2.3: https://docs-staging.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.3/using-tkg/workload-clusters-advanced-vsphere.html#windows
- Extras: https://github.com/jaimegag/tkg-zone.git 

(note these instructions include developer build steps so they may not be 100% reproducible for you)

## Cloudbase init

The `image-builder` repository (https://github.com/kubernetes-sigs/image-builder/) Defines a make target:

```
images/capi/Makefile:   $(if $(findstring windows,$@),
packer build $(PACKER_WINDOWS_NODE_FLAGS)
-var-file="packer/ova/packer-common.json"
-var-file="$(abspath packer/ova/$(subst build-node-ova-vsphere-,,$@).json)"
-only=file $(ABSOLUTE_PACKER_VAR_FILES)

packer/ova/packer-windows.json,)
```

The `packer-windows.json` (https://github.com/kubernetes-sigs/image-builder/blob/main/images/capi/packer/ova/packer-windows.json) file is , the input to the `packer build` step , which builds a windows OVA:

## packer windwow json

This file has multiple "builders"...

Four types of provisioners are used:

- Ansible provisioner for running the node_windows.yml playbook with specific extra arguments.
- Windows-restart provisioner to restart the Windows system.
- Goss provisioner for running GOSS tests with a specific configuration. This could potentially include Kubernetes-related tests depending on what's defined in the user's goss_tests_dir.


```
{
  "builders": [
    {
      "content": "{\n \"unattend_timezone\" : \"{{user `unattend_timezone`}}\"\n}",
      "target": "./packer_cache/unattend.json",
      "type": "file"
    },
    {
      "boot_wait": "{{user `boot_wait`}}",
      "communicator": "winrm",
      "cpus": "{{user `cpu`}}",
      "disk_adapter_type": "scsi",
      "disk_size": "{{user `disk_size`}}",
      "disk_type_id": "{{user `disk_type_id`}}",
      "floppy_dirs": [
        "./packer/ova/windows/pvscsi"
      ],
      "floppy_files": [
      ],
      "guest_os_type": "{{user `local_guest_os_type`}}",
      "http_port_max": "{{user `http_port_max`}}",
      "http_port_min": "{{user `http_port_min`}}",
      "iso_checksum": "{{user `iso_checksum` }}",
      "iso_urls": [
        "{{user `os_iso_url`}}"
      ],
      "memory": "{{user `memory`}}",
      "name": "vmware-iso",
      "output_directory": "{{user `output_dir`}}",
      "shutdown_command": "powershell A:/sysprep.ps1",
      "shutdown_timeout": "1h",
      "type": "vmware-iso",
      "version": "{{user `vmx_version`}}",
      "vm_name": "{{user `build_version`}}",
      "vmdk_name": "{{user `build_version`}}",
      "vmx_data": {
        "numvcpus": "4",
        "scsi0.virtualDev": "pvscsi"
      },
      "winrm_password": "S3cr3t0!",
      "winrm_timeout": "4h",
      "winrm_username": "Administrator"
    },
    {
      "CPUs": "{{user `cpu`}}",
      "RAM": "{{user `memory`}}",
      ...
      "disk_controller_type": "{{user `disk_controller_type`}}",
      "export": {
        "force": true,
        "output_directory": "{{user `output_dir`}}"
      },
      "firmware": "bios",
      "floppy_dirs": [
        "./packer/ova/windows/pvscsi"
      ],
```
we next see that the "floppy_files" sectino has all the stuff you will make as inputs to a base windwos machine that
an IT dept might setup, for example, instructions on autoattend or winrm configuration. 
```
      "floppy_files": [
        "./packer/ova/windows/{{user `build_name`}}/autounattend.xml",
        "./packer/ova/windows/disable-network-discovery.cmd",
        "./packer/ova/windows/disable-winrm.ps1",
        "./packer/ova/windows/enable-winrm.ps1",
        "./packer/ova/windows/sysprep.ps1"
      ],
      "folder": "{{user `folder`}}",
      "guest_os_type": "{{user `vsphere_guest_os_type`}}",
      "host": "{{user `host`}}",
      "http_port_max": "{{user `http_port_max`}}",
      "http_port_min": "{{user `http_port_min`}}",
      "insecure_connection": "{{user `insecure_connection`}}",
      "iso_paths": [
        "{{user `os_iso_path`}}",
        "{{user `vmtools_iso_path`}}"
      ],
      "name": "vsphere",
      "network_adapters": [
        {
          "network": "{{user `network`}}",
          "network_card": "{{user `network_card`}}"
        }
      ],
      "password": "{{user `password`}}",
      "resource_pool": "{{user `resource_pool`}}",
      "shutdown_command": "powershell A:/sysprep.ps1",
      "shutdown_timeout": "1h",
      "storage": [
        {
          "disk_size": "{{user `disk_size`}}",
          "disk_thin_provisioned": "{{user `disk_thin_provisioned`}}"
        }
      ],
```
next we setup winRM login data so that we can remotely run commands against this machine
after booting it, to begin getting it ready to be a k8s node
```
      "type": "vsphere-iso",
      "username": "{{user `username`}}",
      "vcenter_server": "{{user `vcenter_server`}}",
      "vm_name": "{{user `build_version`}}",
      "vm_version": "{{user `vmx_version`}}",
      "winrm_insecure": true,
      "winrm_password": "S3cr3t0!",
      "winrm_timeout": "4h",
      "winrm_username": "Administrator"
    }
  ],
```
We define provisioners in this file which 
```
  "provisioners": [
    {
      "except": "file",
      "extra_arguments": [
        "-e",
        "ansible_winrm_server_cert_validation=ignore",
        "--extra-vars",
        "{{user `ansible_common_vars`}}",
        "--extra-vars",
        "{{user `ansible_extra_vars`}}",
        "--extra-vars",
        "{{user `ansible_user_vars`}}"
      ],
      "playbook_file": "ansible/windows/node_windows.yml",
      "type": "ansible",
      "use_proxy": false,
      "user": "Administrator"
    },
    {
      "except": "file",
      "restart_check_command": "powershell -command \"& {if ((get-content C:\\ProgramData\\lastboot.txt) -eq (Get-WmiObject win32_operatingsystem).LastBootUpTime) {Write-Output 'Sleeping for 600 seconds to wait for reboot'; start-sleep 600} else {Write-Output 'Reboot complete'}}\"",
      "restart_command": "powershell \"& {(Get-WmiObject win32_operatingsystem).LastBootUpTime > C:\\ProgramData\\lastboot.txt; Restart-Computer -force}\"",
      "type": "windows-restart"
    },
...
      "inline": [
        "rm -Force -Recurse C:\\var\\log\\kubelet\\*"
      ],
      "type": "powershell"
    }
  ],
...
```
We'll see the rm -Force command run later when we look at all the image-builder run logs.  This is one of the final 
steps in image-building.

Now, we have "variables" that are inputs to this packer windows image builder.  The variables are listed below, and you can 
customize these in when you build your TKG windows image:

```
  "variables": {
    "additional_debug_files": null,
    "ansible_common_vars": "",
    "ansible_extra_vars": "",
    "ansible_user_vars": "",
    "build_name": null,
    "build_timestamp": "{{timestamp}}",
    "build_version": "{{user `build_name`}}-kube-{{user `kubernetes_semver`}}",
    "cloudbase_init_url": "https://github.com/cloudbase/cloudbase-init/releases/download/{{user `cloudbase_init_version`}}/CloudbaseInitSetup_{{user `cloudbase_init_version` | replace_all `.` `_` }}_x64.msi",
    "cloudbase_metadata_services": "cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService",
    "cloudbase_metadata_services_unattend": "cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService",
    "cloudbase_plugins": "cloudbaseinit.plugins.windows.createuser.CreateUserPlugin, cloudbaseinit.plugins.common.setuserpassword.SetUserPasswordPlugin, cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin, cloudbaseinit.plugins.common.userdata.UserDataPlugin, cloudbaseinit.plugins.common.ephemeraldisk.EphemeralDiskPlugin, cloudbaseinit.plugins.common.mtu.MTUPlugin, cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin,  cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin, cloudbaseinit.plugins.common.localscripts.LocalScriptsPlugin, cloudbaseinit.plugins.windows.createuser.CreateUserPlugin, cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin",
    "cloudbase_plugins_unattend": "cloudbaseinit.plugins.common.mtu.MTUPlugin",
    "containerd_sha256": null,
    "containerd_url": "",
    "containerd_version": null,
    "disable_hypervisor": null,
    "disk_size": "81920",
    "http_port_max": "",
    "http_port_min": "",
    "ib_version": "{{env `IB_VERSION`}}",
    "kubernetes_base_url": "https://kubernetesreleases.blob.core.windows.net/kubernetes/{{user `kubernetes_semver`}}/binaries/node/windows/{{user `kubernetes_goarch`}}",
    "kubernetes_http_package_url": "",
    "kubernetes_typed_version": "kube-{{user `kubernetes_semver`}}",
    "manifest_output": "manifest.json",
    "netbios_host_name_compatibility": null,
    "nssm_url": null,
    "output_dir": "./output/{{user `build_version`}}",
    "prepull": null,
    "unattend_timezone": "Pacific Standard Time",
    "windows_service_manager": null,
    "windows_updates_categories": null,
    "windows_updates_kbs": null,
    "wins_url": "https://github.com/rancher/wins/releases/download/v{{user `wins_version`}}/wins.exe"
  }
}
```
We'll see all this happen shortly, when we create a windows image.  First we'll create a Mgmt cluster.
Of course if you already have a Mgmt cluster and only want to make a windows Workload cluster... skip this section.

## Part1: Unpack TKG, install management cluster

- Get the [independent Tanzu CLI](https://github.com/vmware-tanzu/tanzu-cli/releases/download/v0.90.1/tanzu-cli-linux-amd64.tar.gz
- unzip / untar / install it in bin
- wget your OVA from the release,  `govc import.ova ./windows-2019-kube-v1.21.2+vmware.1-tkg.1.ova`
- Mark your OVA as a template

Setup Tanzu cli:
```
kubo@uOFLhGS9YBJ3y:~$ tanzu plugin install --group vmware-tkg/tkg
[i] Reading plugin inventory for "projects.registry.vmware.com/tanzu_cli/plugins/plugin-inventory:latest", this will take a few seconds.
[i] Reading plugin inventory for "projects-stg.registry.vmware.com/tanzu_cli/plugins/sandbox/tkg-ob-468120486233034140-g57e43b84/plugin-inventory:latest", this will take a few seconds.
[!] Skipping the plugins discovery image signature verification for "projects-stg.registry.vmware.com/tanzu_cli/plugins/sandbox/tkg-ob-468120486233034140-g57e43b84/plugin-inventory:latest"
 
[i] Installing plugin 'isolated-cluster:v0.30.0-dev' with target 'global'
[i] Installing plugin 'management-cluster:v0.30.0-dev' with target 'kubernetes'
[i] Installing plugin 'package:v0.30.0-dev' with target 'kubernetes'
[i] Installing plugin 'pinniped-auth:v0.30.0-dev' with target 'global'
[i] Installing plugin 'secret:v0.30.0-dev' with target 'kubernetes'
[i] Installing plugin 'telemetry:v0.30.0-dev' with target 'kubernetes'
[ok] successfully installed all plugins from group 'vmware-tkg/tkg:v2.3.0'
```

Then make a management cluster
```
  1 CLUSTER_PLAN: dev
  2 CNI: antrea
  3 INFRASTRUCTURE_PROVIDER: vsphere
  4 KUBERNETES_VERSION: v1.25.10+vmware.2
  5 #KUBEVIP_LOADBALANCER_CIDRS: 10.180.81.109/32,10.180.81.109/32,10.180.81.109/32,10.180.81.109/32
  6 #KUBEVIP_LOADBALANCER_ENABLE: true
  7 OS_ARCH: amd64
  8 OS_NAME: ubuntu
  9 OS_VERSION: '20.04'
 10 _VSPHERE_CONTROL_PLANE_ENDPOINT: 10.221.159.240
```

You now run:

in this particular vm, my ubuntu image is at
```
## govc find / | grep ubuntu | grep efi
/Workload 3/vm/GPU/ubuntu-2004-efi-kube-v1.26.5             
```
So my creation env vars are......
`/$VSPHERE_FOLDER/$VSPHERE_DATASTORE/`
and the env vars we use to creat our cluster will be
```                                  
export VSPHERE_NETWORK="/Workload 3/network/workload-dhcp-759"
export VSPHERE_PASSWORD="z.....J"
export VSPHERE_RESOURCE_POOL="windows"
export VSPHERE_INSECURE=true
export VSPHERE_FOLDER="/Workload 3/vm"
export VSPHERE_DATASTORE=GPU
export VSPHERE_USERNAME="shepherd-gpu@vtcc.io"
export VSPHERE_SERVER=10.89.160.37
export VSPHERE_DATACENTER="Workload 3"
#####
export DEPLOY_TKG_ON_VSPHERE7=true
export VSPHERE_SSH_AUTHORIZED_KEY="ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6VfBKd6hqd5h7k5f+AtjJSV1hdW5u9/3uAolK3SD2/5GD9+rn+FMSdbtkeaKuuVJPi2HjnsVMO+r8WcuyN5ZSYHywiSoh4S7PamAxra1CLISsFHPYFlGrtdHC70wnoT7+/wAJk2D3CYkCNMWIxs5eR0cefDOytipBfDplhkJByyrcnXuhI8St3XJzpjlXu454diJOxfsk6axanWLOr/WZFmUi1U6V4gRE7XtKG9WFUm1bmNgkgd7lehKzi+isTjnI+b4tnD0yIzKFcsgIvLdGJTI6Lluj33CeBHIocwu0LbvowTyYSqhP6DzGhGuKfK9rMnJh/ll0Bnu1xf/ok0NSQ== Jpeerindex@doolittle-5.local"
tanzu management-cluster create --acknowledge-CEIP -y -f ./cluster.yaml
```

And your cluster will come up shortly.... 
                                                               
## Part2: Building your image

Now time to build your Windows OVA.  
- The official docs for this are here
https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-byoi-windows.html?hWord=N4IghgNiBcIO4EsB2ATA9nAziAvkA#windows-wc

First, lets Download ISOs... 

- You can get your ISO from https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022
- You can get your VMTools ISO from https://customerconnect.vmware.com/en/downloads/details?downloadGroup=VMTOOLS1225&productId=1259&rPId=106172 

- Then , click on the "hamburger" icon (database) in your VCenter UI, and find your data store (named the same as your VSPHERE_DATASTORE above) and
- click "Upload Files"... at which  point youll now launch a windows VM in vsphere, 
- record the "Path" value, which will be something like `"os_iso_path": "[GPU] 98c6b664-fa96-9911-0c82-bc97e17948c0/SERVER_EVAL_x64FRE_en-us.iso",`.  This will be the value of the `os_iso_path` in your image builder
json input.;

Which will be customized by image builder like so.... 

## 1 windows burrito
Create your burrito
```
# Create the windows "burrito" 
apiVersion: v1
kind: Namespace
metadata:
 name: imagebuilder
---
apiVersion: v1
kind: Service
metadata:
 name: imagebuilder-wrs
 namespace: imagebuilder
spec:
 selector:
   app: image-builder-resource-kit
 type: NodePort
 ports:
 - port: 3000
   targetPort: 3000
   nodePort: 30008
---
apiVersion: apps/v1
kind: Deployment
metadata:
 name: image-builder-resource-kit
 namespace: imagebuilder
spec:
 selector:
   matchLabels:
     app: image-builder-resource-kit
 template:
   metadata:
     labels:
       app: image-builder-resource-kit
   spec:
     nodeSelector:
       kubernetes.io/os: linux
     containers:
     - name: windows-imagebuilder-resourcekit
       image: projects.registry.vmware.com/tkg/windows-resource-bundle:v1.25.7_vmware.2-tkg.1
       imagePullPolicy: Always
       ports:
         - containerPort: 3000
```

## 2 Make your image builder JSON

This will be the input to image builde,r it tells it
- what the URL of the burrito ( http://10.221.159.247:30008 ) is (Which has all your exe's)
- wether to use containerd or docker
- where the windows ISO you downloaded is
- how to acess vsphere (so that it can make a windows VM)

Once you give it this info, you can use docker to run image-builder (its basically a big ansible
script that jumps into vsphere, makes a windows VM, and then kubernetizes the VM).

Note for TKG 2.3, youll want to use 1.26.5 as the k8s version instead of 1.25....and modify the parameters below
```
-  "build_version": "windows-2019-kube-v1.25.7",
+  "build_version": "windows-2019-kube-v1.26.5",
- "kubernetes_series": "v1.25.7",
+ "kubernetes_semver": "v1.26.5+vmware.2",
+ "kubernetes_series": "v1.26.5",
```
Then the OVA will come out looking like `windows-2019-kube-1.26.5`.
```
{
  "additional_executables_destination_path": "C:\\ProgramData\\Temp",                                                                                             
  "additional_executables_list": "http://10.221.159.247:30008/files/antrea-windows/antrea-windows-advanced.zip,http://10.221.159.247:30008/files/kubernetes/kube-proxy.exe",
  "additional_executables": "true",
  "additional_prepull_images": "mcr.microsoft.com/windows/servercore:ltsc2019",
  "build_version": "windows-2019-kube-v1.25.7",
  "cloudbase_init_url": "http://10.221.159.247:30008/files/cloudbase_init/CloudbaseInitSetup_1_1_4_x64.msi",
  "cluster": "VSPHERE-CLUSTER-NAME",
  "containerd_sha256_windows": "2e0332aa57ebcb6c839a8ec807780d662973a15754573630bea249760cdccf2a",                                                              
  "containerd_url": "http://10.221.159.247:30008/files/containerd/cri-containerd-v1.6.18+vmware.1.windows-amd64.tar",                                             
  "containerd_version": "v1.6.18",
  "goss_inspect_mode": "true",                                                                                                                                    
  "insecure_connection": "true",                                                                                                                                
  "kubernetes_base_url": "http://10.221.159.247:30008/files/kubernetes/",                                                                                         
  "kubernetes_semver": "v1.25.7+vmware.2",                                                                                                                        
  "kubernetes_series": "v1.25.7",                                                                                                                                 
  "linked_clone": "false",                                                                                                                                        
  "load_additional_components": "true",                                                                                                                         
  "netbios_host_name_compatibility": "false",                                                                                                                     
  "network": "/Workload 3/network/workload-dhcp-759",                                                                                                             
  "nssm_url": "http://10.221.159.247:30008/files/nssm/nssm.exe",    
  "os_iso_path": "[GPU] 98c6b664-fa96-9911-0c82-bc97e17948c0/SERVER_EVAL_x64FRE_en-us.iso",
  "password": "xxxxxxxxx",                                                                                                                              
  "pause_image": "mcr.microsoft.com/oss/kubernetes/pause:3.6",                                                                                                    
  "prepull": "false",                                                                                                                                             
  "resource_pool": "",                                                                                                                                          
  "runtime": "containerd",                                                                                                                                        
  "template": "",                                                                                                                                                 
  "username": "shepherd-gpu@vtcc.io",                                                                                                                             
  "vcenter_server": "10.89.160.37",                                                                                                                               
  "vmtools_iso_path": "iso/VMware-tools-windows-12.1.5.iso",                                                                                                    
  "windows_updates_categories": "CriticalUpdates SecurityUpdates UpdateRollups",                                                                                
  "windows_updates_kbs": "",                                                                                                                                    
  "wins_version": ""                                                                                                                                            
}    
```

## 3 image builder command

```
docker run -it \ 
--rm \
--mount type=bind,source=$(pwd)/image-builder/image-builder-input-spec.json,target=/windows.json \
--mount type=bind,source=$(pwd)/image-builder/autounattend.xml,target=/home/imagebuilder/packer/ova/windows/windows-2019/autounattend.xml \
-e PACKER_VAR_FILES="/windows.json" -e IB_OVFTOOL=1 -e IB_OVFTOOL_ARGS='--skipManifestCheck' -e PACKER_FLAGS='-force -on-error=ask' \
-e PACKER_LOG=1  \
-t projects.registry.vmware.com/tkg/image-builder:v0.1.13_vmware.3 build-node-ova-vsphere-windows-2019   
```

Note the PACKER_LOG=1 gives you verbose logs in case things go wrong.

## 4 image builder issues?

- make sure you named your files properly
- getting the value of `cluster`
If you see
```
==> vsphere: error creating vm: resource pool 'vc-w3.vtcc.io/Resources/' not found
Build 'vsphere' errored after 9 seconds 65 milliseconds: error creating vm: resource pool  'vc-w3.vtcc.io/Resources/' not found                                                                                                                        
```
Then you might have the wrong `cluster` value.  To find a value you can use for cluster, try looking
for places where there are `Resources/`.

```
kubo@uOFLhGS9YBJ3y:~$ govc find /  2>&1 |grep -i Resources  
kubo@uOFLhGS9YBJ3y:~$ govc find /  2>&1 |grep -i Resources
/Workload 3/host/GPU/Resources/windows  
/Workload 3/host/GPU/Resources
/Workload 3/host/Airgap/Resources                                                                                                                                                                                                                     
/Workload 3/host/A/Resources                                                                                                                                                                                                                          
/Workload 3/host/Main/Resources                                                                                                                                                                                                                       
/Workload 3/host/Main/Resources/Documentation        
...
```
Then , you can use the prefix, ie `/Workload 3/host/GPU/` as the value for `cluster`.
- if you see `==> vsphere: Step "StepCreateVM" failed creating vm`... 
with `ServerFaultCode: Permission to perform this operation was denied.` 
Then somewhow your account is possibly missing some permissions in vsphere.

## 5 image builder: 5 results

Eventually once image builder fully runs, youll see some output like this: 
```
==> vsphere:     + FullyQualifiedErrorId : RemoveFileSystemItemIOError,Microsoft.PowerShell.Commands.RemoveItemCommand                                                                                                                                                                   2023/07/21 21:08:39 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell -executionpolicy bypass "& { if (Test-Path variable:global:ProgressPreference){set-variable -name variable:global:ProgressPreference -value 'SilentlyContinue'};. c:/Windows/Temp/packe
r-ps-env-vars-64baeaf8-6021-4e36-6123-5e4ea4112cfc.ps1; &'c:/Windows/Temp/packer-cleanup-64baeaf8-8e84-27cd-38a7-a44c1fd2ff68.ps1'; exit $LastExitCode }"                                                                                                                                2023/07/21 21:08:40 packer-builder-vsphere-iso plugin: [INFO] command 'powershell -executionpolicy bypass "& { if (Test-Path variable:global:ProgressPreference){set-variable -name variable:global:ProgressPreference -value 'SilentlyContinue'};. c:/Windows/Temp/packer-ps-env-vars-64baeaf8-6021-4e36-6123-5e4ea4112cfc.ps1; &'c:/Windows/Temp/packer-cleanup-64baeaf8-8e84-27cd-38a7-a44c1fd2ff68.ps1'; exit $LastExitCode }"' exited with code: 0                                                                                                                           2023/07/21 21:08:40 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0                                                                                                                                                                                   
2023/07/21 21:08:40 [INFO] 0 bytes written for 'stderr'                                                                                                                                                                                                                                  2023/07/21 21:08:40 [INFO] 0 bytes written for 'stdout'                                                                                                                                                                                                                                  2023/07/21 21:08:40 [INFO] RPC client: Communicator ended with: 0                                                                                                                                                                                                                        2023/07/21 21:08:40 [INFO] RPC endpoint: Communicator ended with: 0                                                                                                                                                                                                                      
2023/07/21 21:08:40 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stdout'                                                                                                                                                                                            2023/07/21 21:08:40 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stderr'                                                                                                                                                                                            2023/07/21 21:08:40 packer-provisioner-powershell plugin: [INFO] RPC client: Communicator ended with: 0                                                                                                                                                                                  2023/07/21 21:08:40 [INFO] (telemetry) ending powershell                                                                                                                                                                                                                                 ==> vsphere: Executing shutdown command...                                                                                                                                                                                                                                               2023/07/21 21:08:41 packer-builder-vsphere-iso plugin: Shutdown command: powershell A:/sysprep.ps1                                                                                                                                                                                       
2023/07/21 21:08:41 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell A:/sysprep.ps1                                                                                                                                                                         2023/07/21 21:08:41 packer-builder-vsphere-iso plugin: Waiting max 1h0m0s for shutdown to complete                                                                                                                                                                                       2023/07/21 21:08:49 packer-builder-vsphere-iso plugin: [INFO] command 'powershell A:/sysprep.ps1' exited with code: 0                                                                                                                                                                    ==> vsphere: Deleting Floppy drives...                                                                                                                                                                                                                                                   
==> vsphere: Deleting Floppy image...                                                                                                                                                                                                                                                    ==> vsphere: Eject CD-ROM drives...                                                                                                                                                                                                                                                      ==> vsphere: Convert VM into template...                                                                                                                                                                                                                                                     vsphere: Starting export...                                                                                                                                                                                                                                                          
    vsphere: Downloading: windows-2019-kube-v1.25.7-disk-0.vmdk                                                                                                                                                                                                                              vsphere: Exporting file: windows-2019-kube-v1.25.7-disk-0.vmdk                                                                                                                                                                                                                           vsphere: Writing ovf...                                                                                                                                                                                                                                                                  vsphere: Creating manifest...                                                                                                                                                                                                                                                            vsphere: Finished exporting...                                                                                                                                                                                                                                                       
==> vsphere: Clear boot order...                                                                                                                                                                                                                                                         2023/07/21 21:15:25 packer-builder-vsphere-iso plugin: Deleting floppy disk: /tmp/packer2539854374                                                                                                                                                                                       ==> vsphere: Running post-processor: manifest                                                                                                                                                                                                                                            2023/07/21 21:15:26 [INFO] (telemetry) ending vsphere                                                                                                                                                                                                                                    2023/07/21 21:15:26 [INFO] (telemetry) Starting post-processor manifest                                                                                                                                                                                                                  
==> vsphere: Running post-processor: vsphere (type shell-local)                                                                                                                                                                                                                          2023/07/21 21:15:26 [INFO] (telemetry) ending manifest                                                                                                                                                                                                                                   2023/07/21 21:15:26 Flagging to keep original artifact from post-processor 'manifest'                                                                                                                                                                                                    2023/07/21 21:15:26 [INFO] (telemetry) Starting post-processor shell-local                                                                                                                                                                                                               
2023/07/21 21:15:26 packer-post-processor-shell-local plugin: [INFO] (shell-local): Prepending inline script with #!/bin/sh -e                                                                                                                                                           ==> vsphere (shell-local): Running local shell script: /tmp/packer-shell339263217                                                                                                                                                                                                        2023/07/21 21:15:26 packer-post-processor-shell-local plugin: [INFO] (shell-local): starting local command: /bin/sh -c PACKER_BUILDER_TYPE='vsphere-iso' PACKER_BUILD_NAME='vsphere' PACKER_HTTP_ADDR='172.17.0.2:0' PACKER_HTTP_IP='172.17.0.2' PACKER_HTTP_PORT='0'  /tmp/packer-shell339263217                                                                                                                                                                                                                                                                                 
2023/07/21 21:15:26 packer-post-processor-shell-local plugin: [INFO] (shell-local communicator): Executing local shell command [/bin/sh -c PACKER_BUILDER_TYPE='vsphere-iso' PACKER_BUILD_NAME='vsphere' PACKER_HTTP_ADDR='172.17.0.2:0' PACKER_HTTP_IP='172.17.0.2' PACKER_HTTP_PORT='0'  /tmp/packer-shell339263217]                                                                                                                                                                                                                                                                vsphere (shell-local): Opening OVF source: windows-2019-kube-v1.25.7+vmware.2.ovf                                                                                                                                                                                                    
    vsphere (shell-local): Opening OVA target: windows-2019-kube-v1.25.7+vmware.2.ova                                                                                                                                                                                                        vsphere (shell-local): Writing OVA package: windows-2019-kube-v1.25.7+vmware.2.ova                                                                                                                                                                                                       vsphere (shell-local): Transfer Completed                                                                                                                                                                                                                                            
    vsphere (shell-local): Completed successfully                                                                                                                                                                                                                                            vsphere (shell-local): image-build-ova: cd .                                                                                                                                                                                                                                             vsphere (shell-local): image-build-ova: loaded windows-2019-kube-v1.25.7+vmware.2                                                                                                                                                                                                        vsphere (shell-local): image-build-ova: create ovf windows-2019-kube-v1.25.7+vmware.2.ovf                                                                                                                                                                                                vsphere (shell-local): image-build-ova: creating OVA from windows-2019-kube-v1.25.7+vmware.2.ovf using ovftool                                                                                                                                                                       
    vsphere (shell-local): image-build-ova: create ova checksum windows-2019-kube-v1.25.7+vmware.2.ova.sha256                                                                                                                                                                            2023/07/21 21:17:58 [INFO] (telemetry) ending shell-local                                                                                                                                                                                                                                ==> Wait completed after 47 minutes 10 seconds                                                                                                                                                                                                                                           
Build 'vsphere' finished after 47 minutes 10 seconds.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             ==> Wait completed after 47 minutes 10 seconds                                                                                                                                                                                                                                           
==> Builds finished. The artifacts of successful builds are:                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      ==> Builds finished. The artifacts of successful builds are:                                                                                                                                                                                                                             
2023/07/21 21:17:59 machine readable: vsphere,artifact-count []string{"3"}                                                                                                                                                                                                               2023/07/21 21:17:59 machine readable: vsphere,artifact []string{"0", "builder-id", "jetbrains.vsphere"}                                                                                                                                                                                  2023/07/21 21:17:59 machine readable: vsphere,artifact []string{"0", "id", "windows-2019-kube-v1.25.7"}                                                                                                                                                                                  
2023/07/21 21:17:59 machine readable: vsphere,artifact []string{"0", "string", "windows-2019-kube-v1.25.7"}     
```

## 6 image builder logs

Now in the spirit of tanzu install annotaed, lets annnotate all the logs !


## 7 ALL the logs 

The entire log output of 
```
docker run -it \ 
--rm \
--mount type=bind,source=$(pwd)/image-builder/image-builder-input-spec.json,target=/windows.json \
--mount type=bind,source=$(pwd)/image-builder/autounattend.xml,target=/home/imagebuilder/packer/ova/windows/windows-2019/autounattend.xml \
-e PACKER_VAR_FILES="/windows.json" -e IB_OVFTOOL=1 -e IB_OVFTOOL_ARGS='--skipManifestCheck' -e PACKER_FLAGS='-force -on-error=ask' \
-e PACKER_LOG=1  \
-t projects.registry.vmware.com/tkg/image-builder:v0.1.13_vmware.3 build-node-ova-vsphere-windows-2019   
```

from above is below.  Lets see what happens.

First some local hack scripts in the VMWare image-builder container run, to ensure the container has
- the kubernetes-sigs/image-builder repo
- ansible
- goss: a tool for verifying that an OS is setup correctly
- ovftool: used for building OVAs

```
hack/ensure-ansible.sh
Starting galaxy collection install process
Nothing to do. All requested collections are already installed. If you want to reinstall them, consider using `--force`.
hack/ensure-ansible-windows.sh
hack/ensure-packer.sh
hack/ensure-goss.sh
Right version of binary present
hack/ensure-ovftool.sh
```

## image-builder: packer
Next we see that image-builder calls out to packer: 

```
packer build -var-file="/home/imagebuilder/packer/config/kubernetes.json"  -var-file="/home/imagebuilder/packer/config/windows/kubernetes.json"  -var-file="/home/imagebuilder/packer/config/containerd.json"  -var-file="/home/imagebuilder/packer/config/windows/containerd.json"  -var-file="/home/imagebuilder/packer/config/windows/docker.json"  -var-file="/home/imagebuilder/packer/config/windows/ansible-args-windows.json"  -var-file="/home/imagebuilder/packer/config/common.json"  -var-file="/home/imagebuilder/packer/config/windows/common.json"  -var-file="/home/imagebuilder/packer/config/windows/cloudbase-init.json"  -var-file="/home/imagebuilder/packer/config/goss-args.json"  -var-file="/home/imagebuilder/packer/config/additional_components.json"  -force -on-error=ask -color=true -var-file="packer/ova/packer-common.json" -var-file="/home/imagebuilder/packer/ova/windows-2019.json" -only=file -var-file="/windows.json"  packer/ova/packer-windows.json
2023/07/22 00:26:48 [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:48 Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 [TRACE] discovering plugins in /home/imagebuilder/.local/bin
2023/07/22 00:26:48 Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 [TRACE] discovering plugins in /home/imagebuilder/.packer.d/plugins
2023/07/22 00:26:48 [DEBUG] Discovered plugin: goss = /home/imagebuilder/.packer.d/plugins/packer-provisioner-goss
2023/07/22 00:26:48 using external provisioners [goss]
2023/07/22 00:26:48 [TRACE] discovering plugins in .
2023/07/22 00:26:48 [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:48 [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:48 [TRACE] Starting internal plugin packer-builder-file
2023/07/22 00:26:48 Starting plugin: /home/imagebuilder/.local/bin/packer []string{"/home/imagebuilder/.local/bin/packer", "plugin", "packer-builder-file"}
2023/07/22 00:26:48 Waiting for RPC address for: /home/imagebuilder/.local/bin/packer
2023/07/22 00:26:48 packer-builder-file plugin: [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:48 packer-builder-file plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-builder-file plugin: [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:48 packer-builder-file plugin: [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 packer-builder-file plugin: [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 packer-builder-file plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-builder-file plugin: [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:48 packer-builder-file plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-builder-file plugin: args: []string{"packer-builder-file"}
2023/07/22 00:26:48 packer-builder-file plugin: Plugin address: unix /tmp/packer-plugin87314221
2023/07/22 00:26:48 packer-builder-file plugin: Waiting for connection...
2023/07/22 00:26:48 Received unix RPC address for /home/imagebuilder/.local/bin/packer: addr is /tmp/packer-plugin87314221
2023/07/22 00:26:48 packer-builder-file plugin: Serving a plugin connection...
```
Now packer  starts running its plugins:
```
2023/07/22 00:26:48 [TRACE] Starting internal plugin packer-provisioner-powershell
2023/07/22 00:26:48 Starting plugin: /home/imagebuilder/.local/bin/packer []string{"/home/imagebuilder/.local/bin/packer", "plugin", "packer-provisioner-powershell"}
2023/07/22 00:26:48 Waiting for RPC address for: /home/imagebuilder/.local/bin/packer
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Old default config directory found: /home/imagebuilder/.packer.d
```

More packer boiler plate:
```
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:48 packer-provisioner-powershell plugin: args: []string{"packer-provisioner-powershell"}
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Plugin address: unix /tmp/packer-plugin1942378941
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Waiting for connection...
2023/07/22 00:26:48 Received unix RPC address for /home/imagebuilder/.local/bin/packer: addr is /tmp/packer-plugin1942378941
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Serving a plugin connection...
2023/07/22 00:26:48 Preparing build: file
2023/07/22 00:26:48 Build debug mode: false
2023/07/22 00:26:48 Force build: true
2023/07/22 00:26:48 On error: ask
2023/07/22 00:26:48 Waiting on builds to complete...
2023/07/22 00:26:48 Starting build run: file
2023/07/22 00:26:48 Running builder: file
2023/07/22 00:26:48 [INFO] (telemetry) Starting builder file
[1;32mfile: output will be in this color.[0m

2023/07/22 00:26:48 [INFO] (telemetry) Starting provisioner powershell
2023/07/22 00:26:48 Unable to read map[string]interface out of data.Using empty interface: <nil>
[1;32m==> file: Provisioning with Powershell...[0m
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Found command: rm -Force -Recurse C:\var\log\kubelet\*
[1;32m==> file: Provisioning with powershell script: /tmp/powershell-provisioner1383838706[0m
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Opening /tmp/powershell-provisioner1383838706 for reading
2023/07/22 00:26:48 packer-provisioner-powershell plugin: Uploading env vars to c:/Windows/Temp/packer-ps-env-vars-64bb2248-da10-11fc-50c3-90ee97da7c02.ps1
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] 64 bytes written for 'uploadData'
2023/07/22 00:26:48 [INFO] 64 bytes written for 'uploadData'
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] 40 bytes written for 'uploadData'
2023/07/22 00:26:48 [INFO] 40 bytes written for 'uploadData'
2023/07/22 00:26:48 packer-builder-file plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 00:26:48 [INFO] 0 bytes written for 'stdout'
2023/07/22 00:26:48 [INFO] 0 bytes written for 'stderr'
2023/07/22 00:26:48 [INFO] RPC client: Communicator ended with: 0
2023/07/22 00:26:48 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] RPC client: Communicator ended with: 0
2023/07/22 00:26:48 packer-provisioner-powershell plugin: c:/Windows/Temp/script-64bb2248-462d-a2a5-5036-0edad16d13d9.ps1 returned with exit code 0
2023/07/22 00:26:48 [INFO] 511 bytes written for 'uploadData'
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] 511 bytes written for 'uploadData'
2023/07/22 00:26:48 [INFO] 0 bytes written for 'stderr'
2023/07/22 00:26:48 packer-builder-file plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 00:26:48 [INFO] 0 bytes written for 'stdout'
2023/07/22 00:26:48 [INFO] RPC client: Communicator ended with: 0
2023/07/22 00:26:48 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 00:26:48 packer-provisioner-powershell plugin: [INFO] RPC client: Communicator ended with: 0
2023/07/22 00:26:48 [INFO] (telemetry) ending powershell
2023/07/22 00:26:48 [INFO] (telemetry) ending file
==> Wait completed after 11 milliseconds 415 microseconds
==> Builds finished. The artifacts of successful builds are:
2023/07/22 00:26:48 machine readable: file,artifact-count []string{"1"}
[1;32mBuild 'file' finished after 11 milliseconds 377 microseconds.[0m
==> Wait completed after 11 milliseconds 415 microseconds
==> Builds finished. The artifacts of successful builds are:
2023/07/22 00:26:48 machine readable: file,artifact []string{"0", "builder-id", "packer.file"}
--> file: Stored file: ./packer_cache/unattend.json
2023/07/22 00:26:48 machine readable: file,artifact []string{"0", "id", "File"}
2023/07/22 00:26:48 machine readable: file,artifact []string{"0", "string", "Stored file: ./packer_cache/unattend.json"}
2023/07/22 00:26:48 machine readable: file,artifact []string{"0", "files-count", "1"}
2023/07/22 00:26:48 machine readable: file,artifact []string{"0", "file", "0", "./packer_cache/unattend.json"}
2023/07/22 00:26:48 machine readable: file,artifact []string{"0", "end"}
2023/07/22 00:26:48 [INFO] (telemetry) Finalizing.
2023/07/22 00:26:48 waiting for all plugin processes to complete...
2023/07/22 00:26:48 /home/imagebuilder/.local/bin/packer: plugin process exited
2023/07/22 00:26:48 /home/imagebuilder/.local/bin/packer: plugin process exited
hack/windows-ova-unattend.py --unattend-file='./packer/ova/windows/windows-2019/autounattend.xml'
windows-ova-unattend: cd .
windows-ova-unattend: Setting Timezone to Pacific Standard Time
windows-ova-unattend: Updating ./packer/ova/windows/windows-2019/autounattend.xml ...
packer build -var-file="/home/imagebuilder/packer/config/kubernetes.json"  -var-file="/home/imagebuilder/packer/config/windows/kubernetes.json"  -var-file="/home/imagebuilder/packer/config/containerd.json"  -var-file="/home/imagebuilder/packer/config/windows/containerd.json"  -var-file="/home/imagebuilder/packer/config/windows/docker.json"  -var-file="/home/imagebuilder/packer/config/windows/ansible-args-windows.json"  -var-file="/home/imagebuilder/packer/config/common.json"  -var-file="/home/imagebuilder/packer/config/windows/common.json"  -var-file="/home/imagebuilder/packer/config/windows/cloudbase-init.json"  -var-file="/home/imagebuilder/packer/config/goss-args.json"  -var-file="/home/imagebuilder/packer/config/additional_components.json"  -force -on-error=ask -color=true  -var-file="packer/ova/packer-common.json" -var-file="/home/imagebuilder/packer/ova/windows-2019.json" -var-file="packer/ova/vsphere.json"  -except=local -only=vsphere-iso -var-file="/windows.json"  -only=vsphere packer/ova/packer-windows.json
2023/07/22 00:26:48 [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:48 Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 [TRACE] discovering plugins in /home/imagebuilder/.local/bin
2023/07/22 00:26:48 Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 [TRACE] discovering plugins in /home/imagebuilder/.packer.d/plugins
2023/07/22 00:26:48 [DEBUG] Discovered plugin: goss = /home/imagebuilder/.packer.d/plugins/packer-provisioner-goss
2023/07/22 00:26:48 using external provisioners [goss]
2023/07/22 00:26:48 [TRACE] discovering plugins in .
2023/07/22 00:26:48 [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:48 [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:48 [TRACE] Starting internal plugin packer-builder-vsphere-iso
2023/07/22 00:26:48 Starting plugin: /home/imagebuilder/.local/bin/packer []string{"/home/imagebuilder/.local/bin/packer", "plugin", "packer-builder-vsphere-iso"}
2023/07/22 00:26:48 Waiting for RPC address for: /home/imagebuilder/.local/bin/packer
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: args: []string{"packer-builder-vsphere-iso"}
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 Received unix RPC address for /home/imagebuilder/.local/bin/packer: addr is /tmp/packer-plugin2011710679
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: Plugin address: unix /tmp/packer-plugin2011710679
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: Waiting for connection...
2023/07/22 00:26:48 packer-builder-vsphere-iso plugin: Serving a plugin connection...
2023/07/22 00:26:48 [TRACE] Starting internal plugin packer-provisioner-ansible
2023/07/22 00:26:48 Starting plugin: /home/imagebuilder/.local/bin/packer []string{"/home/imagebuilder/.local/bin/packer", "plugin", "packer-provisioner-ansible"}
2023/07/22 00:26:48 Waiting for RPC address for: /home/imagebuilder/.local/bin/packer
2023/07/22 00:26:48 packer-provisioner-ansible plugin: [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:48 packer-provisioner-ansible plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-provisioner-ansible plugin: [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:48 packer-provisioner-ansible plugin: [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 packer-provisioner-ansible plugin: [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:48 packer-provisioner-ansible plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-provisioner-ansible plugin: [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:48 packer-provisioner-ansible plugin: args: []string{"packer-provisioner-ansible"}
2023/07/22 00:26:48 packer-provisioner-ansible plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:48 packer-provisioner-ansible plugin: Plugin address: unix /tmp/packer-plugin583655069
2023/07/22 00:26:48 packer-provisioner-ansible plugin: Waiting for connection...
2023/07/22 00:26:48 Received unix RPC address for /home/imagebuilder/.local/bin/packer: addr is /tmp/packer-plugin583655069
2023/07/22 00:26:48 packer-provisioner-ansible plugin: Serving a plugin connection...
2023/07/22 00:26:48 [TRACE] Starting internal plugin packer-provisioner-windows-restart
2023/07/22 00:26:48 Starting plugin: /home/imagebuilder/.local/bin/packer []string{"/home/imagebuilder/.local/bin/packer", "plugin", "packer-provisioner-windows-restart"}
2023/07/22 00:26:48 Waiting for RPC address for: /home/imagebuilder/.local/bin/packer
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: args: []string{"packer-provisioner-windows-restart"}
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: Plugin address: unix /tmp/packer-plugin3062831735
2023/07/22 00:26:49 Received unix RPC address for /home/imagebuilder/.local/bin/packer: addr is /tmp/packer-plugin3062831735
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: Waiting for connection...
2023/07/22 00:26:49 packer-provisioner-windows-restart plugin: Serving a plugin connection...
2023/07/22 00:26:49 [TRACE] Starting external plugin /home/imagebuilder/.packer.d/plugins/packer-provisioner-goss 
2023/07/22 00:26:49 Starting plugin: /home/imagebuilder/.packer.d/plugins/packer-provisioner-goss []string{"/home/imagebuilder/.packer.d/plugins/packer-provisioner-goss"}
2023/07/22 00:26:49 Waiting for RPC address for: /home/imagebuilder/.packer.d/plugins/packer-provisioner-goss
2023/07/22 00:26:49 packer-provisioner-goss plugin: 2023/07/22 00:26:49 Plugin address: unix /tmp/packer-plugin2038428066
2023/07/22 00:26:49 packer-provisioner-goss plugin: 2023/07/22 00:26:49 Waiting for connection...
2023/07/22 00:26:49 Received unix RPC address for /home/imagebuilder/.packer.d/plugins/packer-provisioner-goss: addr is /tmp/packer-plugin2038428066
2023/07/22 00:26:49 packer-provisioner-goss plugin: 2023/07/22 00:26:49 Serving a plugin connection...
2023/07/22 00:26:49 [TRACE] Starting internal plugin packer-provisioner-powershell
2023/07/22 00:26:49 Starting plugin: /home/imagebuilder/.local/bin/packer []string{"/home/imagebuilder/.local/bin/packer", "plugin", "packer-provisioner-powershell"}
2023/07/22 00:26:49 Waiting for RPC address for: /home/imagebuilder/.local/bin/packer
2023/07/22 00:26:49 packer-provisioner-powershell plugin: [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:49 packer-provisioner-powershell plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-provisioner-powershell plugin: [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:49 packer-provisioner-powershell plugin: [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:49 packer-provisioner-powershell plugin: [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:49 packer-provisioner-powershell plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-provisioner-powershell plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-provisioner-powershell plugin: [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:49 packer-provisioner-powershell plugin: args: []string{"packer-provisioner-powershell"}
2023/07/22 00:26:49 packer-provisioner-powershell plugin: Plugin address: unix /tmp/packer-plugin3509944753
2023/07/22 00:26:49 Received unix RPC address for /home/imagebuilder/.local/bin/packer: addr is /tmp/packer-plugin3509944753
2023/07/22 00:26:49 packer-provisioner-powershell plugin: Waiting for connection...
2023/07/22 00:26:49 packer-provisioner-powershell plugin: Serving a plugin connection...
2023/07/22 00:26:49 [TRACE] Starting internal plugin packer-post-processor-manifest
2023/07/22 00:26:49 Starting plugin: /home/imagebuilder/.local/bin/packer []string{"/home/imagebuilder/.local/bin/packer", "plugin", "packer-post-processor-manifest"}
2023/07/22 00:26:49 Waiting for RPC address for: /home/imagebuilder/.local/bin/packer
2023/07/22 00:26:49 packer-post-processor-manifest plugin: [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:49 packer-post-processor-manifest plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-post-processor-manifest plugin: [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:49 packer-post-processor-manifest plugin: [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:49 packer-post-processor-manifest plugin: [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:49 packer-post-processor-manifest plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-post-processor-manifest plugin: [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:49 packer-post-processor-manifest plugin: args: []string{"packer-post-processor-manifest"}
2023/07/22 00:26:49 packer-post-processor-manifest plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-post-processor-manifest plugin: Plugin address: unix /tmp/packer-plugin1556099236
2023/07/22 00:26:49 Received unix RPC address for /home/imagebuilder/.local/bin/packer: addr is /tmp/packer-plugin1556099236
2023/07/22 00:26:49 packer-post-processor-manifest plugin: Waiting for connection...
2023/07/22 00:26:49 packer-post-processor-manifest plugin: Serving a plugin connection...
2023/07/22 00:26:49 [TRACE] Starting internal plugin packer-post-processor-shell-local
2023/07/22 00:26:49 Starting plugin: /home/imagebuilder/.local/bin/packer []string{"/home/imagebuilder/.local/bin/packer", "plugin", "packer-post-processor-shell-local"}
2023/07/22 00:26:49 Waiting for RPC address for: /home/imagebuilder/.local/bin/packer
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: [INFO] Packer version: 1.8.5 [go1.18.9 linux amd64]
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: [INFO] PACKER_CONFIG env var not set; checking the default config file path
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: [INFO] PACKER_CONFIG env var set; attempting to open config file: /home/imagebuilder/.packerconfig
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: [WARN] Config file doesn't exist: /home/imagebuilder/.packerconfig
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: [INFO] Setting cache directory: /home/imagebuilder/.cache/packer
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: args: []string{"packer-post-processor-shell-local"}
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: Old default config directory found: /home/imagebuilder/.packer.d
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: Plugin address: unix /tmp/packer-plugin516904341
2023/07/22 00:26:49 Received unix RPC address for /home/imagebuilder/.local/bin/packer: addr is /tmp/packer-plugin516904341
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: Waiting for connection...
2023/07/22 00:26:49 packer-post-processor-shell-local plugin: Serving a plugin connection...
2023/07/22 00:26:49 Preparing build: vsphere
2023/07/22 00:26:49 packer-provisioner-ansible plugin: ansible-playbook version: 2.11.5
2023/07/22 00:26:49 Build debug mode: false
[1;32mvsphere: output will be in this color.[0m
2023/07/22 00:26:49 Force build: true
2023/07/22 00:26:49 On error: ask
```
## And now the build actually starts

```
2023/07/22 00:26:49 Waiting on builds to complete...
2023/07/22 00:26:49 Starting build run: vsphere
2023/07/22 00:26:49 Running builder: vsphere-iso
2023/07/22 00:26:49 [INFO] (telemetry) Starting builder vsphere
2023/07/22 00:26:50 packer-builder-vsphere-iso plugin: No URLs were provided to Step Download. Continuing...
2023/07/22 00:26:50 packer-builder-vsphere-iso plugin: No CD files specified. CD disk will not be made.
[1;32m==> vsphere: Creating VM...[0m
[1;32m==> vsphere: Customizing hardware...[0m
[1;32m==> vsphere: Mounting ISO images...[0m
```

If you've gotten this far, it means you are able to make VMs in Vsphere and you set the credentials right.
``` 
2023/07/22 00:26:55 packer-builder-vsphere-iso plugin: Check if ISO path is a Content Library path
2023/07/22 00:26:55 packer-builder-vsphere-iso plugin: ISO path not identified as a Content Library path
2023/07/22 00:26:55 packer-builder-vsphere-iso plugin: Using [GPU] 98c6b664-fa96-9911-0c82-bc97e17948c0/en_windows_server_2019_x64_dvd_4cb967d8.iso as the datastore path
2023/07/22 00:26:55 packer-builder-vsphere-iso plugin: Creating CD-ROM on controller '&{{{} 200 0xc000e563a0 <nil> <nil> <nil> 0 <nil> 0 <nil>} 0 []}' with iso '[GPU] 98c6b664-fa96-9911-0c82-bc97e17948c0/en_windows_server_2019_x64_dvd_4cb967d8.iso'
2023/07/22 00:26:56 packer-builder-vsphere-iso plugin: Check if ISO path is a Content Library path
2023/07/22 00:26:56 packer-builder-vsphere-iso plugin: ISO path not identified as a Content Library path
2023/07/22 00:26:56 packer-builder-vsphere-iso plugin: Using [GPU] 98c6b664-fa96-9911-0c82-bc97e17948c0/VMware-tools-windows-12.1.5.iso as the datastore path
2023/07/22 00:26:56 packer-builder-vsphere-iso plugin: Creating CD-ROM on controller '&{{{} 200 0xc001004a40 <nil> <nil> <nil> 0 <nil> 0 <nil>} 0 [3000]}' with iso '[GPU] 98c6b664-fa96-9911-0c82-bc97e17948c0/VMware-tools-windows-12.1.5.iso'
[1;32m==> vsphere: Adding configuration parameters...[0m
[1;32m==> vsphere: Creating floppy disk...[0m
2023/07/22 00:26:57 packer-builder-vsphere-iso plugin: Floppy path: /tmp/packer499324745
2023/07/22 00:26:57 packer-builder-vsphere-iso plugin: Initializing block device backed by temporary file
2023/07/22 00:26:57 packer-builder-vsphere-iso plugin: Formatting the block device with a FAT filesystem...
2023/07/22 00:26:57 packer-builder-vsphere-iso plugin: Initializing FAT filesystem on block device
2023/07/22 00:26:57 packer-builder-vsphere-iso plugin: Reading the root directory from the filesystem
```

Now we start seeing some windows specific configuration that we actually care about:
- autounattend.xml: our input configuration for our windows OVA.  note that image-builder uses different autounattend.xml files for different parts of the build process... for example image-builder/images/capi/packer/ova/windows
/sysprep.ps1 has another autounattend.xml it writes.
- disable-winrm.ps1: this allows us to kill off winrm after we make this image, b/c its no longer needed anymore (winrm is a remote mgmt tool, like SSH, that we use for configuring the VM on startup when building this OVA - you wouldnt want to winRM into akubelet in production though, that would be insecure!.
- sysprep.ps1: this is the cloudbase init autounattend creation.

image builder works by making a Live VM and configuring it, and then shutting it down and exporting it as an OVA.

Were curently in the process of modifying the disk for that VM...

At this point, we are moddifying local files that we plan to have when we boot this VM.
```
[0;32m    vsphere: Copying files flatly from floppy_files[0m
[0;32m    vsphere: Copying file: ./packer/ova/windows/windows-2019/autounattend.xml[0m
[0;32m    vsphere: Copying file: ./packer/ova/windows/disable-network-discovery.cmd[0m
[0;32m    vsphere: Copying file: ./packer/ova/windows/disable-winrm.ps1[0m
[0;32m    vsphere: Copying file: ./packer/ova/windows/enable-winrm.ps1[0m
[0;32m    vsphere: Copying file: ./packer/ova/windows/sysprep.ps1[0m
[0;32m    vsphere: Done copying files from floppy_files[0m
[0;32m    vsphere: Collecting paths from floppy_dirs[0m
```

Now, let's wait for the VM to come up:
```
[0;32m    vsphere: Resulting paths from floppy_dirs : [./packer/ova/windows/pvscsi][0m
[0;32m    vsphere: Recursively copying : ./packer/ova/windows/pvscsi[0m
[0;32m    vsphere: Done copying paths from floppy_dirs[0m
[0;32m    vsphere: Copying files from floppy_content[0m
[0;32m    vsphere: Done copying files from floppy_content[0m
[1;32m==> vsphere: Uploading created floppy image[0m
[1;32m==> vsphere: Adding generated Floppy...[0m
```

Now the "floppy disk" is configured with all the initial things we need to boot up a minimal windows node and begin
configuring it... So we now wait for an IP and power the virtual machine on. 
```
[1;32m==> vsphere: Set boot order temporary...[0m
[1;32m==> vsphere: Power on VM...[0m
[1;32m==> vsphere: Waiting for IP...[0m
2023/07/22 00:27:04 packer-builder-vsphere-iso plugin: [INFO] Waiting for IP, up to total timeout: 30m0s, settle timeout: 5s
2023/07/22 00:31:24 packer-builder-vsphere-iso plugin: VM IP aquired: 10.221.159.238
2023/07/22 00:31:25 packer-builder-vsphere-iso plugin: VM IP is still the same: 10.221.159.238
2023/07/22 00:31:26 packer-builder-vsphere-iso plugin: VM IP is still the same: 10.221.159.238
2023/07/22 00:31:28 packer-builder-vsphere-iso plugin: VM IP is still the same: 10.221.159.238
2023/07/22 00:31:29 packer-builder-vsphere-iso plugin: VM IP is still the same: 10.221.159.238
2023/07/22 00:31:29 packer-builder-vsphere-iso plugin: VM IP seems stable enough: 10.221.159.238
[1;32m==> vsphere: IP address: 10.221.159.238[0m
[1;32m==> vsphere: Using WinRM communicator to connect: 10.221.159.238[0m
2023/07/22 00:31:29 packer-builder-vsphere-iso plugin: Waiting for WinRM, up to timeout: 4h0m0s
```

Ok, finally, now we can see wether winRM works...

```
[1;32m==> vsphere: Waiting for WinRM to become available...[0m
2023/07/22 00:31:29 packer-builder-vsphere-iso plugin: [INFO] Attempting WinRM connection...
2023/07/22 00:31:29 packer-builder-vsphere-iso plugin: [DEBUG] connecting to remote shell using WinRM
2023/07/22 00:31:34 packer-builder-vsphere-iso plugin: Checking that WinRM is connected with: 'powershell.exe -EncodedCommand JABQAHIAbwBnAHIAZQBzAHMAUAByAGUAZgBlAHIAZQBuAGMAZQAgAD0AIAAnAFMAaQBsAGUAbgB0AGwAeQBDAG8AbgB0AGkAbgB1AGUAJwA7AGkAZgAgACgAVABlAHMAdAAtAFAAYQB0AGgAIAB2AGEAcgBpAGEAYgBsAGUAOgBnAGwAbwBiAGEAbAA6AFAAcgBvAGcAcgBlAHMAcwBQAHIAZQBmAGUAcgBlAG4AYwBlACkAewAkAFAAcgBvAGcAcgBlAHMAcwBQAHIAZQBmAGUAcgBlAG4AYwBlAD0AJwBTAGkAbABlAG4AdABsAHkAQwBvAG4AdABpAG4AdQBlACcAfQA7ACAAZQBjAGgAbwAgACIAVwBpAG4AUgBNACAAYwBvAG4AbgBlAGMAdABlAGQALgAiAA=='
2023/07/22 00:31:34 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell.exe -EncodedCommand JABQAHIAbwBnAHIAZQBzAHMAUAByAGUAZgBlAHIAZQBuAGMAZQAgAD0AIAAnAFMAaQBsAGUAbgB0AGwAeQBDAG8AbgB0AGkAbgB1AGUAJwA7AGkAZgAgACgAVABlAHMAdAAtAFAAYQB0AGgAIAB2AGEAcgBpAGEAYgBsAGUAOgBnAGwAbwBiAGEAbAA6AFAAcgBvAGcAcgBlAHMAcwBQAHIAZQBmAGUAcgBlAG4AYwBlACkAewAkAFAAcgBvAGcAcgBlAHMAcwBQAHIAZQBmAGUAcgBlAG4AYwBlAD0AJwBTAGkAbABlAG4AdABsAHkAQwBvAG4AdABpAG4AdQBlACcAfQA7ACAAZQBjAGgAbwAgACIAVwBpAG4AUgBNACAAYwBvAG4AbgBlAGMAdABlAGQALgAiAA==
[0;32m    vsphere: WinRM connected.[0m
2023/07/22 00:31:35 packer-builder-vsphere-iso plugin: [INFO] command 'powershell.exe -EncodedCommand JABQAHIAbwBnAHIAZQBzAHMAUAByAGUAZgBlAHIAZQBuAGMAZQAgAD0AIAAnAFMAaQBsAGUAbgB0AGwAeQBDAG8AbgB0AGkAbgB1AGUAJwA7AGkAZgAgACgAVABlAHMAdAAtAFAAYQB0AGgAIAB2AGEAcgBpAGEAYgBsAGUAOgBnAGwAbwBiAGEAbAA6AFAAcgBvAGcAcgBlAHMAcwBQAHIAZQBmAGUAcgBlAG4AYwBlACkAewAkAFAAcgBvAGcAcgBlAHMAcwBQAHIAZQBmAGUAcgBlAG4AYwBlAD0AJwBTAGkAbABlAG4AdABsAHkAQwBvAG4AdABpAG4AdQBlACcAfQA7ACAAZQBjAGgAbwAgACIAVwBpAG4AUgBNACAAYwBvAG4AbgBlAGMAdABlAGQALgAiAA==' exited with code: 0
2023/07/22 00:31:35 packer-builder-vsphere-iso plugin: Connected to machine
[1;32m==> vsphere: Connected to WinRM![0m
2023/07/22 00:31:35 packer-builder-vsphere-iso plugin: Running the provision hook
2023/07/22 00:31:35 [INFO] (telemetry) Starting provisioner ansible
[1;32m==> vsphere: Provisioning with Ansible...[0m
[0;32m    vsphere: Not using Proxy adapter for Ansible run:
    vsphere: 	Using WinRM Password from Packer communicator...[0m
    vsphere: 	Using WinRM Password from Packer communicator...[0m
2023/07/22 00:31:35 packer-provisioner-ansible plugin: Creating inventory file for Ansible run...
[1;32m==> vsphere: Executing Ansible: ansible-playbook -e packer_build_name="vsphere" -e packer_builder_type=vsphere-iso -e packer_http_addr=172.17.0.2:0 -e ansible_winrm_server_cert_validation=ignore --extra-vars runtime=containerd docker_ee_version=19.03.12 containerd_url=http://10.221.159.247:30008/files/containerd/cri-containerd-v1.6.18+vmware.1.windows-amd64.tar containerd_sha256=2e0332aa57ebcb6c839a8ec807780d662973a15754573630bea249760cdccf2a pause_image=mcr.microsoft.com/oss/kubernetes/pause:3.6 additional_debug_files="" containerd_additional_settings= custom_role_names="" http_proxy= https_proxy= no_proxy= kubernetes_base_url=http://10.221.159.247:30008/files/kubernetes/ kubernetes_semver=v1.25.7+vmware.2 kubernetes_install_path=c:\k cloudbase_init_url="http://10.221.159.247:30008/files/cloudbase_init/CloudbaseInitSetup_1_1_4_x64.msi" cloudbase_plugins="cloudbaseinit.plugins.windows.createuser.CreateUserPlugin, cloudbaseinit.plugins.common.setuserpassword.SetUserPasswordPlugin, cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin, cloudbaseinit.plugins.common.ephemeraldisk.EphemeralDiskPlugin, cloudbaseinit.plugins.common.mtu.MTUPlugin, cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin,  cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin, cloudbaseinit.plugins.common.userdata.UserDataPlugin, cloudbaseinit.plugins.common.localscripts.LocalScriptsPlugin, cloudbaseinit.plugins.windows.createuser.CreateUserPlugin, cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin" cloudbase_metadata_services="cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService" cloudbase_plugins_unattend="cloudbaseinit.plugins.common.mtu.MTUPlugin" cloudbase_metadata_services_unattend="cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService" prepull=false wins_url= windows_updates_kbs="" windows_updates_categories="CriticalUpdates SecurityUpdates UpdateRollups" windows_service_manager=nssm nssm_url=http://10.221.159.247:30008/files/nssm/nssm.exe distribution_version= netbios_host_name_compatibility=false disable_hypervisor=false cloudbase_logging_serial_port= load_additional_components=true additional_registry_images=false additional_registry_images_list= additional_url_images=false additional_url_images_list= additional_executables=true additional_executables_list=http://10.221.159.247:30008/files/antrea-windows/antrea-windows-advanced.zip,http://10.221.159.247:30008/files/kubernetes/kube-proxy.exe additional_executables_destination_path=C:\ProgramData\Temp ssh_source_url= debug_tools=true --extra-vars  --extra-vars  -e ansible_password=***** -i /tmp/packer-provisioner-ansible1728923704 /home/imagebuilder/ansible/windows/node_windows.yml[0m
```
And now were off to the races.  The rest of these individual ansible roles are all part of the main kubernetes-sigs/image-builder creation
loop for building a windows Kubernetes node....

```


[0;32m    vsphere:[0m
[0;32m    vsphere: PLAY [all] *********************************************************************[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [Gathering Facts] *********************************************************[0m
[0;32m    vsphere: ok: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [Check if cloudbase-init url is set] **************************************[0m
[0;32m    vsphere: ok: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [Check if wins url is set] ************************************************[0m
[0;32m    vsphere: ok: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [Optimise powershell] *****************************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [Get Install Drive] *******************************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [Get Program Files Directory] *********************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [Get All Users profile path] **********************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [Get TEMP Directory] ******************************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [include_role : systemprep] ***********************************************[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Remove Windows updates default registry settings] ***********[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Add Windows update registry path] ***************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Add Windows automatic update registry path] *****************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Disable Windows automatic updates in registry] **************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Set Windows automatic updates to notify only in registry] ***[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Set WinRm Service to delayed start] *************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Update Windows Defender signatures] *************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Install Windows updates based on Categories] ****************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Install OpenSSH] ********************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Set default SSH shell to Powershell] ************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Create SSH program data folder] *****************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Enable ssh login without a password] ************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Set SSH service startup mode to auto and ensure it is started] ***[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Expand dynamic port range to 34000-65535 to avoid port exhaustion] ***[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Add required Windows Features] ******************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Add Hyper-V] ************************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [systemprep : Reboot] *****************************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [include_role : cloudbase-init] *******************************************[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [cloudbase-init : Download Cloudbase-init] ********************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [cloudbase-init : Ensure log directory] ***********************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [cloudbase-init : Install Cloudbase-init] *********************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [cloudbase-init : Set up cloudbase-init unattend configuration] ***********[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [cloudbase-init : Set up cloudbase-init configuration] ********************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [cloudbase-init : Configure set up complete] ******************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [include_role : providers] ************************************************[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [include_role : runtimes] *************************************************[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Download containerd] ******************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Create containerd directory structure] ************************[0m
[0;32m    vsphere: changed: [default] => (item=C:\Program Files\containerd)[0m
```
## image-builder: Setup Containerd

```
[0;32m    vsphere: changed: [default] => (item=C:\\ProgramData\containerd\state)[0m
[0;32m    vsphere: changed: [default] => (item=C:\\ProgramData\containerd\root)[0m
[0;32m    vsphere: changed: [default] => (item=C:/opt/cni/bin)[0m
[0;32m    vsphere: changed: [default] => (item=C:/etc/cni/net.d)[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Check if containerd exists] ***********************************[0m
[0;32m    vsphere: ok: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Unpack containerd binaries] ***********************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Add containerd to path] ***************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Copy containerd config file config.toml] **********************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Check if a Containerd service is installed] *******************[0m
[0;32m    vsphere: ok: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Register Containerd] ******************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Apply SMB Resolution Fix for containerd] **********************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Create Windows Defender Exclusions] ***************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [runtimes : Ensure Containerd Service is running] *************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [include_role : kubernetes] ***********************************************[0m
```


## image-builder: Setup Kubernetes directories

```
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Create kubernetes directory structure] **********************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Download kubernetes binaries] *******************************[0m
[0;32m    vsphere: changed: [default] => (item=kubeadm)[0m
[0;32m    vsphere: changed: [default] => (item=kubectl)[0m
[0;32m    vsphere: changed: [default] => (item=kubelet)[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Add kubernetes folder to path] ******************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Create kubelet directory structure] *************************[0m
[0;32m    vsphere: changed: [default] => (item=C:\var\log\kubelet)[0m
[0;32m    vsphere: changed: [default] => (item=C:\etc\kubernetes)[0m
[0;32m    vsphere: changed: [default] => (item=C:\etc\kubernetes\manifests)[0m
[0;32m    vsphere: changed: [default] => (item=C:\etc\kubernetes\pki)[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Download nssm] **********************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Create kubelet start file for nssm] *************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Install kubelet via nssm] ***********************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Ensure kubelet is installed] ********************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [kubernetes : Add firewall rule for kubelet] ******************************[0m
[0;32m    vsphere: changed: [default][0m
```

## image-builder: Antrea is important 

Its important to understand antrea since it will be something you might need to configure specially on windows nodes.  It's different from antrea on linux.  OVSDB for example that runs as a local windows process might need an SSL library installed on your windowsnode. 

| Feature/Aspect          | Antrea on Linux                                       | Antrea on Windows                                |
|-------------------------|-------------------------------------------------------|--------------------------------------------------|
| **Platform**            | Linux OS                                              | Windows Server w TKG K8s                         |
| **OVS Integration**     | Native OVS support.                                   | Uses OVS Windows port.                           |
| **Networking Mode**     | Uses Linux bridging and routing.                      | Uses OVS for pod-to-pod networking.              |
| **Network Policies**    | Fully supported with OVS.                             | UDP might have issues, also things like antrea egress wont work. loadBalancserSourceRanges largely untested in windows environments   |
| **Runtime**             | Typically container runtimes like containerd, Docker. | Windows Server with containerd, but OVS runs directly on the host and not inside a container. |
| **Supported Protocols** | All protocols supported by OVS on Linux.              | Some protocol limitations due to Windows OS.     |
| **Performance**         | Direct kernel integration offers optimal performance. | May have overhead due to OVS Windows port and Windows networking stack.|



## image-builder: "Additional downloads" i.e. antrea


Next we take the components of the burrito, like antrea, which are non-standard for K8s, and we add them in.
Note that  we can see it pulling antrea from the 10.221.159.247 burrito location ...

```
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [include_role : gmsa] *****************************************************[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [include_role : load_additional_components] *******************************[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [load_additional_components : Create temporary download dir] **************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [load_additional_components : Download additional executables] ************[0m
[0;32m    vsphere: changed: [default] => (item=http://10.221.159.247:30008/files/antrea-windows/antrea-windows-advanced.zip)[0m
[0;32m    vsphere: changed: [default] => (item=http://10.221.159.247:30008/files/kubernetes/kube-proxy.exe)[0m
[0;32m    vsphere:[0m
```
Some other generic windows networking related setup code.  These powershell modules allow us to debug k8s networking issues as they come up.

```
[0;32m    vsphere: TASK [include_role : debug] ****************************************************[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [debug : Add debug tools directory] ***************************************[0m
[0;32m    vsphere: changed: [default][0m
[0;32m    vsphere:[0m
[0;32m    vsphere: TASK [debug : Get debug files] *************************************************[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/microsoft/SDN/raw/master/Kubernetes/windows/debug/collectlogs.ps1)[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/microsoft/SDN/raw/master/Kubernetes/windows/debug/dumpVfpPolicies.ps1)[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/microsoft/SDN/raw/master/Kubernetes/windows/debug/portReservationTest.ps1)[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/microsoft/SDN/raw/master/Kubernetes/windows/debug/starthnstrace.cmd)[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/microsoft/SDN/raw/master/Kubernetes/windows/debug/startpacketcapture.cmd)[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/microsoft/SDN/raw/master/Kubernetes/windows/debug/stoppacketcapture.cmd)[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/Microsoft/SDN/raw/master/Kubernetes/windows/debug/VFP.psm1)[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/microsoft/SDN/raw/master/Kubernetes/windows/helper.psm1)[0m
[0;32m    vsphere: changed: [default] => (item=https://github.com/Microsoft/SDN/raw/master/Kubernetes/windows/hns.psm1)[0m
[0;32m    vsphere: changed: [default] => (item=https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hack/DebugWindowsNode.ps1)[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: PLAY RECAP *********************************************************************[0m
[0;32m    vsphere: default                    : ok=55   changed=50   unreachable=0    failed=0    skipped=40   rescued=0    ignored=0[0m
[0;32m    vsphere:[0m
2023/07/22 01:00:18 [INFO] (telemetry) ending ansible
```

## image-builder: Reboot the node !

Now we can reboot our node since our basic setup is done, and see wether it worked....

```
2023/07/22 01:00:18 [INFO] (telemetry) Starting provisioner windows-restart

[1;32m==> vsphere: Restarting Machine[0m
2023/07/22 01:00:18 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell "& {(Get-WmiObject win32_operatingsystem).LastBootUpTime > C:\ProgramData\lastboot.txt; Restart-Computer -force}"
2023/07/22 01:00:20 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:00:20 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:00:20 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:00:20 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:00:20 packer-builder-vsphere-iso plugin: [INFO] command 'powershell "& {(Get-WmiObject win32_operatingsystem).LastBootUpTime > C:\ProgramData\lastboot.txt; Restart-Computer -force}"' exited with code: 0
2023/07/22 01:00:20 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:00:20 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 01:00:20 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 01:00:20 packer-provisioner-windows-restart plugin: [INFO] RPC client: Communicator ended with: 0
[1;32m==> vsphere: Waiting for machine to restart...[0m
2023/07/22 01:00:20 packer-provisioner-windows-restart plugin: Check if machine is rebooting...
2023/07/22 01:00:20 packer-builder-vsphere-iso plugin: [INFO] starting remote command: shutdown /r /f /t 60 /c "packer restart test"
2023/07/22 01:00:20 packer-builder-vsphere-iso plugin: [INFO] command 'shutdown /r /f /t 60 /c "packer restart test"' exited with code: 1115
2023/07/22 01:00:20 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 1115
2023/07/22 01:00:20 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:00:20 [INFO] 40 bytes written for 'stderr'
2023/07/22 01:00:20 [INFO] RPC client: Communicator ended with: 1115
2023/07/22 01:00:20 [INFO] RPC endpoint: Communicator ended with: 1115
2023/07/22 01:00:20 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 01:00:20 packer-provisioner-windows-restart plugin: [INFO] 40 bytes written for 'stderr'
[1;31m==> vsphere: A system shutdown is in progress.(1115)[0m
2023/07/22 01:00:20 packer-provisioner-windows-restart plugin: [INFO] RPC client: Communicator ended with: 1115
2023/07/22 01:00:20 packer-provisioner-windows-restart plugin: Reboot already in progress, waiting...
2023/07/22 01:00:30 packer-provisioner-windows-restart plugin: Check if machine is rebooting...
2023/07/22 01:01:00 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:01:00 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:01:00 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 01:01:00 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 01:01:00 packer-provisioner-windows-restart plugin: Waiting for machine to reboot with timeout: 5m0s
2023/07/22 01:01:00 packer-provisioner-windows-restart plugin: Waiting for machine to become available...
2023/07/22 01:01:00 packer-provisioner-windows-restart plugin: Checking that communicator is connected with: 'powershell -command "& {if ((get-content C:\ProgramData\lastboot.txt) -eq (Get-WmiObject win32_operatingsystem).LastBootUpTime) {Write-Output 'Sleeping for 600 seconds to wait for reboot'; start-sleep 600} else {Write-Output 'Reboot complete'}}"'
2023/07/22 01:01:35 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:01:35 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:01:35 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 01:01:35 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 01:01:35 packer-provisioner-windows-restart plugin: Communication connection err: unknown error Post "http://10.221.159.238:5985/wsman": dial tcp 10.221.159.238:5985: i/o timeout
2023/07/22 01:02:10 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:02:10 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:10 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 01:02:10 packer-provisioner-windows-restart plugin: Communication connection err: unknown error Post "http://10.221.159.238:5985/wsman": dial tcp 10.221.159.238:5985: i/o timeout
2023/07/22 01:02:10 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:45 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:02:45 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:45 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 01:02:45 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:45 packer-provisioner-windows-restart plugin: Communication connection err: unknown error Post "http://10.221.159.238:5985/wsman": dial tcp 10.221.159.238:5985: i/o timeout
2023/07/22 01:02:50 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell -command "& {if ((get-content C:\ProgramData\lastboot.txt) -eq (Get-WmiObject win32_operatingsystem).LastBootUpTime) {Write-Output 'Sleeping for 600 seconds to wait for reboot'; start-sleep 600} else {Write-Output 'Reboot complete'}}"
2023/07/22 01:02:52 packer-builder-vsphere-iso plugin: [INFO] command 'powershell -command "& {if ((get-content C:\ProgramData\lastboot.txt) -eq (Get-WmiObject win32_operatingsystem).LastBootUpTime) {Write-Output 'Sleeping for 600 seconds to wait for reboot'; start-sleep 600} else {Write-Output 'Reboot complete'}}"' exited with code: 0
2023/07/22 01:02:52 [INFO] 17 bytes written for 'stdout'
2023/07/22 01:02:52 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:02:52 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:52 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:02:52 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:02:52 packer-provisioner-windows-restart plugin: [INFO] 17 bytes written for 'stdout'
2023/07/22 01:02:52 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:52 packer-provisioner-windows-restart plugin: [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:02:52 packer-provisioner-windows-restart plugin: Connected to machine
[0;32m    vsphere: Reboot complete[0m
2023/07/22 01:02:52 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell.exe -EncodedCommand JABQAHIAbwBnAHIAZQBzAHMAUAByAGUAZgBlAHIAZQBuAGMAZQAgAD0AIAAnAFMAaQBsAGUAbgB0AGwAeQBDAG8AbgB0AGkAbgB1AGUAJwA7AGUAYwBoAG8AIAAoACIAewAwAH0AIAByAGUAcwB0AGEAcgB0AGUAZAAuACIAIAAtAGYAIABbAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBEAG4AcwBdADoAOgBHAGUAdABIAG8AcwB0AE4AYQBtAGUAKAApACkA
2023/07/22 01:02:54 packer-builder-vsphere-iso plugin: [INFO] command 'powershell.exe -EncodedCommand JABQAHIAbwBnAHIAZQBzAHMAUAByAGUAZgBlAHIAZQBuAGMAZQAgAD0AIAAnAFMAaQBsAGUAbgB0AGwAeQBDAG8AbgB0AGkAbgB1AGUAJwA7AGUAYwBoAG8AIAAoACIAewAwAH0AIAByAGUAcwB0AGEAcgB0AGUAZAAuACIAIAAtAGYAIABbAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBEAG4AcwBdADoAOgBHAGUAdABIAG8AcwB0AE4AYQBtAGUAKAApACkA' exited with code: 0
2023/07/22 01:02:54 [INFO] 28 bytes written for 'stdout'
2023/07/22 01:02:54 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:02:54 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:54 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:02:54 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:02:54 packer-provisioner-windows-restart plugin: [INFO] 28 bytes written for 'stdout'
2023/07/22 01:02:54 packer-provisioner-windows-restart plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:54 packer-provisioner-windows-restart plugin: [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:02:54 [INFO] (telemetry) ending windows-restart
2023/07/22 01:02:54 [INFO] (telemetry) Starting provisioner goss
[0;32m    vsphere: ADMINIS-HVA1DAR restarted.[0m
[1;32m==> vsphere: Machine successfully restarted, moving on[0m
[1;32m==> vsphere: Provisioning with Goss[0m
[1;32m==> vsphere: Configured to run on Windows[0m

```


## image-builder: GOSS

At this point we have a mostly working VM, but we are going to run goss on it.

```
[0;32m    vsphere: Creating directory: /tmp/goss[0m
2023/07/22 01:02:54 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell /c mkdir -p '/tmp/goss'
2023/07/22 01:02:55 packer-builder-vsphere-iso plugin: [INFO] command 'powershell /c mkdir -p '/tmp/goss'' exited with code: 0
2023/07/22 01:02:55 [INFO] 398 bytes written for 'stdout'
2023/07/22 01:02:55 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:02:55 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:02:55 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:02:55 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:02:55 packer-provisioner-goss plugin: 2023/07/22 01:02:55 [INFO] 398 bytes written for 'stdout'
2023/07/22 01:02:55 packer-provisioner-goss plugin: 2023/07/22 01:02:55 [INFO] 0 bytes written for 'stderr'
[0;32m    vsphere:[0m
2023/07/22 01:02:55 packer-provisioner-goss plugin: 2023/07/22 01:02:55 [INFO] RPC client: Communicator ended with: 0
[0;32m    vsphere:[0m
[0;32m    vsphere:     Directory: C:\tmp[0m
[0;32m    vsphere:[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: Mode                LastWriteTime         Length Name[0m
[0;32m    vsphere: ----                -------------         ------ ----[0m
[0;32m    vsphere: d-----        7/21/2023   6:02 PM                goss[0m
[0;32m    vsphere:[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: Installing Goss from, https://github.com/aelsabbahy/goss/releases/download/v0.3.16/goss-alpha-windows-amd64.exe[0m
[0;32m    vsphere: Downloading Goss to /tmp/goss-0.3.16-windows-amd64.exe[0m
2023/07/22 01:02:55 packer-builder-vsphere-iso plugin: [INFO] starting remote command: curl -L   -o /tmp/goss-0.3.16-windows-amd64.exe https://github.com/aelsabbahy/goss/releases/download/v0.3.16/goss-alpha-windows-amd64.exe || wget   -O /tmp/goss-0.3.16-windows-amd64.exe https://github.com/aelsabbahy/goss/releases/download/v0.3.16/goss-alpha-windows-amd64.exe
[1;31m==> vsphere:   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current[0m
2023/07/22 01:02:57 packer-builder-vsphere-iso plugin: [INFO] command 'curl -L   -o /tmp/goss-0.3.16-windows-amd64.exe https://github.com/aelsabbahy/goss/releases/download/v0.3.16/goss-alpha-windows-amd64.exe || wget   -O /tmp/goss-0.3.16-windows-amd64.exe https://github.com/aelsabbahy/goss/releases/download/v0.3.16/goss-alpha-windows-amd64.exe' exited with code: 0
2023/07/22 01:02:57 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:02:57 [INFO] 640 bytes written for 'stderr'
2023/07/22 01:02:57 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:02:57 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:02:57 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:02:57 packer-provisioner-goss plugin: 2023/07/22 01:02:57 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:02:57 packer-provisioner-goss plugin: 2023/07/22 01:02:57 [INFO] 640 bytes written for 'stderr'
2023/07/22 01:02:57 packer-provisioner-goss plugin: 2023/07/22 01:02:57 [INFO] RPC client: Communicator ended with: 0
[1;31m==> vsphere:                                  Dload  Upload   Total   Spent    Left  Speed[0m
[1;31m==> vsphere:   0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0[0m
[1;31m==> vsphere:   0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0[0m
[1;31m==> vsphere: 100 11.7M  100 11.7M    0     0  8133k      0  0:00:01  0:00:01 --:--:-- 49.4M[0m
2023/07/22 01:02:58 packer-builder-vsphere-iso plugin: [INFO] starting remote command: chmod 555 /tmp/goss-0.3.16-windows-amd64.exe && /tmp/goss-0.3.16-windows-amd64.exe --version
2023/07/22 01:02:58 packer-builder-vsphere-iso plugin: [INFO] command 'chmod 555 /tmp/goss-0.3.16-windows-amd64.exe && /tmp/goss-0.3.16-windows-amd64.exe --version' exited with code: 1
2023/07/22 01:02:58 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 1
2023/07/22 01:02:58 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:02:58 [INFO] 96 bytes written for 'stderr'
2023/07/22 01:02:58 [INFO] RPC client: Communicator ended with: 1
2023/07/22 01:02:58 [INFO] RPC endpoint: Communicator ended with: 1
2023/07/22 01:02:58 packer-provisioner-goss plugin: 2023/07/22 01:02:58 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:02:58 packer-provisioner-goss plugin: 2023/07/22 01:02:58 [INFO] 96 bytes written for 'stderr'
2023/07/22 01:02:58 packer-provisioner-goss plugin: 2023/07/22 01:02:58 [INFO] RPC client: Communicator ended with: 1
[1;31m==> vsphere: 'chmod' is not recognized as an internal or external command,[0m
[1;31m==> vsphere: operable program or batch file.[0m
[1;32m==> vsphere: Uploading goss tests...[0m
[0;32m    vsphere: Uploading vars file packer/goss/goss-vars.yaml[0m
2023/07/22 01:02:58 packer-builder-vsphere-iso plugin: Uploading file to '/tmp/goss/goss-vars.yaml'
2023/07/22 01:02:58 packer-provisioner-goss plugin: 2023/07/22 01:02:58 [INFO] 12779 bytes written for 'uploadData'
2023/07/22 01:02:58 [INFO] 12779 bytes written for 'uploadData'
[0;32m    vsphere: Inline variables are --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''}[0m
[0;32m    vsphere: Env variables are set "GOSS_USE_ALPHA=1" && set "GOSS_MAX_CONCURRENT=1" &&[0m
[0;32m    vsphere: Uploading Dir packer/goss[0m
[0;32m    vsphere: Creating directory: /tmp/goss/goss[0m
2023/07/22 01:03:01 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell /c mkdir -p '/tmp/goss/goss'
2023/07/22 01:03:02 packer-builder-vsphere-iso plugin: [INFO] command 'powershell /c mkdir -p '/tmp/goss/goss'' exited with code: 0
2023/07/22 01:03:02 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:03:02 [INFO] 403 bytes written for 'stdout'
2023/07/22 01:03:02 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:03:02 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:03:02 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:03:02 packer-provisioner-goss plugin: 2023/07/22 01:03:02 [INFO] 403 bytes written for 'stdout'
2023/07/22 01:03:02 packer-provisioner-goss plugin: 2023/07/22 01:03:02 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:03:02 packer-provisioner-goss plugin: 2023/07/22 01:03:02 [INFO] RPC client: Communicator ended with: 0
[0;32m    vsphere:[0m
[0;32m    vsphere:[0m
[0;32m    vsphere:     Directory: C:\tmp\goss[0m
[0;32m    vsphere:[0m
[0;32m    vsphere:[0m
[0;32m    vsphere: Mode                LastWriteTime         Length Name[0m
[0;32m    vsphere: ----                -------------         ------ ----[0m
[0;32m    vsphere: d-----        7/21/2023   6:03 PM                goss[0m
[0;32m    vsphere:[0m
[0;32m    vsphere:[0m
2023/07/22 01:03:02 packer-builder-vsphere-iso plugin: Uploading dir 'packer/goss/' to '/tmp/goss/goss'
[1;32m==> vsphere: 
==> vsphere: 
==> vsphere: 
==> vsphere: Running goss tests...[0m
==> vsphere: 
==> vsphere: 
==> vsphere: Running goss tests...[0m
[1;32m==> vsphere: Running GOSS validate command: cd /tmp/goss &&  set "GOSS_MAX_CONCURRENT=1" && set "GOSS_USE_ALPHA=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} validate --retry-timeout 0s --sleep 1s -f json -o pretty[0m
2023/07/22 01:03:24 packer-builder-vsphere-iso plugin: [INFO] starting remote command: cd /tmp/goss &&  set "GOSS_MAX_CONCURRENT=1" && set "GOSS_USE_ALPHA=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} validate --retry-timeout 0s --sleep 1s -f json -o pretty
2023/07/22 01:04:08 packer-builder-vsphere-iso plugin: [INFO] command 'cd /tmp/goss &&  set "GOSS_MAX_CONCURRENT=1" && set "GOSS_USE_ALPHA=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} validate --retry-timeout 0s --sleep 1s -f json -o pretty' exited with code: 1
[0;32m    vsphere: {[0m
2023/07/22 01:04:08 [INFO] 40090 bytes written for 'stdout'
2023/07/22 01:04:08 [INFO] 0 bytes written for 'stderr'
[0;32m    vsphere:     "results": [[0m
2023/07/22 01:04:08 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 1
2023/07/22 01:04:08 [INFO] RPC client: Communicator ended with: 1
2023/07/22 01:04:08 [INFO] RPC endpoint: Communicator ended with: 1
2023/07/22 01:04:08 packer-provisioner-goss plugin: 2023/07/22 01:04:08 [INFO] 0 bytes written for 'stderr'
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1676969000,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
```
And now we see the packer goss tests running, confirming that various aspects of the filesystem are correct...

```

[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Check HNS Control Flag",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Check HNS Control Flag: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": null,[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Check HNS Control Flag",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 1,[0m
[0;32m    vsphere:             "successful": false,[0m
[0;32m    vsphere:             "summary-line": "Command: Check HNS Control Flag: stdout: patterns not found: [True]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 449385200,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows Port Range is Expanded",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Port Range is Expanded: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "Start Port      : 34000",[0m
[0;32m    vsphere:                 "Number of Ports : 31536"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "Start Port      : 34000",[0m
[0;32m    vsphere:                 "Number of Ports : 31536"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows Port Range is Expanded",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Port Range is Expanded: stdout: matches expectation: [Start Port      : 34000 Number of Ports : 31536]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 2621537300,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Check Windows Defender Exclusions are in place",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Check Windows Defender Exclusions are in place: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "\\Program Files\\containerd\\containerd.exe,",[0m
[0;32m    vsphere:                 "\\Program Files\\containerd\\ctr.exe"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "\\Program Files\\containerd\\containerd.exe,",[0m
[0;32m    vsphere:                 "\\Program Files\\containerd\\ctr.exe"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Check Windows Defender Exclusions are in place",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Check Windows Defender Exclusions are in place: stdout: matches expectation: [\\Program Files\\containerd\\containerd.exe, \\Program Files\\containerd\\ctr.exe]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1783006900,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - containerd",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - containerd: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "Automatic",[0m
[0;32m    vsphere:                 "Running"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "Running",[0m
[0;32m    vsphere:                 "Automatic"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - containerd",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - containerd: stdout: matches expectation: [Automatic Running]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1585575200,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - cloudbase-init",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - cloudbase-init: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "Manual",[0m
[0;32m    vsphere:                 "Stopped"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "Stopped",[0m
[0;32m    vsphere:                 "Manual"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - cloudbase-init",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - cloudbase-init: stdout: matches expectation: [Manual Stopped]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 180293900,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "kubelet --version",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: kubelet --version: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "v1.25.7+vmware.2"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "v1.25.7+vmware.2"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "kubelet --version",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: kubelet --version: stdout: matches expectation: [v1.25.7+vmware.2]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1364138600,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "automatic updates are disabled with correct type",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: automatic updates are disabled with correct type: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "automatic updates are disabled with correct type",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: automatic updates are disabled with correct type: stdout: matches expectation: [True]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1381813900,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Check SMB CompartmentNamespace Flag",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Check SMB CompartmentNamespace Flag: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Check SMB CompartmentNamespace Flag",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Check SMB CompartmentNamespace Flag: stdout: matches expectation: [True]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 152559900,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Correct Containerd config",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Correct Containerd config: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "sandbox_image = \"mcr.microsoft.com/oss/kubernetes/pause:3.6\"",[0m
[0;32m    vsphere:                 "conf_dir = \"C:/etc/cni/net.d\"",[0m
[0;32m    vsphere:                 "bin_dir = \"C:/opt/cni/bin\"",[0m
[0;32m    vsphere:                 "root = \"C:\\\\ProgramData\\\\containerd\\\\root\"",[0m
[0;32m    vsphere:                 "state = \"C:\\\\ProgramData\\\\containerd\\\\state\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "root = \"C:\\\\ProgramData\\\\containerd\\\\root\"",[0m
[0;32m    vsphere:                 "state = \"C:\\\\ProgramData\\\\containerd\\\\state\"",[0m
[0;32m    vsphere:                 "sandbox_image = \"mcr.microsoft.com/oss/kubernetes/pause:3.6\"",[0m
[0;32m    vsphere:                 "bin_dir = \"C:/opt/cni/bin\"",[0m
[0;32m    vsphere:                 "conf_dir = \"C:/etc/cni/net.d\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Correct Containerd config",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Correct Containerd config: stdout: matches expectation: [sandbox_image = \"mcr.microsoft.com/oss/kubernetes/pause:3.6\" conf_dir = \"C:/etc/cni/net.d\" bin_dir = \"C:/opt/cni/bin\" root = \"C:\\\\ProgramData\\\\containerd\\\\root\" state = \"C:\\\\ProgramData\\\\containerd\\\\state\"]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1304619200,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "automatic updates set to notify",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: automatic updates set to notify: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "automatic updates set to notify",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: automatic updates set to notify: stdout: matches expectation: [True]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1543349400,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - sshd",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - sshd: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "Automatic",[0m
[0;32m    vsphere:                 "Running"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "Running",[0m
[0;32m    vsphere:                 "Automatic"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - sshd",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - sshd: stdout: matches expectation: [Automatic Running]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1503919500,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - vmtools",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - vmtools: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "Automatic",[0m
[0;32m    vsphere:                 "Running"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "Running",[0m
[0;32m    vsphere:                 "Automatic"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - vmtools",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - vmtools: stdout: matches expectation: [Automatic Running]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1682697200,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Check WCIFS Flag",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Check WCIFS Flag: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": null,[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Check WCIFS Flag",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 1,[0m
[0;32m    vsphere:             "successful": false,[0m
[0;32m    vsphere:             "summary-line": "Command: Check WCIFS Flag: stdout: patterns not found: [True]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 16475772900,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows Feature - Containers",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Feature - Containers: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "Installed"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "Installed"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows Feature - Containers",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Feature - Containers: stdout: matches expectation: [Installed]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1646857900,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - kubelet",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - kubelet: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "Automatic",[0m
[0;32m    vsphere:                 "/RequiredServices.+:.+(containerd|docker)/"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "/RequiredServices.+:.+(containerd|docker)/",[0m
[0;32m    vsphere:                 "Automatic"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows Service - kubelet",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Service - kubelet: stdout: matches expectation: [Automatic /RequiredServices.+:.+(containerd|docker)/]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1306332700,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "automatic updates are disabled",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: automatic updates are disabled: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "automatic updates are disabled",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: automatic updates are disabled: stdout: matches expectation: [True]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 136150500,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Correct Containerd Version",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Correct Containerd Version: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "v1.6.18"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "v1.6.18"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Correct Containerd Version",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Correct Containerd Version: stdout: matches expectation: [v1.6.18]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 607307700,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "kubectl version --client",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: kubectl version --client: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "v1.25.7+vmware.2",[0m
[0;32m    vsphere:                 "windows",[0m
[0;32m    vsphere:                 "amd64"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "v1.25.7+vmware.2",[0m
[0;32m    vsphere:                 "windows",[0m
[0;32m    vsphere:                 "amd64"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "kubectl version --client",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: kubectl version --client: stdout: matches expectation: [v1.25.7+vmware.2 windows amd64]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 587780700,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "kubeadm version",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: kubeadm version: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "v1.25.7+vmware.2",[0m
[0;32m    vsphere:                 "windows",[0m
[0;32m    vsphere:                 "amd64"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "v1.25.7+vmware.2",[0m
[0;32m    vsphere:                 "windows",[0m
[0;32m    vsphere:                 "amd64"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "kubeadm version",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: kubeadm version: stdout: matches expectation: [v1.25.7+vmware.2 windows amd64]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1405654300,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "automatic updates set to notify with correct type",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: automatic updates set to notify with correct type: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "automatic updates set to notify with correct type",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: automatic updates set to notify with correct type: stdout: matches expectation: [True]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 2207130600,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows Feature - Hyper-V-PowerShell",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Feature - Hyper-V-PowerShell: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "Installed"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "Installed"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows Feature - Hyper-V-PowerShell",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows Feature - Hyper-V-PowerShell: stdout: matches expectation: [Installed]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1378032800,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "0"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exit-status",[0m
[0;32m    vsphere:             "resource-id": "Windows build version is high enough",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows build version is high enough: exit-status: matches expectation: [0]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "True"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "stdout",[0m
[0;32m    vsphere:             "resource-id": "Windows build version is high enough",[0m
[0;32m    vsphere:             "resource-type": "Command",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "Command: Windows build version is high enough: stdout: matches expectation: [True]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exists",[0m
[0;32m    vsphere:             "resource-id": "c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init-unattend.conf",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init-unattend.conf: exists: matches expectation: [true]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1002000,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "\"file\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "\"file\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "filetype",[0m
[0;32m    vsphere:             "resource-id": "c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init-unattend.conf",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init-unattend.conf: filetype: matches expectation: [\"file\"]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 7001200,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "metadata_services=cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "metadata_services=cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "contains",[0m
[0;32m    vsphere:             "resource-id": "c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init-unattend.conf",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init-unattend.conf: contains: matches expectation: [metadata_services=cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exists",[0m
[0;32m    vsphere:             "resource-id": "c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init.conf",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init.conf: exists: matches expectation: [true]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "\"file\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "\"file\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "filetype",[0m
[0;32m    vsphere:             "resource-id": "c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init.conf",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init.conf: filetype: matches expectation: [\"file\"]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 6994500,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "",[0m
[0;32m    vsphere:                 "cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.ephemeraldisk.EphemeralDiskPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.mtu.MTUPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.userdata.UserDataPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.localscripts.LocalScriptsPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.windows.createuser.CreateUserPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "",[0m
[0;32m    vsphere:                 "cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.userdata.UserDataPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.ephemeraldisk.EphemeralDiskPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.mtu.MTUPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.common.localscripts.LocalScriptsPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.windows.createuser.CreateUserPlugin",[0m
[0;32m    vsphere:                 "cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "contains",[0m
[0;32m    vsphere:             "resource-id": "c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init.conf",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/program files/Cloudbase Solutions/Cloudbase-init/conf/cloudbase-init.conf: contains: matches expectation: [ cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService cloudbaseinit.plugins.common.ephemeraldisk.EphemeralDiskPlugin cloudbaseinit.plugins.common.mtu.MTUPlugin cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin cloudbaseinit.plugins.common.userdata.UserDataPlugin cloudbaseinit.plugins.common.localscripts.LocalScriptsPlugin cloudbaseinit.plugins.windows.createuser.CreateUserPlugin cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin]",[0m
[0;32m    vsphere:             "test-type": 2,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exists",[0m
[0;32m    vsphere:             "resource-id": "c:/etc/kubernetes",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/etc/kubernetes: exists: matches expectation: [true]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "\"directory\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "\"directory\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "filetype",[0m
[0;32m    vsphere:             "resource-id": "c:/etc/kubernetes",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/etc/kubernetes: filetype: matches expectation: [\"directory\"]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exists",[0m
[0;32m    vsphere:             "resource-id": "c:/etc/kubernetes/manifests",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
2023/07/22 01:04:08 packer-provisioner-goss plugin: 2023/07/22 01:04:08 [INFO] 40090 bytes written for 'stdout'
2023/07/22 01:04:08 packer-provisioner-goss plugin: 2023/07/22 01:04:08 [INFO] RPC client: Communicator ended with: 1
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/etc/kubernetes/manifests: exists: matches expectation: [true]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "\"directory\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "\"directory\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "filetype",[0m
[0;32m    vsphere:             "resource-id": "c:/etc/kubernetes/manifests",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/etc/kubernetes/manifests: filetype: matches expectation: [\"directory\"]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 1001700,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exists",[0m
[0;32m    vsphere:             "resource-id": "c:/etc/kubernetes/pki",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/etc/kubernetes/pki: exists: matches expectation: [true]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "\"directory\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "\"directory\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "filetype",[0m
[0;32m    vsphere:             "resource-id": "c:/etc/kubernetes/pki",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/etc/kubernetes/pki: filetype: matches expectation: [\"directory\"]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "true"[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "exists",[0m
[0;32m    vsphere:             "resource-id": "c:/var/log/kubelet",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/var/log/kubelet: exists: matches expectation: [true]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         },[0m
[0;32m    vsphere:         {[0m
[0;32m    vsphere:             "duration": 0,[0m
[0;32m    vsphere:             "err": null,[0m
[0;32m    vsphere:             "expected": [[0m
[0;32m    vsphere:                 "\"directory\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "found": [[0m
[0;32m    vsphere:                 "\"directory\""[0m
[0;32m    vsphere:             ],[0m
[0;32m    vsphere:             "human": "",[0m
[0;32m    vsphere:             "meta": null,[0m
[0;32m    vsphere:             "property": "filetype",[0m
[0;32m    vsphere:             "resource-id": "c:/var/log/kubelet",[0m
[0;32m    vsphere:             "resource-type": "File",[0m
[0;32m    vsphere:             "result": 0,[0m
[0;32m    vsphere:             "successful": true,[0m
[0;32m    vsphere:             "summary-line": "File: c:/var/log/kubelet: filetype: matches expectation: [\"directory\"]",[0m
[0;32m    vsphere:             "test-type": 0,[0m
[0;32m    vsphere:             "title": ""[0m
[0;32m    vsphere:         }[0m
[0;32m    vsphere:     ],[0m
[0;32m    vsphere:     "summary": {[0m
[0;32m    vsphere:         "failed-count": 2,[0m
[0;32m    vsphere:         "summary-line": "Count: 58, Failed: 2, Duration: 42.997s",[0m
[0;32m    vsphere:         "test-count": 58,[0m
[0;32m    vsphere:         "total-duration": 42996884700[0m
[0;32m    vsphere:     }[0m
[0;32m    vsphere: }[0m
[1;32m==> vsphere: Goss validate failed[0m
[1;32m==> vsphere: Inspect mode on : proceeding without failing Packer[0m
[1;32m==> vsphere: Running GOSS render command: cd /tmp/goss && set "GOSS_MAX_CONCURRENT=1" && set "GOSS_USE_ALPHA=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} render > /tmp/goss-spec.yaml[0m
2023/07/22 01:04:08 packer-builder-vsphere-iso plugin: [INFO] starting remote command: cd /tmp/goss && set "GOSS_MAX_CONCURRENT=1" && set "GOSS_USE_ALPHA=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} render > /tmp/goss-spec.yaml
2023/07/22 01:04:08 packer-builder-vsphere-iso plugin: [INFO] command 'cd /tmp/goss && set "GOSS_MAX_CONCURRENT=1" && set "GOSS_USE_ALPHA=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} render > /tmp/goss-spec.yaml' exited with code: 0
2023/07/22 01:04:08 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:04:08 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:04:08 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:04:08 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:04:08 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:04:08 packer-provisioner-goss plugin: 2023/07/22 01:04:08 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:04:08 packer-provisioner-goss plugin: 2023/07/22 01:04:08 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:04:08 packer-provisioner-goss plugin: 2023/07/22 01:04:08 [INFO] RPC client: Communicator ended with: 0
[1;32m==> vsphere: Goss render ran successfully[0m
[1;32m==> vsphere: Running GOSS render debug command: cd /tmp/goss && set "GOSS_USE_ALPHA=1" && set "GOSS_MAX_CONCURRENT=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} render -d > /tmp/debug-goss-spec.yaml[0m
2023/07/22 01:04:09 packer-builder-vsphere-iso plugin: [INFO] starting remote command: cd /tmp/goss && set "GOSS_USE_ALPHA=1" && set "GOSS_MAX_CONCURRENT=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} render -d > /tmp/debug-goss-spec.yaml
2023/07/22 01:04:09 packer-builder-vsphere-iso plugin: [INFO] command 'cd /tmp/goss && set "GOSS_USE_ALPHA=1" && set "GOSS_MAX_CONCURRENT=1" &&  /tmp/goss-0.3.16-windows-amd64.exe --gossfile goss/goss.yaml --vars /tmp/goss/goss-vars.yaml --vars-inline {'OS':'windows','PROVIDER':'ova','containerd_version':'v1.6.18','distribution_version':'2019','docker_ee_version':'19.03.12','kubernetes_version':'v1.25.7+vmware.2','pause_image':'mcr.microsoft.com/oss/kubernetes/pause:3.6','runtime':'containerd','ssh_source_url':''} render -d > /tmp/debug-goss-spec.yaml' exited with code: 0
2023/07/22 01:04:09 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:04:09 [INFO] 0 bytes written for 'stderr'
[1;32m==> vsphere: Goss render debug ran successfully[0m
2023/07/22 01:04:09 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:04:09 [INFO] RPC client: Communicator ended with: 0
[1;32m==> vsphere: 
==> vsphere: 
==> vsphere: 
==> vsphere: Downloading spec file and debug info[0m
2023/07/22 01:04:09 [INFO] RPC endpoint: Communicator ended with: 0
[0;32m    vsphere: Downloading Goss specs from, /tmp/goss-spec.yaml and /tmp/debug-goss-spec.yaml to current dir[0m
2023/07/22 01:04:09 packer-provisioner-goss plugin: 2023/07/22 01:04:09 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:04:09 packer-provisioner-goss plugin: 2023/07/22 01:04:09 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:04:09 packer-provisioner-goss plugin: 2023/07/22 01:04:09 [INFO] RPC client: Communicator ended with: 0
==> vsphere: 
==> vsphere: 
==> vsphere: Downloading spec file and debug info[0m
2023/07/22 01:04:10 [INFO] 6319 bytes written for 'downloadWriter'
2023/07/22 01:04:10 packer-provisioner-goss plugin: 2023/07/22 01:04:10 [INFO] 6319 bytes written for 'downloadWriter'
2023/07/22 01:04:12 [INFO] 12954 bytes written for 'downloadWriter'
[1;32m==> vsphere: Provisioning with Powershell...[0m
[1;32m==> vsphere: Provisioning with powershell script: /tmp/powershell-provisioner3064732808[0m
2023/07/22 01:04:12 packer-provisioner-goss plugin: 2023/07/22 01:04:12 [INFO] 12954 bytes written for 'downloadWriter'
2023/07/22 01:04:12 [INFO] (telemetry) ending goss
2023/07/22 01:04:12 [INFO] (telemetry) Starting provisioner powershell
```

## image-builder: cleanup

At this point you can see image builder beggining to clean up after itself.... removing unnecessary logs etc...

```
2023/07/22 01:04:12 packer-provisioner-powershell plugin: Found command: rm -Force -Recurse C:\var\log\kubelet\*
2023/07/22 01:04:12 packer-provisioner-powershell plugin: Opening /tmp/powershell-provisioner3064732808 for reading
2023/07/22 01:04:12 packer-provisioner-powershell plugin: Uploading env vars to c:/Windows/Temp/packer-ps-env-vars-64bb2249-3d18-f23f-8374-3c9e8e1aa78c.ps1
2023/07/22 01:04:12 packer-provisioner-powershell plugin: [INFO] 173 bytes written for 'uploadData'
2023/07/22 01:04:12 [INFO] 173 bytes written for 'uploadData'
2023/07/22 01:04:12 packer-builder-vsphere-iso plugin: Uploading file to 'c:/Windows/Temp/packer-ps-env-vars-64bb2249-3d18-f23f-8374-3c9e8e1aa78c.ps1'
2023/07/22 01:04:15 packer-provisioner-powershell plugin: [INFO] 40 bytes written for 'uploadData'
2023/07/22 01:04:15 [INFO] 40 bytes written for 'uploadData'
2023/07/22 01:04:15 packer-builder-vsphere-iso plugin: Uploading file to 'c:/Windows/Temp/script-64bb2249-df68-4924-800b-1bac089e35d9.ps1'
2023/07/22 01:04:18 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell -executionpolicy bypass "& { if (Test-Path variable:global:ProgressPreference){set-variable -name variable:global:ProgressPreference -value 'SilentlyContinue'};. c:/Windows/Temp/packer-ps-env-vars-64bb2249-3d18-f23f-8374-3c9e8e1aa78c.ps1; &'c:/Windows/Temp/script-64bb2249-df68-4924-800b-1bac089e35d9.ps1'; exit $LastExitCode }"
2023/07/22 01:04:20 packer-builder-vsphere-iso plugin: [INFO] command 'powershell -executionpolicy bypass "& { if (Test-Path variable:global:ProgressPreference){set-variable -name variable:global:ProgressPreference -value 'SilentlyContinue'};. c:/Windows/Temp/packer-ps-env-vars-64bb2249-3d18-f23f-8374-3c9e8e1aa78c.ps1; &'c:/Windows/Temp/script-64bb2249-df68-4924-800b-1bac089e35d9.ps1'; exit $LastExitCode }"' exited with code: 0
2023/07/22 01:04:20 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:04:20 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:04:20 [INFO] 1120 bytes written for 'stderr'
2023/07/22 01:04:20 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:04:20 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:04:20 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 01:04:20 packer-provisioner-powershell plugin: [INFO] 1120 bytes written for 'stderr'
[1;31m==> vsphere: rm : Cannot remove item C:\var\log\kubelet\kubelet.err.log: The process cannot access the file[0m
2023/07/22 01:04:20 packer-provisioner-powershell plugin: [INFO] RPC client: Communicator ended with: 0
[1;31m==> vsphere: 'C:\var\log\kubelet\kubelet.err.log' because it is being used by another process.[0m
[1;31m==> vsphere: At C:\Windows\Temp\script-64bb2249-df68-4924-800b-1bac089e35d9.ps1:1 char:1[0m
[1;31m==> vsphere: + rm -Force -Recurse C:\var\log\kubelet\*[0m
[1;31m==> vsphere: + ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~[0m
[1;31m==> vsphere:     + CategoryInfo          : WriteError: (C:\var\log\kubelet\kubelet.err.log:FileInfo) [Remove-Item], IOException[0m
[1;31m==> vsphere:     + FullyQualifiedErrorId : RemoveFileSystemItemIOError,Microsoft.PowerShell.Commands.RemoveItemCommand[0m
[1;31m==> vsphere: rm : Cannot remove item C:\var\log\kubelet\kubelet.log: The process cannot access the file[0m
[1;31m==> vsphere: 'C:\var\log\kubelet\kubelet.log' because it is being used by another process.[0m
[1;31m==> vsphere: At C:\Windows\Temp\script-64bb2249-df68-4924-800b-1bac089e35d9.ps1:1 char:1[0m
[1;31m==> vsphere: + rm -Force -Recurse C:\var\log\kubelet\*[0m
[1;31m==> vsphere: + ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~[0m
[1;31m==> vsphere:     + CategoryInfo          : WriteError: (C:\var\log\kubelet\kubelet.log:FileInfo) [Remove-Item], IOException[0m
[1;31m==> vsphere:     + FullyQualifiedErrorId : RemoveFileSystemItemIOError,Microsoft.PowerShell.Commands.RemoveItemCommand[0m
2023/07/22 01:04:20 packer-provisioner-powershell plugin: c:/Windows/Temp/script-64bb2249-df68-4924-800b-1bac089e35d9.ps1 returned with exit code 0
2023/07/22 01:04:20 packer-provisioner-powershell plugin: [INFO] 511 bytes written for 'uploadData'
2023/07/22 01:04:20 [INFO] 511 bytes written for 'uploadData'
2023/07/22 01:04:20 packer-builder-vsphere-iso plugin: Uploading file to 'c:/Windows/Temp/packer-cleanup-64bb2249-dfaa-9b9c-9a5c-e08dd811d426.ps1'
2023/07/22 01:04:23 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell -executionpolicy bypass "& { if (Test-Path variable:global:ProgressPreference){set-variable -name variable:global:ProgressPreference -value 'SilentlyContinue'};. c:/Windows/Temp/packer-ps-env-vars-64bb2249-3d18-f23f-8374-3c9e8e1aa78c.ps1; &'c:/Windows/Temp/packer-cleanup-64bb2249-dfaa-9b9c-9a5c-e08dd811d426.ps1'; exit $LastExitCode }"
2023/07/22 01:04:25 packer-builder-vsphere-iso plugin: [INFO] command 'powershell -executionpolicy bypass "& { if (Test-Path variable:global:ProgressPreference){set-variable -name variable:global:ProgressPreference -value 'SilentlyContinue'};. c:/Windows/Temp/packer-ps-env-vars-64bb2249-3d18-f23f-8374-3c9e8e1aa78c.ps1; &'c:/Windows/Temp/packer-cleanup-64bb2249-dfaa-9b9c-9a5c-e08dd811d426.ps1'; exit $LastExitCode }"' exited with code: 0
2023/07/22 01:04:25 packer-builder-vsphere-iso plugin: [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:04:25 [INFO] 0 bytes written for 'stdout'
2023/07/22 01:04:25 [INFO] 0 bytes written for 'stderr'
2023/07/22 01:04:25 [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:04:25 [INFO] RPC endpoint: Communicator ended with: 0
2023/07/22 01:04:25 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stdout'
2023/07/22 01:04:25 packer-provisioner-powershell plugin: [INFO] 0 bytes written for 'stderr'
2023/07/22 01:04:25 packer-provisioner-powershell plugin: [INFO] RPC client: Communicator ended with: 0
2023/07/22 01:04:25 [INFO] (telemetry) ending powershell
[1;32m==> vsphere: Executing shutdown command...[0m
2023/07/22 01:04:25 packer-builder-vsphere-iso plugin: Shutdown command: powershell A:/sysprep.ps1
2023/07/22 01:04:25 packer-builder-vsphere-iso plugin: [INFO] starting remote command: powershell A:/sysprep.ps1
2023/07/22 01:04:25 packer-builder-vsphere-iso plugin: Waiting max 1h0m0s for shutdown to complete
2023/07/22 01:04:34 packer-builder-vsphere-iso plugin: [INFO] command 'powershell A:/sysprep.ps1' exited with code: 0
[1;32m==> vsphere: Deleting Floppy drives...[0m
[1;32m==> vsphere: Deleting Floppy image...[0m
[1;32m==> vsphere: Eject CD-ROM drives...[0m
[1;32m==> vsphere: Convert VM into template...[0m
[0;32m    vsphere: Starting export...[0m
[0;32m    vsphere: Downloading: windows-2019-kube-v1.25.7-disk-0.vmdk[0m
[0;32m    vsphere: Exporting file: windows-2019-kube-v1.25.7-disk-0.vmdk[0m
[0;32m    vsphere: Writing ovf...[0m
[0;32m    vsphere: Creating manifest...[0m
[0;32m    vsphere: Finished exporting...[0m
[1;32m==> vsphere: Clear boot order...[0m
2023/07/22 01:11:10 packer-builder-vsphere-iso plugin: Deleting floppy disk: /tmp/packer499324745
2023/07/22 01:11:11 [INFO] (telemetry) ending vsphere
2023/07/22 01:11:11 [INFO] (telemetry) Starting post-processor manifest
[1;32m==> vsphere: Running post-processor: manifest[0m
2023/07/22 01:11:11 [INFO] (telemetry) ending manifest
2023/07/22 01:11:11 Flagging to keep original artifact from post-processor 'manifest'
[1;32m==> vsphere: Running post-processor: vsphere (type shell-local)[0m
2023/07/22 01:11:11 [INFO] (telemetry) Starting post-processor shell-local
2023/07/22 01:11:11 packer-post-processor-shell-local plugin: [INFO] (shell-local): Prepending inline script with #!/bin/sh -e
[1;32m==> vsphere (shell-local): Running local shell script: /tmp/packer-shell198832943[0m
2023/07/22 01:11:11 packer-post-processor-shell-local plugin: [INFO] (shell-local): starting local command: /bin/sh -c PACKER_BUILDER_TYPE='vsphere-iso' PACKER_BUILD_NAME='vsphere' PACKER_HTTP_ADDR='172.17.0.2:0' PACKER_HTTP_IP='172.17.0.2' PACKER_HTTP_PORT='0'  /tmp/packer-shell198832943
2023/07/22 01:11:11 packer-post-processor-shell-local plugin: [INFO] (shell-local communicator): Executing local shell command [/bin/sh -c PACKER_BUILDER_TYPE='vsphere-iso' PACKER_BUILD_NAME='vsphere' PACKER_HTTP_ADDR='172.17.0.2:0' PACKER_HTTP_IP='172.17.0.2' PACKER_HTTP_PORT='0'  /tmp/packer-shell198832943]
```
## image-builder: making the OVA

```
[0;32m    vsphere (shell-local): Opening OVF source: windows-2019-kube-v1.25.7+vmware.2.ovf[0m
[0;32m    vsphere (shell-local): Opening OVA target: windows-2019-kube-v1.25.7+vmware.2.ova[0m
[0;32m    vsphere (shell-local): Writing OVA package: windows-2019-kube-v1.25.7+vmware.2.ova[0m
[0;32m    vsphere (shell-local): Transfer Completed[0m
[0;32m    vsphere (shell-local): Completed successfully[0m
[0;32m    vsphere (shell-local): image-build-ova: cd .[0m
[0;32m    vsphere (shell-local): image-build-ova: loaded windows-2019-kube-v1.25.7+vmware.2[0m
[0;32m    vsphere (shell-local): image-build-ova: create ovf windows-2019-kube-v1.25.7+vmware.2.ovf[0m
[0;32m    vsphere (shell-local): image-build-ova: creating OVA from windows-2019-kube-v1.25.7+vmware.2.ovf using ovftool[0m
[0;32m    vsphere (shell-local): image-build-ova: create ova checksum windows-2019-kube-v1.25.7+vmware.2.ova.sha256[0m
2023/07/22 01:13:43 [INFO] (telemetry) ending shell-local
[1;32mBuild 'vsphere' finished after 46 minutes 53 seconds.[0m
==> Wait completed after 46 minutes 53 seconds

==> Wait completed after 46 minutes 53 seconds

==> Builds finished. The artifacts of successful builds are:
==> Builds finished. The artifacts of successful builds are:
2023/07/22 01:13:43 machine readable: vsphere,artifact-count []string{"3"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "builder-id", "jetbrains.vsphere"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "id", "windows-2019-kube-v1.25.7"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "string", "windows-2019-kube-v1.25.7"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "files-count", "7"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "file", "0", "output/windows-2019-kube-v1.25.7/packer-manifest.json"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "file", "1", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ova"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "file", "2", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ova.sha256"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "file", "3", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ovf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "file", "4", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7-disk-0.vmdk"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "file", "5", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7.mf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "file", "6", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7.ovf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"0", "end"}
--> vsphere: windows-2019-kube-v1.25.7
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "builder-id", "jetbrains.vsphere"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "id", "windows-2019-kube-v1.25.7"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "string", "windows-2019-kube-v1.25.7"}
--> vsphere: windows-2019-kube-v1.25.7
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "files-count", "7"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "file", "0", "output/windows-2019-kube-v1.25.7/packer-manifest.json"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "file", "1", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ova"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "file", "2", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ova.sha256"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "file", "3", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ovf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "file", "4", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7-disk-0.vmdk"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "file", "5", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7.mf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "file", "6", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7.ovf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"1", "end"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "builder-id", "jetbrains.vsphere"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "id", "windows-2019-kube-v1.25.7"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "string", "windows-2019-kube-v1.25.7"}
--> vsphere: windows-2019-kube-v1.25.7
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "files-count", "7"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "file", "0", "output/windows-2019-kube-v1.25.7/packer-manifest.json"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "file", "1", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ova"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "file", "2", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ova.sha256"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "file", "3", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7+vmware.2.ovf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "file", "4", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7-disk-0.vmdk"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "file", "5", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7.mf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "file", "6", "output/windows-2019-kube-v1.25.7/windows-2019-kube-v1.25.7.ovf"}
2023/07/22 01:13:43 machine readable: vsphere,artifact []string{"2", "end"}
2023/07/22 01:13:43 [INFO] (telemetry) Finalizing.
2023/07/22 01:13:43 waiting for all plugin processes to complete...
2023/07/22 01:13:43 /home/imagebuilder/.local/bin/packer: plugin process exited
2023/07/22 01:13:43 /home/imagebuilder/.packer.d/plugins/packer-provisioner-goss: plugin process exited
2023/07/22 01:13:43 /home/imagebuilder/.local/bin/packer: plugin process exited
2023/07/22 01:13:43 /home/imagebuilder/.local/bin/packer: plugin process exited
2023/07/22 01:13:43 /home/imagebuilder/.local/bin/packer: plugin process exited
2023/07/22 01:13:43 /home/imagebuilder/.local/bin/packer: plugin process exited
2023/07/22 01:13:43 [ERR] Error decoding response stream 19: EOF
2023/07/22 01:13:43 /home/imagebuilder/.local/bin/packer: plugin process exited
```

And thats it.  The ova is now loaded in our windows cluster and can be used by TKG to make a Windows Cluster!

# Now you can make a TKG Cluster !

Follow the guide in https://docs-staging.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.3/using-tkg/workload-clusters-advanced-vsphere.html#windows .  In particular you will
- make an OSImage that is connected to a TKR, and points to the OVA you made above via the `<VERSION>` 
- Create a cluster where the worker nodes use that windows OSImage (by saying you want Windows).


## Windows OSImage debugging

Now, if you get far enough that you didnt get "Couldnt find TKR/OSImage" errors (in other words, your metadata for the TKR and OSImage are correct), you might see this...
when running kubectl get vspheremachines to see why your windows node didnt come up:
```
default      windows-cluster-control-plane-qcpj8-g28wg                   windows-cluster                   true    vsphere://42146165-7eb2-c15d-6f57-d451dbb37110   4m39s                                                                             
default      windows-cluster-md-0-infra-s4t55-z9c6x                      windows-cluster                                                                            4m42s    
```

in this case you might see
```
  - lastTransitionTime: "2023-08-02T15:28:35Z"                                                                                                                                                                                                        
    message: 'unable to find template by name "/dc0/vm/windows-2019-kube-v1.26.5+vmware.1-tkg.1":                                                                                                                                                     
      vm ''/dc0/vm/windows-2019-kube-v1.26.5+vmware.1-tkg.1'' not found'                                                                                                                                                                              
    reason: CloningFailed                                                                                                                                                                                                                             
    severity: Warning                                                                                                                                                                                                                                 
    status: "False"                                                                                                                                                                                                                                   
    type: Ready
```

This would imply that the path you gave to your OSImage was incorrect.  In otherwords, you didnt put the right value in for the `template` of the Windows image.

## CLoud INIT logs

A succesfull run of cloudbase init, when making a cluster can be viewed .  If you dont see your windows nodes coming up , 

ssh into one of your nodes (ssh capv@1.2.3.4).... and run
```
cat "C:\Program Files\Cloudbase-Solutions\Cloudbase-Init\log"
```

You should then see logs which look something like this: 

```
2023-08-09 06:22:49.892 3696 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.osutils.windows.WindowsUtils' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:50.612 3696 DEBUG cloudbaseinit.osutils.windows [-] Checking if service exists: cloudbase-init check_service_exists C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\osutils\windows.py:1097
2023-08-09 06:22:50.612 3696 DEBUG cloudbaseinit.osutils.windows [-] Getting service username: cloudbase-init get_service_username C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\osutils\windows.py:1230
2023-08-09 06:22:50.612 3696 DEBUG cloudbaseinit.osutils.windows [-] Resetting password for service user: .\cloudbase-init reset_service_password C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\osutils\windows.py:1253
2023-08-09 06:22:50.659 3696 DEBUG cloudbaseinit.osutils.windows [-] Setting service credentials: cloudbase-init set_service_credentials C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\osutils\windows.py:1213
2023-08-09 06:22:50.674 3696 INFO cloudbaseinit.init [-] Respawning current process with updated credentials.
2023-08-09 06:22:50.674 3696 DEBUG cloudbaseinit.osutils.windows [-] Creating logon session for user: .\cloudbase-init create_user_logon_session C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\osutils\windows.py:651
2023-08-09 06:22:50.721 3696 DEBUG cloudbaseinit.osutils.windows [-] Executing process as user, command line: ['C:\\Program Files\\Cloudbase Solutions\\Cloudbase-Init\\Python\\Scripts\\cloudbase-init', '--config-file', 'C:\\Program Files\\Cloudbase Solutions\\Cloudbase-Init\\conf\\cloudbase-init.conf', '--noreset_service_password'] execute_process_as_user C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\osutils\windows.py:706
2023-08-09 06:22:52.337 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.osutils.windows.WindowsUtils' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:52.430 2808 INFO cloudbaseinit.init [-] Cloudbase-Init version: 1.1.4
2023-08-09 06:22:52.430 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdata.UserDataPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:52.691 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.createuser.CreateUserPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:52.753 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.setuserpassword.SetUserPasswordPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.263 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.291 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.ephemeraldisk.EphemeralDiskPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.302 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.mtu.MTUPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.338 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.365 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.379 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdata.UserDataPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.384 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.localscripts.LocalScriptsPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.createuser.CreateUserPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 INFO cloudbaseinit.init [-] Executing plugins for stage 'PRE_NETWORKING':
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdata.UserDataPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.createuser.CreateUserPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.setuserpassword.SetUserPasswordPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.ephemeraldisk.EphemeralDiskPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.mtu.MTUPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdata.UserDataPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.localscripts.LocalScriptsPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.createuser.CreateUserPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.410 2808 INFO cloudbaseinit.init [-] Executing plugins for stage 'PRE_METADATA_DISCOVERY':
2023-08-09 06:22:53.410 2808 INFO cloudbaseinit.init [-] Executing plugin 'MTUPlugin'
2023-08-09 06:22:53.410 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.osutils.windows.WindowsUtils' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.423 2808 DEBUG cloudbaseinit.plugins.common.mtu [-] Could not obtain the MTU configuration via DHCP for interface "00:50:56:94:DB:E9" execute C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\plugins\common\mtu.py:45
2023-08-09 06:22:53.423 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.metadata.services.vmwareguestinfoservice.VMwareGuestInfoService' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.493 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.osutils.windows.WindowsUtils' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.629 2808 DEBUG cloudbaseinit.metadata.services.vmwareguestinfoservice [-] Decoding key metadata: encoding base64 _get_guest_data C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\metadata\services\vmwareguestinfoservice.py:97
2023-08-09 06:22:53.701 2808 DEBUG cloudbaseinit.metadata.services.vmwareguestinfoservice [-] Decoding key userdata: encoding base64 _get_guest_data C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\metadata\services\vmwareguestinfoservice.py:97
2023-08-09 06:22:53.703 2808 INFO cloudbaseinit.init [-] Metadata service loaded: 'VMwareGuestInfoService'
2023-08-09 06:22:53.704 2808 INFO cloudbaseinit.init [-] Reporting provisioning started
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.init [-] Instance id: windows-cluster-md-0-lmncw-5d8988948xd8c9q-r8sq9 configure_host C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\init.py:202
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdata.UserDataPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.createuser.CreateUserPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.setuserpassword.SetUserPasswordPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.ephemeraldisk.EphemeralDiskPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.mtu.MTUPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdata.UserDataPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.localscripts.LocalScriptsPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.createuser.CreateUserPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:53.704 2808 INFO cloudbaseinit.init [-] Executing plugins for stage 'MAIN':
2023-08-09 06:22:53.704 2808 INFO cloudbaseinit.init [-] Executing plugin 'UserDataPlugin'
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.plugins.common.userdata [-] User data content length: 9593 execute C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\plugins\common\userdata.py:50
2023-08-09 06:22:53.704 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.utils.template_engine.jinja2_template.Jinja2TemplateEngine' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.151 2808 INFO cloudbaseinit.utils.template_engine.factory [-] Using template engine: jinja
2023-08-09 06:22:54.155 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.parthandler.PartHandlerPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.177 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfig.CloudConfigPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.201 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudboothook.CloudBootHookPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.222 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.shellscript.ShellScriptPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.243 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.multipartmixed.MultipartMixedPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.257 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.heat.HeatPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.270 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.write_files.WriteFilesPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.301 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.set_timezone.SetTimezonePlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.317 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.set_timezone.SetTimezonePlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.317 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.set_hostname.SetHostnamePlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.337 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.set_hostname.SetHostnamePlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.337 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.set_ntp.SetNtpPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.337 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.groups.GroupsPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.368 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.users.UsersPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.381 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.runcmd.RunCmdPlugin' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.408 2808 WARNING cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.write_files [-] Fail to process permissions None, assuming 420
2023-08-09 06:22:54.409 2808 WARNING cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.write_files [-] Fail to process permissions None, assuming 420
2023-08-09 06:22:54.409 2808 WARNING cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.write_files [-] Fail to process permissions None, assuming 420
2023-08-09 06:22:54.409 2808 WARNING cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.write_files [-] Fail to process permissions None, assuming 420
2023-08-09 06:22:54.409 2808 WARNING cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.write_files [-] Fail to process permissions None, assuming 420
2023-08-09 06:22:54.418 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.osutils.windows.WindowsUtils' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:54.603 2808 DEBUG cloudbaseinit.osutils.windows [-] Creating logon session for user: .\capv create_user_logon_session C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\osutils\windows.py:651
2023-08-09 06:22:55.516 2808 INFO cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.users [-] Writing SSH public keys in: C:\Users\capv\.ssh\authorized_keys
2023-08-09 06:22:55.516 2808 INFO cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.runcmd [-] Running cloud-config runcmd entries.
2023-08-09 06:22:55.516 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.osutils.windows.WindowsUtils' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
2023-08-09 06:22:55.516 2808 INFO cloudbaseinit.plugins.common.userdataplugins.cloudconfigplugins.runcmd [-] Found 8 cloud-config runcmd entries.
2023-08-09 06:22:55.532 2808 DEBUG cloudbaseinit.utils.classloader [-] Loading class 'cloudbaseinit.osutils.windows.WindowsUtils' load_class C:\Program Files\Cloudbase Solutions\Cloudbase-Init\Python\lib\site-packages\cloudbaseinit\utils\classloader.py:27
```

## PACKAGES !!

After your node comes up , you might not have a CNI running if theres an antrea issue.  


In that case youll see: 

```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Node
metadata:
  annotations:
    kubeadm.alpha.kubernetes.io/cri-socket: npipe:////./pipe/containerd-containerd
    node.alpha.kubernetes.io/ttl: "0"
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: "2023-08-16T12:01:46Z"
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: windows
    image-type: ova
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: windows-cluster-md-0-lmncw-5d8988948xd8c9q-mvdjn
    kubernetes.io/os: windows
    node.kubernetes.io/windows-build: 10.0.17763
    os-name: windows
    os-type: windows
    run.tanzu.vmware.com/os-image: v1.26.5---vmware.1-tkg.1-windows
  name: windows-cluster-md-0-lmncw-5d8988948xd8c9q-mvdjn
  resourceVersion: "2581482"
  uid: 105dc1c7-0d4e-41a2-bcdf-064e0f0791b0
```
Now reading more of the node YAML , we will see the windows pods arent schedulable.... 
```
spec:
  podCIDR: 100.99.88.0/24
  podCIDRs:
  - 100.99.88.0/24
  taints:
  - effect: NoSchedule
    key: os
    value: windows
  - effect: NoSchedule   ##### <--------------------------- UNINITIALIZED !!!
    key: node.cloudprovider.kubernetes.io/uninitialized
    value: "true"
status:
  addresses:
  - address: 10.221.159.225
    type: InternalIP
  - address: windows-cluster-md-0-lmncw-5d8988948xd8c9q-mvdjn
    type: Hostname
  allocatable:
    cpu: "2"
    ephemeral-storage: "38322513038"
    memory: 4091380Ki
    pods: "110"
  capacity:
    cpu: "2"
    ephemeral-storage: 41582588Ki
```

## What is an uninitialized node? 

Above we can see this on our windows node
```
  taints:
  - effect: NoSchedule
    key: os
    value: windows
  - effect: NoSchedule
    key: node.cloudprovider.kubernetes.io/uninitialized
    value: "true"
status:
```
If you go to https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/cloud-provider/api/well_known_taints.go, then you'll find that 
this happens if 
- the kubelet is started with a `--cloud-controller-manager` option and
- the `--cloud-controller-manager` option has not yet verified that the node is ready

Ok.  So, why would that be?  Looking at the `vsphere-cloud-controller-manager-kjmz5` logs in my workload cluster, I saw:

```
I0816 14:48:00.339503       1 search.go:186] Did not find node windows-cluster-md-0-lmncw-5d8988948xd8c9q-gs2qk in vc=10.89.160.37 and datacenter=Workload 3
E0816 14:48:00.339522       1 nodemanager.go:152] WhichVCandDCByNodeID failed using VM name. Err: No VM found
E0816 14:48:00.339530       1 nodemanager.go:197] shakeOutNodeIDLookup failed. Err=No VM found
E0816 14:48:00.339539       1 node_controller.go:258] Error getting instance metadata for node addresses: error fetching node by provider ID: node not found, and error by node name: node not found
```
Looking closely at the node, we can see the 
- DNS Name: `WIN-VG0GDVSO6FE`
- but cloud provider wants to look up `DNS Name:	windows-cluster-dt6bx-89h92` to identify the node
<img width="1522" alt="image" src="https://github.com/jayunit100/tanzu-install-annotated/assets/826111/ebda913a-0e1e-4566-9020-0e9b61a8a8f0">

A chain of events that can cause this uninitialized taint is as follows: 
- The reason why the node is marked uninitialized, is because the windows *DNS name* is wrong
- The *DNS name* is wrong because *cloud init doesnt complete*
- *cloud init doesnt complete*, because postKubeadmCommands didnt complete
- postKubeadmCommands might not complete if *antrea installation* fails , because we install antrea in postKubeadmLogic!

Thus its likely that Antrea is not installing properly in cases where you see ` node.cloudprovider.kubernetes.io/uninitialized` HOWEVEr, dont
mistake this - its not because the cloudprovider cares about CNI, its just, because the node wont be properly rebooted (which is required to consume DNS cloud-init's DNS fixes) until ALL OF THE postKubeadm commands have finished!

.... Next, we'll troubelshoot why antrea installation might fail in a windows cluster's postKubeadmCommand...

We're going to look at MHCs and how to make them more forgiving, as a hack to let us keep the node on for longer...

But before we do that, you should know here that `Node startup` is defined as "the moment that a kubernetes node has a providerID".
- thus `nodeStartupTimeout` really is should be called to `providerIDWaitTimeout` :)
- The CPI is what adds the providerid + internal IP + external ip to a node
- And  removes taint node.cloudprovider.kubernetes.io/uninitialized from the node
THUS
- our DNS name wont get fixed until cloudbase completes and reboots the node
- if cloudbase-init cant complete
- DNS name wont be correct
- and thus the cloud-controller-manager wont set the `providerId` on a node
- and the unset providerID on the node will result in CAPI reprovisioning the machine
Meaning, its harder for us to debug where the cloudbase-init (root cause of all this) is failing, to begin with, bc the machine
will keep dissapearing. 

## MHC and nodeStartupTimeout 

This is relevant on windows bc the startup is less predictable.
Before we troubleshoot, we must make our windows machinedeployment easier to handle.  CAPV will be (rightfully) recreating our nodes, b/c it will detect that
our machines aren't healthy: So, edit the MHC (on your CLUSTER CLASS) not manually...

For example, I just edit the one that i used to create the cluster, after the fact.... 
```
 kubectl-m edit clusterclass tkg-vsphere-default-v1.1.1
```
And change:
```
        type: object
  workers:
    machineDeployments:
    - class: tkg-worker
      machineHealthCheck:
        maxUnhealthy: 100%
        nodeStartupTimeout: 20m0s
        unhealthyConditions:
        - status: Unknown
          timeout: 5m0s
          type: Ready
        - status: "False"
          timeout: 12m0s
          type: Ready
      strategy:
        type: RollingUpdate
      template:
        bootstrap:
          ref:
            apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
            kind: KubeadmConfigTemplate
            name: tkg-vsphere-default-v1.1.1-md-config
            namespace: default
        infrastructure:
          ref:
            apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
            kind: VSphereMachineTemplate
            name: tkg-vsphere-default-v1.1.1-worker
            namespace: default
        metadata: {}
    - class: tkg-worker-windows ############################### <------------- CHANGE THE tkg-worker-windows 
      machineHealthCheck:
        maxUnhealthy: 100%
        nodeStartupTimeout: 6h40m0s
        unhealthyConditions:
        - status: Unknown
          timeout: 6h40m0s ###### <---------- Give it 6 hours, plenty of time to hack around with your VM without it being recreated !!!!!
          type: Ready
        - status: "False"
          timeout: 6h40m0s
          type: Ready
      strategy:
        type: RollingUpdate
      template:
```
  
That is:

1. node..providerID is set
2. CAPI can match a Machine to a Node and sets Machine.status.nodeRef
3. MHC controller can see that the Node for a Machine exists
According to Stefan Büringer: "if 2 & 3 don't happen the Machine is remediated after nodeStartupTimeout"

```
nodeStartupTimeout: 120m ### <-- important THIS is what will cause CAPI to otherwise, every 20 minutes rebuild your node!
...
  unhealthyConditions:
  - status: Unknown
    timeout: 20m0s <-- make a long time before things are determined as unhealthy
    type: Ready
  - status: "False"
    timeout: 60m0s <-- make a large timeout
    type: Ready
```

This means we'll have an hour or so to debug these nodes before they get recreated... 

## Troubleshooting: Wheres the Antrea installation?

A simple way to start looking for the antrea CNI on our windows nodes is to run 
```
PS C:\Users\capv> Get-ChildItem -Path C:\ -Recurse -File -ErrorAction SilentlyContinue | Select-String -Pattern "antreA"

C:\etc\cni\net.d\10-antrea.conflist:3:    "name": "antrea",
C:\etc\cni\net.d\10-antrea.conflist:6:            "type": "antrea",
C:\k\antrea_cleanup.ps1:1:stop-service antrea-agent -ErrorAction SilentlyContinue
C:\k\antrea_cleanup.ps1:2:C:\k\antrea\Clean-AntreaNetwork.ps1
C:\k\register_antrea_cleanup.ps1:1:$methodScript = "C:\k\antrea\Clean-AntreaNetwork.ps1"
C:\k\register_antrea_cleanup.ps1:3:    $cleanScriptPath = "C:\k\antrea_cleanup.ps1"
```
We can then check to see when these files were made:
```
PS C:\Users\capv> ls C:\etc\cni\
    Directory: C:\etc\cni
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        8/16/2023  10:46 PM                net.d
```
And there is a `/var/log/kubelet`...
```
PS C:\Users\capv> ls C:\var\log\kubelet\kubelet.log
    Directory: C:\var\log\kubelet
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        8/16/2023  10:45 PM              0 kubelet.log
```

In the above we can clearly see: 
- C:\etc\cni\net.d\ is **indeed** written to . that means AT SOME POINT, someone wrote this file out...
- It started **about 1 minute** after the kubelet itself, started

So next, we'll need to find out: What did antrea do, and, why did it stop?.... 

## Go to kubeadmConfig yaml to understand antrea installation 

So why is it that Antrea appears to be ready but ... its not running as a service?  TLDR because you **can trick containerd** by writing a file, and containerd 
will then tell kubelet's that "yeah, my network is setup" 

Well, lets **annotate** the powershell installer for TKGs Windows nodes, which runs as a `postKubeadmCommand`:

- First thing we do is falsely mark the kubelet as ready. If anything this should happen at the end or as late in the flow as possible !!!! otherwise, CNI installation failures wont be flagged by Containerd, and thus, by the kubelet ........

note *SQUEED* has suggested that we move away from monitoring folders like `etc/cni/net.d`, and instead, use https://hackmd.io/@squeed/cri-cni#Defaulting-and-initialization, that is something like:
```
message NetworkStatus {
    // if true, the network is ready to handle
    // sandbox creation and deletion requests
    bool sandbox_management = 1;
    
    // if true, existing sandboxes have connectivity
    bool connectivity = 2;
}
```

If you had an explicit networkStatus object, then the creation of the `C:/etc/cni/net.d` directory would be
an implementation detail, not a semaphore ...


```
2196 Expand-Archive -Force -Path $antreaZipFile -DestinationPath C:\k\antrea
2197 cp C:\k\antrea\bin\antrea-cni.exe C:\opt\cni\bin\antrea.exe -Force
2198 cp C:\k\antrea\bin\host-local.exe C:\opt\cni\bin\host-local.exe -Force

### This set's up antrea so that its the default CNI called by containerd
2199 cp C:\k\antrea\etc\antrea-cni.conflist C:\etc\cni\net.d\10-antrea.conflist -Force #####. <-------- makes the kubelet think CNI is installed
2200
...
2219 ### If this fails, then the kubeAPIServerOverride will never happen
2220 ### And then, antrea-agent will never start
2221 # Wait for antrea-agent token to be ready, the token will be used by Install-AntreaAgent
2222 $AntreaAgentToken = (WaitForSaToken $KubeConfigFile 'antrea-agent') ###### <--------  UHOH !!!!!!!!!!!!!
...

####### This line never reached
####### Parse kube-apiserver address, and use it as the value of "kubeAPIServerOverride" in antrea-agent.conf

2226 $KubeAPIServer = $(kubectl --kubeconfig=$KubeConfigFile config view -o jsonpath='{.clusters[0].cluster.server}') ### <---- THIS never happens
2227 $find = "#kubeAPIServerOverride: `"`""
2228 $replace = "kubeAPIServerOverride: `"$KubeAPIServer`""
2229 $antreaConfigFile = "C:\k\antrea\etc\antrea-agent.conf"
2230 (Get-Content $antreaConfigFile) -replace $find, $replace | Out-File -encoding ASCII $antreaConfigFile
2231
######################################
# RESULT is that this never happens #
######################################
2232 # Install antrea-agent & OVS
2233 Import-Module C:\k\antrea\helper.psm1
2234 & Install-AntreaAgent -KubernetesHome "C:\k" -KubeConfig "C:\etc\kubernetes\kubelet.conf" -AntreaHome "C:\k\antrea" -AntreaVersion "1.11.1"
2235 & C:\k\antrea\Install-OVS.ps1 -ImportCertificate $false -LocalFile C:\k\antrea\ovs-win64.zip
2236
2237 # Setup Services
2238 $nssm = (Get-Command nssm).Source
2239 & $nssm set kubelet start SERVICE_AUTO_START
2240 & $nssm install antrea-agent "C:\k\antrea\bin\antrea-agent.exe" "--config=C:\k\antrea\etc\antrea-agent.conf --logtostderr=false --log_dir=C:\var\log\antrea --alsologtostderr --log_file_max_size=100 --log_file_max_num=4"
2241 & $nssm set antrea-agent DependOnService ovs-vswitchd
2242 & $nssm set antrea-agent Start SERVICE_AUTO_START
2243
2244 # Start Services
2245 start-service kubelet
2246 start-service antrea-agent
2247 - op: add
```
Now, we should note that in a windows kubelet, the Containerd configuration looks like this.


```
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "mcr.microsoft.com/oss/kubernetes/pause:3.6"
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "C:/opt/cni/bin"
      conf_dir = "C:/etc/cni/net.d"
And we do exactly that in our json patches for our kubeadmPostCommands :slightly_smiling_face:
cp C:\k\antrea\etc\antrea-cni.conflist C:\etc\cni\net.d\10-antrea.conflist -Force
```

Ok.  So, the conclusion for this cluster is
- Antrea installation never happened
- Our kubelet has a false sense of readiness , because the CNI dir was created

Next step, you'll need to figure out why Antrea Agent was never started, and Antrea service wasnt installed...... 

## So can we root cause the "Antrea Installation failure" ? 

looking back at the above windows configuration in the `kubeadmConfig` object...

```

...
2219 ### If this fails, then the kubeAPIServerOverride will never happen
2220 ### And then, antrea-agent will never start
2221 # Wait for antrea-agent token to be ready, the token will be used by Install-AntreaAgent
2222 $AntreaAgentToken = (WaitForSaToken $KubeConfigFile 'antrea-agent') ###### <--------  UHOH !!!!!!!!!!!!!
...
```

Theres a possibility that, when we are in the `WaitForSaToken`  function, we never are able to get the `antrea-agent`.  This could be:
- because there is no SA token for antrea-agent or
- because we dont have RBAC on the /etc/kubernetes/kubelet.conf, to access the secrets for the `kube-system` namespace.
- 

Well, maybe we can try.  Lets make an *audit log* and see if anything is failing.  But, you must run that audit log on your WORKLOAD CLUSTER!
Otherwise, if you run it on a management cluster you wont see the failure.  You see:
- The windows workload node needs to talk to the windows-cluster's APIserver.
- If you audit these logs, you'll see that the calls to the secret API is failing

We won't show you how to audit log here, but you can reference this blog post for an example https://jayunit100.blogspot.com/2023/08/getting-audit-policies-to-work.html of how to setup audit logging for a TKG cluster...

## Ok so we used audit logs to find that our kubelet.conf wasnt authorized to read secrets!
Now what? 

To be double sure...

- you can reproduce this failure,by GOING INTO THE windows node, and trying to run `kubeconfig get secrets --kubeconfig=/etc/kubernetes/kubelet.conf`.  
- we can try to simulate what this `WaitForSaToken` command does, by `ssh` into a capv node, and running ...

```
PS C:\Users\capv> kubectl get secrets -A --kubeconfig=/etc/kubernetes/kubelet.conf
E0821 10:42:15.588208   19052 memcache.go:287] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0821 10:42:15.657809   19052 memcache.go:121] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0821 10:42:15.681487   19052 memcache.go:121] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0821 10:42:15.700183   19052 memcache.go:121] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
Error from server (Forbidden): secrets is forbidden: User "system:node:windows-cluster-md-0-lmncw-5d8988948xd8c9q-wk9tv" cannot list resource "secrets" in API group "" at the cluster scope: can only read namespaced object of this type
```
## RBAC Hacky workaround , will it work? 

**MAKE SURE FIRST to change to your WORKLOAD CLUSTER Kubeconfig**
And then you can run this `god rbac` creation snippet.  Just run `kubectl create -f ./god-rbac.yaml` with the file below


```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: superuser-for-testing-only-be-careful
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: superuser-binding
subjects:
- kind: Group
  name: "system:anonymous"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: superuser-for-testing-only-be-careful
  apiGroup: rbac.authorization.k8s.io
```

After making this blob of YAML we will:
- See if there are any apiserver calls that are failing
- Likely fix APIServer calls that are failing bc of the ClusterRoleBinding for ANYONE who is `system:authenticated`.
- now lets create the YAML and see what happens.

now, the API call works....  FROM the windows node.... 

```
PS C:\Users\capv> kubectl get secrets -A --kubeconfig=/etc/kubernetes/kubelet.conf
E0821 10:45:46.280116   17424 memcache.go:287] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0821 10:45:46.290202   17424 memcache.go:121] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0821 10:45:46.310545   17424 memcache.go:121] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0821 10:45:46.331127   17424 memcache.go:121] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
NAMESPACE           NAME                                               TYPE                                  DATA   AGE
kube-system         antctl-service-account-token                       kubernetes.io/service-account-token   3      17d
kube-system         antrea-agent-service-account-token                 kubernetes.io/service-account-token   3      17d
kube-system         cloud-provider-vsphere-credentials                 Opaque                                3      17d
tkg-system          windows-cluster-antrea-data-values                 Opaque                                1      17d
tkg-system          windows-cluster-antrea-fetch-0                     kubernetes.io/dockerconfigjson        1      17d
tkg-system          windows-cluster-capabilities-data-values           Opaque                                1      17d
tkg-system          windows-cluster-capabilities-fetch-0               kubernetes.io/dockerconfigjson        1      17d
tkg-system          windows-cluster-metrics-server-fetch-0             kubernetes.io/dockerconfigjson        1      17d
tkg-system          windows-cluster-pinniped-data-values               Opaque                                1      17d
tkg-system          windows-cluster-pinniped-fetch-0                   kubernetes.io/dockerconfigjson        1      17d
tkg-system          windows-cluster-secretgen-controller-data-values   Opaque                                1      17d
tkg-system          windows-cluster-secretgen-controller-fetch-0       kubernetes.io/dockerconfigjson        1      17d
tkg-system          windows-cluster-tkg-storageclass-data-values       Opaque                                1      17d
tkg-system          windows-cluster-tkg-storageclass-fetch-0           kubernetes.io/dockerconfigjson        1      17d
tkg-system          windows-cluster-vsphere-cpi-data-values            Opaque                                1      17d
tkg-system          windows-cluster-vsphere-cpi-fetch-0                kubernetes.io/dockerconfigjson        1      17d
tkg-system          windows-cluster-vsphere-csi-data-values            Opaque                                1      17d
tkg-system          windows-cluster-vsphere-csi-fetch-0                kubernetes.io/dockerconfigjson        1      17d
vmware-system-csi   vsphere-config-secret                              Opaque                                1      17d
```

## So will CAPV then be able to self heal? 

One way to test all this out will be - let's delete the VM and see if a new one comes up happy , with cloud init working instantly to get the SATOken for antrea.

Note in the TKG 2.3 release, we removed the need for a kube-proxy.exe and kube-proxy.exe SAToken, so now there is only one API call that needs to be made to read secrets from the WorkloadClusters APIServer.

... NEXT.... we'll see what happens when we Delete this vsphere node, and let CAPV make a fresh, new Windows VM that will have a better chance at surviving the postKubeadmCommand !..... (TODO)

## Monitoring cloud controller manager logs

We can start by Looking in cloud contorller manager logs as the node comes up.  Ultimately its the CCM's job to get the node's ID and then set a providerID for it... This providerID is then the key that unlocks cluster API to say "yes, this node is ready to use".... At the start, it turns out that cloud controller manager isnt able to find the corresponding VM...
Run ` kubectl logs   vsphere-cloud-controller-manager-598kw -n kube-system  -f | grep windows` 
and we'll see.... 
```
I0821 18:16:47.098355       1 search.go:186] Did not find node windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg in vc=10.89.160.37 and datacenter=Workload 3
E0821 18:16:47.099503       1 node_controller.go:229] error syncing 'windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg': failed to get provider ID for node windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg at cloudprovider: failed to get instance ID from cloud provider: No VM found, requeuing
...
... we'll wait a few minutes...
...
I0821 18:20:59.248009       1 nodemanager.go:268] Adding Hostname: windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg
I0821 18:20:59.248569       1 nodemanager.go:351] Hostname: windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg UUID: 42142ecb-70a1-c75a-d4f0-9a16fd67a2e4
I0821 18:20:59.283210       1 node_controller.go:484] Successfully initialized node windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg with cloud provider
I0821 18:20:59.284421       1 event.go:294] "Event occurred" object="windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg" fieldPath="" kind="Node" apiVersion="v1" type="Normal" reason="Synced" message="Node synced successfully"
```

Yes !!!! We got our windows node, up and running !!!!!!

## A happy windows node... looks like this!

Now we can look at our happy windows node, with no "uninitialized/ dont schedule here" taints !
```
apiVersion: v1
kind: Node
metadata:
  annotations:
    alpha.kubernetes.io/provided-node-ip: 10.221.159.239
    cluster.x-k8s.io/cluster-name: windows-cluster
    cluster.x-k8s.io/cluster-namespace: default
    cluster.x-k8s.io/labels-from-machine: node.cluster.x-k8s.io/esxi-host
    cluster.x-k8s.io/machine: windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg
    cluster.x-k8s.io/owner-kind: MachineSet
    cluster.x-k8s.io/owner-name: windows-cluster-md-0-lmncw-5d8988948xd8c9q
    kubeadm.alpha.kubernetes.io/cri-socket: npipe:////./pipe/containerd-containerd
    node.alpha.kubernetes.io/ttl: "0"
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: "2023-08-21T18:14:56Z"
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/instance-type: vsphere-vm.cpu-2.mem-4gb.os-win10server
    beta.kubernetes.io/os: windows
    image-type: ova
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg
    kubernetes.io/os: windows
    node.cluster.x-k8s.io/esxi-host: w4-hs3-i0303.eng.vmware.com
    node.kubernetes.io/instance-type: vsphere-vm.cpu-2.mem-4gb.os-win10server
    node.kubernetes.io/windows-build: 10.0.17763
    os-name: windows
    os-type: windows
    run.tanzu.vmware.com/os-image: v1.26.5---vmware.1-tkg.1-windows
  name: windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg
  resourceVersion: "3709770"
  uid: 267265ad-6584-4048-be6f-55c3e5ba9710
spec:
  podCIDR: 100.99.230.0/24
  podCIDRs:
  - 100.99.230.0/24
  providerID: vsphere://42142ecb-70a1-c75a-d4f0-9a16fd67a2e4
  taints:
  - effect: NoSchedule
    key: os
    value: windows
status:
  addresses:
  - address: 10.221.159.239
    type: InternalIP
  - address: 10.221.159.239
    type: ExternalIP
  - address: windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg
    type: Hostname
  allocatable:
    cpu: "2"
    ephemeral-storage: "38322513038"
    memory: 4091380Ki
    pods: "110"
  capacity:
    cpu: "2"
    ephemeral-storage: 41582588Ki
    memory: 4193780Ki
    pods: "110"
  conditions:
  - lastHeartbeatTime: "2023-08-21T18:21:23Z"
    lastTransitionTime: "2023-08-21T18:14:55Z"
    message: kubelet has sufficient memory available
    reason: KubeletHasSufficientMemory
    status: "False"
    type: MemoryPressure
  - lastHeartbeatTime: "2023-08-21T18:21:23Z"
    lastTransitionTime: "2023-08-21T18:14:55Z"
    message: kubelet has no disk pressure
    reason: KubeletHasNoDiskPressure
    status: "False"
    type: DiskPressure
  - lastHeartbeatTime: "2023-08-21T18:21:23Z"
    lastTransitionTime: "2023-08-21T18:14:55Z"
    message: kubelet has sufficient PID available
    reason: KubeletHasSufficientPID
    status: "False"
    type: PIDPressure
  - lastHeartbeatTime: "2023-08-21T18:21:23Z"
    lastTransitionTime: "2023-08-21T18:15:11Z"
    message: kubelet is posting ready status
    reason: KubeletReady
  daemonEndpoints:
    kubeletEndpoint:
      Port: 10250
  images:
  - names:
    - mcr.microsoft.com/oss/kubernetes/pause@sha256:b4b669f27933146227c9180398f99d8b3100637e4a0a1ccf804f8b12f4b9b8df
    - mcr.microsoft.com/oss/kubernetes/pause:3.6
    sizeBytes: 104164397
  nodeInfo:
    architecture: amd64
    bootID: "10"
    containerRuntimeVersion: containerd://1.6.18-1-gdbc99e5b1
    kernelVersion: 10.0.17763.4645
    kubeProxyVersion: v1.26.5+vmware.2
    kubeletVersion: v1.26.5+vmware.2
    machineID: windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg
    operatingSystem: windows
    osImage: Windows Server 2019 Standard
    systemUUID: CB2E1442-A170-5AC7-D4F0-9A16FD67A2E4
```

 Yay!


## Administering a Windows cluster

0) What's the networking model?

TKG Cluster by default use **antrea** as their CNI and **antreaProxy** as their kube proxy implementations.  As of TKG 2.3, we no longer run the kube-proxy.exe for windows.  https://antrea.io/docs/main/docs/antrea-proxy/.

1) How to ssh into nodes?

Like all TKG clusters, you can run `kubectl get nodes -o wide` and run `ssh capv@...` to get into your node.

2) What processes run on the host?

A standard Windows node looks (sometthing like) this... (there are many other processes like svchost and so on
which ive removed for readability)
```
S C:\Users\capv> Get-Process

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
173       8     2044       7388       0.94   1612   0 ovsdb-server
7088      11    12032      19340      52.30   2216   0 ovs-vswitchd
296      20    38652      50844     274.98   2704   0 antrea-agent <--- CNI !!!!
303      22    38488      46280     435.39   1700   0 containerd
193      12    21372      16800       6.70   3144   0 containerd-shim-runhcs-v1
0      16     2380     104296       0.39     88   0 Registry
80       6      928       4572       0.02   2464   3 ServiceMonitor
439      10     3640       8180       0.39    624   0 services
...
160       9     1532       5516       0.19   2680   2 services
53       3      492       1152       0.13    280   0 smss
48       3      484       1172       0.03   2616   0 smss
123      12     1528       7088       0.05   1836   0 sshd
136       9     2136       7440       0.02   4052   0 sshd
130       9     2056       7620       0.03   5616   0 sshd
```

3) How does `antrea-agent` on windows run?

In TKG , we use `nssm` to install antrea-agent as a program that runs locally (antrea-agent.exe).  It uses
the SAToken that was explored above, and:
- It creates IPs for pods, like normal CNIs.
- It also creates OVS routes for services->pods.
- Antrea agent DOES NOT use HNS, instead it uses OVS on windows ! So you wont see HNS rules being written for proxied rules

4) Can I disable **AntreaProxy** ? Yes ! There are two ways to do this.
- You can disable the antrea-proxy when you create a new cluster.
- You can disbable the antrea-proxy after you create a cluster.
However, since **antrea-proxy** runs as a process, you cannot do `kubectl edit antreaconfig...` as you would in a normal cluster. Rather you will need to edit the `C:/k/antrea/etc` configuration file, to disable antrea proxying in the agent.  then You will restart the antrea-agent service.

Run these commands to install VIM on your windows server instance after ssh'ing into it....

```
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

choco install vim
```

Then edit the C:/antrea/etc/antrea-agent.conf file to turn `AntreaProxy: false`.  

Now: we can `Stop-Service *ant*` and then `Start-Service *ant*` and we'll see in the new logs that antreaProxy is NOT enabled.

```
PS C:\Users\capv> Start-Service *ant*
PS C:\Users\capv> ls C:\var\log\antrea\


    Directory: C:\var\log\antrea


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a---l        8/22/2023   5:19 AM              0 antrea-agent.exe.INFO
-a---l        8/22/2023   5:19 AM              0 antrea-agent.exe.WARNING
-a----        8/22/2023   5:20 AM           1715 antrea-agent.exe.windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg.WORKGROUP_WINDOWS-CLUSTER$.log.INFO.20230822-051955.3292
-a----        8/22/2023   5:20 AM            603 antrea-agent.exe.windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg.WORKGROUP_WINDOWS-CLUSTER$.log.WARNING.20230822-051955.3292


PS C:\Users\capv> cat C:\var\log\antrea\antrea-agent.exe.windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg.WORKGROUP_WINDOWS-CLUSTER$.log.INFO.20230822-051955.3292
Log file created at: 2023/08/22 05:19:55
Running on machine: windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg
Binary: Built with gc go1.19.4 for windows/amd64
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I0822 05:19:55.647000    3292 log_file.go:93] Set log file max size to 104857600
W0822 05:19:55.647000    3292 options_windows.go:64] AntreaProxy is not enabled. NetworkPolicies might not be enforced correctly for Service traffic!
I0822 05:19:55.647000    3292 agent.go:98] Starting Antrea agent (version v1.11.1-4776f66.dirty)
W0822 05:19:55.656575    3292 env.go:88] Environment variable POD_NAMESPACE not found
W0822 05:19:55.656575    3292 env.go:126] Failed to get Pod Namespace from environment. Using "kube-system" as the Antrea Service Namespace
I0822 05:19:55.656575    3292 prometheus.go:171] Initializing prometheus metrics
I0822 05:19:55.656575    3292 ovs_client.go:71] Connecting to OVSDB at address \\.\pipe\C:openvswitchvarrunopenvswitchdb.sock
I0822 05:19:55.666637    3292 agent.go:403] Setting up node network
I0822 05:19:55.666637    3292 env.go:56] Environment variable NODE_NAME not found, using hostname instead
I0822 05:19:55.706228    3292 agent.go:1020] "Got Interface MTU" MTU=1450
I0822 05:19:56.507770    3292 net_windows.go:511] Receive Segment Coalescing (RSC) for vSwitch antrea-hnsnetwork is already enabled
I0822 05:19:56.508390    3292 ovs_client.go:114] Bridge exists: 27687748-7c7d-42de-b519-92f78ac4ce46
I0822 05:19:56.568404    3292 agent_windows.go:230] OVS bridge local port br-int already exists, skip the configuration
I0822 05:19:56.568945    3292 agent_windows.go:246] "Uplink already exists, skip the configuration" uplink="Ethernet0" port=3
I0822 05:20:06.528875    3292 net_windows.go:705] "existing netnat in CIDR" name=<

        100.99.230.0/24


 > subnetCIDR="100.99.230.0/24"
I0822 05:20:07.904021    3292 route_windows.go:235] "Added virtual Service IP route" route="LinkIndex: 16, DestinationSubnet: 169.254.0.253/32, GatewayAddress: 0.0.0.0, RouteMetric: 50"
I0822 05:20:09.195483    3292 route_windows.go:248] "Added virtual Service IP neighbor" neighbor="LinkIndex: 16, IPAddress: 169.254.0.253, LinkLayerAddress: aa:bb:cc:dd:ee:ff"
```




5) If Antrea uses OVS, are there still HNSEndpoints for the local pods (note: we dont expect HNSEndpoints for remote pods,... kube-proxies normally make THOSE endpoints) ?

Yes!  As you can see below!
```
NAMESPACE              NAME                                                     READY   STATUS      RESTARTS      AGE     IP               NODE                                               NOMINATED NODE   READINESS GATES
default                nginx-deployment-9f7c4f4f-5hncz                          0/1     Pending     0             16d     <none>           <none>                                             <none>           <none>
default                windows-server-iis-5b4d5c94bd-5dzbm                      1/1     Running     0             16h     100.99.230.2     windows-cluster-md-0-lmncw-5d8988948xd8c9q-hjzmg   <none>           <none>
kube-system            antrea-agent-5x9wb                                       2/2     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            antrea-controller-79d6cc869-4qw5x                        1/1     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            coredns-5656df985f-dxffg                                 1/1     Running     0             17d     100.96.0.4       windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            coredns-5656df985f-kg9cn                                 1/1     Running     0             17d     100.96.0.5       windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            etcd-windows-cluster-dt6bx-89h92                         1/1     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            kube-apiserver-windows-cluster-dt6bx-89h92               1/1     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            kube-controller-manager-windows-cluster-dt6bx-89h92      1/1     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            kube-proxy-hcjr8                                         1/1     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            kube-scheduler-windows-cluster-dt6bx-89h92               1/1     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            kube-vip-windows-cluster-dt6bx-89h92                     1/1     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
kube-system            metrics-server-5d67bcd945-d2xc7                          0/1     Pending     0             17d     <none>           <none>                                             <none>           <none>
kube-system            vsphere-cloud-controller-manager-598kw                   1/1     Running     0             5d19h   10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
secretgen-controller   secretgen-controller-c6649544f-jwjct                     0/1     Pending     0             5d19h   <none>           <none>                                             <none>           <none>
tkg-system             kapp-controller-7bffc94977-fszjh                         2/2     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
tkg-system             tanzu-capabilities-controller-manager-7c84489d6f-4xr4z   1/1     Running     0             17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
vmware-system-antrea   register-placeholder-rmpsr                               0/1     Completed   0             5m10s   100.96.0.194     windows-cluster-dt6bx-89h92                        <none>           <none>
vmware-system-csi      vsphere-csi-controller-b597c9bb4-5z85m                   7/7     Running     0             17d     100.96.0.2       windows-cluster-dt6bx-89h92                        <none>           <none>
vmware-system-csi      vsphere-csi-node-4vk7x                                   3/3     Running     3 (17d ago)   17d     10.221.159.171   windows-cluster-dt6bx-89h92                        <none>           <none>
```

- As we can see above, the `windows-server-iis` endpoint IP is 100.99.230.2....
- If we check HnsEndpoints, we can see that  this endpoint with ID `390DDC46-EE32-4D1D-839D-FB4AFAFC20FC`, exists.
- As described in the antrea docs, OVS is a **forwarding extension** for the Hyper-V switches that are made by **HNS** when pods are created. 
```
PS C:\Users\capv> Get-HnsEndpoint
ActivityId                : 390DDC46-EE32-4D1D-839D-FB4AFAFC20FC
AdditionalParams          :
CreateProcessingStartTime : 133371156758852934
DNSServerList             : 100.64.0.10
DNSSuffix                 : default.svc.cluster.local,svc.cluster.local,cluster.local
EncapOverhead             : 0
GatewayAddress            : 100.99.230.1
Health                    : @{LastErrorCode=0; LastUpdateTime=133371156758777705}
ID                        : 4C09E73E-B94D-4AEF-9E9C-B65A1238BE06
IPAddress                 : 100.99.230.2 <------------------------- THE SAME ENDPOINT EXISTS HERE!!!!!
MacAddress                : 00-15-5D-3B-19-20
Name                      : windows--253c69
Namespace                 : @{ID=CEBBD133-9664-4A9A-825C-0FD83015892F; IsDefault=False}
Policies                  : {}
PrefixLength              : 24
Resources                 : @{AdditionalParams=; AllocationOrder=2; Allocators=System.Object[]; Health=; ID=390DDC46-EE32-4D1D-839D-FB4AFAFC20FC; PortOperationTime=0; State=1;
                            SwitchOperationTime=0; VfpOperationTime=0; parentId=B5128C5E-450C-4480-BFBD-032A10FE3251}
SharedContainers          : {51d90ba63dfade13092ff17b08e5e5a031b6d5aea97ccc7e51c84525cd683137, d31273a1935de3815fdc196f4fcd47bf7c1153866e7e5e4886c4f56d49c5e6d6}
StartTime                 : 133371156846220503
State                     : 3
Type                      : Transparent
Version                   : 38654705669
VirtualNetwork            : 0DCBDE4A-EAD8-4FEE-BEFE-08949F9E5D18
VirtualNetworkName        : antrea-hnsnetwork
```

6) How can I learn more about windows networking?

Check out our sister site, https://windowsnetworking.readthedocs.io/en/latest/ by Daman and Jay !!!

7) Can I modify the way antrea is installed ?

You can do some local experiments on this.  If you  ssh into your windows nodes, then you can go to C:/Temp/ youll see `antrea.ps1` which you can manually run as a powershell command, to see what happens. Note YYMV The `antrea.ps` script in Temp  is quite complex and does quite a few steps.

**Once you like your modification**

You can then create a custom windows clusterclass which define the antrea.ps1 scripts in its own way, deviating from the TKG defaults and use this clusterclass when making new clusters, instead of the default TKG Clusterclasses.


## Antrea on windows flow

The flow for antrea on windows looks like this.  Note the SSL issue, which sometimes can bite you when downloading from fulgan.com.  We have a github issue in place to fix it:
https://github.com/antrea-io/antrea/issues/5479 
![image](https://github.com/jayunit100/tanzu-install-annotated/assets/826111/487b8021-a369-4c79-949c-f4989e75e6e7)

