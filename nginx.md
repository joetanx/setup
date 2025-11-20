## Simple nginx reverse proxy setup

The example [`nginx.conf`](/nginx.conf) performs the following

1. Redirect HTTP to HTTPS

The server block listening on port `80` redirects to https with `return 301 https://$host$request_uri;`

2. TLS encryption offloading

The server block listening on port `443` performs TLS encryption using certificates at `/etc/nginx/tls` for the unencrypted backend at `http://dst_host:dst_port`

## Type 1. Server install

### 1.1. Install nginx package:

```
yum -y install nginx
```

### 1.2. Allow nginx to connect network:

```
setsebool -P httpd_can_network_connect 1
```

> [!Note]
>
> `httpd_can_network_connect` is required for SELinux to allow nginx to connect to backend resource
> 
> Error in nginx `/var/log/nginx/error.log` when this is not enabled:
> 
> ```
> 2024/12/17 07:57:23 [crit] 845#845: *2 connect() to 127.0.0.1:9000 failed (13: Permission denied) while connecting to upstream, client: 192.168.84.11, server: _, request: "GET / HTTP/1.1", upstream: "http://127.0.0.1:9000/", host: "hive.vx"
> ```
>
> This corresponds to SELinux `avc:  denied` event in `/var/log/audit/audit.log`:
> 
> ```
> type=AVC msg=audit(1734393443.740:91): avc:  denied  { name_connect } for  pid=845 comm="nginx" dest=9000 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:http_port_t:s0 tclass=tcp_socket permissive=0
> ```

### 1.3. Prepare nginx config file

Download `nginx.conf` to `/etc/nginx/nginx.conf`

```sh
curl -sLo /etc/nginx/nginx.conf https://github.com/joetanx/setup/raw/refs/heads/main/nginx.conf
```

Edit `dst_host:dst_port` to required locations:

(single-line edit command example)

```sh
cat /etc/nginx/nginx.conf | sed "s/dst_host/localhost/" | sed "s/dst_port/9000/" | tee /etc/nginx/nginx.conf
```

### 1.4. Prepare TLS certificates

Example using certificates from [lab-certs](https://github.com/joetanx/lab-certs/)

```sh
mkdir /etc/nginx/tls
curl -sLo /etc/nginx/tls/server.pem https://github.com/joetanx/lab-certs/raw/refs/heads/main/others/delta.vx.pem
curl -sLo /etc/nginx/tls/server.key https://github.com/joetanx/lab-certs/raw/main/others/key
curl -sLo /etc/nginx/tls/cacert.pem https://github.com/joetanx/lab-certs/raw/main/ca/lab_chain.pem
```

### 1.5. Enable nginx to run on start

```sh
systemctl enable --now nginx
```

> [!Tip]
>
> Changes to `nginx.conf` can be quickly reloaded without restarting the nginx services with `systemctl reload nginx`

## Type 2. Container (Podman)

### 2.1. Install podman and SELinux package

```sh
yum -y install podman policycoreutils-python-utils
systemctl enable --now podman
```

> [!Note]
>
> `policycoreutils-python-utils` is required to edit the SELinux labels on host filesystem to allow container access to files and directories

### 2.2. Prepare container directory

```sh
mkdir -p /opt/nginx/tls
semanage fcontext -a -t svirt_sandbox_file_t "/opt/nginx/tls(/.*)?"
restorecon -Rv /opt/nginx
```

> [!Note]
>
> 1. `semanage fcontext` sets the label, `restorecon` applies the label without reboot
> 2. The `(/.*)?` regex means the SELinux label applies to all sub-directories and files

### 2.3. Prepare nginx config file

Download `nginx.conf` to `/opt/nginx/nginx.conf`

```sh
curl -sLo /opt/nginx/nginx.conf https://github.com/joetanx/setup/raw/refs/heads/main/nginx.conf
```

Edit `dst_host:dst_port` to required locations:

(single-line edit command example)

```sh
cat /opt/nginx/nginx.conf | sed "s/dst_host/n8n/" | sed "s/dst_port/5678/" | tee /opt/nginx/nginx.conf
```

### 2.4. Prepare TLS certificates

Example using certificates from [lab-certs](https://github.com/joetanx/lab-certs/)

```sh
mkdir /opt/nginx/tls
curl -sLo /opt/nginx/tls/server.pem https://github.com/joetanx/lab-certs/raw/refs/heads/main/others/delta.vx.pem
curl -sLo /opt/nginx/tls/server.key https://github.com/joetanx/lab-certs/raw/main/others/key
curl -sLo /opt/nginx/tls/cacert.pem https://github.com/joetanx/lab-certs/raw/main/ca/lab_chain.pem
```

### 2.5. Run container with the config file and certificates mounted

#### 2.5.1. Example: podman run

Example network:

```sh
podman network create n8n
```

Example app container:

```sh
podman run -d --name n8n -h n8n --network n8n docker.n8n.io/n8nio/n8n:latest
```

nginx container:

```sh
podman run -d --name nginx -h nginx --network n8n -p 80:80 -p 443:443 -v /opt/nginx/tls:/etc/nginx/tls:Z -v /opt/nginx/nginx.conf:/etc/nginx/nginx.conf:Z docker.io/nginx:latest
```

#### 2.5.2. Example: quadlet

Example network:

```sh
cat << EOF > /etc/containers/systemd/n8n.network
[Unit]
Description=n8n app: network
After=network.target

[Network]
NetworkName=n8n

[Install]
WantedBy=multi-user.target
EOF
```

Example app container:

```sh
cat << EOF > /etc/containers/systemd/n8n.container
[Unit]
Description=n8n app: n8n container
Requires=n8n-network.service
After=n8n-network.service

[Container]
ContainerName=n8n
HostName=n8n
Image=docker.n8n.io/n8nio/n8n:latest
Network=n8n.network
Environment=GENERIC_TIMEZONE=Asia/Singapore
Environment=TZ=Asia/Singapore
Volume=/opt/n8n/data:/home/node/.n8n:Z

[Install]
WantedBy=multi-user.target
EOF
```

nginx container:

```sh
cat << EOF > /etc/containers/systemd/nginx.container
[Unit]
Description=n8n app: nginx container
Requires=n8n.service
After=n8n.service

[Container]
ContainerName=nginx
HostName=nginx
Image=docker.io/nginx:latest
Network=n8n.network
PublishPort=80:80
PublishPort=443:443
Volume=/opt/nginx/tls:/etc/nginx/tls:Z
Volume=/opt/nginx/nginx.conf:/etc/nginx/nginx.conf:Z

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl start nginx
```

Start the containers:

```sh
systemctl daemon-reload
systemctl start nginx
```

> [!Note]
>
> ## Quadlets:
> 1. Only the nginx service needs to be started, the application and network services are automatically started as dependencies stated in the unit files
> 2. There is no need to `enable` the services as quadlets would always start on boot
>
> Example error when trying to enable the quadlet service:
>
> `Failed to enable unit: Unit /run/systemd/generator/nginx.service is transient or generated.`
