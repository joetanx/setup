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

## Annex 0. `Invoke-WebRequest` vs `Invoke-RestMethod`

- `Invoke-WebRequest` general web request, puts response in a PowerShell object where content needs to be referenced by `$_.Content` or `| Select-Object -Expand Content`
- `Invoke-RestMethod` works mostly with JSON APIs and would automatically expand a JSON response

## Annex 1. PowerShell data objects

### 1.1. Arrays

Arrays can be put as single-line with commas or multi-line

Single-line:

```pwsh
$array = @( 'item 1', 'item 2', 3 )
```

Multi-line:

```pwsh
$array = @(
  'item 1'
  'item 2'
  3
)
```

The object type is `System.Array`: `Object[]`

```pwsh
PS C:\Users\Joe> $array
item 1
item 2
3
PS C:\Users\Joe> $array.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array
```

JSON representation:

```pwsh
PS C:\Users\Joe> $array | ConvertTo-Json
```

```json
[
    "item 1",
    "item 2",
    3
]
```

### 1.2. Hash tables

A list of key-value items are represented as hash table in PowerShell

```pwsh
$hashtable = @{
  key1 = 'value 1'
  key2 = 'value 2'
  key3 = 3
}
```

The object type is `System.Object`: `Hashtable`

```pwsh
PS C:\Users\Joe> $hashtable

Name                           Value
----                           -----
key3                           3
key1                           value 1
key2                           value 2


PS C:\Users\Joe> $hashtable.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object
```

JSON representation:

```pwsh
PS C:\Users\Joe> $hashtable | ConvertTo-Json
```

```json
{
    "key3":  3,
    "key1":  "value 1",
    "key2":  "value 2"
}
```

### 1.3. Nested hash tables

Hash tables can contain key-value, arrays, or more hash tables:

```pwsh
$nestedhashtable = @{
  key1 = 'value 1'
  object2 = @{
    subkey2a = 'value 2a'
    subkey2b = 'value 2b'
    subkey2c = 3
  }
  array3 = @( 'item 1', 'item 2', 3 )
}
```

The parent object type is `System.Object`: `Hashtable`

```pwsh
PS C:\Users\Joe> $nestedhashtable

Name                           Value
----                           -----
key1                           value 1
object2                        {subkey2c, subkey2a, subkey2b}
array3                         {item 1, item 2, 3}


PS C:\Users\Joe> $nestedhashtable.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object
```

Individual nested objects have the respective data types:

```pwsh
PS C:\Users\Joe> $nestedhashtable.key1.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object


PS C:\Users\Joe> $nestedhashtable.object2.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object


PS C:\Users\Joe> $nestedhashtable.array3.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array
```

JSON representation:

```pwsh
PS C:\Users\Joe> $nestedhashtable | ConvertTo-Json
```

```json
{
    "key1":  "value 1",
    "object2":  {
                    "subkey2c":  3,
                    "subkey2a":  "value 2a",
                    "subkey2b":  "value 2b"
                },
    "array3":  [
                   "item 1",
                   "item 2",
                   3
               ]
}
```

### 1.4. Array of hash tables

Array elements can also be hash tables:

```pwsh
$arrayofhashtable = @(
  @{
    keyA1 = 'value A1'
    keyA2 = 'value A2'
    keyA3 = 13
  }
  @{
    keyB1 = 'value B1'
    keyB2 = 'value B2'
    keyB3 = 23
  }
)
```

```pwsh
PS C:\Users\Joe> $arrayofhashtable

Name                           Value
----                           -----
keyA2                          value A2
keyA1                          value A1
keyA3                          13
keyB3                          23
keyB1                          value B1
keyB2                          value B2


PS C:\Users\Joe> $arrayofhashtable.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array
```

Each element can be referenced with `[#]` and has the respective data types:

```pwsh
PS C:\Users\Joe> $arrayofhashtable[0]

Name                           Value
----                           -----
keyA2                          value A2
keyA1                          value A1
keyA3                          13


PS C:\Users\Joe> $arrayofhashtable[0].GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object


PS C:\Users\Joe> $arrayofhashtable[1]

Name                           Value
----                           -----
keyB3                          23
keyB1                          value B1
keyB2                          value B2


PS C:\Users\Joe> $arrayofhashtable[1].GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object
```

JSON representation:

