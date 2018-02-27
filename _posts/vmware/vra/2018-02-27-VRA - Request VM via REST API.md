# How to request VM from vRealize catalog using REST API
Firstly, just to be clear following procedure is specific to usecase we are currently having at my job , so it's not necessarily the only way to perform this task.

## How To

1) Get Bearer Token
- $vra is your link to your instance

``` Powershell
$properties = @{'username' = $username ; 'password' = $secret ; 'tenant' = 'vsphere.local'}
$uri = "https://$vra/identity/api/tokens"

```