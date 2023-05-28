## Introduction

Setup single-node Kubernetes cluster with RHEL, CRI-O and Flannel

### Software Versions

- RHEL 9.2
- CRI-O 1.27
- Kubernetes 1.27

## 1. Preparation

### 1.1. Disable swap and SELinux

```console
swapoff -a
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 1.2. Configure repositories for CRI-O and Kubernetes

```console
OS=Fedora_38
VERSION=1.27
curl -sLo /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -sLo /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

### 1.3. Configure required kernel modules and tunables

- Ref: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic

```console
cat << EOF > /etc/modules-load.d/kubernetes.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter
cat <<EOF >> /etc/sysctl.d/kubernetes.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 1.4. Configure firewall

Ref:

- https://kubernetes.io/docs/reference/ports-and-protocols/
- https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md#vxlan

> **Note**
> 
> The `cni0` interface needs to be added to the firewalld zone to allow pod-to-pod communication
> 
> Otherwise, the NGINX ingress controller cannot route traffic to backend pods

```console
firewall-cmd --permanent --add-port 2379-2380/tcp
firewall-cmd --permanent --add-port 6443/tcp
firewall-cmd --permanent --add-port 8472/udp
firewall-cmd --permanent --add-port 10250/tcp
firewall-cmd --permanent --add-port 10257/tcp
firewall-cmd --permanent --add-port 10259/tcp
firewall-cmd --permanent --add-port 30000-32767/tcp
firewall-cmd --permanent --add-masquerade
firewall-cmd --permanent --add-interface cni0
firewall-cmd --reload
```

### 1.5. Install cri-o, kubeadm, kubelet and kubectl

Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

> **Note**
> 
> The CRI-O CNI default configuraton uses `10.85.0.0/16` subnet, below `sed` command changes this to `10.244.0.0/16` to align with Flannel default configuration

```console
yum -y install cri-o kubelet kubeadm kubectl
sed -i 's/10.85/10.244/' /etc/cni/net.d/100-crio-bridge.conflist
systemctl enable --now crio
systemctl enable --now kubelet
```

### Optional - Clean-up Repository

```console
rm -f /etc/yum.repos.d/devel* /etc/yum.repos.d/kubernetes.repo
```

## 2. Create Kubernetes cluster

- The `pull` command downloads the required container images, this is optional as the `init` command will also download the images if they are not already present
- The `--pod-network-cidr` option for `init` command aligns the cluster subnet with Flannel default configuration
- Ref: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/>

```console
kubeadm config images pull
kubeadm init --pod-network-cidr 10.244.0.0/16
```

### 2.1. Configure kubectl admin login and allow pods to run on master (single-node Kubernetes)

```console
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 2.2. Install Flannel networking

```console
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

In case of `"cni0" already has an IP address different from 10.244.0.1/24` error:

- This error can occur pods are deployed immediately after creating the Kubernetes cluster
- Reboot the node, or run below commands to recreate the CNI

```console
ip link del cni0
ip link del flannel.1
kubectl delete pod --selector=app=flannel -n kube-flannel
kubectl delete pod --selector=k8s-app=kube-dns -n kube-system
```

## 3. Setup Kubernetes dashboard

Ref https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

### 3.1. Configure the TLS certificates for the dashboard

The original deployment configuration automatically generates the TLS certficates used by the dashboard

This example uses my own certificates `kube.vx.pem` and `kube.vx.key`, change these to your certificate names

```console
kubectl create namespace kubernetes-dashboard
kubectl -n kubernetes-dashboard create secret generic kubernetes-dashboard-certs --from-file=tls.crt=kube.vx.pem --from-file=tls.key=kube.vx.key
```

#### Edit the deployment configuration in manifest to use the certificates

Download the Kubernetes dashboard manifest file:

```console
curl -sLO https://github.com/kubernetes/dashboard/raw/v2.7.0/aio/deploy/recommended.yaml
```

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

### 3.2. Deploy the dashboard

```console
kubectl apply -f recommended.yaml
```

Verify dashboard deployment

```console
kubectl -n kubernetes-dashboard get all
```

### 3.3. Allow external access to Kubernetes dashboard

The original service configuration does not allow the dashboard to be access externally and requires running `kubectl proxy` to access the dashboard

#### Option A: using `hostPort` to map the Kubernetes dashboard pods to host

```console
kubectl -n kubernetes-dashboard patch deploy kubernetes-dashboard --patch '{"spec":{"template":{"spec":{"containers":[{"name": "kubernetes-dashboard","ports":[{"containerPort": 8443,"hostPort":8443,"protocol":"TCP"}]}]}}}}'
kubectl -n kubernetes-dashboard delete service kubernetes-dashboard
```

#### Option B: using `nodePort` to expose the Kubernetes dashboard service

```console
kubectl -n kubernetes-dashboard patch service kubernetes-dashboard --patch '{"spec":{"ports":[{"nodePort":30443,"port":443,"targetPort": "8443"}]}}'
```

### 3.4. Accessing the dashboard

#### 3.4.1. Create a service account to login to Kubernetes dashboard

Download and apply the manifest that will create:
- ServiceAccount: `dashboard-admin`
- Secret (service account token used to login to dashboard) : `dashboard-admin-secret`
- ClusterRoleBinding: binds `dashboard-admin` service account to `cluster-admin` role

Ref:
- <https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md>
- <https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-service-account-api-token>

```console
curl -sLO https://github.com/joetanx/setup/raw/main/dashboard-serviceaccount.yaml
kubectl apply -f dashboard-serviceaccount.yaml
```

Verify items created:

```console
kubectl -n kubernetes-dashboard get serviceaccount
kubectl -n kubernetes-dashboard describe serviceaccount dashboard-admin
```

#### 3.3.2. Login to Kubernetes dashboard

Retrieve the token for `dashboard-admin`:

```console
kubectl -n kubernetes-dashboard describe secrets dashboard-admin-secret
```

Browse to `https://<kubernetes-fqdn>:30043` and login with the token

#### Alternative login method

- The `dashboard-serviceaccount.yaml` downloaded from this repository creates a persistent token stored in Kubernetes secrets for the `dashboard-admin` service account
- You can edit the manifest to remove creation of the secret, and use `kubectl -n kubernetes-dashboard create token dashboard-admin` to generate an ephermeral token for dashboard login

## 4. Setup NGINX ingress controller

Tasks:
- Apply NGINX ingress controller configuration
- Set NGINX ingress controller as default
- Delete the original service
- Map HTTP port from NGINX ingress controller directly to host

```console
kubectl apply -f https://github.com/kubernetes/ingress-nginx/raw/main/deploy/static/provider/baremetal/deploy.yaml
kubectl -n ingress-nginx annotate ingressclasses nginx ingressclass.kubernetes.io/is-default-class="true"
kubectl -n ingress-nginx delete service ingress-nginx-controller
kubectl -n ingress-nginx patch deploy ingress-nginx-controller --patch '{"spec":{"template":{"spec":{"containers":[{"name": "controller","ports":[{"containerPort": 80,"hostPort":80,"name":"http","protocol":"TCP"}]}]}}}}'
```
