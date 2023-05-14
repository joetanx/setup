## 0. Setting up a single-node Openshift

There are a few methods to setup Openshift, depending on the [platforms](https://docs.openshift.com/container-platform/4.12/installing/index.html#supported-platforms-for-openshift-clusters_ocp-installation-overview) to be deployed on.

This guide is for setting up a **single-node Openshift (SNO)** on Hyper-V (or any other) virtual machine

There are 2 methods for such environment:

1. [Assisted Installer](https://docs.openshift.com/container-platform/4.12/installing/installing_sno/install-sno-installing-sno.html#installing-single-node-openshift-using-the-assisted-installer)

2. [Agent-based Installer](https://docs.openshift.com/container-platform/4.12/installing/installing_sno/install-sno-installing-sno.html#installing-single-node-openshift-manually)

In short, select `Create Cluster` for Assisted Installer or `Run Agent-based Installer locally` for Agent-based Installer in the Create Cluster page:

![image](https://github.com/joetanx/setup/assets/90442032/17a06356-0d48-4e31-809e-a5eed44c6c2b)

This guide walks through the **Agent-based Installer** method

## 1. Prepare the Agent Installer ISO

### 1.1. Download the installer and pull secret:

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

```
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

## 2. Installing Openshift with the Agent Installer ISO

### 2.1. Boot the target VM to the generated Agent Installer ISO

The target VM boots into RHCOS and will take a few moments to discover itself:

![image](https://github.com/joetanx/setup/assets/90442032/caefbd04-e38d-40bd-9ec0-5d6c9b2bf4c4)

The installation process will start automatically once the discovery is completed:

![image](https://github.com/joetanx/setup/assets/90442032/8d438050-d7df-4bbe-9fd5-198e40c5a01e)
