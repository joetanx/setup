## 1. Prepare host machine

### 1.1. Software Versions

- RHEL 9.3
- Podman 4.6.1
- Conjur Enterprise 13.1

### 1.2. Setup host prerequisites

- Install Podman
- Upload `conjur-appliance-Rls-v13.1.tar.gz` to the container host: contact your CyberArk representative to acquire the Conjur container image
- Prepare data directories: these directories will be mounted to the Conjur container as volumes
- Setup [Conjur CLI](https://github.com/cyberark/conjur-cli-go): the client tool to interface with Conjur

```sh
yum -y install podman
podman load -i conjur-appliance-Rls-v13.1.tar.gz && rm -f conjur-appliance-Rls-v13.1.tar.gz
mkdir -p /opt/conjur/{security,config,backups,seeds,logs}
curl -sLo /usr/local/bin/conjur https://github.com/cyberark/conjur-cli-go/releases/download/v8.0.12/conjur_linux_amd64
chmod +x /usr/local/bin/conjur
```

### 1.3. Note on SELinux and Container Volumes
- SELinux may prevent the container access to the data directories without the appropriate SELinux labels
- Ref: [podman-run - Labeling Volume Mounts](https://docs.podman.io/en/latest/markdown/podman-run.1.html)
- There are 2 ways to enable container access to the data directories:
  1. Use `semanage fcontext` and `restorecon` to relabel the data directories
    ```sh
    yum install -y policycoreutils-python-utils
    semanage fcontext -a -t svirt_sandbox_file_t "/opt/conjur(/.*)?"
    restorecon -R -v /opt/conjur
    ```
  2. Add `:z` or `:Z` to the volume mounts when running the container so that Podman will automatically label the data directories
    - `:z` - indicates that content is shared among multiple container
    - `:Z` - indicates that content is is private and unshared

## 2. Deploy Conjur master

### 2.1. Run Conjur appliance container

#### 2.1.1. Method 1: Running Conjur master on the default bridge network

- Podman run command:

```sh
podman run --name conjur -d \
--restart=unless-stopped \
-p "443:443" -p "444:444" -p "5432:5432" -p "1999:1999" \
--log-driver journald \
-v /opt/conjur/config:/etc/conjur/config:Z \
-v /opt/conjur/security:/opt/cyberark/dap/security:Z \
-v /opt/conjur/backups:/opt/conjur/backup:Z \
-v /opt/conjur/seeds:/opt/cyberark/dap/seeds:Z \
-v /opt/conjur/logs:/var/log/conjur:Z \
registry.tld/conjur-appliance:13.1.0
```

#### 2.1.2. Method 2: Running Conjur master on the Podman host network

- Podman run command:

```sh
podman run --name conjur -d \
--restart=unless-stopped \
--network host \
--log-driver journald \
-v /opt/conjur/config:/etc/conjur/config:Z \
-v /opt/conjur/security:/opt/cyberark/dap/security:Z \
-v /opt/conjur/backups:/opt/conjur/backup:Z \
-v /opt/conjur/seeds:/opt/cyberark/dap/seeds:Z \
-v /opt/conjur/logs:/var/log/conjur:Z \
registry.tld/conjur-appliance:13.1.0
```

- Add firewall rules on the Podman host

```sh
firewall-cmd --add-service https --permanent
firewall-cmd --add-service postgresql --permanent
firewall-cmd --add-port 444/tcp --permanent
firewall-cmd --add-port 1999/tcp --permanent
firewall-cmd --reload
```

### 2.2. Initialize the Conjur appliance

- Edit the admin account password in `-p` option and the Conjur account (`cyberark`) according to your environment

```sh
podman exec conjur evoke configure master --accept-eula -h conjur.vx --master-altnames "conjur.vx" -p CyberArk123! cyberark
```

### 2.3. Configure container to start on boot

- Run the Conjur container as systemd service and configure it to setup with container host
- Ref: <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/managing_containers/running_containers_as_systemd_services_with_podman>

```sh
podman generate systemd conjur --name --container-prefix="" --separator="" > /etc/systemd/system/conjur.service
systemctl enable conjur
```

### 2.4. Setup Conjur certificates

The [lab-certs repository](https://github.com/joetanx/lab-certs) contains the certificate chain for CA, leader and follower, use the README and assets in this repository as reference to generate a self-signed certificate chain

> [!Note]
> 
> The Common Name of Conjur certificates should be the FQDN of the access endpoint, otherwise errors will occur

|Certificate|Purpose|Common Name|Subject Alternative Names|
|---|---|---|---|
|lab_chain.pem|Lab root and intermediate CAs|Lab Root CA/Lab Issuer||
|conjur.vx.pem / conjur.vx.key|Conjur cluster certificate|conjur.vx|conjur.vx|
|follower.conjur.svc.cluster.local.pem / follower.conjur.svc.cluster.local.key|Conjur follower certificate|follower.conjur.svc.cluster.local|follower.conjur.svc.cluster.local|

> [!Note]
> 
> In event of `error: cert already in hash table`, ensure that the Conjur server/follower certificates do not contain the CA certificates

```sh
podman exec conjur curl -sLO https://github.com/joetanx/lab-certs/raw/main/ca/lab_chain.pem
podman exec conjur curl -sLO https://github.com/joetanx/lab-certs/raw/main/conjur/conjur.vx.pem
podman exec conjur curl -sLO https://github.com/joetanx/lab-certs/raw/main/conjur/conjur.vx.key
podman exec conjur curl -sLO https://github.com/joetanx/lab-certs/raw/main/conjur/follower.conjur.svc.cluster.local.pem
podman exec conjur curl -sLO https://github.com/joetanx/lab-certs/raw/main/conjur/follower.conjur.svc.cluster.local.key
podman exec conjur evoke ca import -fr lab_chain.pem
podman exec conjur evoke ca import -k conjur.vx.key -s conjur.vx.pem
podman exec conjur evoke ca import -k follower.conjur.svc.cluster.local.key follower.conjur.svc.cluster.local.pem
```

- Clean-up

```sh
podman exec conjur rm lab_chain.pem conjur.vx.pem conjur.vx.key follower.conjur.svc.cluster.local.pem follower.conjur.svc.cluster.local.key
```

### 2.5. Allowlist the Conjur default authenticator

Ref: https://docs.cyberark.com/conjur-enterprise/latest/en/Content/Operations/Services/authentication-types.htm

```sh
podman exec conjur sed -i -e '$aauthenticators:' /etc/conjur/config/conjur.yml
podman exec conjur sed -i -e '/authenticators:/a\  - authn' /etc/conjur/config/conjur.yml
podman exec conjur evoke configuration apply
```

### 2.6. Initialize Conjur CLI and login to conjur

```sh
conjur init -u https://conjur.vx
conjur login -i admin -p CyberArk123!
```

## 3. Staging secret variables

- Integration guides in my GitHub uses the secrets set in this step
- Pre-requisites
  - Setup MySQL database according to this guide: https://github.com/joetanx/setup/blob/main/mysql.md
  - Have an AWS IAM user account with programmatic access
- Credentials are configured by `app-vars.yaml` in `world_db` and `aws_api` policies that are defined with the respective secret variables
- Download the Conjur policies

```sh
curl -sLO https://github.com/joetanx/setup/raw/main/app-vars.yaml
```

- Load the policies to Conjur

```sh
conjur policy load -b root -f app-vars.yaml && rm -f app-vars.yaml
```

- Populate the variables

```sh
conjur variable set -i db_cityapp/address -v mysql.vx
conjur variable set -i db_cityapp/username -v cityapp
conjur variable set -i db_cityapp/password -v Cyberark1
conjur variable set -i db_cicd/username -v cicd
conjur variable set -i db_cicd/password -v Cyberark1
conjur variable set -i aws_api/awsakid -v <AWS_ACCESS_KEY_ID>
conjur variable set -i aws_api/awssak -v <AWS_SECRET_ACCESS_KEY>
```
