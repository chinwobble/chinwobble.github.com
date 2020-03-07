---
layout: post
category: scripting
tags: [pwsh, powershell, scripting]
---

{% include JB/setup %}

having recently watched an [extremely cheesy video on powershell 7](https://www.youtube.com/watch?v=at_MagcYK5M)
I got interested and decided to learn a few things about powershell.
I felt that scripting and powershell in particular was a knowledge gap of mine so this was a great reminder to do so.
My goal was to learn enough to help me setup a system where i could:

- Create re-usable scripts and run them anywhere in the command line
- Autoamte common development tasks like looking up an id / name in a database and getting the associated email
- Get ad-hoc tasks done quicker

The following is some of the highlights

## Powershell is now cross platform and runs on Mac and Linux

Beginning with Powershell 6 Core, Powershell runs on windows, Linux and Mac.
This is useful to me since I somehow find myself working across various operating systems either developing for iPhone on mac,
or setting up build scripts on a linux CI environment
or running web applications on a linux virtual machine.
In the past the lack of cross platform was one way a rationalised not bothering to learn powershell.
The new cross platform process name is `pwsh`. `pwsh.exe` on Windows.

## Simple powershell modules are easy to setup

I wanted a simple set of functions I could easily run from anywhere.
To do this in powershell you can create a `module`.
This is a cohesive library of functions that perform common tasks
and are usually stored in the same folder.

You can set this up yourself like this:

```
C:\users\{username}\
|- documents
   |- powershell # this is the new folder for powershell cor
       |- modules
          |- module1
            |- module1.psm1
            |- Get-UsefulInfo.ps1
            |- Shared.ps1
          |- module2
            |- module2.psm1
```

`C:\users\{username}\documents\powershell\modules\` is a special folder that powershell looks for modules (amongst others).

A `.psm1` is a script module file that can be used to export functions, variables, cmdlets and alises.
A simple `.psm1` could look like this:

```powershell
Get-ChildItem -Path $PSScriptRoot\*.ps1 | Foreach-Object { . $_.FullName }

Export-ModuleMember -Alias *
Export-ModuleMember -Function 'Get-MyData'
```

This module file simply uses dot loading to load all the ps1 files in module folder.
Then you can expose all your functions one by one.

## Powershell pipelines are easy to setup

Powershell can deal with structured objects.
These objects can be piped between commands.
This makes it easy to create pipelines of functionality.

The best tool so far I've found to author powershell scripts is the [powershell VS Code plugin](https://marketplace.visualstudio.com/items?itemName=ms-vscode.PowerShell).

Among the most useful benefit is the `ex-cmdlet` and `cmdlet` snippets.
These help setup all the boilerplate and access all features of powershell like aliases, strongly typed parameters, input validation, etc.
Personally I found this helpful since the powershell documentation is a bit of a mess.
The docs have lots of useful information but it is not structured in a way that I expect.

## Powershell is much easier to write than I expected

At one point I had completely written off powershell since it didn't have the `&&` operator
that allowed two commands to be run consecutively.
Further unlike bash where everything in stdout is just text, handling objects is much eaier.

After trying out some real examples like calling web-services, getting tokens I found it easy glue a few things together to get my tasks done.

See the below example:

```powershell
# shared.ps1
function Get-Auth {
    $tokenednpoint = "https://example.com/oauth2/token"

    $auth = Invoke-RestMethod `
        -Uri $tokenednpoint `
        -Method Post `
        -Form @{
            grant_type    = 'password'
            username      = 'xxxxxx'
            password      = 'xxxxxx'
            client_id     = 'xxxxxx'
            client_secret = 'xxxxxx'
        }
    $auth
    $auth.access_token
}

# Get-UsefulInfo.ps1
function Get-UsefulInfo {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [String]$id
    )

    begin {
        $auth = Get-Auth
        write-host begin
    }

    process {
        $url = "/someservice/${id}"
        Write-Debug $url
        $result = Invoke-RestMethod `
            -Uri $url `
            -Method Get `
            -Headers @{ Authorization = 'Bearer ' + $auth.access_token }

        $result.partyList `
        | sort-object -unique -property userId
    }

    end {
        write-host ended
    }
}


new-alias -Name glud -Value Get-UsefulInfo
```

The above code tries to demonstrate how straightforward it is
to compose scripts that can:

- destructure, sort and filter nested data
- call authenticated services
- piped into other commands
- automatically get `-Debug` and `-Verbose` switches that help find issues

The same thing can be done in bash. However you cannot cleanly parse json out of the box in bash.

The above looks like a lot of code but much of it was scaffolded thanks to vscode snippets.

## Summary

Overall I think it was worthwhile spending some time to learn powershell
in the hopes of saving time in the future.
