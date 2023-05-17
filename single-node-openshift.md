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

Let's deploy a sample web server application to verify that the cluster and image registry are functional

The sample web server application will be deployed via templates which uses the [Source-to-image](https://docs.openshift.com/container-platform/4.12/openshift_images/using_images/using-s21-images.html) (S2I) to build the image, and deploy the image into pods

Go to `Developer` view:

![image](https://github.com/joetanx/setup/assets/90442032/951ed605-47d3-43e4-8b11-0ff591cfdd69)

Select `Create Project` in the project drop down menu:

![image](https://github.com/joetanx/setup/assets/90442032/0816b79e-23ec-45f4-a479-5d6301f4c265)

Give the project a name (e.g. `httpd`):

![image](https://github.com/joetanx/setup/assets/90442032/85bb417e-5bac-4bd0-93e8-65aa117bb52d)

Select `+Add` and browse `All services` under `Developer Catalog`:

![image](https://github.com/joetanx/setup/assets/90442032/5a964773-0580-4677-a698-0705314b2520)

Select `Apache HTTP Server` template:

![image](https://github.com/joetanx/setup/assets/90442032/b2ade3bd-59cc-4422-b823-44759c3c3f98)

Select `Instantiate Template`:

![image](https://github.com/joetanx/setup/assets/90442032/56075622-cf5e-4af4-a508-27a44ec8e807)

Accept defaults and click `Create`:

![image](https://github.com/joetanx/setup/assets/90442032/4868a34e-1b04-426c-ab54-6f51f5f997cf)

The build will start with `pending` status:

![image](https://github.com/joetanx/setup/assets/90442032/e06ee394-9a65-467f-842f-53e1e39f729e)

The build will change to `running` after pending for a short while:

> **Note**
> 
> If the build is stuck at `pending`, select `View logs` to investigate; there can be few reasons for the build to be stuck (e.g. non-functional image registry)

![image](https://github.com/joetanx/setup/assets/90442032/17713527-cc95-4086-819c-218ddb650d74)

The build will `complete` after the S2I procedure is completed:

![image](https://github.com/joetanx/setup/assets/90442032/8575abcd-cf46-416c-a381-9d3cc39b94f4)

The deployment will proceed to create pods from the image:

![image](https://github.com/joetanx/setup/assets/90442032/7492aaf2-91e1-45e9-b2a5-739c7d5d191d)

The application is ready when the pod status change to `Running`

![image](https://github.com/joetanx/setup/assets/90442032/921f943b-4243-4cb2-bf4b-5f861449e4db)

Click on the `Location` URL to open the sample web page:

![image](https://github.com/joetanx/setup/assets/90442032/375736e2-b24b-4a3b-a594-468b1fb31d51)

## Appendix A. Troubleshooting kubelet certificate

If the cluster is turned off (e.g. to conserve consumption), the cluster can become unfunctional when starting after some time

A trail of `Error getting node` errors will be logged on the kubelet status:

```console
[core@sno ~]$ systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─01-kubens.conf, 10-mco-default-env.conf, 10-mco-default-madv.conf, 20-logging.conf, 20-nodenet.conf
   Active: active (running) since Wed 2023-05-17 08:31:10 UTC; 28s ago
  Process: 2467 ExecStartPre=/bin/rm -f /var/lib/kubelet/memory_manager_state (code=exited, status=0/SUCCESS)
  Process: 2465 ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state (code=exited, status=0/SUCCESS)
  Process: 2463 ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
 Main PID: 2469 (kubelet)
    Tasks: 19 (limit: 101898)
   Memory: 127.7M
      CPU: 1.921s
   CGroup: /system.slice/kubelet.service
           └─2469 /usr/bin/kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/var/lib/kubelet/kubeconfig --container-runtime=remote --container-runti>
May 17 08:31:38 sno kubenswrapper[2469]: E0517 08:31:38.553161    2469 kubelet.go:2471] "Error getting node" err="node \"sno\" not found"
May 17 08:31:38 sno kubenswrapper[2469]: E0517 08:31:38.570304    2469 event.go:267] Server rejected event '&v1.Event{TypeMeta:v1.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:v1.ObjectMeta{Name:"etcd-sno.175f>
May 17 08:31:38 sno kubenswrapper[2469]: E0517 08:31:38.653358    2469 kubelet.go:2471] "Error getting node" err="node \"sno\" not found"
May 17 08:31:38 sno kubenswrapper[2469]: E0517 08:31:38.754047    2469 kubelet.go:2471] "Error getting node" err="node \"sno\" not found"
May 17 08:31:38 sno kubenswrapper[2469]: E0517 08:31:38.768191    2469 event.go:267] Server rejected event '&v1.Event{TypeMeta:v1.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:v1.ObjectMeta{Name:"etcd-sno.175f>
May 17 08:31:38 sno kubenswrapper[2469]: E0517 08:31:38.854999    2469 kubelet.go:2471] "Error getting node" err="node \"sno\" not found"
May 17 08:31:38 sno kubenswrapper[2469]: I0517 08:31:38.952113    2469 csi_plugin.go:993] Failed to contact API server when waiting for CSINode publishing: csinodes.storage.k8s.io "sno" is forbidden: User "sy>
May 17 08:31:38 sno kubenswrapper[2469]: E0517 08:31:38.955330    2469 kubelet.go:2471] "Error getting node" err="node \"sno\" not found"
May 17 08:31:38 sno kubenswrapper[2469]: E0517 08:31:38.967940    2469 event.go:267] Server rejected event '&v1.Event{TypeMeta:v1.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:v1.ObjectMeta{Name:"etcd-sno.175f>
May 17 08:31:39 sno kubenswrapper[2469]: E0517 08:31:39.055449    2469 kubelet.go:2471] "Error getting node" err="node \"sno\" not found"
```

The node will also be in a `NotReady` state:

```console
[core@sno ~]$ oc get nodes
NAME   STATUS     ROLES                         AGE     VERSION
sno    NotReady   control-plane,master,worker   2d20h   v1.25.8+37a9a08
```

A common cause of this is due to the expiry of kubelet certificates

The kubelet certificates generated from the SNO installation has a validity of about 13 hours for some reason:

```console
[core@sno ~]$ ls -l /var/lib/kubelet/pki/
total 8
-rw-------. 1 root root 1139 May 14 12:26 kubelet-client-2023-05-14-12-26-39.pem
lrwxrwxrwx. 1 root root   59 May 14 12:26 kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2023-05-14-12-26-39.pem
-rw-------. 1 root root 1167 May 14 12:26 kubelet-server-2023-05-14-12-26-55.pem
lrwxrwxrwx. 1 root root   59 May 14 12:26 kubelet-server-current.pem -> /var/lib/kubelet/pki/kubelet-server-2023-05-14-12-26-55.pem
[core@sno ~]$ sudo openssl x509 -noout -enddate -in /var/lib/kubelet/pki/kubelet-client-current.pem
notAfter=May 15 01:59:46 2023 GMT
[core@sno ~]$ sudo openssl x509 -noout -enddate -in /var/lib/kubelet/pki/kubelet-server-current.pem
notAfter=May 15 01:59:46 2023 GMT
```

The certificate should be rotated automatically when it reaches 80 percent of its validity

But if the cluster is turned off when it was due for certificate rotation, the certificate expires

Kubelet will request for new certificates, but it is left as `pending` and needs manual approval:

```console
[core@sno ~]$ oc get csr
NAME                                             AGE     SIGNERNAME                                    REQUESTOR                                                                         REQUESTEDDURATION   CONDITION
csr-f4zmj                                        2d20h   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
csr-r7zc2                                        2d20h   kubernetes.io/kubelet-serving                 system:node:sno                                                                   <none>              Approved,Issued
csr-thn42                                        58s     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Pending
system:openshift:openshift-authenticator-t2hnf   2d20h   kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-authentication-operator:authentication-operator   <none>              Approved,Issued
system:openshift:openshift-monitoring-z4gd5      2d20h   kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-monitoring:cluster-monitoring-operator            <none>              Approved,Issued
```

Approve the first request from `system:serviceaccount:openshift-machine-config-operator:node-bootstrapper`:

```console
[core@sno ~]$ oc adm certificate approve csr-thn42
certificatesigningrequest.certificates.k8s.io/csr-thn42 approved
```

After the first certificate is approved, kubelet will re-initialize and request for a second certificate after some time (about 3-5 mins):

```console
[core@sno ~]$ oc get csr
NAME                                             AGE     SIGNERNAME                                    REQUESTOR                                                                         REQUESTEDDURATION   CONDITION
csr-f4zmj                                        2d20h   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
csr-n2mkk                                        11s     kubernetes.io/kubelet-serving                 system:node:sno                                                                   <none>              Pending
csr-r7zc2                                        2d20h   kubernetes.io/kubelet-serving                 system:node:sno                                                                   <none>              Approved,Issued
csr-thn42                                        3m45s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
system:openshift:openshift-authenticator-t2hnf   2d20h   kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-authentication-operator:authentication-operator   <none>              Approved,Issued
system:openshift:openshift-monitoring-z4gd5      2d20h   kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-monitoring:cluster-monitoring-operator            <none>              Approved,Issued
```

Approve the second request from `system:node:sno`:

```console
[core@sno ~]$ oc adm certificate approve csr-n2mkk
certificatesigningrequest.certificates.k8s.io/csr-n2mkk approved
```

Verify that the expiry of the new certificates:

```console
[core@sno ~]$ sudo openssl x509 -noout -enddate -in /var/lib/kubelet/pki/kubelet-client-current.pem
notAfter=Jun 16 08:29:26 2023 GMT
[core@sno ~]$ sudo openssl x509 -noout -enddate -in /var/lib/kubelet/pki/kubelet-server-current.pem
notAfter=Jun 16 08:30:32 2023 GMT
```

After the new certificates are done, the SNO will take about 20-30 mins to do some work before the console is available
