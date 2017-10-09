# Powershell error handling techniques

I guess every one of us got stuck at point where you needed to figure out what is the best way to handle errors in your function/script .

And of course so was I , and suprisingly not just once, but many times, because as it sooner or later always turnes out you can't always use the same technique, or at least it's not always the best possible approach.

So lets see some options, shall we ?
But First it's necessary to make sure you understand the behaviour of errors in Powershell language.

## Powershell errors
This is the thing that makes begginers always confused - terminating and non-terminating errors.

Best way to explain is through examples.

Try to execute following piece of code.
``` Powershell
Get-Process -Name 'test'
Write-Output 'test'
```
Assuming that you dont have process called 'test' running on your system, this would be result.

```
Get-Process : Cannot find a process with the name "test". Verify the process name and call the cmdlet again.
At line:1 char:1
+ Get-Process -Name 'test'
+ ~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (test:String) [Get-Process], ProcessCommandException
    + FullyQualifiedErrorId : NoProcessFoundForGivenName,Microsoft.PowerShell.Commands.GetProcessCommand
 
test
```

See the error and after it the message `test` ? How it is possible that the first part failed and second was executed anyway ?

In this case, nothing bad happens if the second part is executed , but what if the second part would rely on data from the first one, then you have a problem, which is especially confusing for people new to Powershell.

So why the error did not stop execution ? Because this error is non terminating error, meaning that even though the issue happend , it does not stop the execution and it does not terminate pipeline.

The reason for this behaviour is usage of `WriteError` method instead of throwing exceptions. If cmdlet would throw exception or use `ThrowTerminatingError` method, the execution would immediately stop as exception are considered as terminating error in Powershell world.

You can always read more on MSDN :)
[https://msdn.microsoft.com/en-us/library/dd878240(v=vs.85).aspx]()

