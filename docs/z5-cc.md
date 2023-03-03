# CC

TKG 2.1+ Uses clusterclasses by default and we reference them many times here.

## At its core

The simplest possible cluster class you can concieve is below... 

```
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  name: docker-clusterclass-v0.1.0
```

When making CAPI clusters, you need to make several objects:

- ControlPlaneVMs
- Workers
- WorkerVM type 1 (usually a linux node)
- WorkerVM type 2 (maybe a windows node) 
- ... (maybe other types of nodes in your cluster)

Each of (controlpane/worker) needs:

- A VM Template definition
- CPU
- Memory

These are defined in our cluster class here (see the [official clusterclass docs](https://cluster-api.sigs.k8s.io/tasks/experimental-features/cluster-class/write-clusterclass.html)
for details... 

```
spec:
  controlPlane:
    ref:
      ...
    machineInfrastructure:
      ref:
        ...
  infrastructure:
    ref:
       ...
 workers:
    machineDeployments:
    - class: default-worker
      template:
        bootstrap:
          ref:
            ...
    infrastructure:
          ref:
            ...
```

# ClusterClasses


A TKG Cluster class (for vsphere) looks a little more complex.  We can see a few differences...

- There are lots of json patches to the objects
- It uses a different `kind` for the machineInfrastructure, infrastructure fields

```
spec:
  controlPlane:
    machineHealthCheck:
      maxUnhealthy: 100%
      nodeStartupTimeout: 20m0s
      unhealthyConditions:
        ...
    machineInfrastructure:
      ref:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: VSphereMachineTemplate
        name: tkg-vsphere-default-v1.0.0-control-plane
        namespace: default
    metadata: {}
    ref:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta1
      kind: KubeadmControlPlaneTemplate
      name: tkg-vsphere-default-v1.0.0-kcp
      namespace: default
  infrastructure:
    ref:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: VSphereClusterTemplate
      name: tkg-vsphere-default-v1.0.0-cluster
      namespace: default
  patches:
  - definitions:
    - jsonPatches:
      - op: add
        path: /spec/template/spec/kubeadmConfigSpec/clusterConfiguration/apiServer/extraVolumes
        value: []
      selector:
        apiVersion: controlplane.cluster.x-k8s.io/v1beta1
        kind: KubeadmControlPlaneTemplate
        matchResources:
          controlPlane: true
    name: KCP_INIT_APISERVER_EMPTY_EXTRAVOLUMES_ARRAY
```
The JSON PAtches then begin, and theres... alof of them.  These go in and modify the `KubeadmControlPlaneTemplate` and other items.

## JSON PATCHES

```
  - definitions:
    - jsonPatches:
      - op: add
        path: /spec/template/spec/kubeadmConfigSpec/clusterConfiguration/etcd/local/extraArgs
        valueFrom:
          template: |
            {{ $containCipherSuites := false }}
            {{- range $key, $val := .etcdExtraArgs }}
            {{- if eq $key "cipher-suites" }}
              {{- $containCipherSuites = true }}
            {{- end }}
            {{ $key -}} : "{{ $val }}"
            {{- end }}
            {{- if not $containCipherSuites }}
            cipher-suites: "{{ .tlsCipherSuites }}"
            {{- end }}
      selector:
        apiVersion: controlplane.cluster.x-k8s.io/v1beta1
        kind: KubeadmControlPlaneTemplate
        matchResources:
          controlPlane: true
    name: etcdExtraArgs
  - definitions:
    - jsonPatches:
      - op: add
        path: /spec/template/spec/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs
        valueFrom:
          template: |
            {{ $containCipherSuites := false }}
            {{ $containCloudProvider := false }}
            {{- range $key, $val := .apiServerExtraArgs }}
            {{- if eq $key "tls-cipher-suites" }}
              {{- $containCipherSuites = true }}
            {{- end }}
            {{- if eq $key "cloud-provider" }}
... (100s more json patches) ...
```

# JSON PAtches for VSPHERE Clusters

These patches can be divided up like so (this YAML was reformatted using a python script here https://gist.github.com/jayunit100/d63944b2acda1797e1cc63bd07344283) .

There are 4 major patches we apply: 

## KubeadmControlPlaneTemplate

There are around 70 of these... 

```
  selector:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlaneTemplate
    matchResources:
      controlPlane: 'true'
```

## KubeadmConfigTemplate

There are about 25  or so of these...

```
  selector:
    apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
    kind: KubeadmConfigTemplate
    matchResources:
      machineDeploymentClass:
        names:
        - tkg-worker
```

## VsphereMachineTemplate

There are about 40 VSphereMachineTemplate...

```
  selector:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: VSphereMachineTemplate
    matchResources:
      machineDeploymentClass:
        names:
        - tkg-worker
        - tkg-worker-windows
```

## VSphereClusterTemplate

There are 7 of these. 

```
  selector:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: VSphereClusterTemplate
    matchResources:
      infrastructureCluster: 'true'
```


# KubeadmControlPlaneTemplate Patch Details

There are kubeadmConfigSpec changes for two different types of objectS:
- the controlplane nodes
- the worker nodes

There are LOTS of these.... like about 70.... 

These patches all effect the `kubeadmConfigSpec` specifically on the **control plane** nodes.... 
Thus, they have special controlplane related items in them, like:
- RBAC specific additions we enable for APIServers
- Things related to etcd , which only needs to be running on the control plane
- Admission controllers that configure the APISErvers behaviour

```
KubeadmControlPlaneTemplate:
  jsonPatches:
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraVolumes":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraVolumes"
      value: []
```

## etcd related patches

```
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/etcd/local/extraArgs":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/etcd/local/extraArgs"
      valueFrom:
        template: |
          {{ $containCipherSuites := false }}
          {{- range $key, $val := .etcdExtraArgs }}
          {{- if eq $key "cipher-suites" }}
            {{- $containCipherSuites = "true" }}
          {{- end }}
          {{ $key -}} : "{{ $val }}"
          {{- end }}
          {{- if not $containCipherSuites }}
          cipher-suites: "{{ .tlsCipherSuites }}"
          {{- end }}
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs":
```
## apiserver patches

```
   - op: add
     path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs"
      valueFrom:
        template: |
          {{ $containCipherSuites := false }}
          {{ $containCloudProvider := false }}
          {{- range $key, $val := .apiServerExtraArgs }}
          {{- if eq $key "tls-cipher-suites" }}
            {{- $containCipherSuites = "true" }}
          {{- end }}
          {{- if eq $key "cloud-provider" }}
            {{- $containCloudProvider = "true" }}
          {{- end }}
          {{ $key -}} : "{{ $val }}"
          {{- end }}
          {{- if not $containCipherSuites }}
          tls-cipher-suites: "{{ .tlsCipherSuites }}"
          {{- end }}
          {{- if not $containCloudProvider }}
          cloud-provider: external
          {{- end }}
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/scheduler/extraArgs":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/scheduler/extraArgs"
      valueFrom:
        template: |
          {{ $containCipherSuites := false }}
          {{- range $key, $val := .kubeSchedulerExtraArgs }}
          {{- if eq $key "tls-cipher-suites" }}
            {{- $containCipherSuites = "true" }}
          {{- end }}
          {{ $key -}} : "{{ $val }}"
          {{- end }}
          {{- if not $containCipherSuites }}
          tls-cipher-suites: "{{ .tlsCipherSuites }}"
          {{- end }}
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/controllerManager/extraArgs":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/controllerManager/extraArgs"
      valueFrom:
        template: |
          {{ $containCipherSuites := false }}
          {{ $containCloudProvider := false }}
          {{- range $key, $val := .kubeControllerManagerExtraArgs }}
          {{- if eq $key "tls-cipher-suites" }}
            {{- $containCipherSuites = "true" }}
          {{- end }}
          {{- if eq $key "cloud-provider" }}
            {{- $containCloudProvider = "true" }}
          {{- end }}
          {{ $key -}} : "{{ $val }}"
          {{- end }}
          {{- if not $containCipherSuites }}
          tls-cipher-suites: "{{ .tlsCipherSuites }}"
          {{- end }}
          {{- if not $containCloudProvider }}
          cloud-provider: external
          {{- end }}
    "/s/t/s/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs"
      valueFrom:
        template: |
          {{ $containCipherSuites := false }}
          {{ $containCloudProvider := false }}
          {{- range $key, $val := .controlPlaneKubeletExtraArgs }}
          {{- if eq $key "tls-cipher-suites" }}
            {{- $containCipherSuites = "true" }}
          {{- end }}
          {{- if eq $key "cloud-provider" }}
            {{- $containCloudProvider = "true" }}
          {{- end }}
          {{ $key -}} : "{{ $val }}"
          {{- end }}
          {{- if not $containCipherSuites }}
          tls-cipher-suites: "{{ .tlsCipherSuites }}"
          {{- end }}
          {{- if not $containCloudProvider }}
          cloud-provider: external
          {{- end }}
    "/s/t/s/kubeadmConfigSpec/joinConfiguration/nodeRegistration/kubeletExtraArgs":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/joinConfiguration/nodeRegistration/kubeletExtraArgs"
      valueFrom:
        template: |
          {{ $containCipherSuites := false }}
          {{ $containCloudProvider := false }}
          {{- range $key, $val := .controlPlaneKubeletExtraArgs }}
          {{- if eq $key "tls-cipher-suites" }}
            {{- $containCipherSuites = "true" }}
          {{- end }}
          {{- if eq $key "cloud-provider" }}
            {{- $containCloudProvider = "true" }}
          {{- end }}
          {{ $key -}} : "{{ $val }}"
          {{- end }}
          {{- if not $containCipherSuites }}
          tls-cipher-suites: "{{ .tlsCipherSuites }}"
          {{- end }}
          {{- if not $containCloudProvider }}
          cloud-provider: external
          {{- end }}
    "/s/t/s/kubeadmConfigSpec/users":
    - op: replace
      path: "/s/t/s/kubeadmConfigSpec/users"
      valueFrom:
        template: |
          - name: capv
            sshAuthorizedKeys:
            {{- range .user.sshAuthorizedKeys }}
            - ' {{- . -}} '
            {{- end }}
            sudo: ALL=(ALL) NOPASSWD:ALL
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/imageRepository":
    - op: replace
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/imageRepository"
      valueFrom:
        template: "{{(index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.imageRepository}}"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/etcd/local/imageRepository":
    - op: replace
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/etcd/local/imageRepository"
      valueFrom:
        template: "{{(index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.imageRepository}}"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/etcd/local/imageTag":
    - op: replace
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/etcd/local/imageTag"
      valueFrom:
        template: "{{(index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.etcd.imageTag}}"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/dns/imageRepository":
    - op: replace
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/dns/imageRepository"
      valueFrom:
        template: "{{(index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.imageRepository}}"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/dns/imageTag":
    - op: replace
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/dns/imageTag"
      valueFrom:
        template: "{{(index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.coredns.imageTag}}"
    "/s/t/s/kubeadmConfigSpec/files/-":
```
## VIP configuration
```
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      valueFrom:
        template: |
          owner: root:root
          path: /etc/kubernetes/manifests/kube-vip.yaml
          content: |
            ---
            apiVersion: v1
            kind: Pod
            metadata:
              creationTimestamp: null
              name: kube-vip
              namespace: kube-system
            spec:
              containers:
              - args:
                - manager
                env:
                - name: cp_enable
                  value: "true"
                - name: svc_enable
                  value: "{{ .kubeVipLoadBalancerProvider }}"
                - name: vip_arp
                  value: "true"
                - name: vip_leaderelection
                  value: "true"
                - name: address
                  value: {{ .apiServerEndpoint }}
                {{- if and (not .aviControlPlaneHAProvider) .apiServerPort }}
                - name: port
                  value: "{{ .apiServerPort }}"
                {{- end }}
                - name: vip_interface
                  value: {{ .vipNetworkInterface }}
                - name: vip_leaseduration
                  value: "30"
                - name: vip_renewdeadline
                  value: "20"
                - name: vip_retryperiod
                  value: "4"
                image: {{(index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.imageRepository}}/kube-vip:{{(index (index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec "kube-vip").imageTag}}
                imagePullPolicy: IfNotPresent
                name: kube-vip
                resources: {}
                securityContext:
                  capabilities:
                    add:
                    - NET_ADMIN
                    - NET_RAW
                volumeMounts:
                - mountPath: /etc/kubernetes/admin.conf
                  name: kubeconfig
              hostNetwork: "true"
              hostAliases:
              - hostnames:
                - kubernetes
                ip: 127.0.0.1
              volumes:
              - hostPath:
                  path: /etc/kubernetes/admin.conf
                  type: FileOrCreate
                name: kubeconfig
            status: {}
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      value:
        content: ''
        owner: root:root
        path: "/etc/sysconfig/kubelet"
        permissions: '0640'
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      valueFrom:
        template: |
          content: |
            [Service]
            Environment="HTTP_PROXY= {{- .proxy.httpProxy -}} "
            Environment="HTTPS_PROXY= {{- .proxy.httpsProxy -}} "
            Environment="NO_PROXY= {{- list "localhost" "127.0.0.1" ".svc" ".svc.cluster.local" ((list "IPv6" "DualStack" | has .builtin.cluster.network.ipFamily) | ternary  "::1" nil) | concat .proxy.noProxy .builtin.cluster.network.services .builtin.cluster.network.pods | uniq | sortAlpha | join "," -}} "
          owner: root:root
          path: /etc/systemd/system/containerd.service.d/http-proxy.conf
          permissions: "0640"
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      valueFrom:
        template: |
          content: |
            [Service]
            Environment="HTTP_PROXY= {{- .proxy.httpProxy -}} "
            Environment="HTTPS_PROXY= {{- .proxy.httpsProxy -}} "
            Environment="NO_PROXY= {{- list "localhost" "127.0.0.1" ".svc" ".svc.cluster.local" ((list "IPv6" "DualStack" | has .builtin.cluster.network.ipFamily) | ternary  "::1" nil) | concat .proxy.noProxy .builtin.cluster.network.services .builtin.cluster.network.pods | uniq | sortAlpha | join "," -}} "
          owner: root:root
          path: /usr/lib/systemd/system/kubelet.service.d/http-proxy.conf
          permissions: "0640"
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      valueFrom:
        template: |
          path: /etc/ssl/certs/tkg-custom-ca.pem
          {{- $proxy := "" }}
          {{- range .trust.additionalTrustedCAs }}
            {{- if eq .name "proxy" }}
              {{- $proxy = .data }}
            {{- end }}
          {{- end }}
          content: {{ $proxy }}
          encoding: base64
          permissions: "0444"
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      valueFrom:
        template: |
          path: /etc/containerd/ {{- index (or .imageRepository.host (index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.imageRepository | splitList "/") 0 -}} .crt
          {{- $proxy := "" }}
          {{- $image := "" }}
          {{- range .trust.additionalTrustedCAs }}
            {{- if eq .name "proxy" }}
              {{- $proxy = .data }}
            {{- end }}
            {{- if eq .name "imageRepository" }}
              {{- $image = .data }}
            {{- end }}
          {{- end }}
          content: {{or $proxy $image}}
          encoding: base64
          permissions: "0444"
```

## Control Plane RBAC !!!

One of the largest patches is the creation of RBAC rules for the kubeadm configuration: 

```
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      value:
        content: |
          ---
          apiVersion: audit.k8s.io/v1
          kind: Policy
          rules:
            #! The following requests were manually identified as high-volume and low-risk,
            #! so drop them.
            - level: None
              users: ["system:serviceaccount:kube-system:kube-proxy"]
              verbs: ["watch"]
              resources:
                - group: "" #! core
                  resources: ["endpoints", "services", "services/status"]
            - level: None
              userGroups: ["system:nodes"]
              verbs: ["get"]
              resources:
                - group: "" #! core
                  resources: ["nodes", "nodes/status"]
            - level: None
              users:
                - system:kube-controller-manager
                - system:kube-scheduler
                - system:serviceaccount:kube-system:endpoint-controller
              verbs: ["get", "update"]
              namespaces: ["kube-system"]
              resources:
                - group: "" #! core
                  resources: ["endpoints"]
            - level: None
              users: ["system:apiserver"]
              verbs: ["get"]
              resources:
                - group: "" #! core
                  resources: ["namespaces", "namespaces/status", "namespaces/finalize"]
            #! Don't log HPA fetching metrics.
            - level: None
              users:
                - system:kube-controller-manager
              verbs: ["get", "list"]
              resources:
                - group: "metrics.k8s.io"
            #! Don't log these read-only URLs.
            - level: None
              nonResourceURLs:
                - /healthz*
                - /version
                - /swagger*
            #! Don't log events requests.
            - level: None
              resources:
                - group: "" #! core
                  resources: ["events"]
            #! Don't log TMC service account performing read operations because they are high-volume.
            - level: None
              userGroups: ["system:serviceaccounts:vmware-system-tmc"]
              verbs: ["get", "list", "watch"]
            #! Don't log read requests from garbage collector because they are high-volume.
            - level: None
              users: ["system:serviceaccount:kube-system:generic-garbage-collector"]
              verbs: ["get", "list", "watch"]
            #! node and pod status calls from nodes are high-volume and can be large, don't log responses for expected updates from nodes
            - level: Request
              userGroups: ["system:nodes"]
              verbs: ["update","patch"]
              resources:
                - group: "" #! core
                  resources: ["nodes/status", "pods/status"]
              omitStages:
                - "RequestReceived"
            #! deletecollection calls can be large, don't log responses for expected namespace deletions
            - level: Request
              users: ["system:serviceaccount:kube-system:namespace-controller"]
              verbs: ["deletecollection"]
              omitStages:
                - "RequestReceived"
            #! Secrets, ConfigMaps, and TokenReviews can contain sensitive & binary data,
            #! so only log at the Metadata level.
            - level: Metadata
              resources:
                - group: "" #! core
                  resources: ["secrets", "configmaps"]
                - group: authentication.k8s.io
                  resources: ["tokenreviews"]
              omitStages:
                - "RequestReceived"
            #! Get repsonses can be large; skip them.
            - level: Request
              verbs: ["get", "list", "watch"]
              resources:
                - group: "" #! core
                - group: "admissionregistration.k8s.io"
                - group: "apiextensions.k8s.io"
                - group: "apiregistration.k8s.io"
                - group: "apps"
                - group: "authentication.k8s.io"
                - group: "authorization.k8s.io"
                - group: "autoscaling"
                - group: "batch"
                - group: "certificates.k8s.io"
                - group: "extensions"
                - group: "metrics.k8s.io"
                - group: "networking.k8s.io"
                - group: "policy"
                - group: "rbac.authorization.k8s.io"
                - group: "settings.k8s.io"
                - group: "storage.k8s.io"
              omitStages:
                - "RequestReceived"
            #! Default level for known APIs
            - level: RequestResponse
              resources:
                - group: "" #! core
                - group: "admissionregistration.k8s.io"
                - group: "apiextensions.k8s.io"
                - group: "apiregistration.k8s.io"
                - group: "apps"
                - group: "authentication.k8s.io"
                - group: "authorization.k8s.io"
                - group: "autoscaling"
                - group: "batch"
                - group: "certificates.k8s.io"
                - group: "extensions"
                - group: "metrics.k8s.io"
                - group: "networking.k8s.io"
                - group: "policy"
                - group: "rbac.authorization.k8s.io"
                - group: "settings.k8s.io"
                - group: "storage.k8s.io"
              omitStages:
                - "RequestReceived"
            #! Default level for all other requests.
            - level: Metadata
              omitStages:
                - "RequestReceived"
        owner: root:root
        path: "/etc/kubernetes/audit-policy.yaml"
        permissions: '0600'
```
## Admission control
```
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      valueFrom:
        template: |-
          path: /etc/kubernetes/admission-control-config.yaml
          content: |-
            apiVersion: apiserver.config.k8s.io/v1
            kind: AdmissionConfiguration
            plugins:
            {{- if and (not .podSecurityStandard.deactivated) (semverCompare ">= v1.24" .builtin.controlPlane.version) }}
            {{ $namespace_exemptions := printf "%q, %q" "kube-system" "tkg-system" -}}
            {{ $defaultWarnAudit := "baseline" }}
            {{- if .podSecurityStandard.exemptions.namespaces -}}
              {{ range $namespace := .podSecurityStandard.exemptions.namespaces -}}
                {{ $namespace_exemptions = printf "%s, %q" $namespace_exemptions $namespace -}}
              {{- end -}}
            {{- end -}}
            - name: PodSecurity
              configuration:
                apiVersion: pod-security.admission.config.k8s.io/v1beta1
                kind: PodSecurityConfiguration
                defaults:
                  enforce: "{{ if .podSecurityStandard.enforce -}}
                      {{ .podSecurityStandard.enforce }}
                    {{- end }}"
                  enforce-version: "{{ .podSecurityStandard.enforceVersion -}}"
                  audit: "{{ if .podSecurityStandard.audit -}}
                      {{ .podSecurityStandard.audit }}
                    {{- else -}}
                      {{ $defaultWarnAudit }}
                    {{- end }}"
                  audit-version: "{{ .podSecurityStandard.auditVersion -}}"
                  warn: "{{ if .podSecurityStandard.warn -}}
                      {{ .podSecurityStandard.warn }}
                    {{- else -}}
                      {{ $defaultWarnAudit }}
                    {{- end }}"
                  warn-version: "{{ .podSecurityStandard.warnVersion -}}"
                exemptions:
                  usernames: []
                  runtimeClasses: []
                  namespaces: [{{ $namespace_exemptions }}]
            {{- end }}
            {{- if .eventRateLimitConf }}
            - name: EventRateLimit
              path: eventConfig.yaml
            {{- end }}
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/files/-"
      valueFrom:
        template: |-
          path: /etc/kubernetes/eventConfig.yaml
          encoding: base64
          content: {{ .eventRateLimitConf}}
```
## containerd and proxying
```
    "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: echo "::1         localhost" >> /etc/hosts
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: echo "KUBELET_EXTRA_ARGS=--node-ip=$(ip -6 -json addr show dev eth0 scope
        global | jq -r .[0].addr_info[0].local)" >> /etc/sysconfig/kubelet
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: systemctl daemon-reload
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: systemctl stop containerd
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: systemctl start containerd
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      valueFrom:
        template: 'export HTTP_PROXY= {{- .proxy.httpProxy }}
          '
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      valueFrom:
        template: 'export HTTPS_PROXY= {{- .proxy.httpsProxy }}
          '
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      valueFrom:
        template: 'export NO_PROXY= {{- list "localhost" "127.0.0.1" ".svc" ".svc.cluster.local"
          ((list "IPv6" "DualStack" | has .builtin.cluster.network.ipFamily) | ternary  "::1"
          nil) | concat .proxy.noProxy .builtin.cluster.network.services .builtin.cluster.network.pods
          | uniq | sortAlpha | join "," }}
          '
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: "! which rehash_ca_certificates.sh 2>/dev/null || rehash_ca_certificates.sh"
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: "! which update-ca-certificates 2>/dev/null || (mv /etc/ssl/certs/tkg-custom-ca.pem
        /usr/local/share/ca-certificates/tkg-custom-ca.crt && update-ca-certificates)"
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: "! which update-ca-trust 2>/dev/null || (update-ca-trust force-enable
        && mv /etc/ssl/certs/tkg-custom-ca.pem /etc/pki/ca-trust/source/anchors/tkg-custom-ca.crt
        && update-ca-trust extract)"
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: systemctl restart containerd
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      valueFrom:
        template: 'sed -i ''s|".*/pause|" {{- or .imageRepository.host (index .TKR_DATA
          .builtin.controlPlane.version).kubernetesSpec.imageRepository -}} /pause|''
          /etc/containerd/config.toml
          '
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      valueFrom:
        template: |
          {{- $host := index (or .imageRepository.host (index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.imageRepository | splitList "/") 0 -}}
          echo '[plugins."io.containerd.grpc.v1.cri".registry.configs." {{- $host -}} ".tls]' >> /etc/containerd/config.toml
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      valueFrom:
        template: |
          {{- $host := index (or .imageRepository.host (index .TKR_DATA .builtin.controlPlane.version).kubernetesSpec.imageRepository | splitList "/") 0 }}
          {{- $val := list "ca_file = \"/etc/containerd/" $host ".crt\"" | join "" }}
          {{- with .imageRepository }}
            {{- if .tlsCertificateValidation | eq false }}
              {{- $val = "insecure_skip_verify = "true"" }}
            {{- end }}
          {{- end -}}
          {{- define "echo" -}}
            echo '  {{ . -}} ' >> /etc/containerd/config.toml
          {{- end }}
          {{- template "echo" $val -}}
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/preKubeadmCommands/-"
      value: systemctl restart containerd
    "/s/t/s/kubeadmConfigSpec/initConfiguration/localAPIEndpoint":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/initConfiguration/localAPIEndpoint"
      valueFrom:
        template: |
          {{ if .builtin.cluster.network.ipFamily | eq "IPv6" | or (.builtin.cluster.network.ipFamily | eq "DualStack" | and (.network.ipv6Primary | default false)) -}}
            advertiseAddress: '::/0'
          {{- else -}}
            advertiseAddress: '0.0.0.0'
          {{- end }}
          bindPort: {{ .apiServerPort }}
    "/s/t/s/kubeadmConfigSpec/joinConfiguration/controlPlane":
```

## kubeadmJoining parameters

```
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/joinConfiguration/controlPlane"
      valueFrom:
        template: |
          localAPIEndpoint:
            {{ if .builtin.cluster.network.ipFamily | eq "IPv6" | or (.builtin.cluster.network.ipFamily | eq "DualStack" | and (.network.ipv6Primary | default false)) -}}
              advertiseAddress: '::/0'
            {{- else -}}
              advertiseAddress: '0.0.0.0'
            {{- end }}
            bindPort: {{ .apiServerPort }}
    "/s/t/s/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs/node-ip":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs/node-ip"
      value: "::"
    "/s/t/s/kubeadmConfigSpec/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-ip":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-ip"
      value: "::"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/advertise-address":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/advertise-address"
      valueFrom:
        variable: apiServerEndpoint
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/bind-address":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/bind-address"
      value: "::"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/controllerManager/extraArgs/bind-address":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/controllerManager/extraArgs/bind-address"
      value: "::"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/scheduler/extraArgs/bind-address":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/scheduler/extraArgs/bind-address"
      value: "::"
    "/s/t/s/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs/node-labels":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/initConfiguration/nodeRegistration/kubeletExtraArgs/node-labels"
      valueFrom:
        template: |
          {{ $first := "true" }}
          {{- range $key, $val := (index .TKR_DATA .builtin.controlPlane.version).labels }}
          {{- if regexMatch "^(?:[a-zA-z])(?:[-\\w\\.]*[a-zA-z])$" $val }}
          {{- if $first }}
            {{- $first = false }}
          {{- else -}}
            ,
          {{- end }}
          {{- $key -}} = {{- $val }}
          {{- end }}
          {{- end }}
          {{- if .controlPlane.nodeLabels -}}
            {{- if (index .TKR_DATA .builtin.controlPlane.version).labels -}}
            ,
            {{- end -}}
            {{- $first := "true" }}
            {{- range .controlPlane.nodeLabels }}
              {{- if $first }}
                {{- $first = false }}
              {{- else -}}
                ,
              {{- end }}
              {{- .key -}} = {{- .value -}}
            {{ end }}
          {{ end }}
    "/s/t/s/kubeadmConfigSpec/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-labels":
```
## Node labelling and audit logging

```
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-labels"
      valueFrom:
        template: |
          {{ $first := "true" }}
          {{- range $key, $val := (index .TKR_DATA .builtin.controlPlane.version).labels }}
          {{- if regexMatch "^(?:[a-zA-z])(?:[-\\w\\.]*[a-zA-z])$" $val }}
          {{- if $first }}
            {{- $first = false }}
          {{- else -}}
            ,
          {{- end }}
          {{- $key -}} = {{- $val }}
          {{- end }}
          {{- end }}
          {{- if .controlPlane.nodeLabels -}}
            {{- if (index .TKR_DATA .builtin.controlPlane.version).labels -}}
            ,
            {{- end -}}
            {{- $first := "true" }}
            {{- range .controlPlane.nodeLabels }}
              {{- if $first }}
                {{- $first = false }}
              {{- else -}}
                ,
              {{- end }}
              {{- .key -}} = {{- .value -}}
            {{ end }}
          {{ end }}
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-log-path":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-log-path"
      value: "/var/log/kubernetes/audit.log"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-policy-file":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-policy-file"
      value: "/etc/kubernetes/audit-policy.yaml"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-log-maxage":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-log-maxage"
      value: '30'
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-log-maxbackup":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-log-maxbackup"
      value: '10'
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-log-maxsize":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/audit-log-maxsize"
      value: '100'
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraVolumes/-":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraVolumes/-"
      value:
        hostPath: "/etc/kubernetes/audit-policy.yaml"
        mountPath: "/etc/kubernetes/audit-policy.yaml"
        name: audit-policy
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraVolumes/-"
      value:
        hostPath: "/var/log/kubernetes"
        mountPath: "/var/log/kubernetes"
        name: audit-logs
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraVolumes/-"
      valueFrom:
        template: |
          name: admin-control-conf
          hostPath: /etc/kubernetes/admission-control-config.yaml
          mountPath: /etc/kubernetes/admission-control-config.yaml
          readOnly: "true"
          pathType: "File"
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraVolumes/-"
      valueFrom:
        template: |
          name: event-conf
          hostPath: /etc/kubernetes/eventConfig.yaml
          mountPath: /etc/kubernetes/eventConfig.yaml
          readOnly: "true"
          , pathType: "File"
    "/s/t/s/kubeadmConfigSpec/initConfiguration/nodeRegistration/taints":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/initConfiguration/nodeRegistration/taints"
      value: []
    "/s/t/s/kubeadmConfigSpec/joinConfiguration/nodeRegistration/taints":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/joinConfiguration/nodeRegistration/taints"
      value: []
    "/s/t/s/rolloutBefore":
```
## Certificates and NTP
```
    - op: add
      path: "/s/t/s/rolloutBefore"
      valueFrom:
        template: 'certificatesExpiryDays: {{ .controlPlaneCertificateRotation.daysBefore
          }}
          '
    "/s/t/s/kubeadmConfigSpec/ntp":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/ntp"
      valueFrom:
        template: |
          enabled: "true"
          servers:
            {{- range .ntpServers }}
          - {{ . }}
            {{- end }}
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/certSANs":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/certSANs"
      valueFrom:
        template: |
          {{- range .additionalFQDN }}
          - {{ . }}
          {{- end }}
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/admission-control-config-file":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/admission-control-config-file"
      value: "/etc/kubernetes/admission-control-config.yaml"
    "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/enable-admission-plugins":
    - op: add
      path: "/s/t/s/kubeadmConfigSpec/clusterConfiguration/apiServer/extraArgs/enable-admission-plugins"
      valueFrom:
        template: |
          {{ $containEnableAdmissionPlugin := false }}
          {{- $admissionPlugins := "" }}
          {{- range $key, $val := .apiServerExtraArgs }}
          {{- if eq $key "enable-admission-plugins" }}
            {{- $containEnableAdmissionPlugin = "true" }}
            {{- $admissionPlugins = $val }}
          {{- end }}
          {{- end }}
          {{- if not $containEnableAdmissionPlugin }}
          NodeRestriction,EventRateLimit
          {{- else -}}
          {{- $admissionPlugins -}},EventRateLimit
          {{- end }}
```

## Selector

And of course, our selector which tells the patches to apply all of these to the `KubeadmControlTemplate`.
```
  selector:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlaneTemplate
    matchResources:
      controlPlane: 'true'
```

# KubeadmConfigTemplate Patch Details

This is how we customize the kubelets that run for WORKER nodes.

- HTTP proxy info
- Custom CAs
- Windows Workload cluster modifications
- ...


``` 
KubeadmConfigTemplate:
  jsonPatches:
    "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs":
    - op: add
      path: "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs"
      valueFrom:
        template: |
          {{ $containCipherSuites := false }}
          {{ $containCloudProvider := false }}
          {{- range $key, $val := .workerKubeletExtraArgs }}
          {{- if eq $key "tls-cipher-suites" }}
            {{- $containCipherSuites = "true" }}
          {{- end }}
          {{- if eq $key "cloud-provider" }}
            {{- $containCloudProvider = "true" }}
          {{- end }}
          {{ $key -}} : "{{ $val }}"
          {{- end }}
          {{- if not $containCipherSuites }}
          tls-cipher-suites: "{{ .tlsCipherSuites }}"
          {{- end }}
          {{- if not $containCloudProvider }}
          cloud-provider: external
          {{- end }}
    "/s/t/s/users":
    - op: replace
      path: "/s/t/s/users"
      valueFrom:
        template: |
          - name: capv
            sshAuthorizedKeys:
            {{- range .user.sshAuthorizedKeys }}
            - ' {{- . -}} '
            {{- end }}
            sudo: ALL=(ALL) NOPASSWD:ALL
    - op: replace
      path: "/s/t/s/users"
      valueFrom:
        template: |
          - name: capv
            groups: Administrators
            sshAuthorizedKeys:
            {{- range .user.sshAuthorizedKeys }}
            - ' {{- . -}} '
            {{- end }}
            sudo: ALL=(ALL) NOPASSWD:ALL
```
## HTTP Proxy and CAs 

```
    "/s/t/s/preKubeadmCommands/-":
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      value: echo "::1         localhost" >> /etc/hosts
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      value: systemctl daemon-reload
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      value: systemctl restart containerd
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      valueFrom:
        template: 'export HTTP_PROXY= {{- .proxy.httpProxy }}
          '
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      valueFrom:
        template: 'export HTTPS_PROXY= {{- .proxy.httpsProxy }}
          '
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      valueFrom:
        template: 'export NO_PROXY= {{- list "localhost" "127.0.0.1" ".svc" ".svc.cluster.local"
          ((list "IPv6" "DualStack" | has .builtin.cluster.network.ipFamily) | ternary  "::1"
          nil) | concat .proxy.noProxy .builtin.cluster.network.services .builtin.cluster.network.pods
          | uniq | sortAlpha | join "," }}
          '
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      value: "! which rehash_ca_certificates.sh 2>/dev/null || rehash_ca_certificates.sh"
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      value: "! which update-ca-certificates 2>/dev/null || (mv /etc/ssl/certs/tkg-custom-ca.pem
        /usr/local/share/ca-certificates/tkg-custom-ca.crt && update-ca-certificates)"
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      value: "! which update-ca-trust 2>/dev/null || (update-ca-trust force-enable
        && mv /etc/ssl/certs/tkg-custom-ca.pem /etc/pki/ca-trust/source/anchors/tkg-custom-ca.crt
        && update-ca-trust extract)"
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      value: systemctl restart containerd
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      valueFrom:
        template: 'sed -i ''s|".*/pause|" {{- or .imageRepository.host (index .TKR_DATA
          .builtin.machineDeployment.version).kubernetesSpec.imageRepository -}} /pause|''
          /etc/containerd/config.toml
          '
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      valueFrom:
        template: |
          {{- $host := index (or .imageRepository.host (index .TKR_DATA .builtin.machineDeployment.version).kubernetesSpec.imageRepository | splitList "/") 0 -}}
          echo '[plugins."io.containerd.grpc.v1.cri".registry.configs." {{- $host -}} ".tls]' >> /etc/containerd/config.toml
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      valueFrom:
        template: |
          {{- $host := index (or .imageRepository.host (index .TKR_DATA .builtin.machineDeployment.version).kubernetesSpec.imageRepository | splitList "/") 0 }}
          {{- $val := list "ca_file = \"/etc/containerd/" $host ".crt\"" | join "" }}
          {{- with .imageRepository }}
            {{- if .tlsCertificateValidation | eq false }}
              {{- $val = "insecure_skip_verify = "true"" }}
            {{- end }}
          {{- end -}}
          {{- define "echo" -}}
            echo '  {{ . -}} ' >> /etc/containerd/config.toml
          {{- end }}
          {{- template "echo" $val -}}
```

## Containerd and Node stuff

```
    - op: add
      path: "/s/t/s/preKubeadmCommands/-"
      value: systemctl restart containerd
    "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-ip":
    - op: add
      path: "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-ip"
      value: "::"
    "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-labels":
    - op: add
      path: "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs/node-labels"
      valueFrom:
        template: |
          {{ $first := "true" }}
          {{- range $key, $val := (index .TKR_DATA .builtin.machineDeployment.version).labels }}
          {{- if regexMatch "^(?:[a-zA-z])(?:[-\\w\\.]*[a-zA-z])$" $val }}
          {{- if $first }}
            {{- $first = false }}
          {{- else -}}
            ,
          {{- end }}
          {{- $key -}} = {{- $val }}
          {{- end }}
          {{- end }}
          {{- if .nodePoolLabels -}}
            ,
            {{- $first := "true" }}
            {{- range .nodePoolLabels }}
              {{- if $first }}
                {{- $first = false }}
              {{- else -}}
                ,
              {{- end }}
              {{- .key -}} = {{- .value -}}
            {{ end }}
          {{ end }}
    "/s/t/s/files/-":
    - op: add
      path: "/s/t/s/files/-"
      valueFrom:
        template: |
          content: |
            [Service]
            Environment="HTTP_PROXY= {{- .proxy.httpProxy -}} "
            Environment="HTTPS_PROXY= {{- .proxy.httpsProxy -}} "
            Environment="NO_PROXY= {{- list "localhost" "127.0.0.1" ".svc" ".svc.cluster.local" ((list "IPv6" "DualStack" | has .builtin.cluster.network.ipFamily) | ternary  "::1" nil) | concat .proxy.noProxy .builtin.cluster.network.services .builtin.cluster.network.pods | uniq | sortAlpha | join "," -}} "
          owner: root:root
          path: /etc/systemd/system/containerd.service.d/http-proxy.conf
          permissions: "0640"
    - op: add
      path: "/s/t/s/files/-"
      valueFrom:
        template: |
          content: |
            [Service]
            Environment="HTTP_PROXY= {{- .proxy.httpProxy -}} "
            Environment="HTTPS_PROXY= {{- .proxy.httpsProxy -}} "
            Environment="NO_PROXY= {{- list "localhost" "127.0.0.1" ".svc" ".svc.cluster.local" ((list "IPv6" "DualStack" | has .builtin.cluster.network.ipFamily) | ternary  "::1" nil) | concat .proxy.noProxy .builtin.cluster.network.services .builtin.cluster.network.pods | uniq | sortAlpha | join "," -}} "
          owner: root:root
          path: /usr/lib/systemd/system/kubelet.service.d/http-proxy.conf
          permissions: "0640"
    - op: add
      path: "/s/t/s/files/-"
      valueFrom:
        template: |
          path: /etc/ssl/certs/tkg-custom-ca.pem
          {{- $proxy := "" }}
          {{- range .trust.additionalTrustedCAs }}
            {{- if eq .name "proxy" }}
              {{- $proxy = .data }}
            {{- end }}
          {{- end }}
          content: {{ $proxy }}
          encoding: base64
          permissions: "0444"
    - op: add
      path: "/s/t/s/files/-"
      valueFrom:
        template: |
          path: /etc/containerd/{{ index (or .imageRepository.host (index .TKR_DATA .builtin.machineDeployment.version).kubernetesSpec.imageRepository | splitList "/") 0 }}.crt
          {{- $proxy := "" }}
          {{- $image := "" }}
          {{- range .trust.additionalTrustedCAs }}
            {{- if eq .name "proxy" }}
              {{- $proxy = .data }}
            {{- end }}
            {{- if eq .name "imageRepository" }}
              {{- $image = .data }}
            {{- end }}
          {{- end }}
          content: {{or $proxy $image}}
          encoding: base64
          permissions: "0444"
```

## WINDOWS !!!

Note that kubeaddmConfig (the worker nodes) has the information for windows specific changes.  These write out 
files to the node when it starts up that are required for certain windows things (like installing antrea).

```
    - op: add
      path: "/s/t/s/files/-"
      value:
        content: 'Set-Service -Name "wuauserv" -StartupType Disabled -Status Stopped
          '
        path: C:\k\prevent_windows_updates.ps1
    - op: add
      path: "/s/t/s/files/-"
      value:
        content: |
          function WaitForSaToken($KubeCfgFile, $ServiceAcctName) {
              $SaToken = $null
              $LoopCount = 400
              do {
                  $LoopCount = $LoopCount - 1
                  if ($LoopCount -eq 0) {
                      break
                  }
                  sleep 5
                  $SaToken=$(kubectl --kubeconfig=$KubeCfgFile get secrets -n kube-system -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='$ServiceAcctName')].data.token}")
              } while ($SaToken -eq $null)
              return $SaToken
          }
          # Disable firewall temporarily for SSH and other internal ports access
          Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
          $TempFolder = 'C:\programdata\temp'
          $AntreaInTempFolder = "$TempFolder\antrea-windows-advanced.zip"
          $KubeproxyInTempFolder = "$TempFolder\kube-proxy.exe"
          # Create Folders
          $folders = @('C:\k\antrea', 'C:\var\log\antrea', 'C:\k\antrea\bin', 'C:\var\log\kube-proxy', 'C:\opt\cni\bin', 'C:\etc\cni\net.d')
          foreach ($f in $folders) {
              New-Item -ItemType Directory -Force -Path $f
          }
          # Add Windows Defender Options
          $avexceptions = @('C:\program files\containerd\ctr.exe', 'C:\program files\containerd\containerd.exe')
          foreach ($e in $avexceptions) {
                Add-MpPreference -ExclusionProcess $e
          }
          # Extract Antrea, Antrea binary should be packed into windows OVA already
          $antreaZipFile = 'C:\k\antrea\antrea-windows-advanced.zip'
          if (!(Test-Path $antreaZipFile)) {
              cp $AntreaInTempFolder $antreaZipFile
          }
          Expand-Archive -Force -Path $antreaZipFile -DestinationPath C:\k\antrea
          cp C:\k\antrea\bin\antrea-cni.exe C:\opt\cni\bin\antrea.exe -Force
          cp C:\k\antrea\bin\host-local.exe C:\opt\cni\bin\host-local.exe -Force
          cp C:\k\antrea\etc\antrea-cni.conflist C:\etc\cni\net.d\10-antrea.conflist -Force
          # Get HostIP and set in kubeadm-flags.env
          [Environment]::SetEnvironmentVariable("NODE_NAME", (hostname).ToLower())
          $env:HostIP = (
              Get-NetIPConfiguration |
              Where-Object {
                  $_.IPv4DefaultGateway -ne $null -and $_.NetAdapter.Status -ne "Disconnected"
              }
          ).IPv4Address.IPAddress
          $file = 'C:\var\lib\kubelet\kubeadm-flags.env'
          $newstr = "--node-ip=" + $env:HostIP
          $raw = Get-Content -Path $file -TotalCount 1
          $raw = $raw -replace ".$"
          $new = "$($raw) $($newstr)`""
          Set-Content $file $new
          $KubeConfigFile = 'C:\etc\kubernetes\kubelet.conf'
          # Wait for antrea-agent token to be ready, the token will be used by Install-AntreaAgent
          $AntreaAgentToken = (WaitForSaToken $KubeConfigFile 'antrea-agent')
          # Setup Kube-Proxy config file
          $KubeProxyToken = (WaitForSaToken $KubeConfigFile 'kube-proxy-windows')
          $KubeProxyConfig = 'C:\k\antrea\etc\kube-proxy.conf'
          $KubeAPIServer = $(kubectl --kubeconfig=$KubeConfigFile config view -o jsonpath='{.clusters[0].cluster.server}')
          $KubeProxyToken = $([System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($KubeProxyToken)))
          kubectl config --kubeconfig=$KubeProxyConfig set-cluster kubernetes --server=$KubeAPIServer --insecure-skip-tls-verify
          kubectl config --kubeconfig=$KubeProxyConfig set-credentials kube-proxy-windows --token=$KubeProxyToken
          kubectl config --kubeconfig=$KubeProxyConfig set-context kube-proxy-windows@kubernetes --cluster=kubernetes --user=kube-proxy-windows
          kubectl config --kubeconfig=$KubeProxyConfig use-context kube-proxy-windows@kubernetes
          # kube-proxy.exe should be packed into windows OVA
          if (!(Test-Path 'C:\k\kube-proxy.exe')) {
              cp $KubeproxyInTempFolder 'C:\k\kube-proxy.exe'
          }
          # Install antrea-agent & OVS
          Import-Module C:\k\antrea\helper.psm1
          & Install-AntreaAgent -KubernetesHome "C:\k" -KubeConfig "C:\etc\kubernetes\kubelet.conf" -AntreaHome "C:\k\antrea" -AntreaVersion "1.7.1"
          New-KubeProxyServiceInterface
          & C:\k\antrea\Install-OVS.ps1 -ImportCertificate $false -LocalFile C:\k\antrea\ovs-win64.zip
          # Setup Services
          $nssm = (Get-Command nssm).Source
          & $nssm set kubelet start SERVICE_AUTO_START
          & $nssm install kube-proxy "C:\k\kube-proxy.exe" "--proxy-mode=userspace --kubeconfig=$KubeProxyConfig --log-dir=C:\var\log\kube-proxy --logtostderr=false --alsologtostderr"
          & $nssm install antrea-agent "C:\k\antrea\bin\antrea-agent.exe" "--config=C:\k\antrea\etc\antrea-agent.conf --logtostderr=false --log_dir=C:\var\log\antrea --alsologtostderr --log_file_max_size=100 --log_file_max_num=4"
          & $nssm set antrea-agent DependOnService kube-proxy ovs-vswitchd
          & $nssm set antrea-agent Start SERVICE_AUTO_START
          # Start Services
          start-service kubelet
          start-service kube-proxy
          start-service antrea-agent
        path: C:\Temp\antrea.ps1
```
## Certificates and NTP

```
    "/s/t/s/useExperimentalRetryJoin":
    - op: remove
      path: "/s/t/s/useExperimentalRetryJoin"
    "/s/t/s/joinConfiguration/nodeRegistration/criSocket":
    - op: add
      path: "/s/t/s/joinConfiguration/nodeRegistration/criSocket"
      value: npipe:////./pipe/containerd-containerd
    "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs/tls-cipher-suites":
    - op: remove
      path: "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs/tls-cipher-suites"
    "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs/register-with-taints":
    - op: add
      path: "/s/t/s/joinConfiguration/nodeRegistration/kubeletExtraArgs/register-with-taints"
      value: os=windows:NoSchedule
    "/s/t/s/joinConfiguration/nodeRegistration/name":
    - op: replace
      path: "/s/t/s/joinConfiguration/nodeRegistration/name"
      value: "{{ ds.meta_data.hostname }}"
    "/s/t/s/preKubeadmCommands":
    - op: replace
      path: "/s/t/s/preKubeadmCommands"
      valueFrom:
        template: |
          - echo | set /p="::1         ipv6-localhost ipv6-loopback localhost6 localhost6.localdomain6" > C:\etc\hosts & echo. >> C:\etc\hosts
          - echo | set /p="127.0.0.1   {{" {{ ds.meta_data.hostname }} "}} localhost localhost.localdomain localhost4 localhost4.localdomain4" >> C:\etc\hosts
    "/s/t/s/postKubeadmCommands/-":
    - op: add
      path: "/s/t/s/postKubeadmCommands/-"
      value: powershell c:/k/prevent_windows_updates.ps1 -ExecutionPolicy Bypass
    - op: add
      path: "/s/t/s/postKubeadmCommands/-"
      value: powershell C:/Temp/antrea.ps1 -ExecutionPolicy Bypass
    "/s/t/s/ntp":
    - op: add
      path: "/s/t/s/ntp"
      valueFrom:
        template: |
          enabled: "true"
          servers:
            {{- range .ntpServers }}
          - {{ . }}
            {{- end }}
```

And thats it, heres the selector
```
  selector:
    apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
    kind: KubeadmConfigTemplate
    matchResources:
      machineDeploymentClass:
        names:
        - tkg-worker
```

# VSPhereMachineTemplate Patch Details

Now we have the changes which we put into vsphere machine templates. These support things like GPU (not shown here) and PCI Passthrough.
There are maybe 30 or so of these patches.

- Vsphere user
- Vsphere cloneMode
- Vsphere memory
- Vsphere CPU
- PCI/VMX passthrough information
- other VSphere specific parameters

```
VSphereMachineTemplate:
  jsonPatches:
```

## Standard VSPHERE parameters
```
    "/s/t/s/numCPUs":
    - op: replace
      path: "/s/t/s/numCPUs"
      valueFrom:
        variable: controlPlane.machine.numCPUs
    - op: replace
      path: "/s/t/s/numCPUs"
      valueFrom:
        variable: worker.machine.numCPUs
    "/s/t/s/diskGiB":
    - op: replace
      path: "/s/t/s/diskGiB"
      valueFrom:
        variable: controlPlane.machine.diskGiB
    - op: replace
      path: "/s/t/s/diskGiB"
      valueFrom:
        variable: worker.machine.diskGiB
    "/s/t/s/memoryMiB":
    - op: replace
      path: "/s/t/s/memoryMiB"
      valueFrom:
        variable: controlPlane.machine.memoryMiB
    - op: replace
      path: "/s/t/s/memoryMiB"
      valueFrom:
        variable: worker.machine.memoryMiB
    "/s/t/s/cloneMode":
    - op: replace
      path: "/s/t/s/cloneMode"
      valueFrom:
        variable: vcenter.cloneMode
    - op: replace
      path: "/s/t/s/cloneMode"
      valueFrom:
        variable: vcenter.cloneMode
    "/s/t/s/network":
    - op: replace
      path: "/s/t/s/network"
      valueFrom:
        variable: vcenter.network
    - op: replace
      path: "/s/t/s/network"
      valueFrom:
        variable: vcenter.network
    - op: replace
      path: "/s/t/s/network"
      valueFrom:
        template: |
          devices:
          - networkName: {{ .vcenter.network }}
            {{ if .controlPlane.network.nameservers -}}
            nameservers:
              {{- range .controlPlane.network.nameservers }}
            - {{ . }}
              {{- end }}
            {{- end }}
            {{ if .controlPlane.network.searchDomains -}}
            searchDomains:
              {{- range .controlPlane.network.searchDomains }}
            - {{ . }}
              {{- end }}
            {{- end }}
            {{ if list "IPv4" "DualStack" | has .builtin.cluster.network.ipFamily | and (empty .network.addressesFromPools) -}} dhcp4: "true" {{- end }}
            {{ if list "IPv6" "DualStack" | has .builtin.cluster.network.ipFamily | and (empty .network.addressesFromPools) -}} dhcp6: "true" {{- end }}
            {{ if .network.addressesFromPools  -}}
            addressesFromPools:
              {{- range .network.addressesFromPools }}
            - apiGroup: {{ .apiGroup }}
              kind: {{ .kind }}
              name: {{ .name }}
              {{- end }}
            {{- end }}
```
## DualStack network configuration
```
    - op: add
      path: "/s/t/s/network"
      valueFrom:
        template: |
          devices:
          - networkName: {{ .vcenter.network }}
            {{ if .worker.network.nameservers -}}
            nameservers:
              {{- range .worker.network.nameservers }}
            - {{ . }}
              {{- end }}
            {{- end }}
            {{ if .controlPlane.network.searchDomains -}}
            searchDomains:
              {{- range .controlPlane.network.searchDomains }}
            - {{ . }}
              {{- end }}
            {{- end }}
            {{ if list "IPv4" "DualStack" | has .builtin.cluster.network.ipFamily | and (empty .network.addressesFromPools) -}} dhcp4: "true" {{- end }}
            {{ if list "IPv6" "DualStack" | has .builtin.cluster.network.ipFamily | and (empty .network.addressesFromPools) -}} dhcp6: "true" {{- end }}
            {{ if .network.addressesFromPools  -}}
            addressesFromPools:
              {{- range .network.addressesFromPools }}
            - apiGroup: {{ .apiGroup }}
              kind: {{ .kind }}
              name: {{ .name }}
              {{- end }}
            {{- end }}
    "/s/t/s/datacenter":
    - op: replace
      path: "/s/t/s/datacenter"
      valueFrom:
        variable: vcenter.datacenter
    - op: replace
      path: "/s/t/s/datacenter"
      valueFrom:
        variable: vcenter.datacenter
    "/s/t/s/datastore":
    - op: replace
      path: "/s/t/s/datastore"
      valueFrom:
        variable: vcenter.datastore
    - op: replace
      path: "/s/t/s/datastore"
      valueFrom:
        variable: vcenter.datastore
    "/s/t/s/folder":
    - op: replace
      path: "/s/t/s/folder"
      valueFrom:
        variable: vcenter.folder
    - op: replace
      path: "/s/t/s/folder"
      valueFrom:
        variable: vcenter.folder
    "/s/t/s/resourcePool":
    - op: replace
      path: "/s/t/s/resourcePool"
      valueFrom:
        variable: vcenter.resourcePool
    - op: replace
      path: "/s/t/s/resourcePool"
      valueFrom:
        variable: vcenter.resourcePool
    "/s/t/s/storagePolicyName":
    - op: replace
      path: "/s/t/s/storagePolicyName"
      valueFrom:
        variable: vcenter.storagePolicyID
    - op: replace
      path: "/s/t/s/storagePolicyName"
      valueFrom:
        variable: vcenter.storagePolicyID
    "/s/t/s/server":
    - op: replace
      path: "/s/t/s/server"
      valueFrom:
        variable: vcenter.server
    - op: replace
      path: "/s/t/s/server"
      valueFrom:
        variable: vcenter.server
    "/s/t/s/template":
    - op: replace
      path: "/s/t/s/template"
      valueFrom:
        template: "{{ (index .TKR_DATA .builtin.controlPlane.version).osImageRef.template
          }}"
    - op: replace
      path: "/s/t/s/template"
      valueFrom:
        template: "{{ (index .TKR_DATA .builtin.machineDeployment.version).osImageRef.template
          }}"
