# Powershell error handling techniques

I guess every one of us got stuck at point where you needed to figure out what is the best way to handle errors in your function/script .
And of course so was I , and suprisingly not just once, but many times, because as it sooner or later always turnes out you can't always use the same technique, or at least it's not always the best possible approach.

``` Powershell
function test-function {
    write-output 'test'
}
```