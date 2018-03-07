# How to request VM from vRealize catalog using REST API
Firstly, just to be clear following procedure is specific to usecase we are currently having at my job , so it's not necessarily the only way to perform this task.

## How To

1) Get Bearer Token
- Create request body
  - $username and $password are clear , $tenant is usually ***vsphere.local***
- $vra is link to your instance

``` Powershell
$headers = @{'Content-Type' = 'application/json'; 'Accept' = 'application/json'}
$uriGetToken = "https://$vra/identity/api/tokens" = "https://$vra/identity/api/tokens"
$properties = @{'username' = $username ; 'password' = $decodedSecret ; 'tenant' = $tenant}
$bodyJson = New-Object -TypeName psobject -Property $properties |  ConvertTo-Json

$invokeGetToken = Invoke-WebRequest -Uri $uriGetToken -Method POST -Headers $headers -Body $bodyJson
$tokenId = ($invokeGetToken.content | ConvertFrom-Json).id # content propery contains unparsed server response - in headers is specified to accept json response , therefore we convert content from json to PS Object
```

Bearer token is used in all following steps, it is unique identifier used to authenticate you.

Token is not indefinitely valid. Token validity can be configured on the vRealize Automation appliance by modifying `/etc/vcac/security.properties file`
- add the following line to mentioned file
  - N is lifetime in hours
```
identity.basic.token.lifetime.hours=N
```
[Read more about token lifetime](https://docs.vmware.com/en/vRealize-Automation/7.1/com.vmware.vra.programming.doc/GUID-FDB47B40-F651-4FBD-8D1E-AC6FC8FA7A96.html)

2) Get Request Template
