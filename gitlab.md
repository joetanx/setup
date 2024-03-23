## Overview

#### Components

- GitLab community edition
- GitLab runner using Podman (Docker) executor

#### Software Versions

- RHEL 9.3
- Podman 4.6.1
- GitLab 16.10.0

## 1. Setup Gitlab instance

### 1.1. Install GitLab instance

Ref: <https://computingforgeeks.com/how-to-install-and-configure-gitlab-on-centos-rhel/>

|Steps|Commands|
|---|---|
|Download repository script|`curl -sLO https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh`|
|Add execute permission to script|`chmod u+x script.rpm.sh`|
|Setup GitLab repository|`os=el dist=9 ./script.rpm.sh`|
|Install GitLab|`yum -y install gitlab-ce`|
|Add firewall rules|`firewall-cmd --permanent --add-port 9443/tcp && firewall-cmd --reload`|
|Clean-up|`rm -f script.rpm.sh /etc/yum.repos.d/gitlab_gitlab-ce.repo`|

### 1.2. Configure GitLab instance

Edit the GitLab configuration file at `/etc/gitlab/gitlab.rb`

#### External URL

```console
external_url 'https://gitlab.net.vx:9443'
```

#### Initial root password

```console
gitlab_rails['initial_root_password'] = "Forti123"
```

#### Nginx SSL certificate

Ref: <https://computingforgeeks.com/how-to-secure-gitlab-server-with-ssl-certificate/>

```console
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.vx.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.vx.key"
```

#### Nginx listening port

```console
nginx['listen_port'] = 9443
```

#### Commit GitLab configuration

```console
gitlab-ctl reconfigure
```

## 2. Setup GitLab runner

We'd be using Docker (Podman) executor for our GitLab runner

### 2.1. Prepare Podman environment

1. Install podman: `yum -y install podman`
2. Enable it to start on boot-up: `systemctl enable --now podman.socket`

### 2.2. Install GitLab runner

|Steps|Commands|
|---|---|
|Download repository script|`curl -sLO https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh`|
|Add execute permission to script|`chmod u+x script.rpm.sh`|
|Setup GitLab repository|`os=el dist=9 ./script.rpm.sh`|
|Install GitLab|`yum -y install gitlab-runner`|
|Allow `gitlab-runner` user to sudo without password prompt|`echo 'gitlab-runner ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers.d/gitlab-runner`|
|Clean-up|`rm -f  script.rpm.sh /etc/yum.repos.d/runner_gitlab-runner.repo`|

### 2.3. Prepare config template for GitLab runner registration

```console
cat  << EOF >> /tmp/test-config.template.toml
[[runners]]
[runners.docker]
host = "unix:///run/podman/podman.sock"
extra_hosts = ["linsvr.net.vx:10.0.20.23","mysql.net.vx:10.0.20.23","pgsql.net.vx:10.0.20.23","gitlab.net.vx:10.0.20.23"]
EOF
```

