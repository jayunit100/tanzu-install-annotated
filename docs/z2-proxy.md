# HTTP_PROXY

## Who needs the HTTP PROXY ? 

When running TKG, the HTTP_PROXY variable is used for restricted internet environments.
- Containerd pulls image through a proxy that is allowed to access the outside world.
- Kubelet talks to the cloud control plane (if running the in-tree CPI, which we do for Azure and AWS) through an HTTP_PROXY
- Kapp Controller pulls down package informations through registries via the HTTP_PROXY

There shouldnt be any other pods (at least, not that i know of) or processes that need to use this HTTP_PROXY.

## What happens to TKG's configuration, when we set HTTP_PROXY ? 

IF we set the HTTP_PROXY variable when creating a cluster like so... 

```
CLUSTER_PLAN: dev
CNI: cluster
TKG_HTTP_PROXY_ENABLED: true <-- 
TKG_HTTP_PROXY: "http://asdf" <-- 
TKG_HTTPS_PROXY: "https://asdf2" <-- 
```

TKG generates alot of changes to the various API objects it makes when you add these 3 parameters.  

The places where TKG plumbs your proxy information are:
- KubeadmControlPlane (containerd and the kubelet)
- KubeadmConfig (containerd and the kubelet)
- Kapp controller (it talkes to external registries, just like contianerd does)
- CPI (it talks to vsphere)
- CSI (it talkes to vsphere) 


# What are the changes ?

Add the Proxy information to the Kubeadm Controlplane...  so that the env vars
are exported into kubeadm before it runs, and so that containerd and  kubelet 
both run with http proxy environment variables.

Interestingly , we can see HTTP_PROXY also is part of the kubeadm startup environment variables ... 
... This might be required because of https://github.com/kubernetes/kubeadm/issues/2765 , wherein 
kubeadm appears to run some commands to pull some images down .  

```
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
spec:
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
      dns:
      etcd:
    files:
    - content: |
    - content: |
        [Service]
        Environment="HTTP_PROXY=http://asdf"
        Environment="HTTPS_PROXY=https://asdf2"
        Environment="NO_PROXY=.svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11"
      owner: root:root
      path: /etc/systemd/system/containerd.service.d/http-proxy.conf
      permissions: "0640"
    - content: |
        [Service]
        Environment="HTTP_PROXY=http://asdf"
        Environment="HTTPS_PROXY=https://asdf2"
        Environment="NO_PROXY=.svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11"
      owner: root:root
      path: /usr/lib/systemd/system/kubelet.service.d/http-proxy.conf
      permissions: "0640"
    name: '{{ ds.meta_data.hostname }}'
    preKubeadmCommands:
    - hostname "{{ ds.meta_data.hostname }}"
    - echo "::1         ipv6-localhost ipv6-loopback" >/etc/hosts
    ...
    - systemctl start containerd
    # https://github.com/kubernetes/kubeadm/issues/2765 <-- possibly the reason why these are exported before running kubeadm on the controlplane
    - export HTTP_PROXY='http://asdf'
    - export HTTPS_PROXY='https://asdf2'
```

And then, to the KubeadmConfigTemplate.  This will make it so that kubeadm configuration for worker nodes
also folows the same proxy conventions as the control plane nodes... 

```
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: cluster-md-0
  namespace: default
spec:
  template:
    spec:
      files:

```

We can see here the two systemd unit files for containerd and kubelet are modified, same as above for the controlplane nodes:
```
      - content: |
          [Service]
          Environment="HTTP_PROXY=http://asdf"
          Environment="HTTPS_PROXY=https://asdf2"
          Environment="NO_PROXY=.svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11"
        owner: root:root
        path: /etc/systemd/system/containerd.service.d/http-proxy.conf
        permissions: "0640"
      - content: |
          [Service]
          Environment="HTTP_PROXY=http://asdf"
          Environment="HTTPS_PROXY=https://asdf2"
          Environment="NO_PROXY=.svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11"
        owner: root:root
        path: /usr/lib/systemd/system/kubelet.service.d/http-proxy.conf
        permissions: "0640"
      preKubeadmCommands:
      - hostname "{{ ds.meta_data.hostname }}"
      - echo "::1         ipv6-localhost ipv6-loopback" >/etc/hosts
      ...
      - export HTTP_PROXY='http://asdf'
      - export HTTPS_PROXY='https://asdf2'
      - export NO_PROXY='.svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11'
      - sed -i 's|".*/pause|"projects.registry.vmware.com/tkg/pause|' /etc/containerd/config.toml
      - systemctl restart containerd
      useExperimentalRetryJoin: true
      users:
      - name: capv
        sshAuthorizedKeys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQCZ6M2makyArGMd8lRoodwlAx5tpIaEBaj6l3b/St73WMlJYeDemuWfwPKiOFNQi0LGu751GDPHYRMN+flX8z6mioa9Apuir9f+1f7E9OOcG9R3XAZ5O4rOFbK8CQQDz0snppGUC7cRx7l7/Kr9sepELLj/Vwhb3/g/POl6cyWOmQ==
        sudo: ALL=(ALL) NOPASSWD:ALL
```

