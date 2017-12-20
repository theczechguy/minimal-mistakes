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
    1/0
}
catch{
    write-warning 'Something failed'
}
```

### CATCH
Catch is the place where you deal with exceptions.
Here you decide what to do in case of fatal error - maybe some detailed logging of the error and then the most critical thing , you decide here how your code should proceed further . Meaning, is this error so fatal that I need to stop the execution, or it's a minor issue and code can proceed .

When your code enteres this block there is a special variable filled with the exception object accessible via `$_` or `$PSItem`

For now all you need to know that this object contains all the details about the exception, in next articles I'll dig little bit deeper into this.

So compared to previous example you can see some difference in the `catch` block. Now we display more accurate information about what happend -> ```write-warning $_.Exception.Message```

But ``` Exception.Message``` property is not the only one that can be found in this object.
You can quickly examine object by using ```Format-List``` cmdlet or if you want to see details about the object use ```Format-Custom``` and specify desired `Depth` as a parameter.

``` Powershell
$_ | Format-list

$_ | Format-Custom -Depth 2
```

``` Powershell
try{
    1/0
}
catch{
    write-warning 'Operationg : 1/0 failed'
    write-warning $_.Exception.Message
}
```

### FINALLY