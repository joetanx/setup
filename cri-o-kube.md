## 0. Introduction

Setup
- Single-node Kubernetes cluster on RHEL with CRI-O container runtime and Flannel CNI
- [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx/) for exposing services
- [cert-manager](https://cert-manager.io/) for automatic certificate issuance to services
- [Kubernetes dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) for UI management to the cluster

### 0.1. Software Versions

- RHEL 9.4
- CRI-O 1.30
- Kubernetes 1.30

## 1. Preparation

### 1.1. Disable swap and configure required kernel modules and tunables

Ref: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic

```sh
swapoff -a
cat << EOF > /etc/modules-load.d/kubernetes.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter
cat << EOF > /etc/sysctl.d/kubernetes.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 1.2. Configure repositories for CRI-O and Kubernetes

Ref:
- https://github.com/cri-o/packaging
- https://kubernetes.io/blog/2023/10/10/cri-o-community-package-infrastructure/

```sh
KUBERNETES_VERSION=v1.30
PROJECT_PATH=stable:/v1.30
cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF
cat << EOF > /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/rpm/repodata/repomd.xml.key
EOF
```

### 1.3. Configure firewall

Ref:

- https://kubernetes.io/docs/reference/ports-and-protocols/
- https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md#vxlan

> [!NOTE]
> 
> Calico interfaces needs to be added to the firewalld zone using `--add-interface cali+` to allow pod-to-pod communication
> 
> Otherwise, the NGINX ingress controller cannot route traffic to backend pods
>
> Ref: https://stackoverflow.com/questions/67701010/kubernetes-cluster-with-firewall-enabled-on-centoscalico-not-working

```sh
firewall-cmd --permanent --add-port 2379-2380/tcp
firewall-cmd --permanent --add-port 6443/tcp
firewall-cmd --permanent --add-port 8472/udp
firewall-cmd --permanent --add-port 10250/tcp
firewall-cmd --permanent --add-port 10257/tcp
firewall-cmd --permanent --add-port 10259/tcp
firewall-cmd --permanent --add-port 30000-32767/tcp
firewall-cmd --permanent --add-masquerade
firewall-cmd --permanent --add-interface "cali+"
firewall-cmd --reload
```

### 1.4. Install cri-o, kubeadm, kubelet and kubectl

Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

```sh
yum -y install container-selinux cri-o kubelet kubeadm kubectl
systemctl enable --now crio
systemctl enable --now kubelet
```

#### Optional - Clean-up Repository

```sh
rm -f /etc/yum.repos.d/kubernetes.repo /etc/yum.repos.d/cri-o.repo
```

## 2. Create Kubernetes cluster

- The `pull` command downloads the required container images, this is optional as the `init` command will also download the images if they are not already present
- The `--pod-network-cidr` option for `init` command aligns the cluster subnet with Calico default configuration
- Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

```sh
kubeadm config images pull
kubeadm init --node-name $(hostname) --pod-network-cidr=10.85.0.0/16
```

### 2.1. Configure kubectl admin login and allow pods to run on master (single-node Kubernetes)

```sh
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 2.2. Install Calico networking

```sh
LATEST=$(curl -sI https://github.com/projectcalico/calico/releases/latest | grep location | cut -d '/' -f 8 | tr -d '\r')
curl -sL https://github.com/projectcalico/calico/raw/$LATEST/manifests/calico.yaml | sed '/CALICO_IPV4POOL_CIDR/s/# //' | sed '/192.168.0.0/s/# //' | sed 's/192.168/10.85/' | kubectl create -f -
```

## 3. Setup NGINX ingress controller

Ref: https://kubernetes.github.io/ingress-nginx/deploy/

### 3.1. Deploy NGINX ingress controller

```sh
kubectl create -f https://github.com/kubernetes/ingress-nginx/raw/main/deploy/static/provider/baremetal/deploy.yaml
```

### 3.2. Set as default ingress class

```sh
kubectl -n ingress-nginx annotate ingressclasses nginx ingressclass.kubernetes.io/is-default-class="true"
```

### 3.3. Expose the ingress controller with `hostPort` on HTTP (`80`) and HTTPS (`443`)

- The NGINX ingress controller will be the only ingress to the cluster and the deployment runs only 1 replica
- The commands below deletes the default ingress controller service and update the deployment ports configuration to `hostPort`

```sh
kubectl -n ingress-nginx delete service ingress-nginx-controller
kubectl -n ingress-nginx patch deploy ingress-nginx-controller --patch '{"spec":{"template":{"spec":{"containers":[{"name": "controller","ports":[{"containerPort": 80,"hostPort":80,"name":"http","protocol":"TCP"},{"containerPort": 443,"hostPort":443,"name":"https","protocol":"TCP"}]}]}}}}'
```

## 4. Setup cert-manager

### 4.1. Deploy cert-manager

```sh
LATEST=$(curl -sI https://github.com/cert-manager/cert-manager/releases/latest | grep location | cut -d '/' -f 8 | tr -d '\r')
kubectl create -f https://github.com/cert-manager/cert-manager/releases/download/$LATEST/cert-manager.yaml
```

### 4.2. Configure the certificate issuer

- cert-manager supports various [issuers](https://cert-manager.io/docs/configuration/)
- The commands below uses the Kubernetes CA as the [CA issuer](https://cert-manager.io/docs/configuration/ca/)
- Replace the `CACERT` and `CAKEY` variables accordingly with the CA in the environment

```sh
CACERT=$(cat /etc/kubernetes/pki/ca.crt | base64 -w0)
CAKEY=$(cat /etc/kubernetes/pki/ca.key | base64 -w0)
cat << EOF > ca-anchor.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ca-key-pair
  namespace: cert-manager
data:
  tls.crt: $CACERT
  tls.key: $CAKEY
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: ca-key-pair
EOF
kubectl apply -f ca-anchor.yaml && rm -f ca-anchor.yaml
```

## 5. Setup helm

```sh
VERSION=$(curl -sI https://github.com/helm/helm/releases/latest | grep location: | cut -d / -f 8 | tr -d '\r')
curl -O https://get.helm.sh/helm-$VERSION-linux-amd64.tar.gz
tar xvf helm-$VERSION-linux-amd64.tar.gz && rm -f helm-$VERSION-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/ && rm -rf linux-amd64
```

## 6. Setup Kubernetes dashboard

Ref: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

### 6.1. Optional: prepare override values for helm installation

Below example values performs the following:
- Enable the ingress and configure it to use the ingress class named `nginx`
- Configure cert-manager to use the cluster issuer named `ca-issuer` to issue certificates

```yaml
app:
  ingress:
    enabled: true
    hosts:
      - delta.vx
    ingressClassName: nginx
    useDefaultIngressClass: false
    # Can also use "useDefaultIngressClass: true"
    issuer:
      name: ca-issuer
      scope: cluster
    tls:
      secretName: kubernetes-dashboard-tls
```

### 6.2. Deploy Kubernetes dashboard

```sh
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard -f values.yaml
```

### 6.3. Accessing the dashboard

#### 6.3.1. Create a service account to login to Kubernetes dashboard

Download and apply the manifest file, which creates:
- ServiceAccount: `dashboard-admin`
- Secret (service account token used to login to dashboard) : `dashboard-admin-secret`
- ClusterRoleBinding: binds `dashboard-admin` service account to `cluster-admin` role

Ref:
- https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-service-account-api-token

```sh
curl -sLO https://github.com/joetanx/setup/raw/main/dashboard-serviceaccount.yaml
kubectl apply -f dashboard-serviceaccount.yaml
```

Verify items created:

```sh
kubectl -n kubernetes-dashboard get serviceaccount
kubectl -n kubernetes-dashboard describe serviceaccount dashboard-admin
```

#### 6.2.2. Login to Kubernetes dashboard

Retrieve the token for `dashboard-admin`:

```sh
kubectl -n kubernetes-dashboard describe secrets dashboard-admin-secret
```

Browse to `https://<kubernetes-fqdn>` and login with the token

#### Alternative login method

- The `dashboard-serviceaccount.yaml` downloaded from this repository creates a persistent token stored in Kubernetes secrets for the `dashboard-admin` service account
- You can edit the manifest to remove creation of the secret, and use `kubectl -n kubernetes-dashboard create token dashboard-admin` to generate an ephermeral token for dashboard login
