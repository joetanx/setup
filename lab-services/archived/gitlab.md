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

Tested with Keycloak 24.0.1

### 3.1. 📌 Keycloak - setup authentication client

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

### 3.2. 📌 GitLab - setup authenticator provider

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

> [!Note]
>
> The Keycloak OpenID configuration URL is at `https://<keycloak-fqdn>/realms/<realm>/.well-known/openid-configuration`
>
> The `OpenID Endpoint Configuration` URL can also be opened from `Realm settings`
>
> ![image](https://github.com/joetanx/setup/assets/90442032/e1413804-d13b-4e6c-ac14-522820ddc044)

### 3.3. Test login

![image](https://github.com/joetanx/setup/assets/90442032/6d4218bb-d38c-4738-9043-8015c4390f6e)

![image](https://github.com/joetanx/setup/assets/90442032/7a0cc823-d935-4e2e-ba26-9ddba9c56683)

![image](https://github.com/joetanx/setup/assets/90442032/06e9fc09-cfbf-4d1b-a1a0-0d76735f3f26)

![image](https://github.com/joetanx/setup/assets/90442032/ed63a327-2ef6-411b-8949-9674f05a58b5)

Verify automatically created user:

![image](https://github.com/joetanx/setup/assets/90442032/60652adf-42b7-4776-a3ba-757b176fcb90)

## 4. Keycloak SAML Authenticator Provider

Ref: https://docs.gitlab.com/ee/integration/saml.html

Tested with Keycloak 24.0.1

### 4.1. 📌 Keycloak - setup authentication client

Select `Create client`

![image](https://github.com/joetanx/setup/assets/90442032/3d7d22ca-d6fc-42cb-9e8e-2e74f129ed75)

Enter a `Client ID` (e.g. `https://<gitlab-fdqn>:9443/`), this is required for the GitLab SAML config later

![image](https://github.com/joetanx/setup/assets/90442032/3e29cc44-b5cf-49d9-94a7-1371417dff64)

Client parameters:

|Parameter|Value|Remarks|
|---|---|---|
|Root URL|`https://<gitlab-fqdn>/`|e.g. `https://gitlab.net.vx:9443/`|
|Valid redirect URIs|`https://<gitlab-fqdn>/users/auth/saml/callback`|e.g. `https://gitlab.net.vx:9443/users/auth/saml/callback`|
|IDP-initiated SSO URL name|`<client-name>`|Keycloak enables IDP-initiated SSO at the client path:<br>`https://<keycloak-fqdn>/realms/<realm>/protocol/saml/clients/<client-name>`<br>e.g. for client name `gitlab`, the IDP-initiated SSO client path would be: `https://keycloak.net.vx:8443/realms/master/protocol/saml/clients/gitlab`|
|Master SAML Processing URL|`https://<gitlab-fqdn>/users/auth/saml/callback`|e.g. `https://gitlab.net.vx:9443/users/auth/saml/callback`|

![image](https://github.com/joetanx/setup/assets/90442032/6121cda9-0283-4cef-9ea0-53d0f9afce83)

Disable `Client signature required`

![image](https://github.com/joetanx/setup/assets/90442032/334eee6b-cdc3-402e-a056-5ae92d59a7b6)

![image](https://github.com/joetanx/setup/assets/90442032/77adab20-ab14-434c-9b33-d74b3f7241b0)

### 4.2. 📌 Keycloak - configure client scope

Claims mapping are configured under `client scope` in Keycloak to map user properties to SAML claims

![image](https://github.com/joetanx/setup/assets/90442032/a5cb2ffd-d970-4465-90b6-8e12581dfa39)

![image](https://github.com/joetanx/setup/assets/90442032/68f476f2-d5ee-4c4f-a85c-a155cab08f73)

![image](https://github.com/joetanx/setup/assets/90442032/26c3655a-8ea1-47e0-b065-fb23b909d080)

Create `User Property` mapper for `firstName`, `lastName` and `email`

![image](https://github.com/joetanx/setup/assets/90442032/0ed6cb99-8b6f-4e68-a054-ac6bae3e5883)

|User property|SAML claim|
|---|---|
|`firstName`|`first-name`|
|`lastName`|`last-name`|
|`email`|`email`|

![image](https://github.com/joetanx/setup/assets/90442032/13afe59b-05f3-4d64-a1f5-c109cd03906d)

![image](https://github.com/joetanx/setup/assets/90442032/8bf36fae-fb39-4969-b753-1adaddb25899)

![image](https://github.com/joetanx/setup/assets/90442032/9c9958b0-6ce9-471c-ab60-ac231251fd3a)

![image](https://github.com/joetanx/setup/assets/90442032/c24d4f63-f0eb-4732-87f2-4ff2292ec249)

> [!Note]
>
> Mapping for `username` is not required as the client configuration default settings uses `username` for `Name ID`
>
> ![image](https://github.com/joetanx/setup/assets/90442032/a0824de3-4b30-44d0-a28f-390bef4c98cb)

### 4.3. 📌 Keycloak - add client scope to GitLab client

![image](https://github.com/joetanx/setup/assets/90442032/bca5d489-392f-45da-b884-06a77b80dcbe)

![image](https://github.com/joetanx/setup/assets/90442032/57ec10b5-f735-41e3-be00-1eb8f6b6ae2d)

![image](https://github.com/joetanx/setup/assets/90442032/04422b76-02d2-4c2a-ba2b-bd44c87e138f)

### 4.4. 📌 GitLab - setup authenticator provider

Edit `/etc/gitlab/gitlab.rb` with the following:

```rb
gitlab_rails['omniauth_allow_single_sign_on'] = ['saml']
gitlab_rails['omniauth_block_auto_created_users'] = false

gitlab_rails['omniauth_providers'] = [
  {
    name: "saml",
    label: "Keycloak", # optional label for login button, defaults to "Saml"
    args: {
      assertion_consumer_service_url: "https://gitlab.net.vx:9443/users/auth/saml/callback",
      idp_cert_fingerprint: "87:b7:42:5d:c1:76:57:a8:9a:20:79:0b:2e:f5:a6:c8:64:18:51:0d",
      idp_sso_target_url: "https://keycloak.net.vx:8443/realms/master/protocol/saml",
      issuer: "https://gitlab.net.vx:9443/",
      name_identifier_format: "urn:oasis:names:tc:SAML:2.0:nameid-format:persistent"
    }
  }
]
```

[Reconfigure GitLab](https://docs.gitlab.com/ee/administration/restart_gitlab.html#reconfigure-a-linux-package-installation) using the `gitlab-ctl reconfigure` command

> [!Note]
>
> The Keycloak signing certificate information is in the metadata URL `https://<keycloak-fqdn>/realms/<realm>/protocol/saml/descriptor` under the `<ds:X509Certificate>` tag
>
> Input the certificate into a `.pem` file...
>
> ```sh
> cat << EOF > kc-saml.pem
> -----BEGIN CERTIFICATE-----
> <redacted>
> -----END CERTIFICATE-----
> ```
>
> ... and use `openssl x509 -sha1 -fingerprint -noout -in kc-saml.pem` to get the SHA1 fingerprint
>
> The `SAML 2.0 Identity Provider Metadata` URL can also be opened from `Realm settings`
>
> ![image](https://github.com/joetanx/setup/assets/90442032/e1413804-d13b-4e6c-ac14-522820ddc044)

### 4.5. Test login

#### 4.5.1. SP-initiated login

![image](https://github.com/joetanx/setup/assets/90442032/11c1d844-5f7f-41b5-958c-bda9b01acc2f)

![image](https://github.com/joetanx/setup/assets/90442032/ff504f02-9130-48bd-a280-59948a5c225f)

![image](https://github.com/joetanx/setup/assets/90442032/fc2ddf45-b296-44c1-9d66-725269243ff4)

![image](https://github.com/joetanx/setup/assets/90442032/dd9f50ae-c2d4-45b9-b76d-c51da7313404)

#### 4.5.2. IDP-initiated login

Open client path: `https://<keycloak-fqdn>/realms/<realm>/protocol/saml/clients/<client-name>`

e.g. for client name `gitlab`, the IDP-initiated SSO client path would be: `https://keycloak.net.vx:8443/realms/master/protocol/saml/clients/gitlab`

![image](https://github.com/joetanx/setup/assets/90442032/e6665d7e-7f04-4d68-8fcd-7f6260bed15c)

![image](https://github.com/joetanx/setup/assets/90442032/39930602-8497-4d4b-ba59-4381c97287b7)

![image](https://github.com/joetanx/setup/assets/90442032/e498b455-25de-4929-8907-fcac9e30baaf)

![image](https://github.com/joetanx/setup/assets/90442032/dd9f50ae-c2d4-45b9-b76d-c51da7313404)

#### 4.5.3. Verify automatically created user:

![image](https://github.com/joetanx/setup/assets/90442032/710cbfe4-124a-4867-98ae-beba7bc79d37)

### 4.6. Possible troubleshooting

#### `Email can't be blank`

![image](https://github.com/joetanx/setup/assets/90442032/2d6355e6-4ccd-4f2b-ab7c-b1874c3701eb)

- User that is logging in does not have an email configured
- The email SAML claim mapping is not configured correctly

#### `Invalid Requester`

![image](https://github.com/joetanx/setup/assets/90442032/57068aa8-a8b2-4430-ad2b-72235a6fc6a8)

- `Client signature required` is enabled in the SAML client configuration
