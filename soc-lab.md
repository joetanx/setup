## 1. Elastic Stack

References:
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html>
- <https://www.elastic.co/guide/en/kibana/current/rpm.html>
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/security-minimal-setup.html>

### 1.1. Setup Elastic repository

```sh
cat << EOF > /etc/yum.repos.d/elasticsearch.repo
[elasticsearch]
name=Elasticsearch repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

### 1.2. Install and configure Elastic Stack

Install Elastic search and Kibana

```sh
yum -y install --enablerepo=elasticsearch elasticsearch kibana
```

Allow access to Elastic Stack on firewall:

```sh
firewall-cmd --permanent --add-service=elasticsearch
firewall-cmd --permanent --add-service=kibana
firewall-cmd --reload
```

<details><summary><h3>1.3. Elastic default certificates</h3></summary>

Elasticsearch after version 8.0 are installed with security enabled by default

Ref: <https://www.elastic.co/guide/en/elasticsearch/reference/8.7/configuring-stack-security.html#stack-security-certificates>

Default security settings in `/etc/elasticsearch/elasticsearch.yml`:

```yaml
# Enable security features
xpack.security.enabled: true

xpack.security.enrollment.enabled: true

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
```

Certificates `http.p12` and `transport.p12` at `/etc/elasticsearch/certs` are automatically generated upon installation and are signed by `Elasticsearch security auto-configuration HTTP CA`

Auto generated certificates example:

```console
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore list
autoconfiguration.password_hash
keystore.seed
xpack.security.http.ssl.keystore.secure_password
xpack.security.transport.ssl.keystore.secure_password
xpack.security.transport.ssl.truststore.secure_password
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show autoconfiguration.password_hash
$2a$10$xs1YLeAkIEyOcAwCG7XLCOczeUK7JDo0AyEMgqm68Ex9ZxGeDMHvq
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show keystore.seed
@qHfj*21kelFMztEFXLq
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password
pXoiWrJOQHWvy2OOlpJ_YQ
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.transport.ssl.keystore.secure_password
MotF62oyTN659X-CuEHYyQ
[root@localhost ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.transport.ssl.truststore.secure_password
MotF62oyTN659X-CuEHYyQ
[root@localhost ~]# ls -l /etc/elasticsearch/certs/
total 24
-rw-rw----. 1 root elasticsearch  1915 Apr 25 16:17 http_ca.crt
-rw-rw----. 1 root elasticsearch 10013 Apr 25 16:17 http.p12
-rw-rw----. 1 root elasticsearch  5822 Apr 25 16:17 transport.p12
[root@localhost ~]# keytool -keystore /etc/elasticsearch/certs/http.p12 -list
Enter keystore password:
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 2 entries

http, Apr 25, 2023, PrivateKeyEntry,
Certificate fingerprint (SHA-256): 5E:0E:5C:1C:8A:58:F6:0F:88:64:78:97:7F:FF:48:37:C2:25:70:33:36:DC:88:C0:F0:7B:28:5B:B6:47:A8:B4
http_ca, Apr 25, 2023, PrivateKeyEntry,
Certificate fingerprint (SHA-256): DE:CE:DD:17:1A:86:B7:90:F6:3C:AB:D5:D0:E2:4B:60:86:FD:75:8F:7B:E6:A3:3F:7A:10:4F:BE:4E:6D:78:F0
[root@localhost ~]# keytool -keystore /etc/elasticsearch/certs/transport.p12 -list
Enter keystore password:
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 2 entries

transport, Apr 25, 2023, PrivateKeyEntry,
Certificate fingerprint (SHA-256): 9D:1B:31:2B:60:B1:31:DA:7C:6B:50:C8:27:8E:1E:D0:28:FF:85:2F:2B:FC:18:F0:01:83:93:87:D8:5E:84:BA
transport_ca, Apr 28, 2023, trustedCertEntry,
Certificate fingerprint (SHA-256): 02:4D:34:FE:92:C8:09:7A:8C:AC:D8:8E:73:20:78:D6:57:FA:9D:B9:43:25:23:B8:95:66:A9:AB:5C:75:3B:31
```

</details>

### 1.4. Generate certificates

References:

- <https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup.html>
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup-https.html>
- <https://www.elastic.co/guide/en/elasticsearch/reference/current/update-node-certs-different.html>

The certificates can be created by `elasticsearch-certutil` or signed by a third party certificate authority, the examples below uses a self-signed lab certificate authority `lab_issuer.pem` and `lab_issuer.key`

Read more about generating self-signed certificate authority [here](https://github.com/joetanx/setup/blob/main/self-signed-ca.md)

- Elasticsearch provides `elasticsearch-certutil`, which creates RSA certicates
- Use `openssl` to create ECDSA certificates

<details><summary><h4>1.4.1. Generate RSA certificates with <code>elasticsearch-certutil</code></h4></summary>

##### Transport certificate

|Steps|Commands|
|---|---|
|Create CSR and output to `csr-bundle-transport.zip`|`echo csr-bundle-transport.zip \| /usr/share/elasticsearch/bin/elasticsearch-certutil csr`|
|Unzip CSR bundle to retrieve `instance.csr` and `instance.key`|`unzip /usr/share/elasticsearch/csr-bundle-transport.zip`|
|Create certificate|`openssl x509 -req -sha256 -days 1096 -CA lab_issuer.pem -CAkey lab_issuer.key -CAcreateserial -in instance.csr -out instance.pem`|
|Export certificate and key into `.p12` bundle<br>☝️ Use a strong password for the certificate bundle|`openssl pkcs12 -export -out transport.p12 -inkey instance.key -in instance.pem -name transport -keysig -passout pass:elastic-transport`|
|Import the certificate authority into the certificate bundle with alias as `transport-ca`|`keytool -importcert -trustcacerts -noprompt -keystore transport.p12 -storepass elastic-transport -alias transport-ca -file lab_chain.pem`|
|Verify the resultant certificate bundle|`echo elastic-transport \| keytool -keystore transport.p12 -list`|

##### HTTP certificate

|Steps|Commands|
|---|---|
|Create CSR and output to `csr-bundle-http.zip`|`echo csr-bundle-http.zip \| /usr/share/elasticsearch/bin/elasticsearch-certutil csr --name foxtrot.vx --dns localhost,foxtrot.vx --ip 127.0.0.1,192.168.17.80`|
|Unzip CSR bundle to retrieve `foxtrot.vx.csr` and `foxtrot.vx.key`|`unzip /usr/share/elasticsearch/csr-bundle-http.zip`|
|Create config file for the SANs|`echo "subjectAltName=DNS:foxtrot.vx,DNS:localhost,IP:127.0.0.1,IP:192.168.17.80" > foxtrot.vx.cnf`|
|Create certificate|`openssl x509 -req -sha256 -days 1096 -CA lab_issuer.pem -CAkey lab_issuer.key -CAcreateserial -in foxtrot.vx.csr -extfile foxtrot.vx.cnf -out foxtrot.vx.pem`|
|Export certificate and key into `.p12` bundle<br>☝️ Use a strong password for the certificate bundle|`openssl pkcs12 -export -out http.p12 -inkey foxtrot.vx.key -in foxtrot.vx.pem -name http -keysig -passout pass:elastic-http`|
|Import the certificate authority into the certificate bundle with alias as `http-ca`|`keytool -importcert -trustcacerts -noprompt -keystore http.p12 -storepass elastic-http -alias http-ca -file lab_issuer.pem`|
|Verify the resultant certificate bundle|`echo elastic-http \| keytool -keystore http.p12 -list`|

</details>

#### 1.4.2. Generate ECDSA certificates with `openssl`

##### Transport certificate

|Steps|Commands|
|---|---|
|Create ECDSA key|`openssl ecparam -name secp384r1 -genkey -out transport.key`|
|Create CSR|`openssl req -new -key transport.key -subj "/CN=instance" -out transport.csr`|
|Create certificate|`openssl x509 -req -in transport.csr -CA lab_issuer.pem -CAkey lab_issuer.key -CAcreateserial -days 1096 -sha256 -out transport.pem`|
|Export certificate and key into `.p12` bundle<br>☝️ Use a strong password for the certificate bundle|`openssl pkcs12 -export -out transport.p12 -inkey transport.key -in transport.pem -name transport -keysig -passout pass:elastic-transport`|
|Import the certificate authority into the certificate bundle with alias as `transport-ca`|`keytool -importcert -trustcacerts -noprompt -keystore transport.p12 -storepass elastic-transport -alias transport-ca -file lab_chain.pem`|
|Verify the resultant certificate bundle|`echo elastic-transport \| keytool -keystore transport.p12 -list`|

##### HTTP certificate

|Steps|Commands|
|---|---|
|Create ECDSA key|`openssl ecparam -name secp384r1 -genkey -out http.key`|
|Create CSR|`openssl req -new -key http.key -subj "/CN=foxtrot.vx" -out http.csr`|
|Create config file for the SANs|`echo "subjectAltName=DNS:foxtrot.vx,DNS:localhost,IP:127.0.0.1,IP:192.168.17.80" > http-openssl.cnf`|
|Create certificate|`openssl x509 -req -in http.csr -CA lab_issuer.pem -CAkey lab_issuer.key -CAcreateserial -days 1096 -sha256 -out http.pem -extfile http-openssl.cnf`|
|Export certificate and key into `.p12` bundle<br>☝️ Use a strong password for the certificate bundle|`openssl pkcs12 -export -out http.p12 -inkey http.key -in http.pem -name http -keysig -passout pass:elastic-http`|
|Import the certificate authority into the certificate bundle with alias as `http-ca`|`keytool -importcert -trustcacerts -noprompt -keystore http.p12 -storepass elastic-http -alias http-ca -file lab_chain.pem`|
|Verify the resultant certificate bundle|`echo elastic-http \| keytool -keystore http.p12 -list`|

#### ☝️ About adding certificate authority to the `.p12` bundle using keytool

This step is required; otherwise, the below error will occur:

```
org.elasticsearch.ElasticsearchSecurityException: failed to load SSL configuration [xpack.security.transport.ssl] - the truststore [/etc/elasticsearch/certs/transport.p12] does not contain any trusted certificate entries
```

### 1.5. Configure Elasticsearch

#### 1.5.1. Replace the auto-generated certificates:

```sh
cp lab_issuer.pem /etc/elasticsearch/certs/http_ca.crt
cp http.p12 /etc/elasticsearch/certs/http.p12
cp transport.p12 /etc/elasticsearch/certs/transport.p12
```

#### 1.5.2. Update keystore values:

```sh
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

#### 1.5.3. Edit Elasticsearch configuration file:

Backup original configuraton file: `cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.bak`

|Settings|Commands|
|---|---|
|Bind Elasticsearch to any address (optional)|`sed -i -e '/#network.host: 192.168.0.1/anetwork.bind_host: 0.0.0.0' /etc/elasticsearch/elasticsearch.yml`|
|Configure the stack as a single-node cluster|`sed -i -e '/#discovery.seed_hosts:/adiscovery.type: single-node' /etc/elasticsearch/elasticsearch.yml`|
|Change certificate verification mode to full|`sed -i 's/verification_mode: certificate/verification_mode: full/' /etc/elasticsearch/elasticsearch.yml`|
|Remove `cluster.initial_master_nodes` setting|`sed -i '/^cluster.initial_master_nodes:/d' /etc/elasticsearch/elasticsearch.yml`|

> Remove `cluster.initial_master_nodes` setting is required because setting `discovery.type: single-node` without removing `cluster.initial_master_nodes` will cause `java.lang.IllegalArgumentException: setting [cluster.initial_master_nodes] is not allowed when [discovery.type] is set to [single-node]` error

#### 1.5.4. Enable + start Elasticsearch

```sh
systemctl enable --now elasticsearch
```

### 1.6. Configure Kibana

#### 1.6.1. Copy certificates for Kibana

The same HTTP certificates generated for Elasticsearch are reused for Kibana since it's a single-node stack; generate a separate set of HTTP certificates as required

```sh
cp lab_issuer.pem /etc/kibana/elasticsearch-ca.pem
mkdir /etc/kibana/certs
cp http.pem /etc/kibana/certs/http.crt
cp http.key /etc/kibana/certs/http.key
```

### 1.6.2. Reset password for `kibana_system` user and add the passwords to Kibana keystore

```sh
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
/usr/share/kibana/bin/kibana-keystore add elasticsearch.username
/usr/share/kibana/bin/kibana-keystore add elasticsearch.password
```

#### 1.6.3. Edit Kibana configuration file:

Backup original configuraton file: `cp /etc/kibana/kibana.yml /etc/kibana/kibana.yml.bak`

|Settings|Commands|
|---|---|
|Bind Kibana to any address|`sed -i -e '/#server.host: "localhost"/aserver.host: "0.0.0.0"' /etc/kibana/kibana.yml`|
|Set `publicBaseUrl`|`sed -i -e '/#server.publicBaseUrl:/aserver.publicBaseUrl: https:\/\/foxtrot.vx:5601' /etc/kibana/kibana.yml`|
|Enable Kibana HTTPS|`sed -i -e '/#server.ssl.enabled:/aserver.ssl.enabled: true' /etc/kibana/kibana.yml`|
|Set Kibana HTTPS certificate|`sed -i -e '/#server.ssl.certificate:/aserver.ssl.certificate: \/etc\/kibana\/certs\/http.crt' /etc/kibana/kibana.yml`|
|Set Kibana HTTPS key|`sed -i -e '/#server.ssl.key:/aserver.ssl.key: \/etc\/kibana\/certs\/http.key' /etc/kibana/kibana.yml`|
|Connect to Elasticsearch on localhost via HTTPS|`sed -i -e '/#elasticsearch.hosts:/aelasticsearch.hosts: https:\/\/localhost:9200' /etc/kibana/kibana.yml`|
|Set the Elasticsearch CA|`sed -i -e '/#elasticsearch.ssl.certificateAuthorities:/aelasticsearch.ssl.certificateAuthorities: \/etc\/kibana\/elasticsearch-ca.pem' /etc/kibana/kibana.yml`|
|Uncomment to enable full veritifcation of Elasticsearch certificate|`sed -i '/elasticsearch.ssl.verificationMode:/s/^#//' /etc/kibana/kibana.yml`|

##### 1.6.3.1. Configure Kibana encryption keys

Kibana encryption keys are required for persistent saved objects, running Kibana with encryption keys will lead to following errors:

Under Security > Alerts:

![image](https://github.com/joetanx/setup/assets/90442032/bc3d65d3-00f4-4fa0-9ec9-b53f5862fcb9)

Under Management > Stack Management > Alerts and Insights > Cases:

![image](https://github.com/joetanx/setup/assets/90442032/d00f1fff-d688-4c01-a137-118a7bd5a60a)

Generate Kibana encyption keys: `/usr/share/kibana/bin/kibana-encryption-keys generate`

Append the keys to `/etc/kibana/kibana.yml`

```sh
cat << EOF >> /etc/kibana/kibana.yml
xpack.encryptedSavedObjects.encryptionKey: 6ef2d28868f25ab9fa7772dfb86b258f
xpack.reporting.encryptionKey: e4d0e17626b51e785ee12f2c9aa08adf
xpack.security.encryptionKey: ca716e0bcb992155aefef67fdd44e56e
EOF
```

#### 1.6.4. Enable + start Kibana

```sh
systemctl enable --now kibana
```

#### 1.6.5. Login to Kibana

Reset password for `elastic` and login

```sh
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

## 2. Setup Elastic Agent

Ref: <https://www.elastic.co/guide/en/fleet/current/add-fleet-server-on-prem.html>

### 2.1. Edit Elastic Agent output

Go to `Management` → `Fleet`, select `Edit` on the `default` entry under `Output`

1. Change the `Hosts` URL to `https` with the FQDN which the Elastic Agent will resolve to find Elasticsearch
2. Enter the `Elasticsearch CA trusted fingerprint`

To get certificate fingerprint: `openssl x509 -fingerprint -sha256 -in /path/to/elasticsearch-ca.crt | grep sha256 | sed 's/://g'`

![image](https://github.com/joetanx/setup/assets/90442032/0153e768-389b-4626-8f4b-d8af7fcdbddf)

### 2.2. Setup Fleet Server

Go to `Management` → `Fleet`, select `Add Fleet Server` under `Fleet server hosts`

Enter Fleet Server details and select `Generate Fleet Server policy`

![image](https://github.com/joetanx/setup/assets/90442032/919ac94b-8c0f-49f0-8187-e3c038577106)

The wizard provides Fleet Server installation commands after the policy is created:

![image](https://github.com/joetanx/setup/assets/90442032/2dda3e06-6a2c-4b1a-94a2-25bd8493b042)

To use self-generated certificates, modify the `elastic-agent install` command as stated [here](https://www.elastic.co/guide/en/fleet/current/secure-connections.html#_encrypt_traffic_between_elastic_agents_fleet_server_and_elasticsearch):

```sh
mkdir -p /opt/Elastic/Agent/certs
cp lab_chain.pem /opt/Elastic/Agent/elasticsearch-ca.pem
ln -s /opt/Elastic/Agent/elasticsearch-ca.pem /opt/Elastic/Agent/ca.crt
cp http.pem /opt/Elastic/Agent/certs/fleet-server.crt
cp http.key /opt/Elastic/Agent/certs/fleet-server.key
https://www.elastic.co/guide/en/fleet/current/secure-connections.html#_encrypt_traffic_between_elastic_agents_fleet_server_and_elasticsearch
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.12.0-linux-x86_64.tar.gz
tar xzvf elastic-agent-8.12.0-linux-x86_64.tar.gz
cd elastic-agent-8.12.0-linux-x86_64
./elastic-agent install \
  --url=https://delta.vx:8220 \
  --fleet-server-es=https://delta.vx:9200 \
  --fleet-server-service-token=AAEAAWVsYXN0aWMvZmxlZXQtc2VydmVyL3Rva2VuLTE3MDY4NDUwOTEwNDQ6d2U2UXlvdEpRSkdObkhYSndWQkFrUQ \
  --fleet-server-policy=fleet-server-policy \
  --fleet-server-es-ca-trusted-fingerprint=63DB6C33C00C46A364A1E56F0316E36AABEB290BE5656939C965BC65C9EE755D \
  --fleet-server-es-ca=/opt/Elastic/Agent/elasticsearch-ca.pem \
  --certificate-authorities=/opt/Elastic/Agent/ca.crt \
  --fleet-server-cert=/opt/Elastic/Agent/certs/fleet-server.crt \
  --fleet-server-cert-key=/opt/Elastic/Agent/certs/fleet-server.key \
  --fleet-server-port=8220
```

Example successful installation output:

```console
[root@foxtrot elastic-agent-8.12.0-linux-x86_64]# ./elastic-agent install \
  --url=https://foxtrot.vx:8220 \
  --fleet-server-es=https://foxtrot.vx:9200 \
  --fleet-server-service-token=AAEAAWVsYXN0aWMvZmxlZXQtc2VydmVyL3Rva2VuLTE3MDY4NDUwOTEwNDQ6d2U2UXlvdEpRSkdObkhYSndWQkFrUQ \
  --fleet-server-policy=fleet-server-policy \
  --fleet-server-es-ca-trusted-fingerprint=63DB6C33C00C46A364A1E56F0316E36AABEB290BE5656939C965BC65C9EE755D \
  --fleet-server-es-ca=/etc/elastic-agent/elasticsearch-ca.pem \
  --certificate-authorities=/etc/elastic-agent/ca.crt \
  --fleet-server-cert=/etc/elastic-agent/certs/fleet-server.crt \
  --fleet-server-cert-key=/etc/elastic-agent/certs/fleet-server.key \
  --fleet-server-port=8220
Elastic Agent will be installed at /opt/Elastic/Agent and will run as a service. Do you want to continue? [Y/n]:
[    ] Service Started  [1s] Elastic Agent successfully installed, starting enrollment.
[=   ] Waiting For Enroll...  [3s] {"log.level":"info","@timestamp":"2024-02-02T11:40:43.876+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":461},"message":"Restarting agent daemon, attempt 0","ecs.version":"1.6.0"}
[  ==] Waiting For Enroll...  [5s] {"log.level":"info","@timestamp":"2024-02-02T11:40:45.878+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":804},"message":"Waiting for Elastic Agent to start Fleet Server","ecs.version":"1.6.0"}
[ ===] Waiting For Enroll...  [9s] {"log.level":"info","@timestamp":"2024-02-02T11:40:49.880+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":818},"message":"Fleet Server - Running on policy with Fleet Server integration: fleet-server-policy; missing config fleet.agent.id (expected during bootstrap process)","ecs.version":"1.6.0"}
[  ==] Waiting For Enroll...  [10s] {"log.level":"info","@timestamp":"2024-02-02T11:40:50.606+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":496},"message":"Starting enrollment to URL: https://foxtrot.vx:8220/","ecs.version":"1.6.0"}
[==  ] Waiting For Enroll...  [11s] {"log.level":"info","@timestamp":"2024-02-02T11:40:51.761+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":461},"message":"Restarting agent daemon, attempt 0","ecs.version":"1.6.0"}
{"log.level":"info","@timestamp":"2024-02-02T11:40:51.762+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":285},"message":"Successfully triggered restart on running Elastic Agent.","ecs.version":"1.6.0"}
Successfully enrolled the Elastic Agent.
[==  ] Done  [11s]
Elastic Agent has been successfully installed.
```

![image](https://github.com/joetanx/setup/assets/90442032/ca69f756-86d8-49a8-8303-54ba785ffc0d)

Allow Fleet Server communication on firewall:

```sh
firewall-cmd --add-port 8220/tcp --permanent && firewall-cmd --reload
```

### 2.3. Setup Elastic Agent

Select `Continue enrolling Elastic Agent` from the previous wizard and create an Agent policy:

![image](https://github.com/joetanx/setup/assets/90442032/897c2c2c-708a-4d57-9fc2-92a2b33cc45f)

![image](https://github.com/joetanx/setup/assets/90442032/f8b36b56-95ab-4001-8408-2a165d400174)

The wizard provides Elastic Agent installation commands after the policy is created:

![image](https://github.com/joetanx/setup/assets/90442032/23a52ec9-25cb-4229-b224-53b042526f1e)

To use self-generated certificates, modify the `elastic-agent install` command as stated [here](https://www.elastic.co/guide/en/fleet/current/secure-connections.html#_encrypt_traffic_between_elastic_agents_fleet_server_and_elasticsearch):

```sh
curl -sLo /opt/Elastic/Agent/ca.crt https://github.com/joetanx/lab-certs/raw/main/ca/lab_chain.pem
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.12.0-linux-x86_64.tar.gz
tar xzvf elastic-agent-8.12.0-linux-x86_64.tar.gz
cd elastic-agent-8.12.0-linux-x86_64
./elastic-agent install \
  --url=https://delta.vx:8220 \
  --enrollment-token=UnRqcVo0MEIyUTdXZzV3UVdXR1I6YmRWczM5RXpUOEt0QlJMdWxaaG9sUQ== \
  --certificate-authorities=/opt/Elastic/Agent/ca.crt
