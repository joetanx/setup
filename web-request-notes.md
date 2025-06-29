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

The `authorization` access token can be referenced by `$response.authorization` and placed into header:

```pwsh
$headers = @{
  Authorization = 'Bearer '+$response.authorization
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
  Host = 'ext.server.com'
  Authorization = 'Bearer '+$response.authorization
  'Accept-Encoding' = 'base64'
}
Invoke-RestMethod http://server/resource -Headers $headers -ContentType 'application/json'
```

##### Request sent

```http
GET /resource HTTP/1.1
Authorization: Bearer <access-token-jwt>
Accept-Encoding: gzip, deflate, br
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.26100.3624
Content-Type: application/json
Host: ext.server.com
Connection: keep-alive
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
GET /resource HTTP/1.1
Host: ext.server.com
User-Agent: curl/7.76.1
Accept: */*
Authorization: Bearer <access-token-jwt>
Accept-Encoding: gzip, deflate, br
Content-Type: application/json
Connection: keep-alive
```

## 3. Uploading data

### 3.1. Content-Type: `application/x-www-form-urlencoded` or `multipart/form-data`

Usually for non REST (JSON) APIs such as SOAP API or just API endpoints that take form based input

Let's say an API endpoints takes in:
1. a zip package upload
2. a xml manifest in the `reqxml` parameter such as below:

```xml
<Request Version="2025-06-01">
  <Notification>
    <To>Security Administrator</To>
    <From>Security Alerts</From>
    <Message>This is a test alert</Message>
  </Notification>
</Request>
```

#### 3.1.1. PowerShell

1. `Get-Content` is used to read `test.xml` and the `-Raw` option gets the content as one string, instead of an array of strings
2. `Get-Item` gets the `FileInfo` of the specified file

```pwsh
PS C:\Users\Joe> $reqxml = Get-Content test.xml -Raw
PS C:\Users\Joe> $reqxml
<Request Version="2025-06-01">
  <Notification>
    <To>Security Administrator</To>
    <From>Security Alerts</From>
    <Message>This is a test alert</Message>
  </Notification>
</Request>
PS C:\Users\Joe> $reqxml.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object
```

```pwsh
PS C:\Users\Joe> $file = Get-Item test.zip
PS C:\Users\Joe> $file


    Directory: C:\Users\Joe


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----       13 Jun 2025     07:39            317 test.zip


PS C:\Users\Joe> $file.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     FileInfo                                 System.IO.FileSystemInfo
```

##### Request code

```pwsh
$body=@{
  reqxml = Get-Content test.xml -Raw
  file = Get-Item test.zip
}
```

```pwsh
PS C:\Users\Joe> $body

Name                           Value
----                           -----
reqxml                         <Request Version="2025-06-01">...
file                           C:\Users\Joe\test.zip
```

```pwsh
Invoke-WebRequest http://server/request -Method Post -Body $body
```

##### Request sent

Notice that `Invoke-WebRequest` sends the request as `application/x-www-form-urlencoded`

A new option `-Form` is added in PowerShell 6.1.0, which converts a dictionary to a `multipart/form-data` submission

