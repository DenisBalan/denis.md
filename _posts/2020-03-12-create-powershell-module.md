---
layout: post
title:  "Powershell Module: unrevealing process of developing, testing and publishing"
date:   2020-03-12 00:00:00 +02
image: /assets/images/eur-to-mdl-rate-graph.png
categories: powershell
tags: 
  - powershell 
  - development
  - bnm
---
> *tl;dr* How to develop and publish basic PowerShell 7 module with PSScriptAnalyzer, Pester, Plaster, PSDeploy, BuildHelpers, InvokeBuild invoking CI build on AppVeyor that test the module to not break anything, and CD pipeline to publish to PowershellGallery.

Daily experience with PowerShell implies itself dealing with scripts, being it plain scripts, grouped cmdlets or whole modules. Properly packaging your code into a reusable module is one way of achieving [separation of concers].

> Module that extracts information from [National Bank of Moldova] (Banca Națională a Moldovei - BNM) about today's exchange rate for foreign currencies for integrating into application backends, excel sheets, and more.

## What is a PowerShell module

A module is a set of files containing scripts, resources, dependent assemblies that you want to include in the project. As docs from [msdn] says, there are four different module types each one suitable for certain needs:

### Script Modules

A file with .psm1 extension that contain valid PowerShell code, think of this as packaging your set of utilities, functions, workflows, and other.

### Binary Modules

Assembly that contains compiled code, this type of module that operates at a lower level in comparison to script module, allowing developers to take advantage of full .NET framework features (ex TPL).

### Manifest Modules

Contains meta-information about the module, such as author, requirements, function exports. In this type of module, you can also use the features, such as the ability to load up dependent assemblies or automatically run certain pre-processing scripts.

### Dynamic Modules

Modules that were created in runtime by `New-Module` cmdlet, cannot be accessed by `Get-Module`, they do not need manifests, and most likely no permanent folder to store themself.

For this project i choose script module.

## Know your tools

Building a module nowadays without additional tools, ie "from scratch" is a messy task, below are several tools that helps automate steps.

| Tool        | Used for           | Stage  |
| ------------- |:-------------:| -----:|
| [Plaster]      | module templating      |   develop |
| [BuildHelpers] | dependency restorer      |    build |
| [InvokeBuild] | build automation      |    build |
| [PSScriptAnalyzer]      | static code checker | testing |
| [Pester]      | testing and mocking | testing |
| [Polaris] | HTTP framework for PowerShell      |    testing |
| [PSDeploy] | deployment to AppVeyor      |    deploy |

## Module description

To install the module open `pwsh` console and type following

```powershell
Install-Module BNMoldovaCurrency -scope CurrentUser;
Import-Module BNMoldovaCurrency
```

Below is a table explaining each file's responsability

```powershell
Get-Command -Module BNMoldovaCurrency |% { Get-Help $_ | select name, synopsis, @{ n = 'description'; e = { $_.description[0].Text } } }
```

| Name | Synopsis | Description
| ------------- | --- | ----- |
| Get-BNMConfig | Get the default configuration for BNM. | Get the default configuration for Banca Nationala of Moldova. |
| Get-BNMCurrency | Gets BNM currency for specified date. | Invokes HTTP GET method to the BNM server for reading exchange rates based on configuration file. |
| Save-BNMCurrency | Saves BNM currency for specified date. | Uses Get-BNMCurrency to get data. |
| Set-BNMConfig | Set the default configuration for Banca Nationala of Moldova. | Set the default configuration for BNM server. |

Manifest file contains following basic properties

```powershell
# Author of this module
Author = 'Denis Balan'

# Company or vendor of this module
CompanyName = 'Unknown'

# Copyright statement for this module
Copyright = '(c) 2020 Denis Balan. All rights reserved.'

# Description of the functionality provided by this module
Description = 'Module for interacting with BNM exchange rates.'

# Minimum version of the Windows PowerShell engine required by this module
PowerShellVersion = '5.1'
```

