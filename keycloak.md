## 1. Install Keycloak

Install Java and allow port `8443` on firewalld

```sh
yum -y install java-17-openjdk-devel
firewall-cmd --permanent --add-port 8443/tcp && firewall-cmd --reload
```

Download latest Keycloak release and extract to `/opt/keycloak`

```sh
VERSION=$(curl -sI https://github.com/keycloak/keycloak/releases/latest | grep location: | cut -d / -f 8 | tr -d '\r' | tr -d 'v')
curl -sLO https://github.com/keycloak/keycloak/releases/download/$VERSION/keycloak-$VERSION.tar.gz
tar xvf keycloak-$VERSION.tar.gz && rm -f keycloak-$VERSION.tar.gz
mv keycloak-$VERSION /opt/keycloak
```

## 2. Configure Keycloak

Prepare server certificates

```sh
mkdir /opt/keycloak/tls
curl -sLo /opt/keycloak/tls/server.crt.pem https://github.com/joetanx/lab-certs/raw/main/keycloak/keycloak.vx.pem
curl -sLo /opt/keycloak/tls/server.key.pem https://github.com/joetanx/lab-certs/raw/main/keycloak/keycloak.vx.key
```

Backup initial Keycloak config file

```sh
cp /opt/keycloak/conf/keycloak.conf /opt/keycloak/conf/keycloak.conf.bak
```

Edit Keycloak configuration according to the environment

Values used the configuration example below:

|Parameter|Value|
|---|---|
|Database type|postgres|
|Database username|keycloak|
|Database password|Keycloak123|
|Database server|foxtrot.vx|
|Database name|keycloak|
|Keycloak server certificate path|`/opt/keycloak/tls/server.crt.pem`|
|Keycloak server certificate key path|`/opt/keycloak/tls/server.crt.pem`|

```
⋮
db=postgres
⋮
db-username=keycloak
⋮
db-password=Keycloak123
⋮
db-url=jdbc:postgresql://foxtrot.vx/keycloak
⋮
https-certificate-file=/opt/keycloak/tls/server.crt.pem
⋮
https-certificate-key-file=/opt/keycloak/tls/server.key.pem
⋮
```

Add `keycloak` user and assign as owner of the Keycloak binaries folder

```sh
useradd -r keycloak
chown -R keycloak:keycloak /opt/keycloak
```

## 3. Keycloak first run

The first start procedure assigns the Keycloak admin credentials from `KEYCLOAK_ADMIN` and `KEYCLOAK_ADMIN_PASSWORD`

`sudo -E -u keycloak` runs the start procedure as `keycloak` user - need to run this as `root`

```console
[root@keycloak ~]# export KEYCLOAK_ADMIN=admin
[root@keycloak ~]# export KEYCLOAK_ADMIN_PASSWORD=Keycloak123
[root@keycloak ~]# sudo -E -u keycloak /opt/keycloak/bin/kc.sh start
⋮
2023-10-15 09:00:50,130 INFO  [org.keycloak.services] (main) KC-SERVICES0009: Added user 'admin' to realm 'master'
```

Test access and login to Keyloak

![image](https://github.com/joetanx/setup/assets/90442032/54a647aa-7f12-4616-9900-472fdb229b76)

![image](https://github.com/joetanx/setup/assets/90442032/391b1c72-2a7b-4b9a-bc40-6ab977d0e872)

![image](https://github.com/joetanx/setup/assets/90442032/ec75ff64-940e-4b0f-89d1-a006aed2a3af)

Sign out and use `Ctrl` + `C` to shutdown Keycloak after verification

## 4. Configure Keycloak as systemd service

- Create the systemd service unit file `/usr/lib/systemd/system/keycloak.service`
- Enable Keycloak service to start on boot

```console
[root@vault ~]# cat << EOF > /usr/lib/systemd/system/keycloak.service
[Unit]
Description=Keycloak Application Server
After=syslog.target network.target

[Service]
Type=simple
User=keycloak
Group=keycloak
ExecStart=/bin/bash /opt/keycloak/bin/kc.sh start --optimized
StandardOutput=null

[Install]
WantedBy=multi-user.target
EOF
[root@vault ~]# systemctl enable --now keycloak
Created symlink /etc/systemd/system/multi-user.target.wants/keycloak.service → /usr/lib/systemd/system/keycloak.service.
[root@vault ~]# systemctl status keycloak
● keycloak.service - Keycloak Application Server
     Loaded: loaded (/usr/lib/systemd/system/keycloak.service; enabled; preset: disabled)
     Active: active (running) since Sun 2023-10-15 09:05:27 +08; 6s ago
     ⋮
Oct 15 09:05:27 vault.vx systemd[1]: Started Keycloak Application Server.
```

Test access and login to Keyloak again

## 5. Create user

Add user

![image](https://github.com/joetanx/setup/assets/90442032/af1a6073-5939-43b5-934b-2511ffa3ec36)

![image](https://github.com/joetanx/setup/assets/90442032/30ebdea3-f006-4be1-bc6d-2a1ab6a0fd09)

![image](https://github.com/joetanx/setup/assets/90442032/bf8ecf2e-4480-455c-8803-9ae0d2c22914)

Set temporary password for user

![image](https://github.com/joetanx/setup/assets/90442032/7b7c72d8-9f03-4c3a-9123-a918785e3be5)

![image](https://github.com/joetanx/setup/assets/90442032/f702c217-3759-45da-b721-d821edbe8250)

![image](https://github.com/joetanx/setup/assets/90442032/c0ffcdb8-6f31-4cd0-bf09-f83fa9427f27)

Login and change password

![image](https://github.com/joetanx/setup/assets/90442032/89e2c76e-54e3-4323-af03-0541c32e8fca)

Add TOTP authenticator

![image](https://github.com/joetanx/setup/assets/90442032/38615d3f-cc30-4602-aeaa-e2e2c1817a77)

![image](https://github.com/joetanx/setup/assets/90442032/8c861e1d-f20e-4428-8bc7-20bb5a99f84c)

![image](https://github.com/joetanx/setup/assets/90442032/f11adfab-6da2-4008-b23c-1cabbe49bfcc)

![image](https://github.com/joetanx/setup/assets/90442032/ab61ebd0-ad25-4a5a-a4c9-21398358e11d)

![image](https://github.com/joetanx/setup/assets/90442032/7ee67b10-9aea-4180-8942-6f5cce54a8a5)
