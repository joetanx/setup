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

> [!Tip]
>
> There is another guide on native Windows event forwarding [here](https://github.com/joetanx/setup/blob/main/win-event-forwarding.md)
>
> Test out both the native and Cribl way to learn more and compare

#### 2.2.1. Cribl configuration

##### Create WEF data source and configure certificate

> [!Note]
>
> This example uses test certificate from [lab-certs](https://github.com/joetanx/lab-certs) as configured above
>
> Use your own certificate chain corresponding to your Cribl hostname
>
> The CA certificate chain configured would be used to validate client certificates

![image](https://github.com/user-attachments/assets/bfe32db3-19fc-454e-bce3-b519da5703b7)

##### Configure subscription and the logs to collect

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
> Use your own certificate chain to establish trust between Cribl and client

Download and importthe Lab Issuer package

```pwsh
Invoke-WebRequest https://github.com/joetanx/lab-certs/raw/refs/heads/main/ca/lab_issuer.pfx -OutFile lab_issuer.pfx
certutil -csp "Microsoft Software Key Storage Provider" -p lab -importPFX lab_issuer.pfx
```

Generate the certificates

```pwsh
New-SelfSignedCertificate -KeyAlgorithm nistP384 -Subject 'O=vx Lab, CN=Windows Event Client' `
-DnsName $(hostname) -CertStoreLocation cert:\LocalMachine\My `
-Signer cert:\LocalMachine\My\476F0ABF52FD56722B9C9A833144D9ABB7F55CE9 `
-NotBefore ([datetime]::parseexact('01-Jan-2020','dd-MMM-yyyy',$null)) -NotAfter ([datetime]::parseexact('01-Jan-2050','dd-MMM-yyyy',$null))
```

The `New-SelfSignedCertificate` command above:
- uses `-DnsName` to put the forwarder's machine name in the SAN, this option can only be used to specify a single DNS SAN entry
- uses the imported Lab Issuer to sign the certificates (`-Signer cert:\LocalMachine\My\476F0ABF52FD56722B9C9A833144D9ABB7F55CE9`)
- creates certificates with 25 years validity from 2025 to 2050 (yes, it's overkill)

##### Grant permissions to `NETWORK SERVICE` on event log

```cmd
net localgroup "Event Log Readers" "NETWORK SERVICE" /add
```

> [!Note]
>
> In some cases, `NETWORK SERVICE` may need to be added to `Manage auditing and security log` user rights assignment
>
> Location: Group Policy Editer → Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → User Rights Assignment → Manage auditing and security log
>
> ![image](https://github.com/user-attachments/assets/c9b5da09-eaa3-4f6f-92d6-7af514b78535)

##### Grant permission on the client certificate private keys

Open local machine certificate manager (`certlm.msc`), select `Manage Private Keys` on the client certificate:

![image](https://github.com/user-attachments/assets/2d548988-7b0d-4668-881b-35b1d72ab557)

Assign `Read` permissions to `NETWORK SERVICE`:

![image](https://github.com/user-attachments/assets/2486b438-e1d4-4304-8344-9eab513bdfc1)

##### Configure events forwarding 

Group Policy Editer → Computer Configuration → Policies → Windows Settings → Administrative Templates → Windows Components → Event Forwarding → Configure target Subscription Manager

Select `Enabled`

![image](https://github.com/user-attachments/assets/c7bc8586-cd41-4a93-b55a-73edd6a9da7f)

Select `Show` under `SubscriptionManagers` and enter:

```
Server=https://<cribl-name>:5986/wsman/SubscriptionManager/WEC,Refresh=10,IssuerCA=<issuer-certificate-thumbprint>
```

![image](https://github.com/user-attachments/assets/fb4b0a76-cc0b-48bb-b717-41bfafda612f)

##### Successful WEF connection to Cribl

Event `100`: `The subscription <sbuscription-name> is created successfully.`

![image](https://github.com/user-attachments/assets/04bb154f-3790-42f0-be91-d08f43466e1f)

> [!tip]
>
> For possible troubleshooing, refer to section 6 and 7 of the [Windows event forwarding guide](https://github.com/joetanx/setup/blob/main/win-event-forwarding.md)

#### 2.2.3. Verify events coming in to Cribl

Start a capture in `Live Data` tab of the data source and see if events are coming in

![image](https://github.com/user-attachments/assets/d4c5d7f2-e719-48aa-a93b-b40319d79bd7)

### 2.3. Monitoring data sources

![image](https://github.com/user-attachments/assets/1268b325-e73e-440b-bb14-91bc8d75ac79)