(https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest#-form)

```http
POST /request HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.26100.3624
Content-Type: application/x-www-form-urlencoded
Host: server
Content-Length: 317
Expect: 100-continue
Connection: keep-alive

reqxml=%3CRequest+Version%3D%222025-06-01%22%3E%0D%0A++%3CNotification%3E%0D%0A++++%3CTo%3ESecurity+Administrator%3C%2FTo%3E%0D%0A++++%3CFrom%3ESecurity+Alerts%3C%2FFrom%3E%0D%0A++++%3CMessage%3EThis+is+a+test+alert%3C%2FMessage%3E%0D%0A++%3C%2FNotification%3E%0D%0A%3C%2FRequest%3E&file=C%3A%5CUsers%5CJoe%5Ctest.zip
```

#### 3.1.2. cURL

cURL takes the multiple `-F` (or `--form`) options and submit the request as `multipart/form-data`
- `<` gets the content of the specified file
- `@` attaches the specified file in the post as a file upload

##### Request code

```sh
curl -F 'reqxml=<test.xml' -F 'file=@test.zip' http://server/request
```

##### Request sent

```http
POST /request HTTP/1.1
Host: server
User-Agent: curl/7.76.1
Accept: */*
Content-Length: 835
Content-Type: multipart/form-data; boundary=------------------------15fe14fd6d40dea9
Connection: keep-alive

--------------------------15fe14fd6d40dea9
Content-Disposition: form-data; name="reqxml"
Content-Type: application/xml

<Request Version="2025-06-01">
  <Notification>
    <To>Security Administrator</To>
    <From>Security Alerts</From>
    <Message>This is a test alert</Message>
  </Notification>
</Request>

--------------------------15fe14fd6d40dea9
Content-Disposition: form-data; name="file"; filename="test.zip"
Content-Type: application/octet-stream

<chunk of the test.zip file>
--------------------------15fe14fd6d40dea9--
```

### 3.2. Content-Type: `application/json`

REST (JSON) APIs usually accepts a file as a base-64 encoded string to a parameter rather than the file upload method above

This simplifies the API communication, but would increase content length submitted as base-64 encoding adds about 33% more data

Let's say an API endpoint accepts a file in the `ImportFile` field

#### 3.2.1. PowerShell

A few commands are needed to convert the file to base-64 encoded string and place into JSON in PowerShell:
1. Read the file with `[System.IO.File]::ReadAllBytes('<file>')`
2. Convert to base-64 with `[Convert]::ToBase64String(<byte-array-from-ReadAllBytes>)`
3. Piping PowerShell object to `ConvertTo-JSON`

```pwsh
$body=@{ ImportFile = [Convert]::ToBase64String([System.IO.File]::ReadAllBytes('test.zip')) } | ConvertTo-JSON
```

##### Request code

```pwsh
Invoke-RestMethod http://server/request -Method Post -Body $body -ContentType 'application/json'
```

##### Request sent

```http
POST /request HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.26100.3624
Content-Type: application/json
Host: server
Content-Length: 455
Expect: 100-continue
Connection: keep-alive

{
    "ImportFile":  "UEsDBBQACAAIAJdCzVoAAAAAAAAAAAAAAAAIACAAdGVzdC54bWx1eAsAAQQAAAAABAAAAABVVA0AB95uS2jebkto1m5LaF2OsQrCQBAFe8F/WNKHiwGt1gMbOy002B9x1QWTw91N4d97OU4Q4VXzphg80WsiNbiQKMdxW7VNu66bTd2sKr9cAOAxGt+4D5buTBLroj9TPwnbG3bXgUdWk2BR0KWrSHuJw4/2JDFFl2kxDqQa7uS7ByukBbC5Jcwquu+bK9xfBrpS7j9QSwcIgcIdq4UAAADDAAAAUEsBAhQDFAAIAAgAl0LNWoHCHauFAAAAwwAAAAgAGAAAAAAAAAAAALaBAAAAAHRlc3QueG1sdXgLAAEEAAAAAAQAAAAAVVQFAAHebktoUEsFBgAAAAABAAEATgAAANsAAAAAAA=="
}
```

#### 3.2.2. cURL

Linux can just use the `base64` command to convert the file to base-64 encoded string

```sh
base64 -w 0 <file>
```

> [!Tip]
>
> `base64` line wraps the output [after 76 characters by default](https://linux.die.net/man/1/base64)
> 
> The `-w 0` option disables line wrapping

##### Request code

```sh
body="{\"ImportFile\": \"$(base64 -w 0 test.zip)\"}"
curl -X POST -H 'Content-Type: application/json' -d "$body" http://server/request
```

##### Request sent

```http
POST /request HTTP/1.1
Host: server
User-Agent: curl/7.76.1
Accept: */*
Content-Type: application/json
Content-Length: 446
Connection: keep-alive

{"ImportFile": "UEsDBBQACAAIAJdCzVoAAAAAAAAAAAAAAAAIACAAdGVzdC54bWx1eAsAAQQAAAAABAAAAABVVA0AB95uS2jebkto1m5LaF2OsQrCQBAFe8F/WNKHiwGt1gMbOy002B9x1QWTw91N4d97OU4Q4VXzphg80WsiNbiQKMdxW7VNu66bTd2sKr9cAOAxGt+4D5buTBLroj9TPwnbG3bXgUdWk2BR0KWrSHuJw4/2JDFFl2kxDqQa7uS7ByukBbC5Jcwquu+bK9xfBrpS7j9QSwcIgcIdq4UAAADDAAAAUEsBAhQDFAAIAAgAl0LNWoHCHauFAAAAwwAAAAgAGAAAAAAAAAAAALaBAAAAAHRlc3QueG1sdXgLAAEEAAAAAAQAAAAAVVQFAAHebktoUEsFBgAAAAABAAEATgAAANsAAAAAAA=="}
```

## Annex 1. Getting response content for HTTP errors

PowerShell's `Invoke-WebRequest` treats HTTP errors as exceptions

Hence, it doesn't store the response content when an error occurs

The response content can be capture by exception handling:

```pwsh
try {
    $response = Invoke-WebRequest -Uri $apiendpoint -Method Get
} catch {
    $errorResponse = $_.Exception.Response.GetResponseStream()
    $reader = New-Object System.IO.StreamReader($errorResponse)
    $reader.ReadToEnd()
}
```

## Annex 2. PowerShell data objects

### 2.1. Arrays

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

### 2.2. Hash tables

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

### 2.3. Nested hash tables

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

### 2.4. Array of hash tables

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

### 2.5. Mixing different data types together

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

## Annex 3. CSV and JSON in PowerShell

There are various cmdlets in PowerShell that help with working with CSV and JSON data:
- `GetContent`
- `ConvertFrom-Json`
- `ConvertTo-Json`
- `ConvertFrom-Csv`
- `ConvertTo-Csv`
- `Import-Csv`
- `Export-Csv`

### 3.1. Uniform data

Conversion between PowerShell array, JSON and CSV is consistent and predictable

Example data:

|alpha  |bravo  |charlie |delta  |echo   |foxtrot|
|-------|-------|--------|-------|-------|-------|
|valueA1|valueB1|valueC1 |valueD1|valueE1|valueF1|
|valueA2|valueB2|valueC2 |valueD2|valueE2|valueF2|
|valueA3|valueB3|valueC3'|valueD3|valueE3|valueF3|

#### 3.1.1. Starting with CSV, convert to JSON

Assign the CSV data as a multi-line variable:

```pwsh
PS C:\Users\Joe> $csv = 'alpha,bravo,charlie,delta,echo,foxtrot
>> valueA1,valueB1,valueC1,valueD1,valueE1,valueF1
>> valueA2,valueB2,valueC2,valueD2,valueE2,valueF2
>> valueA3,valueB3,valueC3,valueD3,valueE3,valueF3'
```

The multi-line variable is `string` type:

```pwsh
PS C:\Users\Joe> $csv.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object


PS C:\Users\Joe> $csv
alpha,bravo,charlie,delta,echo,foxtrot
valueA1,valueB1,valueC1,valueD1,valueE1,valueF1
valueA2,valueB2,valueC2,valueD2,valueE2,valueF2
valueA3,valueB3,valueC3,valueD3,valueE3,valueF3
```

Use `ConvertFrom-Csv` to populate data as a PowerShell array:

```pwsh
PS C:\Users\Joe> $psarray = $csv | ConvertFrom-Csv
PS C:\Users\Joe> $psarray.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array


PS C:\Users\Joe> $psarray


alpha   : valueA1
bravo    : valueB1
charlie : valueC1
delta   : valueD1
echo    : valueE1
foxtrot : valueF1

alpha   : valueA2
bravo    : valueB2
charlie : valueC2
delta   : valueD2
echo    : valueE2
foxtrot : valueF2

alpha   : valueA3
bravo    : valueB3
charlie : valueC3
delta   : valueD3
echo    : valueE3
foxtrot : valueF3
```

> [!Tip]
>
> If the data exists in a file `data.csv`, `Import-Csv` can be used to import directly as a PowerShell array
>
> ```pwsh
> $psarray = Import-Csv data.csv 
> ```

`ConvertTo-Json` converts a PowerShell array or oject to a `string` type JSON:

```pwsh
PS C:\Users\Joe> $json = $psarray | ConvertTo-Json
PS C:\Users\Joe> $json.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object


PS C:\Users\Joe> $json
[
    {
        "alpha":  "valueA1",
        "bravo":  "valueB1",
        "charlie":  "valueC1",
        "delta":  "valueD1",
        "echo":  "valueE1",
        "foxtrot":  "valueF1"
    },
    {
        "alpha":  "valueA2",
        "bravo":  "valueB2",
        "charlie":  "valueC2",
        "delta":  "valueD2",
        "echo":  "valueE2",
        "foxtrot":  "valueF2"
    },
    {
        "alpha":  "valueA3",
        "bravo":  "valueB3",
        "charlie":  "valueC3",
        "delta":  "valueD3",
        "echo":  "valueE3",
        "foxtrot":  "valueF3"
    }
]
```


#### 3.1.2. Starting with JSON, convert to CSV

Assign the JSON data as a multi-line variable:

```pwsh
PS C:\Users\Joe> $json='[{"alpha": "valueA1", "bravo": "valueB1", "charlie": "valueC1", "delta": "valueD1", "echo": "valueE1", "foxtrot": "valueF1"},
>> {"alpha": "valueA2", "bravo": "valueB2", "charlie": "valueC2", "delta": "valueD2", "echo": "valueE2", "foxtrot": "valueF2"},
>> {"alpha": "valueA3", "bravo": "valueB3", "charlie": "valueC3", "delta": "valueD3", "echo": "valueE3", "foxtrot": "valueF3"}]'
```

The multi-line variable is `string` type:

```pwsh
PS C:\Users\Joe> $json.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object


PS C:\Users\Joe> $json
[{"alpha": "valueA1", "bravo": "valueB1", "charlie": "valueC1", "delta": "valueD1", "echo": "valueE1", "foxtrot": "valueF1"},
{"alpha": "valueA2", "bravo": "valueB2", "charlie": "valueC2", "delta": "valueD2", "echo": "valueE2", "foxtrot": "valueF2"},
{"alpha": "valueA3", "bravo": "valueB3", "charlie": "valueC3", "delta": "valueD3", "echo": "valueE3", "foxtrot": "valueF3"}]
```

Use `ConvertFrom-Json` to populate data as a PowerShell array:

```pwsh
PS C:\Users\Joe> $psarray = $json | ConvertFrom-Json
PS C:\Users\Joe> $psarray.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array


PS C:\Users\Joe> $psarray


alpha   : valueA1
bravo    : valueB1
charlie : valueC1
delta   : valueD1
echo    : valueE1
foxtrot : valueF1

alpha   : valueA2
bravo    : valueB2
charlie : valueC2
delta   : valueD2
echo    : valueE2
foxtrot : valueF2

alpha   : valueA3
bravo    : valueB3
charlie : valueC3
delta   : valueD3
echo    : valueE3
foxtrot : valueF3
```

> [!Tip]
>
> If the data exists in a file `data.json`, `Get-Content` can be used with `ConvertFrom-Json` to import directly as a PowerShell array
>
> ```pwsh
> $psarray = Get-Content data.json | ConvertFrom-Json
> ```

`ConvertTo-Csv` converts a PowerShell array or oject to a `array` type CSV:

```pwsh

PS C:\Users\Joe> $csv = $psarray | ConvertTo-Csv
PS C:\Users\Joe> $csv.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array


PS C:\Users\Joe> $csv
#TYPE System.Management.Automation.PSCustomObject
"alpha","bravo","charlie","delta","echo","foxtrot"
"valueA1","valueB1","valueC1","valueD1","valueE1","valueF1"
"valueA2","valueB2","valueC2","valueD2","valueE2","valueF2"
"valueA3","valueB3","valueC3","valueD3","valueE3","valueF3"
```

> [!Tip]
>
> A PowerShell array can also be exported to CSV with `Export-Csv`
>
> ```pwsh
> $psarray | Export-Csv data.csv
> ```

### 3.2. Non-Uniform data

The PowerShell cmdlets work well with uniform data, but can produce unintended results with non-uniform data

Example data:

```json
[
  {
  "alpha": "valueA1",
  "bravo": "valueB1",
  "charlie": "valueC1"
  },
  {
  "charlie": "valueC2",
  "delta": "valueD2"
  },
  {
  "delta": "valueD3",
  "foxtrot": "valueF3"
  }
]
```

#### 3.2.1. Starting with JSON, convert to CSV

Assign the JSON data as a multi-line variable:

```pwsh
PS C:\Users\Joe> $json = '[{"alpha": "valueA1", "bravo": "valueB1", "charlie": "valueC1"},
>> {"charlie": "valueC2", "delta": "valueD2"},
>> {"delta": "valueD3", "foxtrot": "valueF3"}]'
```

The multi-line variable is `string` type:

```pwsh
PS C:\Users\Joe> $json.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object


PS C:\Users\Joe> $json
[{"alpha": "valueA1", "bravo": "valueB1", "charlie": "valueC1"},
{"charlie": "valueC2", "delta": "valueD2"},
{"delta": "valueD3", "foxtrot": "valueF3"}]
```

Using `ConvertFrom-Json` to populate data as a PowerShell array has an interesting effect:
- Only columns `alpha`, `bravo` and `charlie` that the first element has data are retained
- The columns `delta`, `echo` and `foxtrot` are missing

```pwsh
PS C:\Users\Joe> $psarray = $json | ConvertFrom-Json
PS C:\Users\Joe> $psarray.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array


PS C:\Users\Joe> $psarray

alpha   bravo   charlie
-----   ----    -------
valueA1 valueB1 valueC1
                valueC2
```

The data are still accessible with `FL` (`Format-List`):

```pwsh
PS C:\Users\Joe> $psarray | FL


alpha   : valueA1
bravo    : valueB1
charlie : valueC1

charlie : valueC2
delta   : valueD2

delta   : valueD3
foxtrot : valueF3
```

`ConvertTo-Csv` has a disastrous effect as the missing data in `FT` (`Format-Table`) view are not populated in CSV:

```pwsh
PS C:\Users\Joe> $csv = $psarray | ConvertTo-Csv
PS C:\Users\Joe> $csv.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array


PS C:\Users\Joe> $csv
#TYPE System.Management.Automation.PSCustomObject
"alpha","bravo","charlie"
"valueA1","valueB1","valueC1"
,,"valueC2"
,,
```

> [!Warning]
>
> Attempting to convert JSON data that has non-uniform fields can result in data loss
>
> JSON is meant to be flexible and contains only fields that are applicable to the object - be careful when forcing JSON into a tabular representation

#### 3.2.2. Starting with CSV, convert to JSON

Let's say the JSON data in previous section is represented as CSV with empty columns:

|alpha  |bravo  |charlie |delta  |echo   |foxtrot|
|-------|-------|--------|-------|-------|-------|
|valueA1|valueB1|valueC1 |       |       |       |
|       |       |valueC2 |valueD2|       |       |
|       |       |        |valueD3|       |valueF3|


Assign the CSV data as a multi-line variable:

```pwsh
PS C:\Users\Joe> $csv = 'alpha,bravo,charlie,delta,echo,foxtrot
valueA1,valueB1,valueC1,,,
,,valueC2,valueD2,,
,,,valueD3,,valueF3'
```

The multi-line variable is `string` type:

```pwsh
PS C:\Users\Joe> $csv.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object


PS C:\Users\Joe> $csv
alpha,bravo,charlie,delta,echo,foxtrot
valueA1,valueB1,valueC1,,,
,,valueC2,valueD2,,
,,,valueD3,,valueF3
```

`ConvertFrom-Csv` has a less disastrous but possibly problematic effect - empty columns are retained as empty fields:

```pwsh
PS C:\Users\Joe> $psarray = $csv | ConvertFrom-Csv
PS C:\Users\Joe> $psarray.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array


PS C:\Users\Joe> $psarray


alpha   : valueA1
bravo    : valueB1
charlie : valueC1
delta   :
echo    :
foxtrot :

alpha   :
bravo    :
charlie : valueC2
delta   : valueD2
echo    :
foxtrot :

alpha   :
bravo    :
charlie :
delta   : valueD3
echo    :
foxtrot : valueF3
```

`ConvertTo-Json` cascades the empty fields to the JSON body - this can be undesirable as having empty fields in JSON can have a different meaning from not having the fields at all:

```pwsh
PS C:\Users\Joe> $json = $psarray | ConvertTo-Json
PS C:\Users\Joe> $json.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object


PS C:\Users\Joe> $json
[
    {
        "alpha":  "valueA1",
        "bravo":  "valueB1",
        "charlie":  "valueC1",
        "delta":  "",
        "echo":  "",
        "foxtrot":  ""
    },
    {
        "alpha":  "",
        "bravo":  "",
        "charlie":  "valueC2",
        "delta":  "valueD2",
        "echo":  "",
        "foxtrot":  ""
    },
    {
        "alpha":  "",
        "bravo":  "",
        "charlie":  "",
        "delta":  "valueD3",
        "echo":  "",
        "foxtrot":  "valueF3"
    }
]
```

> [!Important]
>
> This doesn't mean the conversion is erroneous - the CSV representation contained empty columns, so `ConvertFrom-Csv` and `ConvertTo-Json` correctly interpreted those empty columns as empty fields
>
> This, however, means to be careful with representing JSON data as CSV, as it can have unintended results

### 3.3. Data structure in PowerShell

#### 3.3.1. PowerShell array

The notation to create an array in PowerShell is `@()`

```pwsh
PS C:\Users\Joe> $body = @(
>>   'alpha'
>>   'bravo'
>>   'charlie'
>> )
PS C:\Users\Joe> $body.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array


PS C:\Users\Joe> $body
alpha
bravo
charlie
```

#### 3.3.2. PowerShell object - HashTable

The notation to create a data object in PowerShell is `@{}`

```pwsh
PS C:\Users\Joe> $body = @{
>>   alpha='valueA'
>>   bravo='valueB'
>>   charlie='valueC'
>> }
PS C:\Users\Joe> $body.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object


PS C:\Users\Joe> $body

Name                           Value
----                           -----
alpha                          valueA
bravo                          valueB
charlie                        valueC
```

The object is created as `HashTable` when the type is not specified

`HashTable` is a simple key-value data structure, which may have unintended results such as when the data is supposed to have column-like format:

```pwsh
PS C:\Users\Joe> $body = @(
>>   @{
>>     alpha='valueA1'
>>     bravo='valueB1'
>>     charlie='valueC1'
>>   }
>>   @{
>>     alpha='valueA2'
>>     bravo='valueB2'
>>     charlie='valueC2'
>>   }
>>   @{
>>     alpha='valueA3'
>>     bravo='valueB3'
>>     charlie='valueC3'
>>   }
>> )
```

Instead of an array of objects, the entire data is represented as a list of key-value items:

```pwsh
PS C:\Users\Joe> $body.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object


PS C:\Users\Joe> $body

Name                           Value
----                           -----
alpha                          valueA1
bravo                          valueB1
charlie                        valueC1
alpha                          valueA2
bravo                          valueB2
charlie                        valueC2
alpha                          valueA3
bravo                          valueB3
charlie                        valueC3
```

Retrieving the first element of the array returns that portion of key-value items of that element:

```pwsh
PS C:\Users\Joe> $body[0].GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Hashtable                                System.Object


PS C:\Users\Joe> $body[0]

Name                           Value
----                           -----
alpha                          valueA1
bravo                          valueB1
charlie                        valueC1
```

Being a key-value structure, the object has only `Keys` and `Values` columns:

```pwsh
PS C:\Users\Joe> $body.Keys
alpha
bravo
charlie
alpha
bravo
charlie
alpha
bravo
charlie
PS C:\Users\Joe> $body.Values
valueA1
valueB1
valueC1
valueA2
valueB2
valueC2
valueA3
valueB3
valueC3
PS C:\Users\Joe> $body[0].Keys
alpha
bravo
charlie
PS C:\Users\Joe> $body[0].Values
valueA1
valueB1
valueC1
```

#### 3.3.3. PowerShell object - PSCustomObject

Creating data as `PSCustomObject` would be more suitable and allows more flexibility

This is done by casting the object as `PSCustomObject` by adding `[PSCustomObject]` in front of each of the `@{}` declaration:

```pwsh
PS C:\Users\Joe> $body = @(
>>   [PSCustomObject]@{
>>     alpha='valueA1'
>>     bravo='valueB1'
>>     charlie='valueC1'
>>   }
>>   [PSCustomObject]@{
>>     alpha='valueA2'
>>     bravo='valueB2'
>>     charlie='valueC2'
>>   }
>>   [PSCustomObject]@{
>>     alpha='valueA3'
>>     bravo='valueB3'
>>     charlie='valueC3'
>>   }
>> )
```

The data is represented as an array of objects, each object is a row of the table:

```pwsh
PS C:\Users\Joe> $body.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array


PS C:\Users\Joe> $body

alpha   bravo   charlie
-----   -----   -------
valueA1 valueB1 valueC1
valueA2 valueB2 valueC2
valueA3 valueB3 valueC3
```

Retrieving the an element of the array returns a row of the table:

```pwsh
PS C:\Users\Joe> $body[0].GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     False    PSCustomObject                           System.Object


PS C:\Users\Joe> $body[0]

alpha   bravo   charlie
-----   -----   -------
valueA1 valueB1 valueC1
```

Columns can be referenced by `.<name>`:

```pwsh
PS C:\Users\Joe> $body.alpha
valueA1
valueA2
valueA3
PS C:\Users\Joe> $body.bravo
valueB1
valueB2
valueB3
PS C:\Users\Joe> $body.charlie
valueC1
valueC2
valueC3
```