```

Example successful installation output:

```console
[root@quebec elastic-agent-8.12.0-linux-x86_64]# ./elastic-agent install --url=https://foxtrot.vx:8220 --enrollment-token=UnRqcVo0MEIyUTdXZzV3UVdXR1I6YmRWczM5RXpUOEt0QlJMdWxaaG9sUQ== --certificate-authorities=/etc/elastic-agent/ca.crt
Elastic Agent will be installed at /opt/Elastic/Agent and will run as a service. Do you want to continue? [Y/n]:
[ ===] Service Started  [1s] Elastic Agent successfully installed, starting enrollment.
[==  ] Waiting For Enroll...  [2s] {"log.level":"info","@timestamp":"2024-02-02T11:48:42.962+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":496},"message":"Starting enrollment to URL: https://foxtrot.vx:8220/","ecs.version":"1.6.0"}
[    ] Waiting For Enroll...  [3s] {"log.level":"info","@timestamp":"2024-02-02T11:48:43.961+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":461},"message":"Restarting agent daemon, attempt 0","ecs.version":"1.6.0"}
{"log.level":"info","@timestamp":"2024-02-02T11:48:43.963+0800","log.origin":{"file.name":"cmd/enroll_cmd.go","file.line":285},"message":"Successfully triggered restart on running Elastic Agent.","ecs.version":"1.6.0"}
Successfully enrolled the Elastic Agent.
[   =] Done  [3s]
Elastic Agent has been successfully installed.
```

![image](https://github.com/joetanx/setup/assets/90442032/d7b59068-0eef-46a4-adf8-e4be154071ce)

### 2.4. Example Fleet Server + Elastic Agent output

![image](https://github.com/joetanx/setup/assets/90442032/a85b793d-802f-421a-b664-d494fd224b08)

## 3. Integrate Suricata to Elastic

### 3.1. Setup Suricata Network IDS

References:
- How To Install Suricata on CentOS 8 Stream: <https://www.digitalocean.com/community/tutorials/how-to-install-suricata-on-centos-8-stream>
- Understanding Suricata Signatures: <https://www.digitalocean.com/community/tutorials/understanding-suricata-signatures>

#### 3.1.1. Port mirroring:

Lab network information:
- The lab is based on Hyper-V
- There are 2 networks `Server` and `Home` in the lab
- The outbound traffic for both networks goes to a firewall with an interface in each network
- For Suricata to monitor outbound traffic for both networks, port mirroring is configured to mirror the traffic on the firewall to Suricata

Interface setting on the firewall network adapter:

![image](https://user-images.githubusercontent.com/90442032/235355173-dcf54a67-97ed-419c-a306-1f151ee5ebc2.png)

Interface setting on the Suricata network adapter:

![image](https://user-images.githubusercontent.com/90442032/235355190-55d747e2-b397-4ee5-988d-790e599eaa32.png)

#### 3.1.2. Install Suricata:

```sh
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
yum -y copr enable @oisf/suricata-latest
yum -y install suricata
```

#### 3.1.3. Edit Suricata configuration:

Multiple interfaces: Read "Set the Interface" <https://forum.suricata.io/t/guide-getting-started-on-centos-8-and-centos-7/538>

```sh
cp /etc/suricata/suricata.yaml /etc/suricata/suricata.yaml.bak
sed -i 's/      community-id: false/\      community-id: true/' /etc/suricata/suricata.yaml
echo -e "detect-engine:\n  - rule-reload: true" >> /etc/suricata/suricata.yaml
sed -i 's/-i eth0/-i eth0 -i eth1/' /etc/sysconfig/suricata
echo group:stream-events.rules > /etc/suricata/disable.conf
systemctl daemon-reload
sudo -u suricata suricata-update
```

#### 3.1.4. Enable + start Suricata:

```sh
systemctl enable --now suricata
systemctl status suricata
tail /var/log/suricata/suricata.log
```

#### 3.1.5. Test IDS:

```sh
curl http://testmynids.org/uid/index.html
curl -Lk http://testmyids.com
tail -f /var/log/suricata/fast.log
```

> [!Note]
>
> SSL decryption is needed to inspect https traffic, then mirror the decrypted traffic to Suricata
> 
> Attempting to `curl https://testmyids.com` without decryption wouldn't log any events

