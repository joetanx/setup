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

![image](https://github.com/user-attachments/assets/dd7d834c-eff4-4daf-9a19-184ba0f3dd5f)

![image](https://github.com/user-attachments/assets/0aa72a42-ec31-41e3-9d97-f2f02e6430ee)

#### 1.6.2. Configure TLS with the added certificate

![image](https://github.com/user-attachments/assets/da24e15f-395a-4627-8c18-ae48591a87cb)

![image](https://github.com/user-attachments/assets/4a5e045e-9394-4d6b-a00b-6e3d338fd665)

## 2. Configure data sources

![image](https://github.com/user-attachments/assets/af9acba5-4dbc-4429-b6a2-f79179197b5e)

### 2.1. Syslog

#### 2.1.1. Cribl configuration

A default syslog source is configured for both TCP and UDP on port 9514 and is not enabled:

![image](https://github.com/user-attachments/assets/dbbff765-90cf-406e-bf92-987e143e3d8d)

Enable the data source and clear the UDP port field if not using:

![image](https://github.com/user-attachments/assets/7d936ac9-3c12-40ab-acba-1a72b08dd4e1)

Configure TLS for the TCP syslog listener:

![image](https://github.com/user-attachments/assets/912933fb-a8d1-4149-83e1-4f7412a5359c)

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

![image](https://github.com/user-attachments/assets/71bcd6bb-9f9b-425e-bc6a-76db68177759)

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

![image](https://github.com/user-attachments/assets/b9198c44-1953-44c3-ba7e-4514d5c71a7f)

##### Configure subscription and the logs to collect

![image](https://github.com/user-attachments/assets/d356c23c-9067-4058-a39d-79f74880431d)

> [!Tip]
>
> Cribl uses XML/XPath query to select the events to collect
>
> Simple mode is a simple way to configure it on GUI, and the raw XML would be generated by Cribl
>
> Use `wevtutil el` (`enum-logs`) to see all paths available (read more on [wevtutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/wevtutil))
>
> Read more on [Cribl queries](https://docs.cribl.io/stream/sources-wef/#queries)

#### 2.2.1a. Sentinel Security Events

Sentinel recommends [a standard set of events for auditing purposes](https://learn.microsoft.com/en-us/azure/sentinel/windows-security-event-id-reference).

The XML query for Cribl to collect the same set of events is:

```xml
<QueryList>
  <Query Id="0">
    <Select Path="Security">*[System[(EventID=1) or (EventID=299) or (EventID=300) or (EventID=324) or (EventID=340) or (EventID=403) or (EventID=404) or (EventID=410) or (EventID=411) or (EventID=412) or (EventID=413) or (EventID=431) or (EventID=500) or (EventID=501) or (EventID=1100)]]</Select>
    <Select Path="Security">*[System[(EventID=1102) or (EventID=1107) or (EventID=1108) or (EventID=4608) or (EventID=4610) or (EventID=4611) or (EventID=4614) or (EventID=4622) or (EventID=4624) or (EventID=4625) or (EventID=4634) or (EventID=4647) or (EventID=4648) or (EventID=4649) or (EventID=4657)]]</Select>
    <Select Path="Security">*[System[(EventID=4661) or (EventID=4662) or (EventID=4663) or (EventID=4665) or (EventID=4666) or (EventID=4667) or (EventID=4688) or (EventID=4670) or (EventID=4672) or (EventID=4673) or (EventID=4674) or (EventID=4675) or (EventID=4689) or (EventID=4697) or (EventID=4700)]]</Select>
    <Select Path="Security">*[System[(EventID=4702) or (EventID=4704) or (EventID=4705) or (EventID=4716) or (EventID=4717) or (EventID=4718) or (EventID=4719) or (EventID=4720) or (EventID=4722) or (EventID=4723) or (EventID=4724) or (EventID=4725) or (EventID=4726) or (EventID=4727) or (EventID=4728)]]</Select>
    <Select Path="Security">*[System[(EventID=4729) or (EventID=4733) or (EventID=4732) or (EventID=4735) or (EventID=4737) or (EventID=4738) or (EventID=4739) or (EventID=4740) or (EventID=4742) or (EventID=4744) or (EventID=4745) or (EventID=4746) or (EventID=4750) or (EventID=4751) or (EventID=4752)]]</Select>
    <Select Path="Security">*[System[(EventID=4754) or (EventID=4755) or (EventID=4756) or (EventID=4757) or (EventID=4760) or (EventID=4761) or (EventID=4762) or (EventID=4764) or (EventID=4767) or (EventID=4768) or (EventID=4771) or (EventID=4774) or (EventID=4778) or (EventID=4779) or (EventID=4781)]]</Select>
    <Select Path="Security">*[System[(EventID=4793) or (EventID=4797) or (EventID=4798) or (EventID=4799) or (EventID=4800) or (EventID=4801) or (EventID=4802) or (EventID=4803) or (EventID=4825) or (EventID=4826) or (EventID=4870) or (EventID=4886) or (EventID=4887) or (EventID=4888) or (EventID=4893)]]</Select>
    <Select Path="Security">*[System[(EventID=4898) or (EventID=4902) or (EventID=4904) or (EventID=4905) or (EventID=4907) or (EventID=4931) or (EventID=4932) or (EventID=4933) or (EventID=4946) or (EventID=4948) or (EventID=4956) or (EventID=4985) or (EventID=5024) or (EventID=5033) or (EventID=5059)]]</Select>
    <Select Path="Security">*[System[(EventID=5136) or (EventID=5137) or (EventID=5140) or (EventID=5145) or (EventID=5632) or (EventID=6144) or (EventID=6145) or (EventID=6272) or (EventID=6273) or (EventID=6278) or (EventID=6416) or (EventID=6423) or (EventID=6424) or (EventID=8001) or (EventID=8002)]]</Select>
    <Select Path="Security">*[System[(EventID=8003) or (EventID=8004) or (EventID=8005) or (EventID=8006) or (EventID=8007) or (EventID=8222) or (EventID=26401) or (EventID=30004)]]</Select>
    <Select Path="Microsoft-Windows-AppLocker/EXE and DLL">*[System[(EventID=8001) or (EventID=8002) or (EventID=8003) or (EventID=8004)]]</Select>
    <Select Path="Microsoft-Windows-AppLocker/MSI and Script">*[System[(EventID=8005) or (EventID=8006) or (EventID=8007)]]</Select>
  </Query>
</QueryList>
```

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

![image](https://github.com/user-attachments/assets/90e5e27d-64e3-4f70-ac9d-5a073f46c0bb)

### 2.3. Monitoring data sources

![image](https://github.com/user-attachments/assets/e0b2348b-f37e-4c83-80c0-1555a1dcb16d)

## 3. Output to local file

### 3.1. Prepare directories

|Path|Purpose|
|---|---|
|`/opt/cribl/out_staging/`|Cribl filesystem output staging directory|
|`/opt/cribl/out_syslog/`|Destination for syslog|
|`/opt/cribl/out_wef/`|Destination for WEF|

> [Important]
>
> The `cribl` user **must** have write permissions at these directories

```sh
mkdir /opt/cribl/{out_staging,out_syslog,out_wef}
chown cribl:cribl /opt/cribl/{out_staging,out_syslog,out_wef}
```

### 3.2. Configure filesystem destination

Ref: https://docs.cribl.io/stream/destinations-fs/

> [!Tip]
> 
> The filesystem destination also supports NFS

![image](https://github.com/user-attachments/assets/6ed9378e-b437-452e-9612-003306d0eb27)

**General Settings**:

|Parameter|Explanation|
|---|---|
|Output ID|The name of the destination|
|Output location|The `out_wef` and `out_syslog` directories prepared above|
|Staging location|The `out_staging` directory prepared above|
|Partitioning expression|Default setting partitions by date using `C.Time.strftime(_time ? _time : Date.now()/1000, '%Y/%m/%d')`<br>If this field is empty, Cribl uses the event's `__partition` field value<br>If the event doesn't have `__partition` field, Cribl falls back to the output location itself.
|Compression|Whether to compress: select between `none` or `gzip`|
|File name prefix expression|Customize file name prefix, default: `CriblOut`|
|File name suffix expression|Customize file name suffix<br>The tarnary operator `{__compression === "gzip" ? ".gz" : ""}` automatically adds `.gz` to the file name if `gzip` comppression is selected|

![image](https://github.com/user-attachments/assets/3b25d508-c699-4b34-9427-987400796081)

![image](https://github.com/user-attachments/assets/9dd2330b-0788-43e8-97b5-9c970fd492b4)

**Post-Processing**:

A pipeline can be assigned for post-processing here, we'll leave it empty and assign pipelines based on routes instead

![image](https://github.com/user-attachments/assets/837ed28b-8434-48ab-b4dc-6471b824b79b)

**Advanced Settings**:

There are several _advanced settings_ that can fine tune the behaviour of sizes, durations, performance, etc

Default settings below would suffice

![image](https://github.com/user-attachments/assets/891c907f-fca4-4636-9205-30ffc35ac14d)

### 3.3. Configure route to filesystem destination

|Route|Source|Pipeline|Destination|
|---|---|---|---|
|route_wef_to_file|`__inputId=='wef:in_wef'`|main|filesystem:out_wef|
|route_syslog_to_file|`__inputId.startsWith('syslog:in_syslog:')`|main|filesystem:out_syslog|

![image](https://github.com/user-attachments/assets/b2bba395-bf4d-44ab-8297-a1cc316cabdd)

### 3.4. Verify file output

Cribl creates a folder structure in the staging location for each destination:

```console
[root@delta ~]# ls -lRh /opt/cribl/out_staging/
/opt/cribl/out_staging/:
total 0
drwxr-xr-x. 4 cribl cribl 120 Jun  4 11:42 out_syslog
drwxr-xr-x. 4 cribl cribl 120 Jun  4 11:42 out_wef

/opt/cribl/out_staging/out_syslog:
total 8.0K
drwxr-xr-x. 2 cribl cribl  6 Jun  4 11:38 Bucket_20250604_1130
drwxr-xr-x. 2 cribl cribl  6 Jun  4 11:40 Bucket_20250604_1140
-rw-r--r--. 1 cribl cribl 12 Jun  4 11:42 CriblOpenFiles.0.json
-rw-r--r--. 1 cribl cribl 12 Jun  4 11:42 CriblOpenFiles.1.json

/opt/cribl/out_staging/out_syslog/Bucket_20250604_1130:
total 0

/opt/cribl/out_staging/out_syslog/Bucket_20250604_1140:
total 0

/opt/cribl/out_staging/out_wef:
total 8.0K
drwxr-xr-x. 2 cribl cribl   6 Jun  4 11:40 Bucket_20250604_1130
drwxr-xr-x. 2 cribl cribl  40 Jun  4 11:42 Bucket_20250604_1140
-rw-r--r--. 1 cribl cribl  12 Jun  4 11:42 CriblOpenFiles.0.json
-rw-r--r--. 1 cribl cribl 129 Jun  4 11:42 CriblOpenFiles.1.json

/opt/cribl/out_staging/out_wef/Bucket_20250604_1130:
total 0

/opt/cribl/out_staging/out_wef/Bucket_20250604_1140:
total 192K
-rw-r--r--. 1 cribl cribl 69K Jun  4 11:42 CriblOut-KtUMq1.1.json.tmp
```

As the partitioning expression was left empty, the output files are simply placed on the respective directories, without any subdirectories structure:

```console
[root@delta ~]# ls -lRh /opt/cribl/{out_syslog,out_wef}
/opt/cribl/out_syslog:
total 1.9M
-rw-r--r--. 1 cribl cribl 391K Jun  4 11:38 CriblOut-1s8ueO.0.json
-rw-r--r--. 1 cribl cribl 1.5M Jun  4 11:38 CriblOut-Bq5tGM.1.json
-rw-r--r--. 1 cribl cribl  713 Jun  4 11:46 CriblOut-E5gLLk.0.json
-rw-r--r--. 1 cribl cribl 5.8K Jun  4 11:48 CriblOut-hBnxRN.0.json
-rw-r--r--. 1 cribl cribl 3.5K Jun  4 11:40 CriblOut-JrGwzh.1.json
-rw-r--r--. 1 cribl cribl 4.5K Jun  4 11:40 CriblOut-OfU90U.0.json
-rw-r--r--. 1 cribl cribl 1.8K Jun  4 11:43 CriblOut-OLyKhS.0.json
-rw-r--r--. 1 cribl cribl 6.1K Jun  4 11:48 CriblOut-sLJTLC.1.json

/opt/cribl/out_wef:
total 536K
-rw-r--r--. 1 cribl cribl 25K Jun  4 11:43 CriblOut-11YwFF.1.json
-rw-r--r--. 1 cribl cribl 54K Jun  4 11:38 CriblOut-cfouGF.1.json
-rw-r--r--. 1 cribl cribl 50K Jun  4 11:47 CriblOut-fDCbbT.1.json
-rw-r--r--. 1 cribl cribl 30K Jun  4 11:44 CriblOut-iA1vIP.1.json
-rw-r--r--. 1 cribl cribl 44K Jun  4 11:46 CriblOut-JLPybW.1.json
-rw-r--r--. 1 cribl cribl 19K Jun  4 11:45 CriblOut-jzxOsd.1.json
-rw-r--r--. 1 cribl cribl 69K Jun  4 11:42 CriblOut-KtUMq1.1.json
-rw-r--r--. 1 cribl cribl 97K Jun  4 11:40 CriblOut-wUXe4r.1.json
-rw-r--r--. 1 cribl cribl 60K Jun  4 11:37 CriblOut-zaFZaM.1.json
-rw-r--r--. 1 cribl cribl 71K Jun  4 11:39 CriblOut-zj8BCV.1.json
```

Example syslog output:

```console
[root@delta ~]# tail -n3 /opt/cribl/out_syslog/CriblOut-sLJTLC.1.json | jq
```

```json
{
  "message": "Failed password for invalid user doesnotexist from 192.168.84.11 port 53647 ssh2",
  "severity": 6,
  "facility": 10,
  "host": "kube",
  "appname": "sshd",
  "procid": "7618",
  "severityName": "info",
  "facilityName": "authpriv",
  "_time": 1749008920,
  "_raw": "<86>Jun  4 11:48:40 kube sshd[7618]: Failed password for invalid user doesnotexist from 192.168.84.11 port 53647 ssh2",
  "cribl": "yes"
}
{
  "message": "Connection reset by invalid user doesnotexist 192.168.84.11 port 53647 [preauth]",
  "severity": 6,
  "facility": 10,
  "host": "kube",
  "appname": "sshd",
  "procid": "7618",
  "severityName": "info",
  "facilityName": "authpriv",
  "_time": 1749008921,
  "_raw": "<86>Jun  4 11:48:41 kube sshd[7618]: Connection reset by invalid user doesnotexist 192.168.84.11 port 53647 [preauth]",
  "cribl": "yes"
}
{
  "message": "PAM 2 more authentication failures; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.84.11",
  "severity": 5,
  "facility": 10,
  "host": "kube",
  "appname": "sshd",
  "procid": "7618",
  "severityName": "notice",
  "facilityName": "authpriv",
  "_time": 1749008921,
  "_raw": "<85>Jun  4 11:48:41 kube sshd[7618]: PAM 2 more authentication failures; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.84.11",
  "cribl": "yes"
}
```

Example wef output:

```console
[root@delta ~]# tail -n3 /opt/cribl/out_wef/CriblOut-zj8BCV.1.json | jq
```

```json
{
  "_raw": "<Event xmlns='http://schemas.microsoft.com/win/2004/08/events/event'><System><Provider Name='Microsoft-Windows-Security-Auditing' Guid='{54849625-5478-4994-a5ba-3e3b0328c30d}'/><EventID>4672</EventID><Version>0</Version><Level>0</Level><Task>12548</Task><Opcode>0</Opcode><Keywords>0x8020000000000000</Keywords><TimeCreated SystemTime='2025-06-04T03:56:05.4226889Z'/><EventRecordID>1393</EventRecordID><Correlation/><Execution ProcessID='848' ThreadID='5208'/><Channel>Security</Channel><Computer>DC.lab.vx</Computer><Security/></System><EventData><Data Name='SubjectUserSid'>S-1-5-18</Data><Data Name='SubjectUserName'>DC$</Data><Data Name='SubjectDomainName'>LAB</Data><Data Name='SubjectLogonId'>0xcbdfb1</Data><Data Name='PrivilegeList'>SeSecurityPrivilege\r\n\t\t\tSeBackupPrivilege\r\n\t\t\tSeRestorePrivilege\r\n\t\t\tSeTakeOwnershipPrivilege\r\n\t\t\tSeDebugPrivilege\r\n\t\t\tSeSystemEnvironmentPrivilege\r\n\t\t\tSeLoadDriverPrivilege\r\n\t\t\tSeImpersonatePrivilege\r\n\t\t\tSeDelegateSessionUserImpersonatePrivilege\r\n\t\t\tSeEnableDelegationPrivilege</Data></EventData></Event>",
  "sourcetype": "wef",
  "source": "wef:in_wef",
  "host": "192.168.17.20",
  "sourceMachineID": "DC.lab.vx",
  "_time": 1749009367.761,
  "cribl": "yes"
}
{
  "_raw": "<Event xmlns='http://schemas.microsoft.com/win/2004/08/events/event'><System><Provider Name='Microsoft-Windows-Security-Auditing' Guid='{54849625-5478-4994-a5ba-3e3b0328c30d}'/><EventID>4624</EventID><Version>3</Version><Level>0</Level><Task>12544</Task><Opcode>0</Opcode><Keywords>0x8020000000000000</Keywords><TimeCreated SystemTime='2025-06-04T03:56:05.4228398Z'/><EventRecordID>1394</EventRecordID><Correlation/><Execution ProcessID='848' ThreadID='5208'/><Channel>Security</Channel><Computer>DC.lab.vx</Computer><Security/></System><EventData><Data Name='SubjectUserSid'>S-1-0-0</Data><Data Name='SubjectUserName'>-</Data><Data Name='SubjectDomainName'>-</Data><Data Name='SubjectLogonId'>0x0</Data><Data Name='TargetUserSid'>S-1-5-18</Data><Data Name='TargetUserName'>DC$</Data><Data Name='TargetDomainName'>LAB.VX</Data><Data Name='TargetLogonId'>0xcbdfb1</Data><Data Name='LogonType'>3</Data><Data Name='LogonProcessName'>Kerberos</Data><Data Name='AuthenticationPackageName'>Kerberos</Data><Data Name='WorkstationName'>-</Data><Data Name='LogonGuid'>{956ba283-e74f-d8fc-d7db-48603cebc1e2}</Data><Data Name='TransmittedServices'>-</Data><Data Name='LmPackageName'>-</Data><Data Name='KeyLength'>0</Data><Data Name='ProcessId'>0x0</Data><Data Name='ProcessName'>-</Data><Data Name='IpAddress'>127.0.0.1</Data><Data Name='IpPort'>61073</Data><Data Name='ImpersonationLevel'>%%1833</Data><Data Name='RestrictedAdminMode'>-</Data><Data Name='RemoteCredentialGuard'>-</Data><Data Name='TargetOutboundUserName'>-</Data><Data Name='TargetOutboundDomainName'>-</Data><Data Name='VirtualAccount'>%%1843</Data><Data Name='TargetLinkedLogonId'>0x0</Data><Data Name='ElevatedToken'>%%1842</Data></EventData></Event>",
  "sourcetype": "wef",
  "source": "wef:in_wef",
  "host": "192.168.17.20",
  "sourceMachineID": "DC.lab.vx",
  "_time": 1749009367.761,
  "cribl": "yes"
}
{
  "_raw": "<Event xmlns='http://schemas.microsoft.com/win/2004/08/events/event'><System><Provider Name='Microsoft-Windows-Security-Auditing' Guid='{54849625-5478-4994-a5ba-3e3b0328c30d}'/><EventID>4634</EventID><Version>0</Version><Level>0</Level><Task>12545</Task><Opcode>0</Opcode><Keywords>0x8020000000000000</Keywords><TimeCreated SystemTime='2025-06-04T03:56:05.4245315Z'/><EventRecordID>1395</EventRecordID><Correlation/><Execution ProcessID='848' ThreadID='904'/><Channel>Security</Channel><Computer>DC.lab.vx</Computer><Security/></System><EventData><Data Name='TargetUserSid'>S-1-5-18</Data><Data Name='TargetUserName'>DC$</Data><Data Name='TargetDomainName'>LAB</Data><Data Name='TargetLogonId'>0xcbdfb1</Data><Data Name='LogonType'>3</Data></EventData></Event>",
  "sourcetype": "wef",
  "source": "wef:in_wef",
  "host": "192.168.17.20",
  "sourceMachineID": "DC.lab.vx",
  "_time": 1749009367.761,
  "cribl": "yes"
}
```

## 4. Microsoft Sentinel Integration

Configure Cribl to send events to Sentinel: https://github.com/joetanx/sentinel/blob/main/cribl.md
