# Introduction

Setup single-node Kubernetes cluster (with RHEL, CRI-O and Flannel)

### Software Versions

- RHEL 9.2
- CRI-O 1.27
- Kubernetes 1.27

# 1. Preparation

## 1.1. Install CRI-O

```console
OS=Fedora_37
VERSION=1.26
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
yum -y install cri-o
systemctl enable --now crio
```

## 1.2. Disable swap and SELinux

```console
swapoff -a
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## 1.3. Configure required kernel modules and tunables

- Ref: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#letting-iptables-see-bridged-traffic>

```console
cat <<EOF >> /etc/modules-load.d/k8s.conf
br_netfilter
EOF
modprobe br_netfilter
cat <<EOF >> /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

## 1.4. Install kubeadm, kubelet and kubectl

- Ref: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl>
- Note:
  - CRI-O uses the systemd cgroup driver per default. Ref: <https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-driver>
  - There are several ways to configure the cgroup driver, this guide configures it in `/etc/default/kubelet`. Ref: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#the-kubelet-drop-in-file-for-systemd>

```console
cat <<EOF >> /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
yum -y install kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```

### Optional - Clean-up Repository

```console
rm -f /etc/yum.repos.d/devel* /etc/yum.repos.d/kubernetes.repo
```

## 1.5. Configure firewall

- Ref: <https://kubernetes.io/docs/reference/ports-and-protocols/>
- Ref: <https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md#vxlan>

```console
firewall-cmd --add-port 2379-2380/tcp --permanent
firewall-cmd --add-port 6443/tcp --permanent
firewall-cmd --add-port 8472/udp --permanent
firewall-cmd --add-port 10250/tcp --permanent
firewall-cmd --add-port 10257/tcp --permanent
firewall-cmd --add-port 10259/tcp --permanent
firewall-cmd --add-port 30000-32767/tcp --permanent
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
```

# 2. Create Kubernetes cluster

- The `pull` command downloads the required container images, this is optional as the `init` command will also download the images if they are not already present
- The `--pod-network-cidr` option for `init` command is required for Flannel networking
- Ref: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/>

```console
kubeadm config images pull
kubeadm init --pod-network-cidr 10.244.0.0/16
```

## 2.1. Configure kubectl admin login and allow pods to run on master (single-node Kubernetes)

```console
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 2.2. Install Flannel networking

```console
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
- In case of `"cni0" already has an IP address different from 10.244.0.1/24` error
  - This error occurs when you deploy a pod immediately after creating the Kubernetes cluster
  - You can either reboot the node, or run below commands to recreate the CNI

```console
ip link del cni0
ip link del flannel.1
kubectl delete pod --selector=app=flannel -n kube-flannel
kubectl delete pod --selector=k8s-app=kube-dns -n kube-system
```

# 3. OPTIONAL - Setup Kubernetes dashboard

Ref <https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/>

## 3.1. Prepare the Kubernetes dashboard manifest file

Download the Kubernetes dashboard manifest file

```console
curl -O https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### 3.1.2. Edit the service configuration in manifest to allow external access

The original service configuration does not allow the dashboard to be access externally and requires running `kubectl proxy` to access the dashboard:

```console
⋮
kind: Service
⋮
  ports:
    - port: 443
      targetPort: 8443
⋮
```

It is much easier to just change service configuration to expose the dashboard via NodePort:

```console
⋮
kind: Service
⋮
  - nodePort: 30443
    port: 443
    protocol: TCP
    targetPort: 8443
  type: NodePort
⋮
```

### 3.1.3. Using own TLS certificates for the dashboard

The original deployment configuration automatically generates the TLS certficates used by the dashboard

#### Prepare the secrets for the TLS certificates:

This example uses my own certificates `kube.vx.pem` and `kube.vx.key`, change these to your certificate names

```console
kubectl create namespace kubernetes-dashboard
kubectl -n kubernetes-dashboard create secret generic kubernetes-dashboard-certs --from-file=tls.crt=kube.vx.pem --from-file=tls.key=kube.vx.key
```

#### Edit the deployment configuration in manifest to use the certificates

Original configuration:

```console
⋮
kind: Deployment
⋮
          args:
            - --auto-generate-certificates
⋮
```

Change configuration to:

```console
⋮
kind: Deployment
⋮
          args:
            - --tls-cert-file=/tls.crt
            - --tls-key-file=/tls.key
⋮
```

## 3.2. Deploy the dashboard

```console
kubectl apply -f recommended.yaml
```

Verify dashboard deployment

```console
kubectl -n kubernetes-dashboard get all
```

## 3.3. Accessing the dashboard

### 3.3.1. Create a service account to login to Kubernetes dashboard

Download and apply the manifest that will create:
- ServiceAccount: `dashboard-admin`
- Secret (service account token used to login to dashboard) : `dashboard-admin-secret`
- ClusterRoleBinding: binds `dashboard-admin` service account to `cluster-admin` role

Ref:
- <https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md>
- <https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-service-account-api-token>

```console
curl -O https://raw.githubusercontent.com/joetanx/setup/main/dashboard-serviceaccount.yaml
kubectl apply -f dashboard-serviceaccount.yaml
```

Verify items created:

```console
kubectl -n kubernetes-dashboard get serviceaccount
kubectl -n kubernetes-dashboard describe serviceaccount dashboard-admin
```

### 3.3.2. Login to Kubernetes dashboard

- Retrieve the token for `dashboard-admin`

```console
kubectl -n kubernetes-dashboard describe secrets dashboard-admin-secret
```

- Browse to `https://<kubernetes-fqdn>:30043` and login with the token

### Alternative login method

- The `dashboard-serviceaccount.yaml` downloaded from this repository creates a persistent token stored in Kubernetes secrets for the `dashboard-admin` service account
- You can edit the manifest to remove creation of the secret, and use `kubectl -n kubernetes-dashboard create token dashboard-admin` to generate an ephermeral token for dashboard login