```
## Selector

And finally of course the selector
```
  selector:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: VSphereMachineTemplate
    matchResources:
      machineDeploymentClass:
        names:
        - tkg-worker
        - tkg-worker-windows
```



# VSphereClusterTemplate Patch Details

Next we have all the changes to the VSphere Cluster...   There are about 6 of these.

- ControlPlaneEndpoints
- Thumbprints
- Cluster Name
- apiServerPort

```
VSphereClusterTemplate:
  jsonPatches:
```
## ControlPlane Endpoint

```

    "/s/t/s/controlPlaneEndpoint":
    - op: add
      path: "/s/t/s/controlPlaneEndpoint"
      valueFrom:
        template: |
          host: '{{ .apiServerEndpoint }}'
          port: 6443
    "/s/t/s/thumbprint":
    - op: replace
      path: "/s/t/s/thumbprint"
      valueFrom:
        variable: vcenter.tlsThumbprint
    "/s/t/s/server":
    - op: replace
      path: "/s/t/s/server"
      valueFrom:
        variable: vcenter.server
```
## Identity of the VSphere Cluster 

Necessary so we can map it back to the name of the CAPI cluster... there are internal functionality that depend on this ... 

```
    "/s/t/s/identityRef":
    - op: add
      path: "/s/t/s/identityRef"
      valueFrom:
        template: |
          {{ if .identityRef -}}
          kind: {{ .identityRef.kind }}
          name: {{ .identityRef.name }}
          {{- else -}}
          kind: Secret
          name: '{{ .builtin.cluster.name }}'
          {{- end }}
    "/s/t/s/controlPlaneEndpoint/port":
    - op: replace
      path: "/s/t/s/controlPlaneEndpoint/port"
      valueFrom:
        variable: apiServerPort
```
## Selector
```
  selector:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: VSphereClusterTemplate
    matchResources:
      infrastructureCluster: 'true'
```

# Thats it !

The above JSON patches are from 2.1.0.. but there are more patches in 2.1.1, including patches to support PCI Passthrough and other new fixes
which make 2.1.1 more usable then 2.1.0 for the full TKG feature matrix.


