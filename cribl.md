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

![image](https://github.com/user-attachments/assets/c696e59e-8924-4872-9981-134beea9f87f)

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

![image](https://github.com/user-attachments/assets/ab09131f-b622-4682-9799-0a4537abfd0a)

![image](https://github.com/user-attachments/assets/5dd854e1-3f1b-4968-8eef-0e6d064a39f0)

![image](https://github.com/user-attachments/assets/7989426a-14bd-4ab0-9704-6c4957ebe444)

![image](https://github.com/user-attachments/assets/78c8c42f-a1d0-4e49-8a32-6f6f9413bcd2)

#### 3.3.2. Authentication

|Field|Description|
|---|---|
|Login URL|The token API endpoint for the Microsoft identity platform. Use the string: `https://login.microsoftonline.com/<tenant_id>/oauth2/v2.0/token`, substituting `<tenant_id>` with Entra ID tenant ID.<br>The Directory (tenant) ID listed on the app's Overview page.<br>![image](https://github.com/user-attachments/assets/fe35d5c5-6112-476e-88d1-b03bb51c1649)|
|OAuth secret|The client secret generated in [3.1.2. Create client secret](#312-create-client-secret)|
|Client ID|The Application (client) ID listed on the app's Overview page.<br>![image](https://github.com/user-attachments/assets/e4ef9523-65e4-4c81-8195-d5a5e2e029ab)|

> [!Tip]
>
> The client ID is entered as a json constant (i.e. enclosing the value with backticks <code>`</code>)

![image](https://github.com/user-attachments/assets/bf230cf4-df3e-4c65-8e78-b0b23d07844b)

#### 3.3.3. Test the data destination

![image](https://github.com/user-attachments/assets/86a05cc5-1e74-4a94-a5b4-33bd17bd803d)

![image](https://github.com/user-attachments/assets/e17f730e-c4b7-4c7f-9eb7-41e02198e623)

### 3.4. Get Cribl packs for Sentinel

Processing → Packs → Add Pack → Add from Dispensary

![image](https://github.com/user-attachments/assets/4db69995-ca12-4b5c-9728-e57da448270a)

Search for `Sentinel`

![image](https://github.com/user-attachments/assets/00edba8c-9490-4ff3-bfa3-cbbcf8a2940d)

The `Microsoft Sentinel` pack by Christoph Dittmann (cdittmann@cribl.io) works well to parse Windows events to column in the SecurityEvent table

![image](https://github.com/user-attachments/assets/84926765-dac0-4221-932f-82b5c348d7fc)

The `Microsoft Sentinel Syslog` pack by Dan Schmitz (dschmitz@cribl.io) works well to parse Linux events to column in the Syslog table

![image](https://github.com/user-attachments/assets/df9c2744-e070-4f34-95e6-61212a6bf783)

### 3.5. Configure routes

![image](https://github.com/user-attachments/assets/8796bfd4-2352-456b-b95a-683d911677ea)

### 3.6. Verify data flow in Cribl

Sources:

![image](https://github.com/user-attachments/assets/0476b027-801d-499d-9f39-aa9dce492f8c)

Routes:

![image](https://github.com/user-attachments/assets/60fd98a5-25df-4072-b750-8d9c5fc7194d)

Packs:

![image](https://github.com/user-attachments/assets/27e5faaa-2067-46cd-8588-c79ad3488236)

Destinations:

![image](https://github.com/user-attachments/assets/1d06afeb-5d12-4222-ab94-b3b574282e53)

### 3.7. Verify events ingested in Sentinel

SecurityEvent table:

![image](https://github.com/user-attachments/assets/6ab50489-e4ca-44ca-a35c-8e2e20bcbdd6)

Syslog table:

![image](https://github.com/user-attachments/assets/56a42c72-051e-491a-b0ed-54e7d982d21f)
