# Kubeadm Config

When clusters bootstrap they have a `KubeadmConfig` object that is created .

```
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfig
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: tkg-mgmt-vc
    cluster.x-k8s.io/control-plane: ""
  name: tkg-mgmt-vc-control-plane-hbhrs
  namespace: tkg-system
```
- It is owned by the *KubeadmControlPlane* object if its a Controlplane node
- It is owned by the *Machine* object if its a worker node
```
  ownerReferences:
  #  kubectl edit kubeadmconfig tkg-mgmt-vc-md-0-hptfc  -n tkg-system ... shows this owner
  - apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: tkg-mgmt-vc-control-plane
    uid: 59a6cc5e-9be7-4a2f-b971-f6db54eb9f31
  - apiVersion: cluster.x-k8s.io/v1beta1
...
spec:
  clusterConfiguration:
    timeoutForControlPlane: 8m0s
    clusterName: tkg-mgmt-vc
    controlPlaneEndpoint: 10.215.33.114:6443
```

Each of the controlplane elements: 
- apiserver
- scheduler
- controller manager
- etcd
... 

... Have specific configuration knobs inside the KubeadmConfig object:

APISErver and KCM 
```
  scheduler:
      extraArgs:
        tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
   apiServer:
      extraArgs:
        cloud-provider: external
        tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
    controllerManager:
      extraArgs:
        cloud-provider: external
        tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
```
Add ons are also 1st class citizens (dns and etcd).
```
    dns:
      imageRepository: projects.registry.vmware.com/tkg
      imageTag: v1.8.4_vmware.9
    etcd:
      local:
        dataDir: /var/lib/etcd
        extraArgs:
          cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
          election-timeout: "2000"
          experimental-initial-corrupt-check: "true"
          heartbeat-interval: "300"
        imageRepository: projects.registry.vmware.com/tkg
        imageTag: v3.5.4_vmware.2
```
Note that these dont allow us to set a securityContext, though, and other fields in etcd or apiserver that
might be settable for a pod arent passed through from the KubeadmInitConfig object.

```
imageRepository: projects.registry.vmware.com/tkg
    kubernetesVersion: v1.22.9+vmware.1
    networking:
      podSubnet: 100.96.0.0/11
      serviceSubnet: 100.64.0.0/13
  initConfiguration:
    localAPIEndpoint: {}
    nodeRegistration:
      criSocket: /var/run/containerd/containerd.sock
      kubeletExtraArgs:
        cloud-provider: external
        tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
      name: '{{ ds.meta_data.hostname }}'
  preKubeadmCommands:
  - hostname "{{ ds.meta_data.hostname }}"
  - echo "::1         ipv6-localhost ipv6-loopback" >/etc/hosts
  - echo "127.0.0.1   localhost" >>/etc/hosts
  - echo "127.0.0.1   {{ ds.meta_data.hostname }}" >>/etc/hosts
  - echo "{{ ds.meta_data.hostname }}" >/etc/hostname
  - sed -i 's|".*/pause|"projects.registry.vmware.com/tkg/pause|' /etc/containerd/config.toml
  - systemctl restart containerd
```
finally, all clusters as we know allow SSH key injection so we can SSH into nodes:
```
  useExperimentalRetryJoin: true
  users:
  - name: capv
    sshAuthorizedKeys:
    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQCZ6M2makyArGMd8lRoodwlAx5tpIaEBaj6l3b/St73WMlJYeDemuWfwPKiOFNQi0LGu751GDPHYRMN+flX8z6mioa9Apuir9f+1f7E9OOcG9R3XAZ5O4rOFbK8CQQDz0snppGUC7cRx7l7/Kr9sepELLj/Vwhb3/g/POl6cyWOmQ==
    sudo: ALL=(ALL) NOPASSWD:ALL
```

## Patching

(untested....) 

You can patch CAPI objects to solve problems that dont have solutions in the existing configuration for KubeadmConfig init.  For example, if I wanted have `etcd` run as a high
level user instead of root, i could modify the following file in /etc/kubernetes/manifests/etcd.yaml... to run like so:
```
 spec:
      containers: 
      - name: etcd
        securityContext:
          runAsUser: 1500 
```
Of course if doing this i'd also need to set the etcd permissions for its root directory... In any case you could 
add this patch to your new clusters like so:
1) modify `preKubeadmCommands` to output a "patch" file to /etc/kubernetes/patches/etcd.yaml
2) Add something like this to the above file:
```

The kind and name fields of the pod will be matched, and then the changes
in the spec.containers... will be applied
```
apiVersion: v1
kind: Pod <-- match 1
  name: etcd <--- match 2 
  namespace: kube-system
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: etcd <-- this will need to match to for the patch to be valid 
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
```

## What is this patches field? 

- introduced in Kubernetes v1.22
- allows you to specify patches to apply to the static pod manifests for
  - the control plane components (kube-apiserver, kube-controller-manager, kube-scheduler, and etcd) during the kubeadm init and kubeadm upgrade operations.
- Each patch = 
  - Directory: the directory where patch files are located.
  - Inline: an array of inline patch spec, where each item in the array needs to specify kind, patch, and target.

The Directory field is a string that specifies the file system path where kubeadm should look for files that contain patches. The files can be either strategic merge patches or JSON 6902 patches, and they must have file extensions .json, .yaml, or .yml.

## kubeadm init?

Thus 
- kubeadm will read the patch files from the directory 
- and apply the patches to the  pod manifest files, matching them  