More IDS tests: <https://github.com/3CORESec/testmynids.org>

### 3.2. Add Suricata integration to Elastic Agent

Go to the Agent policy created previous and select `Add integration`

![image](https://user-images.githubusercontent.com/90442032/235355414-2c00b19c-8d28-4731-8e68-c03f750023c1.png)

Search for Suricata and select `Add Suricata`

![image](https://user-images.githubusercontent.com/90442032/235355455-f127632e-4fb7-44bf-a7d5-f32b0a0fdcbd.png)

![image](https://user-images.githubusercontent.com/90442032/235355475-577d9074-83c2-4395-ba74-3f6f129b5102.png)

### 3.3. Visualizing Suricata events

Simply search for Suricata returns `Alerts` and `Events` views in both `Dashboard` and `Discover` views

![image](https://user-images.githubusercontent.com/90442032/235355507-f97e57b8-2b98-4692-824d-7c2b5bc0309d.png)

#### 3.3.1. Dashboard > Alerts

![image](https://user-images.githubusercontent.com/90442032/235355657-4c50d8c3-3145-4cc7-8222-a5cd83e34d1f.png)

#### 3.3.2. Dashboard > Events

![image](https://user-images.githubusercontent.com/90442032/235355663-b6d45fb6-7ea8-452a-bed3-83624a4303bd.png)

#### 3.3.3. Discover > Alerts

![image](https://user-images.githubusercontent.com/90442032/235355668-9a122a04-fe90-49db-b290-ac38aa6876b3.png)

#### 3.3.4. Discover > Events

![image](https://user-images.githubusercontent.com/90442032/235355672-4874fbb7-b912-4675-98c3-f64b78515ca8.png)

#### 3.3.5. Network view

![image](https://user-images.githubusercontent.com/90442032/235355693-7eaa236d-0cc5-4b81-83f1-8a2502f2b8e5.png)

## 4. Syslog integration with CyberArk PAM

Ref: https://docs.elastic.co/en/integrations/cyberarkpas

![image](https://github.com/joetanx/setup/assets/90442032/b9649772-6e2e-49a9-a1c8-ac11e2fcb647)

![image](https://github.com/joetanx/setup/assets/90442032/d30bfef8-7ed0-41d5-9500-82a316288a15)

![image](https://github.com/joetanx/setup/assets/90442032/065703a6-8f14-4c63-be36-ab774c5b237f)

![image](https://github.com/joetanx/setup/assets/90442032/187d3113-e6e5-436a-a3d0-2ee607f15dd0)

This integration sets up a tcp/udp listener on the Elastic Agent, allow communication to the configured ports on firewall to allow the logs from PAM to flow in

```sh
firewall-cmd --permanent --add-port=9301/tcp
firewall-cmd --permanent --add-port=9301/udp
firewall-cmd --reload
```

## 5. Additional Elastic configurations

### 5.1. Stack monitoring

![image](https://user-images.githubusercontent.com/90442032/235392510-5c625734-903a-4b89-8db7-3af11d69a068.png)

Stack monitoring can be fulfilled by adding Elasticsearch and Kibana integrations to the Elastic Agent policy

#### 5.1.1. Prepare integration authentication

The integrations require authentication to connect to Elasticsearch and Kibana

Reset password for `remote_monitoring_user`:

```sh
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u remote_monitoring_user
```

#### 5.1.2. Adding Elasticsearch

Search for Elasticsearch under Integrations:

![image](https://user-images.githubusercontent.com/90442032/235392785-c51204da-e0be-47f9-95c1-0c627b233a40.png)

Configure Elasticsearch URL and `remote_monitoring_user` credentials:

![image](https://user-images.githubusercontent.com/90442032/235392797-fae52017-3eb2-4513-90c8-08b133f9c50a.png)

#### 5.1.3. Adding Kibana

Search for Kibana under Integrations:

![image](https://user-images.githubusercontent.com/90442032/235393010-0b73fae4-9630-4121-a7a3-7d5b7d2c2616.png)

Configure Kibana URL and `remote_monitoring_user` credentials:

![image](https://user-images.githubusercontent.com/90442032/235393015-b6e0d7ca-b9e9-4471-a01f-64d5f7201408.png)

#### 5.1.4. Stack monitoring view

![image](https://user-images.githubusercontent.com/90442032/235393042-4bfacf7d-cc87-4718-a619-9477ec970051.png)

### 5.2. Elastic Defend

Hosts with Elastic Agents can be secured with Elastic Defend

Search for Elastic Defend under Integrations:

![image](https://user-images.githubusercontent.com/90442032/235393087-25c4120e-8d48-45e9-9402-1deefc525ac8.png)

Configure monitoring settings:

![image](https://user-images.githubusercontent.com/90442032/235393096-16e4bd77-a732-4f11-bd79-9b89f01aaba1.png)

Verify Elastic Defend under Security > Manage > Endpoints:

![image](https://user-images.githubusercontent.com/90442032/235393106-84cb4b3a-8094-4d21-9741-43cca35f3c90.png)

## 6. Future: Host-based IDS: Wazuh

References:
- <https://documentation.wazuh.com/current/proof-of-concept-guide/index.html>

## 7. Future: Performance monitoring: Grafana + Prometheus
