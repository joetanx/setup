## 1. Setup single-node Openshift with agent-based installer

Openshift can be deployed on various [platforms](https://docs.openshift.com/container-platform/4.12/installing/index.html#supported-platforms-for-openshift-clusters_ocp-installation-overview)

This guide walks through the [agent-based installer](https://docs.openshift.com/container-platform/4.12/installing/installing_sno/install-sno-installing-sno.html#installing-single-node-openshift-manually) method, which is suitable for generic VM-based installation

### 1.1. Download the installer and pull secret:

Select `Run Agent-based Installer locally` in the `Create Cluster` page:

![image](https://github.com/joetanx/setup/assets/90442032/17a06356-0d48-4e31-809e-a5eed44c6c2b)

Download the installer and pull secret:

![image](https://github.com/joetanx/setup/assets/90442032/1abc902b-4028-4b0f-98bc-b665b62e7f1c)

### 1.2. Prepare the configuration files

There are 2 configuration files required to generate the agent-based installer:

1. `install-config.yaml`: Specifies clustering details of the Openshift cluster platform
2. `agent-config.yaml`: Specifies networking details of the Openshift cluster nodes

Examples of the configuration files can be found in steps 12 and 13 of <https://docs.openshift.com/container-platform/4.12/installing/installing_with_agent_based_installer/installing-with-agent-based-installer.html>

<details><summary>Configuration files used for this lab</summary>

`install-config.yaml`

```yaml
apiVersion: v1
baseDomain: vx
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: sno
  replicas: 1
metadata:
  name: sno
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.17.0/24
  networkType: OVNKubernetes 
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: 'redacted'
sshKey: 'redacted'
```

`agent-config.yaml`

```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: sno
hosts: 
  - hostname: sno
    interfaces:
      - name: eth1
        macAddress: 00:15:5d:00:00:5d
    rootDeviceHints: 
      deviceName: /dev/sda
    networkConfig:
      interfaces:
        - name: eth1
          type: ethernet
          state: up
          mac-address: 00:15:5d:00:00:5d
          ipv4:
            enabled: true
            address:
              - ip: 192.168.17.93
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.17.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.17.1
            next-hop-interface: eth1
            table-id: 254
```

</details>

### 1.3. Generate the Agent Installer ISO

A separate machine (not the target VM where the SNO is to be deployed on) is needed to generate the Agent Installer ISO

Install nmstatectl: `yum -y install /usr/bin/nmstatectl`

Unpack the installer package:

```console
[root@foxtrot ~]# tar xvf openshift-install-linux.tar.gz
README.md
openshift-install
```

Check the unpacked files and note that `install-config.yaml` and `agent-config.yaml` are in the same directory as `openshift-install`

```console
[root@foxtrot ~]# ls -lh
total 2.0G
-rw-r--r--. 1 root root  760 May 14 08:01 agent-config.yaml
-rw-r--r--. 1 root root 3.4K May 14 08:01 install-config.yaml
-rwxr-xr-x. 1 root root 473M Apr 19 15:32 openshift-install
-rw-r--r--. 1 root root 335M May 14 07:58 openshift-install-linux.tar.gz
-rw-r--r--. 1 root root  706 Apr 19 15:32 README.md
```

Generate the Agent Installer ISO:

```console
[root@foxtrot ~]# ./openshift-install agent create image
INFO The rendezvous host IP (node0 IP) is 192.168.17.93
INFO Extracting base ISO from release payload
WARNING Failed to extract base ISO from release payload - check registry configuration
INFO Downloading base ISO
INFO Consuming Install Config from target directory
INFO Consuming Agent Config from target directory
```

Verify that `agent.x86_64.iso` and `auth` files are generated: 

```console
[root@foxtrot ~]# ls -lh
total 2.0G
-rw-r--r--. 1 root root 1.2G May 14 08:00 agent.x86_64.iso
drwxr-x---. 2 root root   50 May 14 08:00 auth
-rwxr-xr-x. 1 root root 473M Apr 19 15:32 openshift-install
-rw-r--r--. 1 root root 335M May 14 07:58 openshift-install-linux.tar.gz
-rw-r--r--. 1 root root  706 Apr 19 15:32 README.md
[root@foxtrot ~]# ls -lh auth
total 16K
-rw-r-----. 1 root root   23 May 14 08:00 kubeadmin-password
-rw-r-----. 1 root root 8.8K May 14 08:00 kubeconfig
```

### 1.4. Boot the target VM to the agent installer ISO

Specifications of the SNO VM:

|Specification|Size|Remarks|
|---|---|---|
|vCPU|8||
|Memory|16GB||
|/dev/sda|120GB||
|/dev/sdb|120GB|Persistent storage for image registry|

The target VM boots into RHCOS and will take a few moments to discover itself:

![image](https://github.com/joetanx/setup/assets/90442032/caefbd04-e38d-40bd-9ec0-5d6c9b2bf4c4)

The installation process will start automatically once the discovery is completed:

![image](https://github.com/joetanx/setup/assets/90442032/8d438050-d7df-4bbe-9fd5-198e40c5a01e)

### 1.5. Installation completed

The VM will reboot twice during the installation process

It will look nothing is happening after the second reboot, but it is still running the setup

After about an hour, the cluster registration on https://console.redhat.com/ should complete and show its status:

![image](https://github.com/joetanx/setup/assets/90442032/7538fba9-dcc0-4152-a5b8-9c0721c9aaaf)

### 1.6. Accessing the SNO

#### 1.6.1. Web UI

The cluster management page can be accessed at https://console-openshift-console.apps.cluster-domain/

Login to the cluster management page using `kubeadmin` user and the password from `auth/kubeadmin-password` file that was generated during the agent installer ISO creation

![image](https://github.com/joetanx/setup/assets/90442032/f9b038d8-3c1f-4e8d-a1ff-e629ea1e6b16)

> **Note**
>
> DNS conditional forwarder needs to be configured in the environment DNS server to forward requests for `*.cluster-domain` to the SNO

#### 1.6.2. SSH

The only user in RHCOS is `core` and the `sshKey` specified in `install-config.yaml` added to the `.ssh/authorized_keys` of this user

```console
[root@foxtrot ~]# ssh -i id_ecdsa core@sno.vx
Warning: Permanently added 'sno.vx' (ED25519) to the list of known hosts.
Red Hat Enterprise Linux CoreOS 412.86.202304260244-0
  Part of OpenShift 4.12, RHCOS is a Kubernetes native operating system
  managed by the Machine Config Operator (`clusteroperator/machine-config`).

WARNING: Direct SSH access to machines is not recommended; instead,
make configuration changes via `machineconfig` objects:
  https://docs.openshift.com/container-platform/4.12/architecture/architecture-rhcos.html

---
[core@sno ~]$ id
uid=1000(core) gid=1000(core) groups=1000(core),4(adm),10(wheel),16(sudo),190(systemd-journal) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

#### 1.6.3. oc client

To manage the cluster with oc client, setup authentication with the `auth/kubeconfig` file that was generated during the agent installer ISO creation

oc client can also be used from another other platform: https://docs.openshift.com/container-platform/4.12/cli_reference/openshift_cli/getting-started-cli.html

```console
[core@sno ~]$ mkdir .kube
[core@sno ~]$ cat << EOF > .kube/config
> clusters:
> - cluster:
>     certificate-authority-data: <redacted>
>     server: https://api.sno.vx:6443
>   name: sno
> contexts:
> - context:
>     cluster: sno
>     user: admin
>   name: admin
> current-context: admin
> preferences: {}
> users:
> - name: admin
>   user:
>     client-certificate-data: <redacted>
>     client-key-data: <redacted>
> EOF
[core@sno ~]$ oc get nodes
NAME   STATUS   ROLES                         AGE   VERSION
sno    Ready    control-plane,master,worker   11h   v1.25.8+37a9a08
```

## 2. Configure image registry

The OpenShift image registry operator bootstraps itself as `Removed` during the installation as the SNO on VM does not provide a shareable object storage

To complete the cluster setup, the image registry needs to be functional

### 2.1. Configure persistent storage using local volumes

A 120GB disk was provisioned to the VM and available at `/dev/sdb` of the VM; the most straightforward method to use this disk is to configure it as local volume

Ref: https://docs.openshift.com/container-platform/4.12/storage/persistent_storage/persistent_storage_local/persistent-storage-local.html#local-create-cr-manual_persistent-storage-local

Example persistent volume configuration:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-image-registry
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  volumeName: pv-image-registry
  storageClassName: local-storage
  local:
    path: /dev/sdb
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - sno
```

Example persistent volume claim configuration

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-image-registry
  namespace: openshift-image-registry
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 100Gi
  storageClassName: local-storage
  volumeName: pv-image-registry
```

Applying pv and pvc together, and verify creation:

```console
[core@sno ~]$ cat << EOF | oc apply -f -
> apiVersion: v1
> kind: PersistentVolume
> metadata:
>   name: pv-image-registry
> spec:
>   capacity:
>     storage: 100Gi
>   accessModes:
>     - ReadWriteOnce
>   persistentVolumeReclaimPolicy: Delete
>   volumeName: pv-image-registry
>   storageClassName: local-storage
>   local:
>     path: /dev/sdb
>   nodeAffinity:
>     required:
>       nodeSelectorTerms:
>       - matchExpressions:
>         - key: kubernetes.io/hostname
>           operator: In
>           values:
>           - sno
> ---
> kind: PersistentVolumeClaim
> apiVersion: v1
> metadata:
>   name: pvc-image-registry
>   namespace: openshift-image-registry
> spec:
>   accessModes:
>   - ReadWriteOnce
>   volumeMode: Filesystem
>   resources:
>     requests:
>       storage: 100Gi
>   storageClassName: local-storage
>   volumeName: pv-image-registry
> EOF
persistentvolume/pv-image-registry created
persistentvolumeclaim/pvc-image-registry created
[core@sno ~]$ oc get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                         STORAGECLASS    REASON   AGE
pv-image-registry   100Gi      RWO            Delete           Bound    openshift-image-registry/pvc-image-registry   local-storage            14s
[core@sno ~]$ oc -n openshift-image-registry get pvc
NAME                 STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-image-registry   Bound    pv-image-registry   100Gi      RWO            local-storage   19s
```

### 2.2. Configure the image registry’s management state

#### 2.2.1. Using emptyDir{}

If persistent storage configuration from [2.1.](#21-configure-persistent-storage-using-local-volumes) is not available, image registry can still be run without persistent storage by using `emptyDir{}`

`emptyDir{}` is a memory-backed non-persistent storage, using this is a quick way to get image registry running, but all images are lost if the registry restarts

Ref: https://docs.openshift.com/container-platform/4.12/registry/configuring_registry_storage/configuring-registry-storage-baremetal.html#installation-registry-storage-non-production_configuring-registry-storage-baremetal

```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}},"managementState":"Managed"}}'
```

#### 2.2.2. Using the persistent storage configuration from [2.1.](#21-configure-persistent-storage-using-local-volumes)

```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"pvc":{"claim":"pvc-image-registry"}},"rolloutStrategy":"Recreate","replicas":1,"managementState":"Managed"}}'
```

### 2.3. Verify image registry pods

`image-registry-##########-#####` pod should be `Running` if the configuration is successful

```console
[core@sno ~]$ oc -n openshift-image-registry get pods
NAME                                              READY   STATUS    RESTARTS   AGE
cluster-image-registry-operator-b9dd5984c-fdhfq   1/1     Running   1          11h
image-registry-58b4b57fc6-b46m4                   1/1     Running   0          25s
node-ca-642vg                                     1/1     Running   1          11h
```

## 3. Test app deployment
