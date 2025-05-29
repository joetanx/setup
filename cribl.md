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

The xPathQueries used by the security events data connector is (go to the JSON view of the DCR to see this):

```
"Security!*[System[(EventID=1) or (EventID=299) or (EventID=300) or (EventID=324) or (EventID=340) or (EventID=403) or (EventID=404) or (EventID=410) or (EventID=411) or (EventID=412) or (EventID=413) or (EventID=431) or (EventID=500) or (EventID=501) or (EventID=1100)]]",
"Security!*[System[(EventID=1102) or (EventID=1107) or (EventID=1108) or (EventID=4608) or (EventID=4610) or (EventID=4611) or (EventID=4614) or (EventID=4622) or (EventID=4624) or (EventID=4625) or (EventID=4634) or (EventID=4647) or (EventID=4648) or (EventID=4649) or (EventID=4657)]]",
"Security!*[System[(EventID=4661) or (EventID=4662) or (EventID=4663) or (EventID=4665) or (EventID=4666) or (EventID=4667) or (EventID=4688) or (EventID=4670) or (EventID=4672) or (EventID=4673) or (EventID=4674) or (EventID=4675) or (EventID=4689) or (EventID=4697) or (EventID=4700)]]",
"Security!*[System[(EventID=4702) or (EventID=4704) or (EventID=4705) or (EventID=4716) or (EventID=4717) or (EventID=4718) or (EventID=4719) or (EventID=4720) or (EventID=4722) or (EventID=4723) or (EventID=4724) or (EventID=4725) or (EventID=4726) or (EventID=4727) or (EventID=4728)]]",
"Security!*[System[(EventID=4729) or (EventID=4733) or (EventID=4732) or (EventID=4735) or (EventID=4737) or (EventID=4738) or (EventID=4739) or (EventID=4740) or (EventID=4742) or (EventID=4744) or (EventID=4745) or (EventID=4746) or (EventID=4750) or (EventID=4751) or (EventID=4752)]]",
"Security!*[System[(EventID=4754) or (EventID=4755) or (EventID=4756) or (EventID=4757) or (EventID=4760) or (EventID=4761) or (EventID=4762) or (EventID=4764) or (EventID=4767) or (EventID=4768) or (EventID=4771) or (EventID=4774) or (EventID=4778) or (EventID=4779) or (EventID=4781)]]",
"Security!*[System[(EventID=4793) or (EventID=4797) or (EventID=4798) or (EventID=4799) or (EventID=4800) or (EventID=4801) or (EventID=4802) or (EventID=4803) or (EventID=4825) or (EventID=4826) or (EventID=4870) or (EventID=4886) or (EventID=4887) or (EventID=4888) or (EventID=4893)]]",
"Security!*[System[(EventID=4898) or (EventID=4902) or (EventID=4904) or (EventID=4905) or (EventID=4907) or (EventID=4931) or (EventID=4932) or (EventID=4933) or (EventID=4946) or (EventID=4948) or (EventID=4956) or (EventID=4985) or (EventID=5024) or (EventID=5033) or (EventID=5059)]]",
"Security!*[System[(EventID=5136) or (EventID=5137) or (EventID=5140) or (EventID=5145) or (EventID=5632) or (EventID=6144) or (EventID=6145) or (EventID=6272) or (EventID=6273) or (EventID=6278) or (EventID=6416) or (EventID=6423) or (EventID=6424) or (EventID=8001) or (EventID=8002)]]",
"Security!*[System[(EventID=8003) or (EventID=8004) or (EventID=8005) or (EventID=8006) or (EventID=8007) or (EventID=8222) or (EventID=26401) or (EventID=30004)]]",
"Microsoft-Windows-AppLocker/EXE and DLL!*[System[(EventID=8001) or (EventID=8002) or (EventID=8003) or (EventID=8004)]]",
"Microsoft-Windows-AppLocker/MSI and Script!*[System[(EventID=8005) or (EventID=8006) or (EventID=8007)]]"
```

To collect the same set of events, the XML qury for Cribl is:

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

## 3. Microsoft Sentinel Integration

Ref:
- https://docs.cribl.io/stream/usecase-azure-workspace/
- https://docs.cribl.io/stream/destinations-sentinel/

### 3.1. Setup Entra Identity for Cribl

#### 3.1.1. Create app registration

> [!Note]
>
> Take note of the `Application (client) ID` and `Directory (tenant) ID`; these will be required later.

