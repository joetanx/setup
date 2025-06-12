## 1. Requests with body

### 1.1. Content-Type: `application/x-www-form-urlencoded`

#### 1.1.1. PowerShell

##### Request code

```pwsh
$body=@{
  username = "test@example.com"
  password = "SuperPassword"
}
Invoke-RestMethod http://server/login -Method Post -Body $body
```

##### Request sent

```http
POST /login HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.26100.3624
Content-Type: application/x-www-form-urlencoded
Host: server
Content-Length: 50
Expect: 100-continue
Connection: keep-alive

password=SuperPassword&username=test%40example.com
```

#### 1.1.2. cURL

##### Request code

```sh
curl -d 'username=test@example.com' -d 'password=SuperPassword'  http://server/login
```

##### Request sent

```http
POST /login HTTP/1.1
Host: server
User-Agent: curl/7.76.1
Accept: */*
Content-Length: 48
Content-Type: application/x-www-form-urlencoded
Connection: keep-alive

username=test@example.com&password=SuperPassword
```

### 1.2. Content-Type: `application/json`

#### 1.2.1. PowerShell

##### Request code

```pwsh
$body=@{
  username = "test@example.com"
  password = "SuperPassword"
} | ConvertTo-Json
Invoke-RestMethod http://server/login -Method Post -Body $body -ContentType 'application/json'
```

##### Request sent

```http
POST /login HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.26100.3624
Content-Type: application/json
Host: server
Content-Length: 76
Expect: 100-continue
Connection: keep-alive

{
    "password":  "SuperPassword",
    "username":  "test@example.com"
}
```

#### 1.2.2. cURL

##### Request code

```sh
curl -H 'Content-Type: application/json' -d '{"username": "test@example.com", "password": "SuperPassword"}' http://server/login
```

##### Request sent

```http
POST /login HTTP/1.1
Host: server
User-Agent: curl/7.76.1
Accept: */*
Content-Type: application/json
Content-Length: 61
Connection: keep-alive

{"username": "test@example.com", "password": "SuperPassword"}
```

## 2. Requests with headers

### 2.1. Using the access token from authentication

Let's say an API endpoint returns a JSON response such as below:

```json
{
  "authentication": "Successful",
  "authorization": "<access-token-jwt>",
  "message": "Welcome, test@example.com"
}
```

#### 2.1.1. PowerShell: Invoke-WebRequest vs Invoke-RestMethod

> TL:DR
> 
> - `Invoke-WebRequest` general web request, puts response in a PowerShell object where content needs to be referenced by `$_.Content` or `| Select-Object -Expand Content`
> - `Invoke-RestMethod` works well with JSON APIs by automatically expanding the JSON content into `PSCustomObject`

`Invoke-WebRequest` returns the response with the body under `Content`

```readline
StatusCode        : 200
StatusDescription : OK
Content           : {"authentication": "Successful","authorization": "<access-token-...
RawContent        : HTTP/1.1 200 OK
                    Strict-Transport-Security: max-age=31536000; includeSubDomains
                    X-Content-Type-Options: nosniff
                    Access-Control-Allow-Origin: *
                    Access-Control-Allow-Methods: GET, OPTIONS
                    x-ms-reque...
Forms             : {}
Headers           : {[Strict-Transport-Security, max-age=31536000; includeSubDomains], [X-Content-Type-Options, nosniff], [Access-Control-Allow-Origin, *], [Access-Control-Allow-Methods, GET, OPTIONS]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 253
```

A few more steps are required to get the `authorization` access token:

```pwsh
$response = Invoke-WebRequest http://server/login -Method Post -Body $body
$jsonData = $response.Content | ConvertFrom-Json
$accessToken = $jsonData.authorization
```

`InvokeRestMethod` returns the body as a `PSCustomObject`:

```pwsh
PS C:\Users\Joe> $response = Invoke-RestMethod http://server/login -Method Post -Body $body
PS C:\Users\Joe> $response
authentication : Successful
authorization  : <access-token-jwt>
message        : Welcome, test@example.com
PS C:\Users\Joe> $response.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     False    PSCustomObject                           System.Object
```

##### Request code

The `authorization` access token can be reference by `$response.authorization` and placed into header:

