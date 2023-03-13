# Generate a self-signed certificate chain with openssl

## Example used for this guide: Conjur
This guide is applicable for both Conjur Open Source Suite (OSS) and Enterprise Edition

### Conjur Open Source Suite
- Conjur OSS relies on the nginx proxy container to provide communication over TLS
- The default method to deploy Conjur OSS using quick start includes an openssl container in the Docker Compose manifest
  - Read: <https://github.com/cyberark/conjur-quickstart>
- In my method of deploying a minimal Conjur OSS using `podman play kube`, you can generate the certificate using openssl, then put it in a volume mount to `/etc/nginx/tls` of the nginx container
  - Read: <https://joetanx.github.com/conjur-oss>

### Conjur Enterprise Edition
- Conjur EE uses certificates for communication between the Master, Standby, and follower nodes in Conjur cluster.
- To understand Conjur certificate architecture, read: <https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Deployment/HighAvailability/certificate-architecture.htm>

### Certificate to be generated in this guide

| Role | Common Name | Subject Alternative Name |
| --- | --- | --- |
| Certificate Authority | vx Lab Certificate Authority | |
| Conjur Server | conjur.vx | conjur.vx, cj1.vx, cj2.vx, cj3.vx |
| Conjur Follower | follower.vx | follower.vx, flr1.vx, flr2.vx, follower.conjur.svc.cluster.local |

## 1. Generate a self-signed certificate authority
### 1.1 Generate private key
- RSA
```console
openssl genrsa -out vxLabCA.key 3072
```
- ECDSA
```console
openssl ecparam -name secp384r1 -genkey -out vxLabCA.key
```

### 1.2 Prepare openssl config
> **Note**:
> - Change the common name of the certificate according to your environment
> - There is a sample openssl config file at `/etc/pki/tls/openssl.cnf` that you can refer to for more details

```console
cat << EOF >> vxLabCA.cnf
[ req ]
prompt = no
distinguished_name = req_distinguished_name
[ req_distinguished_name ]
commonName = vx Lab Certificate Authority
EOF
```

### 1.3 Generate certificate
```console
openssl req -x509 -new -nodes -sha256 -days 365 -key vxLabCA.key -config vxLabCA.cnf -out vxLabCA.pem
```

## 2. Generate certificate for Conjur Server
> **Note**:
> - Change the common name/subject alternative name of the certificate according to your environment
> - The name must match the Conjur Server FQDN that the clients or followers will be using to communicate to the Conjur leader or cluster
> - If your Conjur Enterprise deployment includes master/standby setup, you need to include the FQDNs of the load balancer, master server, and standby servers in the certificate SAN

### 2.1 Generate private key
- **Note**: Conjur currently supports only RSA keys
```console
openssl genrsa -out conjur.vx.key 2048
```

### 2.2 Prepare openssl config
> **Note**:
> - Change the common name of the certificate according to your environment
> - There is a sample openssl config file at `/etc/pki/tls/openssl.cnf` that you can refer to for more details

```console
cat << EOF >> conjur.vx.cnf
[ req ]
prompt = no
distinguished_name = req_distinguished_name
req_extensions = req_ext
[ req_distinguished_name ]
commonName = conjur.vx
[ req_ext ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = conjur.vx
DNS.2 = cjr1.vx
DNS.3 = cjr2.vx
DNS.4 = cjr3.vx
EOF
```

> Note: `subjectAltName = DNS:conjur.vx, DNS:cjr1.vx, DNS:cjr2.vx, DNS:cjr3.vx` will also work for the subject alternative names

### 2.3 Create certificate signing request
```console
openssl req -new -config conjur.vx.cnf -key conjur.vx.key -out conjur.vx.csr
```

### 2.4 Generate certificate
```console
openssl x509 -req -sha256 -days 365 -CA vxLabCA.pem -CAkey vxLabCA.key -CAcreateserial -in conjur.vx.csr -extfile conjur.vx.cnf -extensions req_ext -out conjur.vx.pem
```

## 3. Generate certificate for Conjur Server
> **Note**:
> - Change the common name/subject alternative name of the certificate according to your environment
> - The name must match the follower FQDN that the follower will be using to communicate to the Conjur leader or cluster
> - Include the FQDN for all follower nodes in the subject alternative name
> - For follower deployment in Kubernetes, the name will be the Kubernetes service FQDN in the form of `<service-name>.<namespace>.svc.cluster.local`
> - Conjur OSS does not support followers

### 3.1 Generate private key
- **Note**: Conjur currently supports only RSA keys
```console
openssl genrsa -out follower.vx.key 2048
```

### 3.2 Prepare openssl config
> **Note**:
> - Change the common name of the certificate according to your environment
> - There is a sample openssl config file at `/etc/pki/tls/openssl.cnf` that you can refer to for more details

```console
cat << EOF >> follower.vx.cnf
[ req ]
prompt = no
distinguished_name = req_distinguished_name
req_extensions = req_ext
[ req_distinguished_name ]
commonName = follower.vx
[ req_ext ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = follower.vx
DNS.2 = flr1.vx
DNS.3 = flr2.vx
DNS.4 = follower.conjur.svc.cluster.local
EOF
```

> Note: `subjectAltName = DNS:follower.vx, DNS:flr1.vx, DNS:flr2.vx, DNS: follower.conjur.svc.cluster.local` will also work for the subject alternative names

### 3.3 Create certificate signing request
```console
openssl req -new -config follower.vx.cnf -key follower.vx.key -out follower.vx.csr
```

### 3.4 Generate certificate
```console
openssl x509 -req -sha256 -days 365 -CA vxLabCA.pem -CAkey vxLabCA.key -CAcreateserial -in follower.vx.csr -extfile follower.vx.cnf -extensions req_ext -out follower.vx.pem
```

## 4. Certificate checking commands

### 4.1 Check certificate
```console
openssl x509 -text -noout -in cert.pem
```

### 4.2 Check private key
- RSA key
```console
openssl rsa -check -in cert.key
```

- EC key
```console
openssl ec -check -in cert.key
```

### 4.3 Check certificate signing request (CSR)
```console
openssl req -text -noout -verify -in cert.csr
```

### 4.4 Check PKCS#12 file (.pfx or .p12)
```console
openssl pkcs12 -info -in cert.pfx
```