![image](https://github.com/user-attachments/assets/241a9a75-0ef9-49d9-ac03-c0c71144ed40)

#### 3.1.2. Create client secret

> [!Important]
>
> The client secret is displayed **only once**, copy and store it securely right after creation
>
> There is no way to retrieve the client secret if it's lost, it will need to be deleted and create a new one

![image](https://github.com/user-attachments/assets/aada95da-c047-41c6-a2f3-2e5a9e235cfd)

### 3.2. Setup data collection

#### 3.2.1. Create DCE (Data Collection Endpoint)

![image](https://github.com/user-attachments/assets/27bcb706-2e48-4bb9-bc89-4a6b22d4e331)

#### 3.2.2. Create DCR (Data Collection Rule) using [Cribl DCR template](https://docs.cribl.io/stream/usecase-webhook-azure-sentinel-dcr-template/)

Go to the created DCE and copy the Resource ID in JSON view:

![image](https://github.com/user-attachments/assets/b5c8c323-30df-4903-a114-0df6f6071a42)

Get the Resource ID for the target LAW (Log Analytics Workspace):

![image](https://github.com/user-attachments/assets/73ede50a-1327-4f6d-808c-f3971db4efe1)

Deploy DCR from [Cribl DCR template](https://docs.cribl.io/stream/usecase-webhook-azure-sentinel-dcr-template/)
- Go to `Deploy a custom template`
- Select `Build your own template in the editor`
- Copy and paste the [Cribl DCR template](https://docs.cribl.io/stream/usecase-webhook-azure-sentinel-dcr-template/)

![image](https://github.com/user-attachments/assets/9ab6e999-733d-4c79-adb1-786b67a1351c)

Paste the DCE and LAW Resource IDs

![image](https://github.com/user-attachments/assets/9c959ea1-896d-4478-89a9-476e39a966ca)

Add role assignment to the DCR

> [!Note]
>
> In Entra, app registration contains information about the application, usually including URLs for SSO (Single Sign-On)
>
> An enterprise application is created automatically when an app is registered
>
> The enterprise application resource is the service prinicipal (i.e. service account or machine identity) of the application
>
> Permissions can be granted to the application by role assignment to the application resource

DCR → Access Control (IAM) → Add role assignment

Select `Monitoring Metrics Publisher` role:

![image](https://github.com/user-attachments/assets/92a4c7f7-3e46-4134-80b5-f8c88fa4655f)

Select the Cribl application:

> [!Tip]
>
> https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal#assign-a-role-to-the-application
>
> By default, Microsoft Entra applications aren't displayed in the available options. Search for the application by name to find it.

![image](https://github.com/user-attachments/assets/73032eb5-7e29-4318-92e6-8fc0460e8200)

### 3.3. Configure data destination to Sentinel in Cribl

![image](https://github.com/user-attachments/assets/3182c6fe-dd13-41d4-8ae6-028c049d590f)

#### 3.3.1. General Settings

Retrieving required information for configuration:

|Field|Description|
|---|---|
|Data collection endpoint|Data collection endpoint (DCE) in the format `https://<endpoint-name>.<identifier>.<region>.ingest.monitor.azure.com`.<br>![image](https://github.com/user-attachments/assets/d5ae0c7c-af03-4035-ba87-17129f707c33)|
|Data collection rule ID|DCR Immutable ID:<br>![image](https://github.com/user-attachments/assets/aa03b33e-2bc9-4ddc-9367-a1a0d92e4a47)|
|Stream name|Name of the Sentinel table in which to store events.<br>![image](https://github.com/user-attachments/assets/5fb1040a-6a5e-451d-8c7f-27c6a6099a39)|

Cribl provides the [Azure Resource Graph Explorer](https://docs.cribl.io/stream/usecase-azure-sentinel/#obtaining-url) to retrieve the required information

```kusto
Resources
| where type =~ 'microsoft.insights/datacollectionrules'
| mv-expand Streams= properties['dataFlows']
| project name, id, DCE = tostring(properties['dataCollectionEndpointId']), ImmutableId = properties['immutableId'], StreamName = Streams['streams'][0]
| join kind=leftouter (Resources
| where type =~ 'microsoft.insights/datacollectionendpoints'
| project name,  DCE = tostring(id), endpoint = properties['logsIngestion']['endpoint']) on DCE
| project name, StreamName, Endpoint = strcat(endpoint, '/dataCollectionRules/',ImmutableId,'/streams/',StreamName,'?api-version=2023-01-01')
```

![image](https://github.com/user-attachments/assets/bf18edcd-0b2f-46c3-b8ca-b7a018047f34)

Configuration of Sentinel as data destination in Cribl can be done using `URL` or `ID`

![image](https://github.com/user-attachments/assets/31764db1-b3f1-4f75-8ef2-32704a058be2)

![image](https://github.com/user-attachments/assets/86faccc4-6354-4316-9303-4989b7847211)

![image](https://github.com/user-attachments/assets/ed19be71-18dd-4c1e-854e-7ded4b63edf2)

![image](https://github.com/user-attachments/assets/25d56410-23c1-40fa-b9f4-95a54bdda486)

#### 3.3.2. Authentication

|Field|Description|
|---|---|
|Login URL|The token API endpoint for the Microsoft identity platform. Use the string: `https://login.microsoftonline.com/<tenant_id>/oauth2/v2.0/token`, substituting `<tenant_id>` with Entra ID tenant ID.<br>The Directory (tenant) ID listed on the app's Overview page.<br>![image](https://github.com/user-attachments/assets/fe35d5c5-6112-476e-88d1-b03bb51c1649)|
|OAuth secret|The client secret generated in [3.1.2. Create client secret](#312-create-client-secret)|
|Client ID|The Application (client) ID listed on the app's Overview page.<br>![image](https://github.com/user-attachments/assets/e4ef9523-65e4-4c81-8195-d5a5e2e029ab)|

> [!Tip]
>
> The client ID is entered as a json constant (i.e. enclosing the value with backticks <code>`</code>)

![image](https://github.com/user-attachments/assets/d03b02d7-1f18-471b-a2df-7e03e345326c)

#### 3.3.3. Test the data destination

![image](https://github.com/user-attachments/assets/65d19943-83b9-47a1-893c-95f32f0a3fbb)

![image](https://github.com/user-attachments/assets/6bf978b7-00d3-40c9-9f43-3815aeb17caa)

### 3.4. Get Cribl packs for Sentinel

Processing → Packs → Add Pack → Add from Dispensary

![image](https://github.com/user-attachments/assets/b6626ad8-98ec-4935-8bf4-0f89b6be4287)

Search for `Sentinel`

![image](https://github.com/user-attachments/assets/4b0f770c-d273-4789-9d6f-3a685a671807)

The `Microsoft Sentinel` pack by Christoph Dittmann (cdittmann@cribl.io) includes a wef pipeline to parse Windows events to columns in the SecurityEvent table

![image](https://github.com/user-attachments/assets/60c366f2-b85e-467a-975c-e19cabc2a6da)

The `Microsoft Sentinel Syslog` pack by Dan Schmitz (dschmitz@cribl.io) includes a syslog pipeline to parse syslog events to columns in the Syslog table

![image](https://github.com/user-attachments/assets/3d1bf2f7-7e5e-47d6-b845-d0029570559c)

### 3.5. Configure pipelines

#### 3.5.1. Syslog pipeline

Go to the `Microsoft Sentinel Syslog` pack and copy the `sentinel_syslog` pipeline

![image](https://github.com/user-attachments/assets/acd092a1-29da-4698-912e-13af64c3538b)

Paste the pipeline

![image](https://github.com/user-attachments/assets/e35e2e92-4825-4ccc-a190-06006a4b7bc3)

![image](https://github.com/user-attachments/assets/4b3bf871-b881-4cf9-80a9-d5021edc0a46)

Edit the `Eval` step of the pipeline:
- Change `String(facility) || facilityName` to `facilityName` for the `Facility` field
  - Sentinel accepts `facilityName` (name) but not `facility` (number) for the `Facility` column
- Add field for `SourceSystem`: `'Cribl'`
- Add `SourceSystem` under `Keep fields`

![image](https://github.com/user-attachments/assets/4a35caea-1dc7-4ab8-a0fa-462547dc6cef)

#### 3.5.2. WEF pipeline

Go to the `Microsoft Sentinel` pack and copy the `wef_security_events` pipeline

![image](https://github.com/user-attachments/assets/8d82e239-55dc-46f4-90a5-57b662bbd333)

Paste the pipeline

![image](https://github.com/user-attachments/assets/db3f3372-ec4c-43c0-8fce-cefb2ffcc2cf)

![image](https://github.com/user-attachments/assets/07913f12-14e1-4622-aa47-1b13018c27a3)

The wef pipeline is usable right away, but additional modification can be done to keep XML or JSON copy of the `EventData` by enabling step 2 or step 6

![image](https://github.com/user-attachments/assets/46ad22a1-6838-44ac-93f2-35f2655287b8)

### 3.6. Configure routes

|Route|Source|Pipeline|Destination|
|---|---|---|---|
|route_wef_to_sentinel|`__inputId=='wef:in_wef'`|sentinel_wef_securityevent|sentinel:out_sentinel_securityevent|
|route_syslog_to_sentinel|`__inputId.startsWith('syslog:in_syslog:')`|sentinel_syslog|sentinel:out_sentinel_syslog|

![image](https://github.com/user-attachments/assets/8fa284e7-bbaf-4caf-8595-26c5652f6fd3)

### 3.7. Verify data flow in Cribl

Sources:

![image](https://github.com/user-attachments/assets/45e85a9a-c9a5-4c63-b622-407a18795439)

Routes:

![image](https://github.com/user-attachments/assets/055b0bc5-1d6f-43b3-a5c0-f2aec81d0bb2)

Pipelines:

![image](https://github.com/user-attachments/assets/40d452e4-352d-44df-a3a5-3b65bbc0a397)

Destinations:

![image](https://github.com/user-attachments/assets/6ec3c85f-c7cf-49d6-8c63-86facd0e9bfc)

### 3.8. Verify events ingested in Sentinel

SecurityEvent table:

![image](https://github.com/user-attachments/assets/1362af83-bdeb-413c-97f3-75c96d115c96)

Syslog table:

![image](https://github.com/user-attachments/assets/81bff44e-3407-4096-9783-dc8ec62a5eb4)