Configurations used:
- `host = "unix:///run/podman/podman.sock"` is required to [use Podman to run Docker commands](https://docs.gitlab.com/runner/executors/docker.html#use-podman-to-run-docker-commands)
- `extra_hosts = ["linsvr.net.vx:10.0.20.23","mysql.net.vx:10.0.20.23","pgsql.net.vx:10.0.20.23","gitlab.net.vx:10.0.20.23"]` to [add the host records](https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runnersdocker-section) to `/etc/hosts` of the containers
  - Detailed explanation on why I need this in my internal lab instead of adding DNS records:
    - Containers queries dual-stacked (`A` and `AAAA`) name resolution
    - My `.vx` domain results in a `NXDOMAIN for the `AAAA` (IPv6), since I only have `A` (IPv4) records of the hosts in my DNS

### 2.4. Register GitLab runner

#### Create runner in UI

![gitlab-runner-1](https://github.com/joetanx/setup/assets/90442032/c093fbdd-090d-4cdf-8dd6-35821f96c0f5)

#### Register runner

![gitlab-runner-2](https://github.com/joetanx/setup/assets/90442032/51aa9da7-88a0-40cc-aa0a-4ce01233b4cf)

Ref: https://docs.gitlab.com/runner/register/#register-a-runner-created-in-the-ui-with-an-authentication-token

```console
gitlab-runner register \
--non-interactive \
--url https://gitlab.vx \
--token <registration-token> --executor docker \
--template-config /tmp/test-config.template.toml \
--docker-image docker.io/library/alpine:latest
```

> [!Note]
> 
> The `authentication token` for registering a GitLab runner is different from personal/project/group acces tokens
> 
> The various tokens in GitLab is described here: https://docs.gitlab.com/ee/security/token_overview.html

#### Verify runner status

![gitlab-runner-3](https://github.com/joetanx/setup/assets/90442032/ffcc5b78-ed95-4ea4-a105-a05280858b85)

## 3. Keycloak OIDC Authenticator Provider

Ref: https://docs.gitlab.com/ee/administration/auth/oidc.html

### 3.1. 📌 Keycloak - Setup authentication client

Select `Create client`

![image](https://github.com/joetanx/setup/assets/90442032/7f7ed29e-bc53-4793-a134-be10817d02bd)

Enter a `Client ID` (e.g. `https://<gitlab-fdqn>:9443/`), this is required for the GitLab OIDC config later

![image](https://github.com/joetanx/setup/assets/90442032/940ef0b4-bf1b-4a60-9b7e-158c56efd4c4)

- Enable `Client authentication`
- Only `Standard flow` (authorization code flow) is needed

![image](https://github.com/joetanx/setup/assets/90442032/86ff1a20-3f04-4bf0-b8e6-1817926b84d8)

- Enter `https://<gitlab-fdqn>:9443/` for `Root URL`
- Add `https://<gitlab-fdqn>:9443/users/auth/openid_connect/callback` for the `Valid redirect URIs`

![image](https://github.com/joetanx/setup/assets/90442032/451d0eb7-0878-4e15-b527-dfc8cc28c5ad)

![image](https://github.com/joetanx/setup/assets/90442032/bcafd5cf-ea5b-4e30-a5d6-cbd0e70f218f)

Copy the `Client secret`, this is required for the GitLab OIDC config later

![image](https://github.com/joetanx/setup/assets/90442032/4c444a3c-cc2e-4e30-96ed-9b9c292cc30a)

### 3.2. 📌 GitLab - Setup Authenticator Provider

Edit `/etc/gitlab/gitlab.rb` with the following:

```rb
gitlab_rails['omniauth_allow_single_sign_on'] = ['openid_connect']
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['env'] = {"SSL_CERT_FILE" => "/etc/pki/ca-trust/source/anchors/lab_root.pem"}

gitlab_rails['omniauth_providers'] = [
  {
    name: "openid_connect", # do not change this parameter
    label: "Keycloak", # optional label for login button, defaults to "Openid Connect"
    args: {
      name: "openid_connect",
      scope: ["openid", "profile", "email"],
      response_type: "code",
      issuer:  "https://keycloak.net.vx:8443/realms/master",
      client_auth_method: "query",
      discovery: true,
      uid_field: "preferred_username",
      pkce: true,
      client_options: {
        identifier: "https://gitlab.net.vx:9443/",
        secret: "6ISsg55MN4GpTDSK770WS3Pg4fNwRZOg",
        redirect_uri: "https://gitlab.net.vx:9443/users/auth/openid_connect/callback",
      }
    }
  }
]
```

[Reconfigure GitLab](https://docs.gitlab.com/ee/administration/restart_gitlab.html#reconfigure-a-linux-package-installation) using the `gitlab-ctl reconfigure` command

### 3.3. Test Login

![image](https://github.com/joetanx/setup/assets/90442032/6d4218bb-d38c-4738-9043-8015c4390f6e)

![image](https://github.com/joetanx/setup/assets/90442032/7a0cc823-d935-4e2e-ba26-9ddba9c56683)

![image](https://github.com/joetanx/setup/assets/90442032/06e9fc09-cfbf-4d1b-a1a0-0d76735f3f26)

![image](https://github.com/joetanx/setup/assets/90442032/ed63a327-2ef6-411b-8949-9674f05a58b5)

Verify automatically created user:

![image](https://github.com/joetanx/setup/assets/90442032/60652adf-42b7-4776-a3ba-757b176fcb90)
