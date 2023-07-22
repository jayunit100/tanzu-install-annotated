# (((( WIP )))) WIN
....... THIS IS A WIP.......

This article was written on TKG 2.3.0.

# Windows Clusters 
  
Installing a TKG Windows cluster is a good way to learn about TKRs, image-builder, and so on.
Let's go.

## Links

https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-byoi-windows.html 
https://github.com/jaimegag/tkg-zone.git 

(note these instructions include developer build steps so they may not be 100% reproducible for you)

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

```
{
  "additional_executables_destination_path": "C:\\ProgramData\\Temp",                                                                                             
  "additional_executables_list": "http://10.221.159.247:30008/files/antrea-windows/antrea-windows-advanced.zip,http://10.221.159.247:30008/files/kubernetes/kube-proxy.exe",                                                                                                                                                      "additional_executables": "true",                                                                                                                               "additional_prepull_images": "mcr.microsoft.com/windows/servercore:ltsc2019",                                                                                   "build_version": "windows-2019-kube-v1.25.7",                                                                                                                   "cloudbase_init_url": "http://10.221.159.247:30008/files/cloudbase_init/CloudbaseInitSetup_1_1_4_x64.msi",                                                      "cluster": "VSPHERE-CLUSTER-NAME",                                                                                                                              "containerd_sha256_windows": "2e0332aa57ebcb6c839a8ec807780d662973a15754573630bea249760cdccf2a",                                                              
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
