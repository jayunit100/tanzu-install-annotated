# cli

... This is a WIP... will add more next week  ....

Tanzu CLI is now independent of TKG  https://github.com/vmware-tanzu/tanzu-cli/releases/tag/v0.90.0 !

As always, were not replacing the docs here, the real docs for tanzu cli are here https://docs.vmware.com/en/VMware-Tanzu-CLI/0.29.0/vmware-tanzu-cli-ref/tanzu-plugin.html. 

## tanzu cli
Lets look at the tanzu CLI.

```
tanzu version
09:06:57  version: v0.90.0-beta.0
09:06:57  buildDate: 2023-05-31
09:06:57  sha: 1fc71327
```

Several of the defaults can be set by simply invoking `tanzu ... set ...`: 
```
09:06:57  INFO:root:====== 85   CMD: tanzu ceip-participation set false
09:06:57  INFO:root:====== 86   CMD: tanzu config eula accept
```

These edit the config file on disk:

```
./.config/tanzu/config-ng.yaml:    ceipOptIn: "false"
...
kubo@MrAVegHFCdimD:~$ cat ~/.config/tanzu/config-ng.yaml 
cli:
    discoverySources:
        - oci:
            name: default
            image: projects.registry.vmware.com/tanzu_cli/plugins/plugin-inventory:latest
    ceipOptIn: "false"
    eulaStatus: accepted
contexts:
    - name: tkg-mgmt-vc
      target: kubernetes
      clusterOpts:
        path: /home/kubo/.kube-tkg/config
        context: tkg-mgmt-vc-admin@tkg-mgmt-vc
        isManagementCluster: true
      discoverySources: []
currentContext:
    kubernetes: tkg-mgmt-vc
```

We see that there are:
- cli
- contexts
- current context
configurations. 

## cli

CLI has *discovery sources*.  these tell the CLI what OCI registries it can pull plugins from.  IF we look at the discovery sources,
we can see there are several tags:
```
ubuntu@jay-build-box-6 [17:39:41] [~/LOCAL/TKRS/geetika/antrea-122]
-> %
imgpkg tag list -i projects.registry.vmware.com/tanzu_cli/plugins/plugin-inventory:latest
Name
latest
sha256-003ea5053cea54060190df60bbbb76351784a680756e3dd984f449180bce5136.imgpkg
sha256-11b71d946722647c92bb77dfd13f99baa1b44778247ee2cda2f2952298d5cd7d.imgpkg
sha256-2e8a6673ce12d3437a0209bb02ca1f5d9b4e856b687445a5c60cc1c7ce6eb7fb.imgpkg
sha256-2e8a6673ce12d3437a0209bb02ca1f5d9b4e856b687445a5c60cc1c7ce6eb7fb.sig
sha256-3cb5cafb116712900dfaf38b561e6f62f26aa14e25252d8e59c9603b324c5008.imgpkg
sha256-3cb5cafb116712900dfaf38b561e6f62f26aa14e25252d8e59c9603b324c5008.sig
sha256-44988cb17f533c87e487fb41aba0dd396083b4ad893d5442cd7f76ec996bdbaf.imgpkg
sha256-46dea93994860ced534867858c3f880bf6a51449fa466722a49c938959f106f7.imgpkg
sha256-46dea93994860ced534867858c3f880bf6a51449fa466722a49c938959f106f7.sig
```

Whats inside of the "latest" tag?  A SQLLite Database !

```
➜  tkg22-cli wget https://storage.googleapis.com/jayunit100/plugin_inventory.db 
--2023-06-23 15:44:56--  https://storage.googleapis.com/jayunit100/plugin_inventory.db
...
Saving to: ‘plugin_inventory.db’
plugin_inventory.db                           100%[===============================================================================================>]  72.00K  --.-KB/s    in 0.02s   
...
➜  tkg22-cli sqlite3 plugin_inventory.db 
SQLite version 3.39.5 2022-10-14 20:58:05
Enter ".help" for usage hints.
sqlite> .tables

PluginBinaries  
PluginGroups  
sqlite> 
```

Ok, so tanzu cli pulls down this `:latest` plugins image, and it has two tables: *PluginBinaries* and *PluginGroups*.  Lets see whats in there.

