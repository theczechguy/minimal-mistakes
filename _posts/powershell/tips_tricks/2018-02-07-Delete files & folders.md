# How to delete missbehaving folders with Powershell

## What is missbehaving folder ?
Let's say you have complex folder structure , the names of folder are just way too long. At some point the path is so long that windows explorer is unable to access the rest of inner folders , and unable to delete the whole structure, because the path is too long.

What "too long path" means ?
Windows has max path lenght limit (***MAX_PATH***) .
Absolute limit is 260 characters . But available for use is actually 256 characters.

Read more about this here

`https://msdn.microsoft.com/en-us/library/windows/desktop/aa365247%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396#maxpath`

## What is the problem ?
The actuall problem is that standard tools , either CMD or Powershell, are unable to handle paths exceeding ***MAX_PATH*** .
But tools like TotalCommander are able to work with them, so it's got to be possible, right ?

It is indeed , it just requires small trick mentioned in the msdn article abowe. And that is prepending path with ***\\\\?\\*** .

So let's take a look on possible implementation.
This is how I've implemented it in our environment , not saying this is the only way :) .

## Implementation
Our developers tend to create complex folder structure's which exceed the ***MAX_PATH*** limit, and as usual they don't cleanup their stuff after themselves . So it's our job to clean it, otherwise they would kill our HDD's soon.

Our case :
- paths too long
- some files are hiden or readonly

There are reasons why I haven't use native Powershell cmdletls .
FOr some reason Powershell , even Version 5,  contains bug in `Remove-Item` cmdlet .

IF you try to run

` Powershell
    Remove-Item -Force -Recurse -Path $directoryPath
`

You will get following message : `Cannot remove item. The directory is not empty.`

It is known bug and officialy is recommended to first use `Get-Childitem` with `recurse` parameter and pipe the results into `Remove-Item` .
This works fine , until you need to perform this stuff automatically.
It asks you to confirm the deletion , which is unacceptable when you need to automate stuff . I've read suggestions to use `-confirm:$false` but I had no luck here so I quickly realized I need to take more low level way and that is the almighty .NET .

It's important to note that this technique is valid only for systems Windows Version 6.3 and higher - Server 2012 R2 & Windows 8.1 .

I will describe here just most important steps , not the whole function.

Function is called ***Remove-FolderContent***
- $Path is my input parameter


- check if path starts with the special characters which enable long path support, if not prepend it.

```Powershell
    if(!($Path.StartsWith('\\?\'))){
        $Path = '\\?\{0}' -f $Path    
    }
```

- enumerate subdirectories  using method from `[System.IO.Directory]` class.
  - according to my tests this is the fastest method to list the directories available in this class
  - if any subdirs are found the function calls itself on the subfolder and this repeats until no subfolders are found

``` Powershell
$subDirs = [System.IO.Directory]::EnumerateDirectories($Path)
    if ($subDirs -ne $Null) {
        foreach ($dir in $subDirs) {
            Remove-FolderContent -Path $dir
        }
    }
```

- now there are no folders so let's try to get list of files, again using `[System.IO.Directory]` class
  - then it loops through the list of found files and changes their attributes = removing ***readonly*** and ***hidden*** which would cause problems when attempting to delete the file .
  - if attributes are cleared it tries to delete the file .
    - exceptions are here not considered as fatal , they are just displayed as warnings
  - one by one, the entire folder is cleared of all files.

``` Powershell
$subFiles = [System.IO.Directory]::EnumerateFiles($Path)
    foreach ($file in $subFiles) {
        try {
            [System.IO.File]::SetAttributes($file , [System.IO.FileAttributes]::Normal)
            [System.IO.File]::Delete($file)
        }
        catch {
            Write-Warning -Message 'Unable to delete : {0} . Error : {1}' -f $file , $_.exception.message
        }
    }
```

- folder is cleared , let's try to delete it
  - this is important , it's not possible to delete folder that is not empty, that is why I first try to clear it.
``` Powershell
try{
    [System.IO.Directory]::Delete($Path,$true)
}
catch{
    Write-Warning -Message 'Unable to delete : {0} . Error : {1}' -f $file , $_.exception.message
}
```

And this way the functions loops through all subfolders and tries to clear and remove them, one by one .

From performance point of view it is not the fastest way to delete folders , but it can
handle this very special case.

Performance could be slightly improved by pipening results of EnumerateFiles and EnumerateDirectories. The speciality of these methods is that they start providing results immediatelly , it does not wait to first collect all data and then return results.


``` Powershell

function Remove-FolderContent {
    [CmdletBinding()]
    param (
        # Specifies a path to one or more locations.
        [Parameter(Mandatory=$true)]
        [ValidateNotNullOrEmpty()]
        [string]
        $Path,

        # delete the top folder as well ?
        [Parameter(Mandatory = $false)]
        [switch]
        $DeleteRootFolder
    )
    process {
        $originalPath = $Path
        try { 
            if(!($Path.StartsWith('\\?\'))){
                $Path = '\\?\{0}' -f $Path    
            }
            
            $subDirs = [System.IO.Directory]::EnumerateDirectories($Path)
            if ($subDirs -ne $Null) {
                foreach ($dir in $subDirs) {
                    Remove-FolderContent -Path $dir
                }
            }
    
            $subFiles = [System.IO.Directory]::EnumerateFiles($Path)
            foreach ($file in $subFiles) {
                try {
                    [System.IO.File]::SetAttributes($file , [System.IO.FileAttributes]::Normal)
                    [System.IO.File]::Delete($file)
                }
                catch {
                    Write-Warning -Message 'Unable to delete : {0} . Error : {1}' -f $file , $_.exception.message
                }
            }
    
            $trimmedPath = $Path.TrimStart('\\?\')
            if($originalPath -eq $trimmedPath){
                if ($DeleteRootFolder.IsPresent) {
                    [System.IO.Directory]::Delete($Path,$true)
                }
            } else {
                [System.IO.Directory]::Delete($Path,$true)
            }
        }
        catch {
            Write-Warning -Message 'Unable to delete : {0} . Error : {1}' -f $Path , $_.exception.message
        }
    }
}
```