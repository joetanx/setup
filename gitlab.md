## Overview

#### Components

- GitLab community edition
- GitLab runner using Podman (Docker) executor

#### Software Versions

- RHEL 9.2
- Podman 4.4.1
- GitLab 16.2.4

## 1. Setup Gitlab instance

### 1.1. Install GitLab instance

Ref: <https://computingforgeeks.com/how-to-install-and-configure-gitlab-on-centos-rhel/>

|Steps|Commands|
|---|---|
|Download repository script|`curl -sLO https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh`|
|Add execute permission to script|`chmod u+x script.rpm.sh`|
|Setup GitLab repository|`os=el dist=9 ./script.rpm.sh`|
|Install GitLab|`yum -y install gitlab-ce`|
|Add firewall rules|`firewall-cmd --add-service=https --permanent && firewall-cmd --reload`|
|Clean-up|`rm -f script.rpm.sh /etc/yum.repos.d/gitlab_gitlab-ce.repo`|

### 1.2. Configure GitLab instance

Edit the GitLab configuration file at `/etc/gitlab/gitlab.rb`

#### External URL

```console
external_url 'https://gitlab.vx'
```

#### Initial root password

```console
gitlab_rails['initial_root_password'] = "Cyberark1"
```

#### Nginx SSL certificate

Ref: <https://computingforgeeks.com/how-to-secure-gitlab-server-with-ssl-certificate/>

```console
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.vx.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.vx.key"
```

#### Nginx listening port

```console
nginx['listen_port'] = 6443
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
extra_hosts = ["conjur.vx:192.168.17.90","gitlab.vx:192.168.17.92"]
EOF
```

Configurations used:
- `host = "unix:///run/podman/podman.sock"` is required to [use Podman to run Docker commands](https://docs.gitlab.com/runner/executors/docker.html#use-podman-to-run-docker-commands)
- `extra_hosts = ["conjur.vx:192.168.17.90","gitlab.vx:192.168.17.91"]` to [add the host records](https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runnersdocker-section) to `/etc/hosts` of the containers
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