```
sqlite> .schema PluginBinaries
CREATE TABLE IF NOT EXISTS "PluginBinaries" (
                "PluginName"         TEXT NOT NULL,
                "Target"             TEXT NOT NULL,
                "RecommendedVersion" TEXT NOT NULL,
                "Version"            TEXT NOT NULL,
                "Hidden"             TEXT NOT NULL,
                "Description"        TEXT NOT NULL,
                "Publisher"          TEXT NOT NULL,
                "Vendor"             TEXT NOT NULL,
                "OS"                 TEXT NOT NULL,
                "Architecture"       TEXT NOT NULL,
                "Digest"             TEXT NOT NULL,
                "URI"                TEXT NOT NULL,
                PRIMARY KEY("PluginName", "Target", "Version", "OS", "Architecture")
);
sqlite> .schema PluginGroups
CREATE TABLE IF NOT EXISTS "PluginGroups" (
                "Vendor"             TEXT NOT NULL,
                "Publisher"          TEXT NOT NULL,
                "GroupName"          TEXT NOT NULL,
                "GroupVersion"       TEXT NOT NULL,
                "Description"        TEXT NOT NULL,
                "PluginName"         TEXT NOT NULL,
                "Target"             TEXT NOT NULL,
                "PluginVersion"      TEXT NOT NULL,
                "Mandatory"          TEXT NOT NULL,
                "Hidden"             TEXT NOT NULL,
                PRIMARY KEY("Vendor", "Publisher", "GroupName", "GroupVersion", "PluginName", "Target")
);
```

Now, looking at the plugin binaries, we can see:

- pinniped-auth is a *global* plugin
- management-cluster is a *kubernetes* plugin
- each of the plugins has a linux, darwin, and windows binary like so:

```
|vmware|
|linux|
|amd64|
|06708ed7793e3ed08e5ade02309a7da39659a9a9f9cfb16e41932bb570074e07|vmware/tkg/linux/amd64/global/pinniped-auth:v0.29.0
```