```pwsh
PS C:\Users\Joe> $arrayofhashtable | ConvertTo-Json
```

```json
[
    {
        "keyA2":  "value A2",
        "keyA1":  "value A1",
        "keyA3":  13
    },
    {
        "keyB3":  23,
        "keyB1":  "value B1",
        "keyB2":  "value B2"
    }
]
```

### 1.5. Mixing different data types together

```pwsh
$complexstructure = @(
  @{
    keyA1 = 'value A1'
    objectA2 = @{
      subkeyA2a = 'value A2a'
      subkeyA2b = 122
      subkeyA2cArray = @( 'item 121', 'item 122', 123 )
    }
    arrayA3 =  @( 'item 131', 'item 132', 133 )
  }
  @{
    keyB1 = 'value B1'
    ArrayB2 = @( 'item221', 'item222', 223 )
    ArrayB3 = @(
      'value 23'
      @{
        B3Table1Key1 = 'B3 table1 value 1'
        B3Table1Key2 = 'B3 table1 value 2'
      }
      @{
        B3Table2Key1 = 'B3 table2 value 1'
        B3Table2Key2 = 2322
      }
      @( 'B3 array3 value 1', 'B3 array3 value 2', 2333 )
    )
    keyB4 = 24
  }
)
```

```pwsh
PS C:\Users\Joe> $complexstructure

Name                           Value
----                           -----
objectA2                       {subkeyA2a, subkeyA2b, subkeyA2cArray}
keyA1                          value A1
arrayA3                        {item 131, item 132, 133}
arrayB3                        {value 23, System.Collections.Hashtable, System.Collections.Hashtable, B3 array3 value 1...}
keyB1                          value B1
keyB4                          24
arrayB2                        {item221, item222, 223}



PS C:\Users\Joe> $complexstructure.Length
2
PS C:\Users\Joe> $complexstructure.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array
```

```pwsh
PS C:\Users\Joe> $complexstructure[0]

Name                           Value
----                           -----
objectA2                       {subkeyA2a, subkeyA2b, subkeyA2cArray}
keyA1                          value A1
arrayA3                        {item 131, item 132, 133}


PS C:\Users\Joe> $complexstructure[0].GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object


PS C:\Users\Joe> $complexstructure[0].objectA2

Name                           Value
----                           -----
subkeyA2a                      value A2a
subkeyA2b                      122
subkeyA2cArray                 {item 121, item 122, 123}
```

Notice that `keyB4`: `24` is "flatten" up from `arrayB3` to `$complexstructure[1]`, while `value 23` is still in `arrayB3`

```pwsh
PS C:\Users\Joe> $complexstructure[1]

Name                           Value
----                           -----
ArrayB3                        {value 23, System.Collections.Hashtable, System.Collections.Hashtable, B3 array3 value 1...}
keyB1                          value B1
keyB4                          24
ArrayB2                        {item221, item222, 223}


PS C:\Users\Joe> $complexstructure[1].GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object
```

The `arrayB3` array length is 6

```pwsh
PS C:\Users\Joe> $complexstructure[1].arrayB3.Length
6
```

Notice that `value 23` is treated seperately from the other items:

```pwsh
PS C:\Users\Joe> $complexstructure[1].arrayB3
value 23

Name                           Value
----                           -----
B3Table1Key2                   B3 table1 value 2
B3Table1Key1                   B3 table1 value 1
B3Table2Key2                   2322
B3Table2Key1                   B3 table2 value 1
B3 array3 value 1
B3 array3 value 2
2333
```

JSON representation:

```pwsh
PS C:\Users\Joe> $complexstructure | ConvertTo-Json
```

```json
[
    {
        "objectA2":  {
                         "subkeyA2a":  "value A2a",
                         "subkeyA2b":  122,
                         "subkeyA2cArray":  "item 121 item 122 123"
                     },
        "keyA1":  "value A1",
        "arrayA3":  [
                        "item 131",
                        "item 132",
                        133
                    ]
    },
    {
        "arrayB3":  [
                        "value 23",
                        "System.Collections.Hashtable",
                        "System.Collections.Hashtable",
                        "B3 array3 value 1",
                        "B3 array3 value 2",
                        2333
                    ],
        "keyB1":  "value B1",
        "keyB4":  24,
        "arrayB2":  [
                        "item221",
                        "item222",
                        223
                    ]
    }
]
```
