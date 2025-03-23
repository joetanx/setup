## 0. Overview

Windows event forwarding uses WinRM protocol, which supports:
- Kerberos authentication in same-domain environments
- Certificate authentication in separate domains or WORKGROUP environments

## 1. Setup collector and forwarder certificates

Requirements:

|Machine|Requirement|
|---|---|
|Collector|This is the service certificate used for the WinRM HTTPS listener.<br>The FQDN or IP that the forwarders would be using to connect to the collector should be in the subject CN or included in the SANs of the certificate.|
|Forwarder|This the client certificate used by `NETWORK SERVICE` to authenticate the machine connecting to the collector.<br>The forwarder's machine name should be in the subject CN or included in the SANs of the certificate.|

This example uses the self-signed certificate chain from [lab-certs](https://github.com/joetanx/lab-certs) repository

|Certificate|Thumbprint|
|---|---|
|Lab Root|`E9AEA1BA21DA8352AA3999AE94891466CD761958`|
|Lab Issuer|`476F0ABF52FD56722B9C9A833144D9ABB7F55CE9`|

> [!Note]
>
> The intermediate CA `Lab Issuer` is used to generate the certificates
>
> The certificate thumbprint of the intermediate CA, **not root CA**, would be use in both collector and forwarder configurations

> [!Tip]
>
> It is also possible to use the `New-SelfSignedCertificate` to generate direct self-signed certificates without any chain hierarchy
>
> In this case, the certificates between collector and forwarder would need to be cross-trusted (i.e. imported in the the Root store), and the certificate thumbprints would need to be cross-configure in the respective configurations

### 1.1. Download and importthe Lab Issuer package

Perform this on both the collector and forwarder machines

```pwsh
Invoke-WebRequest https://github.com/joetanx/lab-certs/raw/refs/heads/main/ca/lab_issuer.pfx -OutFile lab_issuer.pfx
certutil -csp "Microsoft Software Key Storage Provider" -p lab -importPFX lab_issuer.pfx
```

### 1.2. Generate the certificates

Below commands:
- uses the imported Lab Issuer to sign the certificates (`-Signer cert:\LocalMachine\My\476F0ABF52FD56722B9C9A833144D9ABB7F55CE9`)
- creates certificates with 25 years validity from 2025 to 2050 (yes, it's overkill)

#### Collector

```pwsh
New-SelfSignedCertificate -KeyAlgorithm nistP384 -Subject 'O=vx Lab, CN=Windows Event Collector' `
-TextExtension @("2.5.29.17={text}DNS=dc&DNS=dc.lab.vx&IPAddress=192.168.17.20") -CertStoreLocation cert:\LocalMachine\My `
-Signer cert:\LocalMachine\My\476F0ABF52FD56722B9C9A833144D9ABB7F55CE9 `
-NotBefore ([datetime]::parseexact('01-Jan-2025','dd-MMM-yyyy',$null)) -NotAfter ([datetime]::parseexact('01-Jan-2050','dd-MMM-yyyy',$null))
```

- The `-TextExtension` option is used with OID `2.5.29.17` to specifiy entries for the collector's machine name, FQDN and IP address in the SAN
- It is helpful to include all names and IP addresses that forwarders would use to connect to the collector to minimize SSL certificate errors

#### Forwarder

```pwsh
New-SelfSignedCertificate -KeyAlgorithm nistP384 -Subject 'O=vx Lab, CN=Windows Event Client' `
-DnsName $(hostname) -CertStoreLocation cert:\LocalMachine\My `
-Signer cert:\LocalMachine\My\476F0ABF52FD56722B9C9A833144D9ABB7F55CE9 `
-NotBefore ([datetime]::parseexact('01-Jan-2020','dd-MMM-yyyy',$null)) -NotAfter ([datetime]::parseexact('01-Jan-2050','dd-MMM-yyyy',$null))
```

The forwarder certificate is simpler with just using the `-DnsName` to put the forwarder's machine name in the SAN, this option can only be used to specify a single DNS SAN entry

## 2. Configure WinRM on the collector

### 2.X. OPTIONAL - Testing WinRM connnection to the collector

#### 2.X.1. Configure certificate mapping

#### 2.X.2. Test connection

## 3. Configure events subscriber on the collector

## 4. Configure events forwarding on the forwarder

## 5. Troubleshooing

Log location: Applications and Services → Microsoft → Windows → Eventlog-ForwardingPlugin → Operational

### Expected success events sequence:

- 106: `Subscription policy has changed.  Forwarder is adjusting its subscriptions according to the subscription manager(s) in the updated policy.`

  ![image](https://github.com/user-attachments/assets/92061079-e66c-4854-abae-94324289239b)

- 100: `The subscription default is created successfully.`

  ![image](https://github.com/user-attachments/assets/34a35f93-5009-43f7-bb70-08abf6924ac7)

- 104: `The forwarder has successfully connected to the subscription manager at address https://192.168.17.20:5986/wsman/SubscriptionManager/WEC.`

  ![image](https://github.com/user-attachments/assets/dc3de344-0cb9-4023-8708-62a2bcd2b533)

### 107:

`The WS-Management service cannot find the certificate that was requested.`

![image](https://github.com/user-attachments/assets/36017213-1b48-429c-bb61-d6395ec414ca)

Possible cause: There are no certificates in `Cert:\LocalMachine\My` store that are signed by the certificate thumbprint specified by `IssuerCA`.

Resolution: Check that the client certificate is properly configured.

`Keyset does not exist`

![image](https://github.com/user-attachments/assets/fd3f8afe-626a-48ac-87ba-43ecb93d5e1a)

Possible cause: A certificate signed by the certificate thumbprint specified by `IssuerCA` exists, but `NETWORK SERVICE` does not have `Read` permission on the private key

Resolution: Assign `Read` permission on the private key to `NETWORK SERVICE` under `certlm.msc`

### 105:

`The SSL certificate is signed by an unknown certificate authority.`

![image](https://github.com/user-attachments/assets/1f1ce7c9-5ef4-4411-b72a-fba8aa302f9a)

Possible cause: The certificate used by the collector WinRM HTTPS listener is not trusted by the forwarder

Resolution: Import the issuer CA into the forwarder trust stores

`The SSL certificate contains a common name (CN) that does not match the hostname.`

![image](https://github.com/user-attachments/assets/6a574fa4-2fe1-44f0-bd9c-6637634a5a0e)

Possible cause: The FQDN or IP is not present in the subject or subject alternative names (SANs) of the certificate used by the collector WinRM HTTPS listener

Resolution: Generate a collector certificate that contains the FQDN or IP in the subject, or include the FQDN and/or IP in the SANs

`The WinRM client cannot process the request. The destination computer (<collector>:5986) returned an &apos;access denied&apos; error.`

![image](https://github.com/user-attachments/assets/de06ee81-6764-425c-8ea1-ff7166d1bd6a)

Possible cause: The issue CA that signed the forwarder certificate is not added in the subscriber configuration of the collector

Resolution: Check that the CA is properly configured under `Select Computer Groups` in the subscriber configuration of the collector

`The client cannot connect to the destination specified in the request.`

![image](https://github.com/user-attachments/assets/48f7bc63-6fb9-4145-b820-e82ea913f5ab)

Possible cause: The collector is down or the collector WinRM HTTPS listener is not configured

Resolution: This can usually occur when the collector is rebooting, or the collector WinRM HTTPS listener is deleted accidentally

`The WinRM client sent a request to an HTTP server and got a response saying the requested HTTP URL was not available.`

![image](https://github.com/user-attachments/assets/c7def762-c08d-4770-beb8-de17e881377f)

Possible cause:
- This error message is somewhat misleading, this doesn't mean that the collector cannot be reached (which would be the error above)
- This actually means that `NETWORK SERVICE` does not have permissions to one of the selected event channels to collect on the collector

Resolution: Add `NETWORK SERVICE` to the `Event Log Readers` group on the collector

### 101:

`The subscription default is created, but one or more channels in the query could not be read at this time.`

![image](https://github.com/user-attachments/assets/2db49036-7d75-4686-aaad-1e6d9cad7e30)

Possible cause: `NETWORK SERVICE` does not have access to one or more event log sources (usually the `Security` logs) on the forwarder

Resolution: Add `NETWORK SERVICE` to the `Event Log Readers` group on the forwarder