HEres a sampling of many of the entries in the database (you can run `sqllite3 ./plugin_inventory.db` followed by `Select * from PluginBinaries` to see these yourself:

```
pinniped-auth|global||v0.29.0|false|Pinniped authentication operations (usually not directly invoked)|tkg|vmware|linux|amd64|06708ed7793e3ed08e5ade02309a7da39659a9a9f9cfb16e41932bb570074e07|vmware/tkg/linux/amd64/global/pinniped-auth:v0.29.0
pinniped-auth|global||v0.29.0|false|Pinniped authentication operations (usually not directly invoked)|tkg|vmware|darwin|amd64|8c8f81e10b6a388e0b25744167ffb7a938bd8cf9208c63d56ce120692b54b58e|vmware/tkg/darwin/amd64/global/pinniped-auth:v0.29.0
pinniped-auth|global||v0.29.0|false|Pinniped authentication operations (usually not directly invoked)|tkg|vmware|windows|amd64|3529874d693258de2479633313c67e03aaae2795d3e6f173b88d4b00e3b049a8|vmware/tkg/windows/amd64/global/pinniped-auth:v0.29.0 
telemetry|kubernetes||v0.29.0|false|configure cluster-wide settings for vmware tanzu telemetry|tkg|vmware|linux|amd64|0189d6518b172fe08d547141e0862085c771f8fb664d3e3bcc8cf75edac0972a|vmware/tkg/linux/amd64/kubernetes/telemetry:v0.29.0
telemetry|kubernetes||v0.29.0|false|configure cluster-wide settings for vmware tanzu telemetry|tkg|vmware|darwin|amd64|45cc4439fb351091d022e01653d0e7806c0382450c98d9d45c13fc91ddfccad3|vmware/tkg/darwin/amd64/kubernetes/telemetry:v0.29.0
telemetry|kubernetes||v0.29.0|false|configure cluster-wide settings for vmware tanzu telemetry|tkg|vmware|windows|amd64|bc520d1e2f765911aefb1b74d267b6bb2cf715db8b6c449057a58170f1809070|vmware/tkg/windows/amd64/kubernetes/telemetry:v0.29.0
package|kubernetes||v0.29.0|false|Tanzu package management|tkg|vmware|linux|amd64|bdf80daff5dd8cd1165b3a70a80abf02cb0c943c1917f2d0e6d8edbe4e51e2d1|vmware/tkg/linux/amd64/kubernetes/package:v0.29.0
package|kubernetes||v0.29.0|false|Tanzu package management|tkg|vmware|darwin|amd64|abc5da0a813e7d7d5929ceaa092ea627b3320cf08f2a4f1c894b86bd0dddc962|vmware/tkg/darwin/amd64/kubernetes/package:v0.29.0
package|kubernetes||v0.29.0|false|Tanzu package management|tkg|vmware|windows|amd64|f70a5e536e21e6c7685aa0678fbb9a5b9f491b73d28d505f7d6a3467042a818c|vmware/tkg/windows/amd64/kubernetes/package:v0.29.0
secret|kubernetes||v0.29.0|false|Tanzu secret management|tkg|vmware|linux|amd64|60f6bd68fe2bda31d1b79bec2a5b539d145ede92cf911ee6e8c4b169ea71592c|vmware/tkg/linux/amd64/kubernetes/secret:v0.29.0
secret|kubernetes||v0.29.0|false|Tanzu secret management|tkg|vmware|darwin|amd64|e53990e9b64942cb81b5ec3667273b2a4bbee23b142d8fa6c86aca71aa5246f8|vmware/tkg/darwin/amd64/kubernetes/secret:v0.29.0
secret|kubernetes||v0.29.0|false|Tanzu secret management|tkg|vmware|windows|amd64|5a755e59b1a419b4e5600691ffddb4b02c673392befdc0f879225c99878fdfc7|vmware/tkg/windows/amd64/kubernetes/secret:v0.29.0
management-cluster|kubernetes||v0.29.0|false|Kubernetes management cluster operations|tkg|vmware|linux|amd64|f5a99b7c0e7aabadf31ddca8b4d856ea78a9ca6877bd5c7044b5ec115bf11d4f|vmware/tkg/linux/amd64/kubernetes/management-cluster:v0.29.0
management-cluster|kubernetes||v0.29.0|false|Kubernetes management cluster operations|tkg|vmware|darwin|amd64|4e9b92589ad9fa78fb3f893f056e45a017f414074a2daf216f561a3a6063a215|vmware/tkg/darwin/amd64/kubernetes/management-cluster:v0.29.0
management-cluster|kubernetes||v0.29.0|false|Kubernetes management cluster operations|tkg|vmware|windows|amd64|424b2cc90abfffcb08d24526d8203603ed79ca24084d8bbed16e2925f9f54d1b|vmware/tkg/windows/amd64/kubernetes/management-cluster:v0.29.0
cluster|kubernetes||v0.29.0|false|Kubernetes cluster operations|tkg|vmware|linux|amd64|9979b03d231da6728777aff52097cd51adeedb55ee7697d006c009ba42c341c4|vmware/tkg/linux/amd64/kubernetes/cluster:v0.29.0
cluster|kubernetes||v0.29.0|false|Kubernetes cluster operations|tkg|vmware|darwin|amd64|31dc3c3320614d38b68a4f16ebc806b33715e2c03a7ee55c893ba0821fb21fae|vmware/tkg/darwin/amd64/kubernetes/cluster:v0.29.0
cluster|kubernetes||v0.29.0|false|Kubernetes cluster operations|tkg|vmware|windows|amd64|7b49a8acba1ae2e9bf2147334a4865344d7e599c4d322710380f3c1ed1b6e74b|vmware/tkg/windows/amd64/kubernetes/cluster:v0.29.0
```

Ok.  So it looks like:

- tanzu cli is configured to go grab a container that has a large DB in it.
- that DB then has lots of plugin binaries where themselves, are ALSO images.

We can now see this binary gets pulled down from the contents of the `URI` field, in the schema. 

For example `vmware/tkg/linux/amd64/kubernetes/cluster:v0.29.0` can be pulled from `projects.registry.vmware.com/tanzu_cli/plugins/`, and thats how tanzu cli installs the *cluster* plugin.

```
-> % imgpkg pull -i projects.registry.vmware.com/tanzu_cli/plugins/vmware/tkg/linux/amd64/kubernetes/cluster:v0.29.0 --output cluster-pluginnnn
Pulling image 'projects.registry.vmware.com/tanzu_cli/plugins/vmware/tkg/linux/amd64/kubernetes/cluster@sha256:85cec81e5a54df24c65b69d15eece40c4592b7ec71416e8086e0a3c88bb506e2'
Extracting layer 'sha256:6c4d2a106c1cc87f7f1e66492b690961fcf3f5059f5723abac31e9f95dcc31e9' (1/1)

Succeeded
ubuntu@jay-build-box-6 [20:22:33] [~/LOCAL/TKRS/geetika/antrea-122] 
-> % ls cluster-pluginnnn 
tanzu-cluster-linux_amd64
```

What about the other table? ~ `PluginGroups`? The entries for that table is here.

```
vmware|tkg|default|v1.6.0|Plugins for TKG|feature|kubernetes|v0.25.0|false|false
vmware|tkg|default|v1.6.0|Plugins for TKG|pinniped-auth|global|v0.25.0|true|false
vmware|tkg|default|v1.6.0|Plugins for TKG|telemetry|kubernetes|v0.25.0|true|false
vmware|tkg|default|v1.6.0|Plugins for TKG|package|kubernetes|v0.25.0|true|false
vmware|tkg|default|v1.6.0|Plugins for TKG|secret|kubernetes|v0.25.0|true|false
vmware|tkg|default|v1.6.0|Plugins for TKG|management-cluster|kubernetes|v0.25.0|true|false
vmware|tkg|default|v1.6.0|Plugins for TKG|cluster|kubernetes|v0.25.0|false|false
vmware|tkg|default|v1.6.0|Plugins for TKG|kubernetes-release|kubernetes|v0.25.0|false|false
vmware|tkg|default|v1.6.1|Plugins for TKG|feature|kubernetes|v0.25.4|false|false
vmware|tkg|default|v1.6.1|Plugins for TKG|pinniped-auth|global|v0.25.4|true|false
vmware|tkg|default|v1.6.1|Plugins for TKG|telemetry|kubernetes|v0.25.4|true|false
vmware|tkg|default|v1.6.1|Plugins for TKG|package|kubernetes|v0.25.4|true|false
vmware|tkg|default|v1.6.1|Plugins for TKG|secret|kubernetes|v0.25.4|true|false
vmware|tkg|default|v1.6.1|Plugins for TKG|management-cluster|kubernetes|v0.25.4|true|false
vmware|tkg|default|v1.6.1|Plugins for TKG|cluster|kubernetes|v0.25.4|false|false
vmware|tkg|default|v1.6.1|Plugins for TKG|kubernetes-release|kubernetes|v0.25.4|false|false
vmware|tkg|default|v2.1.0|Plugins for TKG|isolated-cluster|global|v0.28.0|true|false
vmware|tkg|default|v2.1.0|Plugins for TKG|feature|kubernetes|v0.28.0|false|false
vmware|tkg|default|v2.1.0|Plugins for TKG|pinniped-auth|global|v0.28.0|true|false
vmware|tkg|default|v2.1.0|Plugins for TKG|telemetry|kubernetes|v0.28.0|true|false
vmware|tkg|default|v2.1.0|Plugins for TKG|package|kubernetes|v0.28.0|true|false
vmware|tkg|default|v2.1.0|Plugins for TKG|secret|kubernetes|v0.28.0|true|false
vmware|tkg|default|v2.1.0|Plugins for TKG|management-cluster|kubernetes|v0.28.0|true|false
vmware|tkg|default|v2.1.0|Plugins for TKG|cluster|kubernetes|v0.28.0|false|false
vmware|tkg|default|v2.1.0|Plugins for TKG|kubernetes-release|kubernetes|v0.28.0|false|false
vmware|tkg|default|v2.1.1|Plugins for TKG|isolated-cluster|global|v0.28.1|true|false
vmware|tkg|default|v2.1.1|Plugins for TKG|feature|kubernetes|v0.28.1|false|false
vmware|tkg|default|v2.1.1|Plugins for TKG|pinniped-auth|global|v0.28.1|true|false
vmware|tkg|default|v2.1.1|Plugins for TKG|telemetry|kubernetes|v0.28.1|true|false
vmware|tkg|default|v2.1.1|Plugins for TKG|package|kubernetes|v0.28.1|true|false
vmware|tkg|default|v2.1.1|Plugins for TKG|secret|kubernetes|v0.28.1|true|false
vmware|tkg|default|v2.1.1|Plugins for TKG|management-cluster|kubernetes|v0.28.1|true|false
vmware|tkg|default|v2.1.1|Plugins for TKG|cluster|kubernetes|v0.28.1|false|false
vmware|tkg|default|v2.1.1|Plugins for TKG|kubernetes-release|kubernetes|v0.28.1|false|false
vmware|tkg|default|v2.2.0|Plugins for TKG|isolated-cluster|global|v0.29.0|true|false
vmware|tkg|default|v2.2.0|Plugins for TKG|feature|kubernetes|v0.29.0|false|false
vmware|tkg|default|v2.2.0|Plugins for TKG|pinniped-auth|global|v0.29.0|true|false
vmware|tkg|default|v2.2.0|Plugins for TKG|telemetry|kubernetes|v0.29.0|true|false
vmware|tkg|default|v2.2.0|Plugins for TKG|package|kubernetes|v0.29.0|true|false
vmware|tkg|default|v2.2.0|Plugins for TKG|secret|kubernetes|v0.29.0|true|false
vmware|tkg|default|v2.2.0|Plugins for TKG|management-cluster|kubernetes|v0.29.0|true|false
vmware|tkg|default|v2.2.0|Plugins for TKG|cluster|kubernetes|v0.29.0|false|false
vmware|tkg|default|v2.2.0|Plugins for TKG|kubernetes-release|kubernetes|v0.29.0|false|false
...
```
more on that, another day.

## Why so complicated? 

I talked to mark khouzham about this.  It turns out that tanzu cli is evolving to be a central place where any product can
put their CLIs.  this means:
- Tanzu CLI has a central repository of child plugins, which support more then just TKG in the future.  For example, TMC and other plugins
can be stored in this central repository.

## Targets and Contexts

- Things like that Tanzu management-cluster plugin, which requires other things to be up to date, now tell tanzu cli "hey, can you sync plugins for me"? That way, it doesnt
have to compile the "tanzu plugin" binary into itself, and thus tanzu cli can easily be updated.
- There are these things called context plugins.... these are dependent on the management cluster.  In other words, the context plugins allow tanzu cli to determine
what version of a plugin it should be running for a given version of a TKG Management cluster.
- Then there is the concept of a *target*.  A target is not the same as a context - a target refers to WHAT installed your management cluster.  For example, if you are using
TMC, then your *target* is `tanzu mission-control cluster list`.  If using TKG, then `tanzu kubernetes cluster list `. 

## Independent Tanzu CLI

The new , independent tanzu cli is more modular than ones in the past. 

## Moving on 

Ok now lets see what happens when we run TKG in the normal manner... We'll now see where the plugin groups shown above
come in...  NOTE THAT in this case, im using a "pre release" of the tanzu plugins.

```
-> % cat plugins/plugin_inventory.db

...
09:06:58  INFO:root:====== 96   CMD: tanzu plugin install --group vmware-tkg/tkg:v2.3.0
09:06:59  [i] Reading plugin inventory for "projects.registry.vmware.com/tanzu_cli/plugins/plugin-inventory:latest", this will take a few seconds.
09:07:01  [i] Reading plugin inventory for "projects-stg.registry.vmware.com/tanzu_cli/plugins/sandbox/tkg-ob-464796462228756226-g885cc5ad/plugin-inventory:latest", this will take a few seconds.
09:07:01  [!] Skipping the plugins discovery image signature verification for "projects-stg.registry.vmware.com/tanzu_cli/plugins/sandbox/tkg-ob-464796462228756226-g885cc5ad/plugin-inventory:latest"
```
Now all of the plugins for this group *2.3.0* start getting installed: 

```
09:07:02  [i] Installing plugin 'isolated-cluster:v0.30.0-dev' with target 'global'
09:07:04  [i] Installing plugin 'management-cluster:v0.30.0-dev' with target 'kubernetes'
09:07:09  [i] Installing plugin 'package:v0.30.0-dev' with target 'kubernetes'
09:07:12  [i] Installing plugin 'pinniped-auth:v0.30.0-dev' with target 'global'
09:07:14  [i] Installing plugin 'secret:v0.30.0-dev' with target 'kubernetes'
09:07:16  [i] Installing plugin 'telemetry:v0.30.0-dev' with target 'kubernetes'
09:07:17  [ok] successfully installed all plugins from group 'vmware-tkg/tkg:v2.3.0'
```

At this point, all the plugins will be listed : 

```
09:07:17  INFO:root:====== 97   CMD: tanzu plugin list
09:07:17  Standalone Plugins
09:07:17    NAME                DESCRIPTION                                                        TARGET      VERSION      STATUS     
09:07:17    isolated-cluster    Prepopulating images/bundle for internet-restricted environments   global      v0.30.0-dev  installed  
09:07:17    pinniped-auth       Pinniped authentication operations (usually not directly invoked)  global      v0.30.0-dev  installed  
09:07:17    management-cluster  Kubernetes management cluster operations                           kubernetes  v0.30.0-dev  installed  
09:07:17    package             Tanzu package management                                           kubernetes  v0.30.0-dev  installed  
09:07:17    secret              Tanzu secret management                                            kubernetes  v0.30.0-dev  installed  
09:07:17    telemetry           configure cluster-wide settings for vmware tanzu telemetry         kubernetes  v0.30.0-dev  installed  
09:07:17  INFO:root:====== 98   CMD: touch .tanzu_plugins_v2.3.0
```


