## 1. Install Cribl

### 1.1. Download and extract Cribl

> [!Note]
>
> The `C` switch specifies the directory to extract to

```sh
yum -y install tar
curl -sLo - $(curl https://cdn.cribl.io/dl/latest-x64) | tar zxvC /opt
```

### 1.2. Add firewall rules

|Port|Protocol|Purpose|
|---|---|---|
|9000|TCP|Cribl web UI|
|9514|TCP, UDP|Cribl default syslog input, change as necessary|
|5986|TCP|WinRM for Windows Event Collector input|

```sh
firewall-cmd --permanent --add-port 9000/tcp
firewall-cmd --permanent --add-port 9514/udp
firewall-cmd --permanent --add-port 9514/tcp
firewall-cmd --permanent --add-port 5986/tcp
firewall-cmd --reload
```

### 1.3. Setup `cribl` user

```sh
useradd cribl
chown -R cribl:cribl /opt/cribl
```

### 1.4. Configure Cribl to run on boot with systemd

```sh
/opt/cribl/bin/cribl boot-start enable -m systemd -u cribl
systemctl start cribl
```

### 1.5. Initial login

> [!Note]
>
> Default login credentials: `admin`: `admin`

![image](https://github.com/user-attachments/assets/5c7c816f-096a-47f5-84fe-aa3abf6f28ba)

![image](https://github.com/user-attachments/assets/17cb1729-d63c-4316-8eaa-203e316a4e35)

![image](https://github.com/user-attachments/assets/41a1ffc9-d286-4d7d-bfd9-b7bbd7c9b9ab)

### 1.6. Configure TLS

> [!Note]
>
> This example installs Cribl on a host `delta.vx` and uses test certificates from [lab-certs](https://github.com/joetanx/lab-certs)
>
> Use your own certificate chain corresponding to your Cribl hostname

#### 1.6.1. Add certificate

![image](https://github.com/user-attachments/assets/b43306c4-524c-4acf-a55d-57621d2c58b0)

![image](https://github.com/user-attachments/assets/93710ca0-4bd0-4f21-a108-9c6512ce75b3)

#### 1.6.2. Configure TLS with the added certificate

![image](https://github.com/user-attachments/assets/f71abdc9-94ba-4bb9-95a9-5b5a77f50d46)

![image](https://github.com/user-attachments/assets/bdc37100-2d92-4909-bdf5-c31b8559dfba)

![image](https://github.com/user-attachments/assets/bf9915fe-9f62-4366-9ea6-4a0d04d077e3)

## 2. Configure data sources

![image](https://github.com/user-attachments/assets/5b6c8fc2-ae06-407c-b22d-ba2149d87ca0)

### 2.1. Syslog

#### 2.1.1. Cribl configuration

A default syslog source is configured for both TCP and UDP on port 9514 and is not enabled:

![image](https://github.com/user-attachments/assets/bd2cdc7f-505f-46d4-b539-d514cd26e80d)

Enable the data source and clear the UDP port field if not using:

![image](https://github.com/user-attachments/assets/5ff9e526-edeb-4ecd-845c-19d032e811ce)

Configure TLS for the TCP syslog listener:

![image](https://github.com/user-attachments/assets/ca81f5dd-33fa-4571-811f-be24d49e3ff0)

#### 2.1.2. Client configuration

##### Rocky / RHEL
- `rsyslog` installed by default
- `rsyslog-gnutls` needs to be installed for TLS
- SELinux needs to be configured to allow syslog port connection
  - `policycoreutils-python-utils` is needed for the `semanage` command used to configure SELinux

```sh
yum -y install rsyslog-gnutls policycoreutils-python-utils
semanage port -a -t syslogd_port_t -p tcp 9514
```

##### Ubuntu
- Both `rsyslog` and `rsyslog-gnutls` need to be installed
- SELinux configuration not required

```sh
apt -y install rsyslog rsyslog-gnutls
```

##### rsyslog configuration

Prepare the certificate chain

> [!Note]
>
> This example uses test certificate from [lab-certs](https://github.com/joetanx/lab-certs) as configured above
>
> Use your own certificate chain corresponding to your Cribl hostname

```sh
mkdir /etc/rsyslog.d/certs
curl -sLo /etc/rsyslog.d/certs/lab_chain.pem https://github.com/joetanx/lab-certs/raw/refs/heads/main/ca/lab_chain.pem
```

Add rsyslog configuration file under `/etc/rsyslog.d/` to set certificate chain and connectivity to Cribl

```sh
cat << EOF > /etc/rsyslog.d/cribl-tls.conf
global(
  defaultNetstreamDriverCAFile="/etc/rsyslog.d/certs/lab_chain.pem"
)

action(
  type="omfwd" protocol="tcp" target="delta.vx" port="9514" StreamDriver="gtls" StreamDriverMode="1" StreamDriverAuthMode="x509/name" StreamDriverPermittedPeers="*"
)
EOF
```

Restart rsyslog

```sh
systemctl restart rsyslog
```

#### 2.1.3. Verify events coming in to Cribl

Start a capture in `Live Data` tab of the data source and see if events are coming in

![image](https://github.com/user-attachments/assets/547aed01-9780-4056-a0e9-b1c6bd903355)

### 2.2. Windows Event Forwarder

> [!Tip]
>
> Cribl WEF data source supports client certificate and kerberos authentication methods
> - Kerberos method is useful for Active Directory environments where clients are domain trusts are established
> - Client certificate method is useful for heterogeneous environments where clients are in WORKGROUP, different forests, or mixture of both
> - Essentially, Kerberos authentication uses Active Directory as trust while client certificate authentication uses the CA certificate chain as trust to validate clients

#### 2.2.1. Cribl configuration

##### Create WEF data source and configure certificate

> [!Note]
>
> This example uses test certificate from [lab-certs](https://github.com/joetanx/lab-certs) as configured above
>
> Use your own certificate chain corresponding to your Cribl hostname
>
> The CA certificate chain configure would be used to validate client certificates

![image](https://github.com/user-attachments/assets/bfe32db3-19fc-454e-bce3-b519da5703b7)

##### Configure subscription and the logs to collect

This configuration is similar to configuring subscription on Event Viewer [here](https://github.com/joetanx/setup/blob/main/win-event-forwarding.md#4-configure-events-subscriber-on-the-collector)

![image](https://github.com/user-attachments/assets/73d4c449-92af-41b7-9e71-fac172c57b83)

> [!Tip]
>
> Cribl uses XML/XPath query to select the events to collect
>
> Simple mode is a simple way to configure it on GUI, and the raw XML would be generated by Cribl
>
> Use `wevtutil el` (`enum-logs`) to see all paths available (read more on [wevtutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/wevtutil))
>
> Read more on [Cribl queries](https://docs.cribl.io/stream/sources-wef/#queries)

#### 2.2.2. Client configuration

##### Configure client certificate

> [!Note]
>
> This example uses test certificate from [lab-certs](https://github.com/joetanx/lab-certs)
>
> The configuration is similar to the steps [here](https://github.com/joetanx/setup/blob/main/win-event-forwarding.md#1-setup-collector-and-forwarder-certificates)



#### 2.2.3. Verify events coming in to Cribl


