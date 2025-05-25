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

#### 3.2.2. Create DCR (Data Collection Rule) using Cribl template

Go to the created DCE and copy the Resource ID in JSON view:

![image](https://github.com/user-attachments/assets/529e388a-2707-4291-896f-3ae278726a94)

Get the Resource ID for the LAW (Log Analytics Workspace):

![image](https://github.com/user-attachments/assets/d6dc3570-beb1-497e-851c-5e23f915d6ff)

Deploy DCR from [Cribl DCR template](https://docs.cribl.io/stream/usecase-webhook-azure-sentinel-dcr-template/)
- Go to `Deploy a custom template`
- Select `Build your own template in the editor`
- Copy and paste the [Cribl DCR template](https://docs.cribl.io/stream/usecase-webhook-azure-sentinel-dcr-template/)

![image](https://github.com/user-attachments/assets/9ab6e999-733d-4c79-adb1-786b67a1351c)

Paste the DCE and LAW Resource IDs

![image](https://github.com/user-attachments/assets/0768144f-c274-4d93-867b-96fe69671b42)

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

![image](https://github.com/user-attachments/assets/3805332f-5f90-4b1e-b075-c52ca0e0477d)

Select the Cribl application:

> [!Tip]
>
> https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal#assign-a-role-to-the-application
>
> By default, Microsoft Entra applications aren't displayed in the available options. Search for the application by name to find it.

![image](https://github.com/user-attachments/assets/74285140-1bca-4a8f-87c7-535c0ef40d22)

### 3.3. Configure data destination to Sentinel in Cribl

![image](https://github.com/user-attachments/assets/c696e59e-8924-4872-9981-134beea9f87f)

#### 3.3.1. General Settings

Retrieving required information for configuration:

|Field|Description|
|---|---|
|Data collection endpoint|Data collection endpoint (DCE) in the format `https://<endpoint-name>.<identifier>.<region>.ingest.monitor.azure.com`.<br>![image](https://github.com/user-attachments/assets/d5ae0c7c-af03-4035-ba87-17129f707c33)|
|Data collection rule ID|DCR Immutable ID:<br>![image](https://github.com/user-attachments/assets/4009798f-e1e3-410a-a2ba-7b745533c06a)|
|Stream name|Name of the Sentinel table in which to store events.<br>![image](https://github.com/user-attachments/assets/3049dc6f-e974-451e-bf0e-afadd15c47b7)|

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

![image](https://github.com/user-attachments/assets/28cf0e79-2d6b-414c-9054-5675082cd05d)

Configuration of Sentinel as data destination in Cribl can be done using `URL` or `ID`

![image](https://github.com/user-attachments/assets/5f072ed8-76c0-41b1-9038-0342a87dfefa)

![image](https://github.com/user-attachments/assets/62406f1c-4d51-4804-a1cc-db4e80581036)

![image](https://github.com/user-attachments/assets/9a97851c-65e5-492a-9cc5-c246bd6b2afe)

![image](https://github.com/user-attachments/assets/db1e49c6-cc0a-44f9-93cb-2767eddc6b1e)

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