Next, proxy information is added to the CPI... That way when Kapp configures CPI,
the CPI pods always talk to vsphere via an http_proxy... 

```
apiVersion: v1
kind: Secret
metadata:
  annotations:
    tkg.tanzu.vmware.com/addon-type: cloud-provider/vsphere-cpi
  labels:
    tkg.tanzu.vmware.com/addon-name: vsphere-cpi
    tkg.tanzu.vmware.com/cluster-name: cluster
  name: cluster-vsphere-cpi-addon
  namespace: default
stringData:
  values.yaml: |
    #@data/values
    #@overlay/match-child-defaults missing_ok=True
    ---
    vsphereCPI:
      http_proxy: http://asdf
      https_proxy: https://asdf2
      no_proxy: .svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11
type: tkg.tanzu.vmware.com/addon
```

And CSI wants to do the same, b/c CPI is so cool... 

```
---
apiVersion: v1
kind: Secret
metadata:
  annotations:
    tkg.tanzu.vmware.com/addon-type: csi/vsphere-csi
  labels:
    tkg.tanzu.vmware.com/addon-name: vsphere-csi
    tkg.tanzu.vmware.com/cluster-name: cluster
  name: cluster-vsphere-csi-addon
  namespace: default
stringData:
  values.yaml: |
    #@data/values
    #@overlay/match-child-defaults missing_ok=True
    ---
    vsphereCSI:
      http_proxy: http://asdf
      https_proxy: https://asdf2
      no_proxy: .svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11
type: tkg.tanzu.vmware.com/addon
```

Kapp controller also needs to talk to the outer cluster, or maybe the internet even, to pull packages, so it needs
an http proxy: 

```
apiVersion: v1
kind: Secret
metadata:
stringData:
  values.yaml: |
    kappController:
      namespace: tkg-system
      createNamespace: true
      globalNamespace: tanzu-package-repo-global
      image:
      deployment:
      config:
        httpProxy: http://asdf
        httpsProxy: https://asdf2
        noProxy: .svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11
type: tkg.tanzu.vmware.com/addon
```

Finally, the cluster configuration values are all stored as a big secret and we can see
the configuration we sent (for http proxy info) is stored there as well: 

```
apiVersion: v1
kind: Secret
metadata:
  labels:
    clusterctl.cluster.x-k8s.io/move: ""
    tkg.tanzu.vmware.com/cluster-name: cluster
  name: cluster-config-values
  namespace: default
stringData:
  value: |
    CLUSTER_NAME: cluster
    CLUSTER_PLAN: dev
    NAMESPACE: default
    INFRASTRUCTURE_PROVIDER: vsphere
    IS_WINDOWS_WORKLOAD_CLUSTER: false
    SIZE: medium
    CONTROLPLANE_SIZE: null
    WORKER_SIZE: null
    ...
    TKG_CUSTOM_IMAGE_REPOSITORY: projects.registry.vmware.com/tkg
    TKG_CUSTOM_IMAGE_REPOSITORY_SKIP_TLS_VERIFY: false
    TKG_CUSTOM_IMAGE_REPOSITORY_CA_CERTIFICATE: ""
    TKG_HTTP_PROXY: http://asdf
    TKG_HTTPS_PROXY: https://asdf2
    TKG_NO_PROXY: .svc.cluster.local,.svc,localhost,127.0.0.1,100.64.0.0/13,100.96.0.0/11
type: addons.cluster.x-k8s.io/resource-set
```



