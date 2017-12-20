# Error handling techniques

I guess every one of us got stuck at point where you needed to figure out what is the best way to handle errors in your function/script .

And of course so was I , and suprisingly not just once, but many times, because as it sooner or later always turnes out, you can't always use the same technique, or at least it's not always the best possible approach.

So lets see some options, shall we ?
But first it's necessary to make sure you understand the behaviour of errors in Powershell language.
If things like terminating and non-terminating errors are unknown terms to you, I would suggest to read previous my previous post about errors.

[Powershell Errors](https://theczechguy.github.io/Powershell-Errors/)


## TRY / CATCH / FINALLY
`Try/Catch`c The classic construct known for years.
Let's just see simple example, so we know what we're dealing with.

``` Powershell
    try{
        #code that can cause TERMINATING error
    }
    catch{
        # code to handle the situation
    }
```

But the header of this section contains one more keyword : `FINALLY`.
Often, this little guy is  overlooked. If you're asking yourself why, the answer is simple , because `try/catch` works fine without `finally`.
Now let's take a look on each keyword and see what they do.

### TRY
Try is the place where you put code, that could potentially throw terminating errors.
- It's important to remember that in this error handling construct, `try` is always mandatory , while the catch/finally are optionall , but at least one of them, or both, must be present
  - `try/catch`
  - `try/finally`
  - `try/catch/finally`

``` Powershell
try{
    get-childitem HKLM:\ -erroraction stop
}
catch{
    write-warning 'Something failed'
}
```

### CATCH
Catch is the place where you deal with exceptions.
Here you decide what to do in case of fatal error - maybe some detailed logging of the error and then the most critical thing , you decide here how your code should proceed further . Meaning, is this error so fatal that I need to stop the execution, or it's a minor issue and code can proceed .

Important to realize is that this block is executed only when code in `try` block produces ***terminating error***

When your code enteres this block there is a special variable filled with the exception object accessible via `$_` or `$PSItem`

For now all you need to know that this object contains all the details about the exception, in next articles I'll dig little bit deeper into this.

So compared to previous example you can see some difference in the `catch` block. Now we display more accurate information about what happend -> ```write-warning $_.Exception.Message```

But ``` Exception.Message``` property is not the only one that can be found in this object.
You can quickly examine object by using ```Format-List``` cmdlet or if you want to see details about the object use ```Format-Custom``` and specify desired `Depth` as a parameter.

``` Powershell
$_ | Format-list
or
$_ | Format-Custom -Depth 2
```

``` Powershell
try{
    get-childitem HKLM:\ -erroraction stop
}
catch{
    write-warning 'Operationg : 1/0 failed'
    write-warning $_.Exception.Message
}
```

### FINALLY
And finally the Finally block. Usually this block is used to perform some cleanup operations, to make sure that no 'garbage' has left after , whatever you run in `try` block, but in the end it's up to you how and if you use it.

Important thing to know is that this block is executed no matter what.
Let me explain on different scenarios that can happen during execution.
  - TRY/CATCH/FINALLY
    - No error in `TRY` -> `CATCH` skipped -> `FINALLY`
    - Error in `TRY` -> `CATCH` -> `FINALLY`
  - TRY/FINALLY
    - No error in `TRY` -> `FINALLY`
    - Error in `TRY` -> unhandled exception is thrown -> `FINALLY`


## Ignore & Check
The name of this technique was used for a reason, as it suggests, you ***ignore*** and then you ***check*** something. 

There are two possible approaches. Let's take a look.

### 1

- `ignore` - erroraction is in this case set to ***silentlycontinue*** , you can use any other than the ***stop*** for this technique to work.
- `check`  - As you can see there is a new parameter used in this example : `ErrorVariable` . 
By Specifying this parameter we tell to PowerShell to hide the error from user and instead save it to the variable called `failed`. Then, the next step is to check if the ```$failed``` variable contains anything, if yes we can deal with the error.

  - notice how the variable is specified -> it is the name without the dollar sign !

``` PowerShell
$result = get-childitem HKLM:\ -erroraction Silentlycontinue -ErrorVariable failed

if ($failed){
    # handle error
}
```
Wondering how to get the details of an error like we did in the `catch` block previously ? .
I assume you've already noticed that powershell has number of automatic variables and one of them is ```$Error```
Error variable contains all errors thrown in current session - it is an array of error objects.
Therefore if you need to know what was the last error just access the first item in array -> ```$Error[0]```

### 2

- `ignore` - erroraction is in this case set to ***silentlycontinue*** , you can use any other than the ***stop*** for this technique to work.

- `check`  - this time we do not fill any *errorvariable* , instead we check if automatic variable ```$Error``` contains anything. If yes, it means that some error occured and we can deal with it.
  - In this case it's very important to clear the ```$Error``` variable after you handle errors, otherwise next time you would perfom such check, even though everything would be ok, the ```$Error``` variable would not be empty , causing fake failures. -> done by the line ```$Error.Clear() ```
    - Instead of clearing the ```$Error``` variable , you could count number of errors, and if the count of errors in ```$Error``` variable is higher than expected then you know there is some new error.

``` PowerShell
$result = get-childitem HKLM:\ -erroraction Silentlycontinue

if($Error){
    #handle error
    $Error.Clear()
}
```