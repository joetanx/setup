> [!Note]
>
> Draft - work in progress

## 1. Example: Sophos Firewall

Content-type: `application/x-www-form-urlencoded`

1. Sophos firewall uses SOAP rather than REST API → `Invoke-WebRequest` instead of `Invoke-RestMethod`
2. `application/x-www-form-urlencoded` uses form request file upload, which is different from uploading file content to `application/json` where the file needs to be byte array and/or base64 encoded

### 1.1. Invoke-WebRequest

Without file upload:

```powershell
$body=@{
  reqxml = Get-Content -Path request.xml -Raw
}
Invoke-WebRequest -Uri "https:/sophos_firewall:4444/webconsole/APIController" -Method Post -Body $body | Select-Object -Expand Content
```

With file upload:

```powershell
$body=@{
  reqxml = Get-Content -Path request.xml -Raw
  file = Get-Item -Path firewall_cert.pfx
}
Invoke-WebRequest -Uri "https://sophos_firewall:4444/webconsole/APIController" -Method Post -Body $body | Select-Object -Expand Content
```

### 1.2. cURL

Without file upload:

```sh
curl -k -F "reqxml=<request.xml" https://sfw.vx:4444/webconsole/APIController

```

With file upload:

```sh
curl -k -F "reqxml=<request.xml" -F "file=@firewall_cert.pfx" https://sfw.vx:4444/webconsole/APIController
curl -k -F "reqxml=<request.xml" -F "file=@root_ca.pem" https://sfw.vx:4444/webconsole/APIController
curl -k -F "reqxml=<request.xml" -F "file=@firewall_cert.pfx" -F "file=@root_ca.pem" -F "file=@intermediate_ca.pem" https://sophos_firewall:4444/webconsole/APIController
```

## 2. Example: Entra identity platform

Content-type: `application/json`

### 2.1. Invoke-RestMethod

> [!Note]
>
> Notice that `ConvertTo-Json` is not needed for Entra APIs

Get Entra endpoints:

```powershell
$tenant = '<tenant_id>'
$openid = Invoke-RestMethod https://login.microsoftonline.com/$tenant/v2.0/.well-known/openid-configuration
```

Authenticate to token endpoint to get access token:

```powershell
$clientid = '<application_id>'
$clientsecret = '<application_secret>'
$body=@{
  client_id = $clientid
  client_secret = $clientsecret
  grant_type = 'client_credentials'
  scope = 'https://monitor.azure.com/.default'
}
$token = Invoke-RestMethod $openid.token_endpoint -Method Post -Body $body
```

### 2.2. cURL

>[!Note]
>
> work in progress

```sh

```

### 3. Example: Azure Monitor Ingestion

Content-type: `application/json`

### 3.1. Invoke-RestMethod

> [!Note]
>
> Notice that `ConvertTo-Json` is not needed for headers but needed for the body

Place the access token into header authorization format:

```powershell
$headers = @{
  Authorization='Bearer '+$token.access_token
}
```

Prepare body to be uploaded:

```powershell
$body = @(
  @{
    Computer='gitlab'
    EventTime=Get-Date ([datetime]::UtcNow) -Format o
    Facility='auth'
    HostIP='192.168.17.21'
    HostName='gitlab'
    ProcessID=1487
    ProcessName='sshd'
    SeverityLevel='info'
    SyslogMessage='Invalid user doesnotexist from 192.168.84.11 port 52892'
    SourceSystem='Cribl'
  }
  @{
    Computer='gitlab'
    EventTime=Get-Date ([datetime]::UtcNow) -Format o
    Facility='auth'
    HostIP='192.168.17.21'
    HostName='gitlab'
    ProcessID=1487
    ProcessName='sshd'
    SeverityLevel='info'
    SyslogMessage='Failed password for invalid user doesnotexist from 192.168.84.11 port 52892 ssh2'
    SourceSystem='Cribl'
  }
  @{
    Computer='kube'
    EventTime=Get-Date ([datetime]::UtcNow) -Format o
    Facility='auth'
    HostIP='192.168.17.22'
    HostName='kube'
    ProcessID=4740
    ProcessName='sshd'
    SeverityLevel='info'
    SyslogMessage='Invalid user doesnotexist from 192.168.84.11 port 52893'
    SourceSystem='Cribl'
  }
  @{
    Computer='kube'
    EventTime=Get-Date ([datetime]::UtcNow) -Format o
    Facility='auth'
    HostIP='192.168.17.22'
    HostName='kube'
    ProcessID=4740
    ProcessName='sshd'
    SeverityLevel='info'
    SyslogMessage='Failed password for invalid user doesnotexist from 192.168.84.11 port 52893 ssh2'
    SourceSystem='Cribl'
  }
) | ConvertTo-Json
```

```powershell
$endpointuri = '<dce_uri>/dataCollectionRules/<dcr_immutable_id>/streams/<stream_name>?api-version=2023-01-01`
Invoke-RestMethod $endpointuri -Method Post -Headers $headers -Body $body -ContentType 'application/json'
```

### 3.2. cURL

>[!Note]
>
> work in progress

```sh

```

### 4. Example: CyberArk PAM API (import connection component)

Content-type: `application/json`

### 4.1. Invoke-RestMethod

> [!Note]
>
> Notice that `ConvertTo-Json` is not needed for headers but needed for the body

Login and get PVWA token:

```powershell
$body=@{
  "username" = "administrator"
  "password" = "password"
} | ConvertTo-Json
$token = Invoke-RestMethod https://cybr.ark.vx/PasswordVault/API/auth/Cyberark/Logon -Method Post -Body $body -ContentType 'application/json'
$headers=@{
  "Authorization" = $token
}
```

The [import connection component API](https://docs.cyberark.com/pam-self-hosted/latest/en/content/webservices/importconncomponent.htm) expects the package to be in a base64 byte array, there is a rather dated but valid KB article on this: https://cyberark-customers.force.com/s/article/How-to-upload-files-using-REST-and-Powershell

Read the file into base64 byte array:

```powershell
$file=[System.IO.File]::ReadAllBytes('<psm_cc>.zip')
```

Reference the byte array to the `ImportFile` field expected by the API:

```powershell
$body=@{ ImportFile=$file } | ConvertTo-JSON
```

Call the API:

```powershell
Invoke-RestMethod https://<pvwa>/PasswordVault/API/ConnectionComponents/Import -Method Post -Headers $headers -Body $cc -ContentType 'application/json'
```

### 4.2. cURL

> [!Note]
>
> work in progress

```sh

```

## X. Additional Notes

### X.1. `Invoke-WebRequest` vs `Invoke-RestMethod`

- `Invoke-WebRequest` general web request, puts response in a PowerShell object where content needs to be referenced by `$_.Content` or `| Select-Object -Expand Content`
- `Invoke-RestMethod` works mostly with JSON APIs and would automatically expand a JSON response

### X.2. PowerShell objects and `ConvertTo-Json`
