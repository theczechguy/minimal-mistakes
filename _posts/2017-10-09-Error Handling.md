# Powershell Errors
## Terminating & Non-Terminating Errors

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

The reason for this behaviour is usage of `WriteError` method instead of throwing exceptions. If cmdlet would throw exception or use `ThrowTerminatingError` method, the execution would immediately stop as exceptions are considered as terminating error in Powershell world.

[Read more about non-terminating errors on MSDN](https://msdn.microsoft.com/en-us/library/dd878240(v=vs.85).aspx)

The first thing that people usually try is to wrap this potentially failing part in the try/catch block. And it makes sense right ?

Well that is the tricky thing about non-terminating errors. It won't help.

Let's take a look how this thing works .


### Try/Catch
``` Powershell
try{
    Get-Process -Name 'test'
}
catch{
    write-warning 'a problem occured'
    return
}

Write-Output 'test'

```

This time our previous code got little bit modified, the `Get-Process` part got wrapped in Try/Catch block. In the Catch we try to write some message to warning stream and then call `return` keyword to exit current scope.

Beginners usually assume that thanks to construction like this the script would not continue in case of failure of the first part, but is that really true ?

```
Get-Process : Cannot find a process with the name "test". Verify the process name and call the cmdlet again.
At line:2 char:5
+     Get-Process -Name 'test'
+     ~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (test:String) [Get-Process], ProcessCommandException
    + FullyQualifiedErrorId : NoProcessFoundForGivenName,Microsoft.PowerShell.Commands.GetProcessCommand
 
test
```

See the output , it's exactly the same as previously , strange...


 Well, this is actually intentional behaviour. Because Get-Process throws non-terminating error , and Catch block captures only terminating ones. This means that whatever is in Try is executed, Catch is ignored, as no terminating error was thrown , and then the engine continues with executing next lines.

 Now you're thinking : "so how am I supposed to make this thing work the way I need ?" .
 The solution is easier than you think.

You might have noticed that cmdlets have parameter called ErrorAction. If you dont know already what it means , let me just tell you that this is exactly what you're looking for.
Using this parameter changes the behaviour of the cmdlet, specifically how errors are handled. And it has couple of possible values , let's go through them.

1. Stop
    * Turn all errors into terminating errors.
2. SilentlyContinue
    * Ignore all errors and do not display them.
3. Inquire
    * Inform user there has been an error and let him decide what to do
4. Continue
    * This is default ErrorAction setting , cmdlet displays errors but does not turn then into terminating ones. Which is exactly the case of our example.


***To make our little piece of code work properly we need to do one small change.***

From
> Get-Process -Name 'test'

To
> Get-Process -Name 'test' -ErrorAction Stop

If you execute the code again, you finally receive the expected Warning message saying `a problem occured` .
Reason now should be clear - changing ErrorAction from default `Continue` to `Stop` turned all errors from *Get-Process* into terminating errors, which are captured by `Catch` block.

But be aware that the ErrorAction was set only this one time, only for this one cmdlet, and it was not set globally , so all other cmdlets will use default `Continue` ***ErrorAction preferece*** , unles you specify the EA as we did here.

But what if you want to set the ***ErrorAction preference*** (yes, that how it's called :D ) globally , because you dont want to specify this parameter for every single cmdlet, and you want all of them to throw always terminating errors.
Powershell has a solution for that as well and it's called `ErrorAction Preference` .

### ErrorAction Preference
In general it is automatically defined variable. Let's take a look what PowerShell says about it.

``` Powershell
    Get-Variable ErrorActionPreference | Format-List -force
```

Output :
```
PSPath        : Microsoft.PowerShell.Core\Variable::ErrorActionPreference
PSDrive       : Variable
PSProvider    : Microsoft.PowerShell.Core\Variable
PSIsContainer : False
Name          : ErrorActionPreference
Description   : Dictates the action taken when an error message is delivered
Value         : Continue
Visibility    : Public
Module        : 
ModuleName    : 
Options       : None
Attributes    : {System.Management.Automation.ArgumentTypeConverterAttribute}
```

As you can see from the Description, this is exactly what we were doing with the `Get-Process` but this affects ErrorAction setting globally .

It was mentioned before that default value fro `$ErrorActionPreference` variable is 'Continue' , and changing it is no more complicated than assigning new string value to variable. Want to change the preference to 'Stop' for your script just run simply this
```
    $ErrorActionPreference  = 'Stop'
```

Since this moment all erorrs are treated as terminating, unles you override this setting by using `ErrorAction` parameter of cmdlet and use some different value like `SilentlyContinue` . This means you can do this .

``` Powershell
$ErrorActionPreference = 'SilentlyContinue'
Stop-Process -Name 'test2'
Get-Process -Name 'test' -ErrorAction Stop
```

- We set the `ErrorActionPreference` in the beggining to `SilentlyContinue`, that means that all errors will be ignored and hidden. 
- Second line is executed, `ErrorAction` parameter is not specified , so Global setting from `ErrorActionPreference` is used.
Process `test2` is not found, but error is ignored and not displayed. 
- Script continues with 3rd line. Here we specified `ErrorAction` parameter, with value ***Stop*** . This overrides Global Preference, and just for this one cmdlet all errors are treated as terminating errors, which would lead to failure of script and termination of pipeline.


Hopefully this shed some light on behaviour of errors in Powershell language.
