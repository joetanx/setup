## 1. Cluster setup

### Optional: use Lab Issur from [lab-certs](https://github.com/joetanx/lab-certs) for cluster certificates

kubeadm detects the CA files in below directories and uses them when creating the cluster

```sh
mkdir -p /etc/kubernetes/pki/etcd
curl -sLo /etc/kubernetes/pki/ca.crt https://github.com/joetanx/lab-certs/raw/main/ca/lab_issuer.pem
curl -sLo /etc/kubernetes/pki/ca.key https://github.com/joetanx/lab-certs/raw/main/ca/lab_issuer.key
cp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/front-proxy-ca.crt
cp /etc/kubernetes/pki/ca.key /etc/kubernetes/pki/front-proxy-ca.key
cp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/etcd/ca.crt
cp /etc/kubernetes/pki/ca.key /etc/kubernetes/pki/etcd/ca.key
openssl pkey -in /etc/kubernetes/pki/ca.key -pubout -out /etc/kubernetes/pki/sa.pub
openssl pkey -in /etc/kubernetes/pki/ca.key -traditional > /etc/kubernetes/pki/sa.key
```

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

### 1.2. Configure Kubernetes repository

Ref: https://github.com/cri-o/packaging

> [!Tip]
>
> `containerd` is already available in Ubuntu default repositories

```sh
KUBERNETES_VERSION=v1.34
curl -sL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
cat << EOF > /etc/apt/sources.list.d/kubernetes.sources
Types: deb
URIs: https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/
Suites: /
Signed-By: /etc/apt/keyrings/kubernetes-apt-keyring.gpg
EOF
```

### 1.3. Configure firewall (if using firewalld)

Ref: https://kubernetes.io/docs/reference/ports-and-protocols/

> [!NOTE]
> 
> Calico interfaces needs to be added to the firewalld zone using `--add-interface cali+` to allow pod-to-pod communication
> 
> Otherwise, the ingress controller cannot route traffic to backend pods
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

### 1.4. Install packages and configure containerd

Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

```sh
apt update && apt -y install vim containerd kubelet kubeadm kubectl
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl enable containerd kubelet
```

### 1.5. Create Kubernetes cluster

- The `pull` command downloads the required container images, this is optional as the `init` command will also download the images if they are not already present
- The `--pod-network-cidr` option for `init` command aligns the cluster subnet with Calico default configuration
- Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

```sh
kubeadm config images pull
kubeadm init --node-name $(hostname) --pod-network-cidr=10.244.0.0/16
```

### 1.6. Configure kubectl admin login and allow pods to run on master (single-node Kubernetes)

```sh
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 1.7. Install Calico networking

```sh
LATEST=$(curl -sI https://github.com/projectcalico/calico/releases/latest | grep location | cut -d '/' -f 8 | tr -d '\r')
curl -sL https://github.com/projectcalico/calico/raw/$LATEST/manifests/calico.yaml | sed '/CALICO_IPV4POOL_CIDR/s/# //' | sed '/192.168.0.0/s/# //' | sed 's/192.168/10.85/' | kubectl create -f -
```

### 1.8. Setup helm

```sh
VERSION=$(curl -sI https://github.com/helm/helm/releases/latest | grep location: | cut -d / -f 8 | tr -d '\r')
curl -O https://get.helm.sh/helm-$VERSION-linux-amd64.tar.gz
tar xvf helm-$VERSION-linux-amd64.tar.gz && rm -f helm-$VERSION-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/ && rm -rf linux-amd64
```

## 2. Setup cert-manager

### 2.1. Deploy cert-manager

```sh
LATEST=$(curl -sI https://github.com/cert-manager/cert-manager/releases/latest | grep location | cut -d '/' -f 8 | tr -d '\r')
kubectl create -f https://github.com/cert-manager/cert-manager/releases/download/$LATEST/cert-manager.yaml
```

### 2.2. Configure the certificate issuer

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

## 3. Setup Traefik

> [!Tip]
>
> Traefik supports both `Gateway` API and `Ingress` Kubernetes resources
>
> It also extends Kubernetes with `IngressRoute` CRD that implements a syntax similar to its own routing configuration definitions
>
> `Ingress` is chosen because:
> 1. Portability in case there is a need to change the ingress management
> 2. `Gateway` certificates are configured at listener level and doesn't support automatic issuance with self-managed CAs
> 3. `Ingress` certificates are configured at individual `Ingress` resources with automatic issuance with self-managed CAs 

```sh
curl -sLO https://github.com/joetanx/setup/raw/refs/heads/main/traefik/helm/values.yaml
cat << EOF >> values.yaml
image:
  registry: docker.io
  repository: traefik
  tag: v3.6.4
EOF
helm repo add traefik https://traefik.github.io/charts
helm upgrade --install traefik traefik/traefik --create-namespace --namespace traefik -f values.yaml && rm -f values.yaml
```

## 4. Setup Kubernetes dashboard

Ref: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

### 4.1. Deploy Kubernetes dashboard

```sh
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard -f values.yaml
```

### 4.2. Create Traefik ingress

The Kubernetes dashboard helm chart supports creation of Nginx ingress (deprecated)

It is possible to use it to create Traefik-based ingress with this `values.yaml`:

```yaml
app:
  ingress:
    enabled: true
    hosts:
      - kube.vx
    ingressClassName: traefik
    issuer:
      name: ca-issuer
      scope: cluster
    tls:
      secretName: dashboard
    annotations:
      cert-manager.io/cluster-issuer: ca-issuer
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      traefik.ingress.kubernetes.io/router.tls: "true"
```

However:
1. It doesn't create the Traefik `ServersTransport` need for `insecureSkipVerify` as nginx ingress had `nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"` label
2. The ingress it creates will have several ` nginx.ingress.kubernetes.io/...` annotations that were meant for nginx ingress

Hence, the ingress (and Traefik `ServersTransport`) for Kubernetes dashboard will be created with [dashboard-ingress.yaml](/kubernetes/dashboard-ingress.yaml):

```sh
kubectl create -f https://github.com/joetanx/setup/raw/refs/heads/main/kubernetes/dashboard-ingress.yaml
```

`insecureSkipVerify` is required because the `kubernetes-dashboard-kong-proxy` services uses an untrusted certificate, which causes backend TLS connection error

Annotate the `kubernetes-dashboard-kong-proxy` service with `traefik.ingress.kubernetes.io/service.serverstransport` annotation to hook it up with the Traefik `ServersTransport`:

```sh
kubectl -n dashboard annotate service kubernetes-dashboard-kong-proxy traefik.ingress.kubernetes.io/service.serverstransport=dashboard-skipverify@kubernetescrd
```

### 4.3. Accessing the dashboard

#### 4.3.1. Create a service account to login to Kubernetes dashboard

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

#### 4.2.2. Login to Kubernetes dashboard

Retrieve the token for `dashboard-admin`:

```sh
kubectl -n kubernetes-dashboard describe secrets dashboard-admin-secret
```

Browse to `https://<kubernetes-fqdn>` and login with the token

#### Alternative login method

- The `dashboard-serviceaccount.yaml` downloaded from this repository creates a persistent token stored in Kubernetes secrets for the `dashboard-admin` service account
- You can edit the manifest to remove creation of the secret, and use `kubectl -n kubernetes-dashboard create token dashboard-admin` to generate an ephermeral token for dashboard login