## Testing

There are several tests in `tests` folder:

- [1.BNMoldovaCurrency.Tests.ps1] - tests file integrity with PSScriptAnalyzer
- [2.Help.Tests.ps1] - requires functions to have examples, description and synopsis
- [Get-BNMConfig.Tests.ps1] - tests `Set-BNMConfig` and `Get-BNMConfig` to return proper builded endpoint
- [Get-BNMCurrency.Tests.ps1] - tests xml parsing of exchange rates to have predefined schema (`Polaris` used as a proxy for mocking BNM server)

## Summary

To build it locally

```powershell
git clone https://github.com/DenisBalan/BNMoldovaCurrency
cd BNMoldovaCurrency && ./build.ps1
```
![building powershell module](/assets/images/building-bnmoldovacurrency-pwsh-module.gif)

### Use cases

#### Generate CSV of EUR for last several days

```powershell
$data = $(100..0 |% {
    $date = "$('{0:dd.MM.yyyy}' -f $(get-date).AddDays(-$_))"
    Set-BNMConfig -Endpoint https://bnm.md/ro/official_exchange_rates -Params @{get_xml = 1; date = $date };
    [pscustomobject]@{date=$date; value = [float]$(Get-BNMCurrency |? { $_.charcode -match 'eur' } |% value)}
});
```

Having `$data` in memory, we can export it to CSV

```powershell

$data | Export-Csv ./rata_eur_mdl.csv -NoTypeInformation
```

Example

| date | value |
| -- | -- |
| 09.03.2020 | 19.7466 |
| 10.03.2020 | 19.8699 |
| 11.03.2020 | 19.7153 |
| 12.03.2020 | 19.7277 |
| ... |

Or even plot it in the terminal

```powershell
# this will install Graphical module
Install-Module Graphical -scope CurrentUser;
Import-Module Graphical
$days100 = $data |% { $_.value * 100; }
Show-Graph -Datapoints $days100 -GraphTitle '100 EUR -> MDL'  
```

You should see something like this
![100 EUR to MDL in time](/assets/images/eur-to-mdl-rate-graph.png)

Writing PowerShell module is not a thing you should scary, but instead enjoy all steps.

Feel free to fork it via GitHub - [BNMoldovaCurrency].

[National Bank of Moldova]: http://bnm.md/en
[separation of concers]: https://en.wikipedia.org/wiki/Separation_of_concerns
[msdn]: https://docs.microsoft.com/en-us/powershell/scripting/developer/module/understanding-a-windows-powershell-module
[Pester]: https://github.com/pester/Pester
[PSDeploy]: https://github.com/RamblingCookieMonster/PSDeploy
[Plaster]: https://github.com/PowerShell/Plaster
[BuildHelpers]: https://github.com/RamblingCookieMonster/BuildHelpers
[InvokeBuild]: https://github.com/nightroman/Invoke-Build
[Polaris]: https://github.com/PowerShell/Polaris
[PSScriptAnalyzer]: https://github.com/PowerShell/PSScriptAnalyzer
[BNMoldovaCurrency]: https://github.com/DenisBalan/BNMoldovaCurrency
[1.BNMoldovaCurrency.Tests.ps1]: https://github.com/DenisBalan/BNMoldovaCurrency/blob/master/tests/1.BNMoldovaCurrency.Tests.ps1
[2.Help.Tests.ps1]: https://github.com/DenisBalan/BNMoldovaCurrency/blob/master/tests/2.Help.Tests.ps1
[Get-BNMConfig.Tests.ps1]: https://github.com/DenisBalan/BNMoldovaCurrency/blob/master/tests/Get-BNMConfig.Tests.ps1
[Get-BNMCurrency.Tests.ps1]: https://github.com/DenisBalan/BNMoldovaCurrency/blob/master/tests/Get-BNMCurrency.Tests.ps1
