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
What you do here is requesting from VRA list of entitled items.

When you're requesting something from catalog , you see list of requestable items available to your user and you then pick one and proceed with request . Here we do the same thing, just programatically :).

- Update header
  First you we need to update header with token we've acquired in previous step.
``` Powershell
  $headers.Add('Authorization' , "Bearer $tokenId")
```

- Now we'll invoke Rest ***GET*** method against ***entitledCatalogItems***.

```Powershell
$uriGetCatalogItems = "https://$vra/catalog-service/api/consumer/entitledCatalogItems/"
$invokeGetCatalogItems = invoke-webrequest -uri $uriGetCatalogItems -method Get -headers $headers
```

- If the request is sucessfull we receive from server list of catalog items available to us.
- If we inspect them received data we can easily find names of all the catalog items, server sent us, so filter out the item you need, simply by using ```where-object``` cmdlet.

``` Powershell

$convertedContent = $invokeGetcatalogItems.Content | convertfrom-json
$catalogItem = $convertedContent.content | where-object {$_.catalogitem.name -eq 'itemname'}

```

- Now we need to retrieve the links for ***template*** and for ***request*** submision.

``` Powershell

$uriGetItemLinks = "https://$vra/catalog-service/api/consumer/entitledcatalogitemviews/$($catalogItem.id)"
$invokeGetItemLinks = invoke-webrequest -uri $urigetitemLinks -method get -headers $headers
$convertedContent = $invokeGetItemLinks.content | convertfrom-json

```

- Now if we examine the `convertedContent` variable, we'll find two links , one for ***GET*** and one for ***POST*** requests.
  - GET link  : will be used to retrieve request template for our item
  - POST link : will be used to submit new request based on template retrieve from GET link

- Extract the links

``` Powershell
$uriRequestTemplate = $convertedContent.links | where-object {$_.rel -like 'GET*'}
$uriSubmitRequest = $convertedContent.links | where-object {$_.rel -like 'POST*'}
```

- Request the template

``` Powershell
# request template
$invokeGetTemplate = invoke-webrequest -uri $uriRequestTemplate -method get -headers $headers
```

3) Submit request
- Now we use template we retrieved from previous step as a body of new request. In return we receive ***request ID*** which will be used in next step.

``` Powershell
#submit new request
$invokeSubmitRequest = invoke-webrequest -uri $uriSubmitRequest -method Post -headers $headers -body $invokeGetTemplate.content

$requestId = $invokeSubmitRequest.content | convertfrom-json | select-object -expandproperty ID

```

4) Wait for request to complete
-  From previous step we have request ID , so now we can periodically ask server about state of our request.
  - Code example here is very simplified , in real life you should check for other states of request than just 'SUCCESSFUL' like 'FAILED' and others.
  Find all possible states here : [vra request states](https://docs.vmware.com/en/vRealize-Automation/7.3/com.vmware.vra.programming.doc/GUID-D4D6665B-EA83-479A-B732-DB9EDAC185E0.html)

```Powershell
$runLoop = $true

$uriGetRequestStatus = "https://$vra/catalog-service/api/consumer/requests/$requestId"

while($runLoop){

  $invokeGetRequestStatus = invoke-webrequest -uri $urigetRequestStatus -method Get -headers $headers
  $requestStatus = $invokeGetRequestStatus.content | convertfrom-json | select -expandproperty State

  if($requestStatus -eq 'SUCCESSFUL'){
    $runLoop = $false # this breaks loop
  }
}
```

5) Celebrate

- Assuming our request completed without any issues we can celebrate and enjoy our new virtual machine.



If you wonder how to decomission the VM using the same programatic approach , as you'll see in next article it's even simpler than requesting it.

Stay tuned.