```pwsh
$headers = @{
  Authorization = 'Bearer '+$response.access_token
}
Invoke-RestMethod http://server/resource -Headers $headers
```

##### Request sent

```http
GET /resource HTTP/1.1
Authorization: Bearer <access-token-jwt>
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.26100.3624
Host: server
Connection: keep-alive
```

#### 2.1.2. cURL

##### Request code

Since the server response is in JSON, cURL can be used with `jq` to get the `authorization` access token

> [!Tip]
>
> `jq` returns the value with double quotes around it (e.g. `"<access-token-jwt>"`), the `-r` (or `--raw-output`) gets the value without the quotes

```console
[root@ubuntu ~]# echo $response | jq -r .authorization
<access-token-jwt>
```

```console
auth="Authorization: Bearer $(echo $response | jq -r .authorization)"
curl -H "$auth" http://server/resource
```

##### Request sent

```http
GET /resource HTTP/1.1
Host: server
User-Agent: curl/7.76.1
Accept: */*
Authorization: Bearer <access-token-jwt>
Connection: keep-alive
```

### 2.2. Requests with multiple headers

Web requests commonly required sending multiple headers

The `Content-Type` and `Authorization` headers were briefly seen above, and below are some more common examples:

|Header|Example values|Purpose|
|---|---|---|
|Host|`example.com`|If the API endpoint is behind reverse proxies, it may require SNI (Server Name Indication)|
|Cookie|Usually a JWT|Maintains session, usually for browser rather than API|
|Accept-Encoding|`base64`|Indicates the type of encoding that the client can understand|
|Authorization|Usually a JWT|Credentials for authentication|
|Content-Type|`application/json`, `application/x-www-form-urlencoded`, `multipart/form-data`|Specifies the type of content sent by the client|

#### 2.2.1. PowerShell

`Invoke-WebRequest` and `Invoke-RestMethod`:
- Supports the `-ContentType` switch to add the Content-Type to the headers of a request
- Uses hash table as input for `-Headers` to submit each of the key-value as headers

##### Request code

```pwsh
$headers = @{
  Host = "ext.server.com"
  Authorization = 'Bearer '+$response.access_token
  Accept-Encoding = "base64"
}
Invoke-RestMethod http://server/resource -Headers $headers -ContentType 'application/json'
```

##### Request sent

```http
work-in-progress
```

#### 2.2.2. cURL

To specify multiple headers with cURL, simple use `-H` (or `--header`) multiple times

##### Request code

```sh
auth="Authorization: Bearer $(echo $response | jq -r .authorization)"
curl -H 'host: ext.server.com' \
  -H "$auth" \
  -H 'Accept-Encoding: base64' \
  -H 'Content-Type: application/json' \
  http://server/resource
```

##### Request sent

```http
work-in-progress
```


## 3. Uploading data

> work-in-progress



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

A list of key-value items is represented as hash table in PowerShell

```pwsh
$hashtable = @{
  key1 = 'value 1'
  key2 = 'value 2'
  key3 = 3
}
```

Notice that the order of the key-value is not preserved:

```pwsh
PS C:\Users\Joe> $hashtable

Name                           Value
----                           -----
key3                           3
key1                           value 1
key2                           value 2
```

The object type is `System.Object`: `Hashtable`

```pwsh
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

The `arrayB3` array length is 6, PowerShell "flattens" elements of array nested below the parent array upwards to the parent array:

```pwsh
PS C:\Users\Joe> $complexstructure[1].arrayB3.Length
6
PS C:\Users\Joe> $complexstructure[1].arrayB3[0]
value 23
PS C:\Users\Joe> $complexstructure[1].arrayB3[1]

Name                           Value
----                           -----
B3Table1Key2                   B3 table1 value 2
B3Table1Key1                   B3 table1 value 1


PS C:\Users\Joe> $complexstructure[1].arrayB3[2]

Name                           Value
----                           -----
B3Table2Key2                   2322
B3Table2Key1                   B3 table2 value 1


PS C:\Users\Joe> $complexstructure[1].arrayB3[3]
B3 array3 value 1
PS C:\Users\Joe> $complexstructure[1].arrayB3[4]
B3 array3 value 2
PS C:\Users\Joe> $complexstructure[1].arrayB3[5]
2333
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

<details><summary><header2>ARCHIVED</header2></summary>

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
</details>
