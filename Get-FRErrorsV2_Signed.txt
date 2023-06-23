<#
.SYNOPSIS
	Find Folder Redirection errors on computers in Microsoft Active Directory.
.DESCRIPTION
	By default, builds a list of all computers where "Server" is in the 
	OperatingSystem property unless the ComputerName or InputFile 
	parameter is used.

	Process each server looking for Folder Redirection errors (Event ID 502) 
	within, by default, the last 30 days.
	
	Builds a list of unique user names and servers unable to process.
	
	Creates the two text files, by default, in the folder where the script is run.
	
	Optionally, can specify the output folder.

	Unless the InputFile parameter is used, needs the ActiveDirectory module.
	
	The script has been tested with PowerShell versions 2, 3, 4, 5, and 5.1.
	The script has been tested with Microsoft Windows Server 2008 R2, 2012, 
	2012 R2, and 2016 and Windows 10 Creators Update.
.PARAMETER ComputerName
	Computer name used to restrict the computer search.
	Script surrounds ComputerName with "*".
	
	For example, if "RDS" is entered, the script uses "*RDS*".
	
	This allows the script to reduce the number of servers searched.
	
	If both ComputerName and InputFile are used, ComputerName is used to filter
	the list of computer names read from InputFile.
	
	Alias is CN
.PARAMETER InputFile
	Specifies the optional input file containing computer account names to search.
	
	Computer account names can be either the NetBIOS or Fully Qualified Domain Name.
	
	ServerName and ServerName.domain.tld are both valid.
	
	If both ComputerName and InputFile are used, ComputerName is used to filter
	the list of computer names read from InputFile.
	
	The computer names contained in the input file are not validated.
	
	Using this parameter causes the script to not check for or load the ActiveDirectory module.
	
	Alias is IF
.PARAMETER OrganizationalUnit
	Restricts the retrieval of computer accounts to a specific OU tree. 
	Must be entered in Distinguished Name format. i.e. OU=XenDesktop,DC=domain,DC=tld. 
	
	The script retrieves computer accounts from the top level OU and all sub-level OUs.
	
	Alias OU
.PARAMETER StartDate
	Start date, in MM/DD/YYYY format.
	Default is today's date minus 30 days.
.PARAMETER EndDate
	End date, in MM/DD/YYYY HH:MM format.
	Default is today's date.
.PARAMETER Folder
	Specifies the optional output folder to save the output reports. 
.EXAMPLE
	PS C:\PSScript > .\Get-FRErrorsV2.ps1
	
.EXAMPLE
	PS C:\PSScript > .\Get-FRErrorsV2.ps1 -StartDate "04/01/2017" -EndDate "04/02/2017" 
	
	Will return all Folder Redirection errors from "04/01/2017" through "04/02/2017".
.EXAMPLE
	PS C:\PSScript > .\Get-FRErrorsV2.ps1 -StartDate "04/01/2017" -EndDate "04/01/2017" 
	
	Will return all Folder Redirection errors from "04/01/2017" through "04/01/2017".
.EXAMPLE
	PS C:\PSScript > .\Get-FRErrorsV2.ps1 -Folder \\FileServer\ShareName

	Retrieves all Folder Redirection errors for the last 30 days.
	
	Output files will be saved in the path \\FileServer\ShareName
.EXAMPLE
	PS C:\PSScript > .\Get-FRErrorsV2.ps1 -ComputerName XEN
	
	Retrieves all Folder Redirection errors for the last 30 days.
	
	The script will only search computers with "XEN" in the DNSHostName.
	
.EXAMPLE
	PS C:\PSScript > .\Get-FRErrorsV2.ps1 -ComputerName RDS -Folder \\FileServer\ShareName -StartDate "05/01/2017" -EndDate "05/15/2017"
	
	The script will only search computers with "RDS" in the DNSHostName.
	
	Output file will be saved in the path \\FileServer\ShareName

	Will return all Folder Redirection errors from "05/01/2017" through "05/15/2017".
	
.EXAMPLE
	PS C:\PSScript > .\Get-FRErrorsV2.ps1 -ComputerName CTX -Folder \\FileServer\ShareName -StartDate "05/01/2017" -EndDate "05/07/2017" -InputFile c:\Scripts\computers.txt
	
	The script will only search computers with "CTX" in the entries contained in the computers.txt file.

	Output file will be saved in the path \\FileServer\ShareName

	Will return all Folder Redirection errors from "05/01/2017" through "05/07/2017".
	
	InputFile causes the script to not check for or use the ActiveDirectory module.

.EXAMPLE
	PS C:\PSScript > .\Get-FRErrorsV2.ps1 -OU "ou=RDS Servers,dc=domain,dc=tld"
	
	Finds Folder Redirection errors in all computers found in the "ou=RDS Servers,dc=domain,dc=tld" OU tree.
	
	Retrieves all Folder Redirection errors for the last 30 days.
	
.INPUTS
	None.  You cannot pipe objects to this script.
.OUTPUTS
	No objects are output from this script.  This script creates two text files.
.NOTES
	NAME: Get-FRErrorsV2.ps1
	VERSION: 2.00
	AUTHOR: Carl Webster
	LASTEDIT: May 18, 2017
#>


#Created by Carl Webster, CTP and independent consultant 01-Mar-2016
#webster@carlwebster.com
#@carlwebster on Twitter
#http://www.CarlWebster.com

[CmdletBinding(SupportsShouldProcess = $False, ConfirmImpact = "None", DefaultParameterSetName = "Default") ]

Param(
	[parameter(ParameterSetName="Default",Mandatory=$False)] 
	[Datetime]$StartDate = ((Get-Date -displayhint date).AddDays(-30)),

	[parameter(ParameterSetName="Default",Mandatory=$False)] 
	[Datetime]$EndDate = (Get-Date -displayhint date),
	
	[parameter(ParameterSetName="Default",Mandatory=$False)] 
	[Alias("CN")]
	[string]$ComputerName,
	
	[parameter(ParameterSetName="Default",Mandatory=$False)] 
	[Alias("IF")]
	[string]$InputFile="",
	
	[parameter(ParameterSetName="Default",Mandatory=$False)] 
	[Alias("OU")]
	[string]$OrganizationalUnit="",
	
	[parameter(ParameterSetName="Default",Mandatory=$False)] 
	[string]$Folder=""
	
	)

Set-StrictMode -Version 2
	
Write-Host "$(Get-Date): Setting up script"

If(![String]::IsNullOrEmpty($InputFile))
{
	Write-Host "$(Get-Date): Validating input file"
	If(!(Test-Path $InputFile))
	{
		Write-Error "Input file specified but $InputFile does not exist. Script cannot continue."
		Exit
	}
}

If($Folder -ne "")
{
	Write-Host "$(Get-Date): Testing folder path"
	#does it exist
	If(Test-Path $Folder -EA 0)
	{
		#it exists, now check to see if it is a folder and not a file
		If(Test-Path $Folder -pathType Container -EA 0)
		{
			#it exists and it is a folder
			Write-Host "$(Get-Date): Folder path $Folder exists and is a folder"
		}
		Else
		{
			#it exists but it is a file not a folder
			Write-Error "Folder $Folder is a file, not a folder. Script cannot continue"
			Exit
		}
	}
	Else
	{
		#does not exist
		Write-Error "Folder $Folder does not exist.  Script cannot continue"
		Exit
	}
}

#test to see if OrganizationalUnit is valid
If(![String]::IsNullOrEmpty($OrganizationalUnit))
{
	Write-Host "$(Get-Date): Validating Organnization Unit"
	try 
	{
		$results = Get-ADOrganizationalUnit -Identity $OrganizationalUnit
	} 
	
	catch
	{
		#does not exist
		Write-Error "Organization Unit $OrganizationalUnit does not exist.`n`nScript cannot continue`n`n"
		Exit
	}	
}

If($Folder -eq "")
{
	$pwdpath = $pwd.Path
}
Else
{
	$pwdpath = $Folder
}

If($pwdpath.EndsWith("\"))
{
	#remove the trailing \
	$pwdpath = $pwdpath.SubString(0, ($pwdpath.Length - 1))
}

Function Check-LoadedModule
#Function created by Jeff Wouters
#@JeffWouters on Twitter
#modified by Michael B. Smith to handle when the module doesn't exist on server
#modified by @andyjmorgan
#bug fixed by @schose
#bug fixed by Peter Bosen
#This Function handles all three scenarios:
#
# 1. Module is already imported into current session
# 2. Module is not already imported into current session, it does exists on the server and is imported
# 3. Module does not exist on the server

{
	Param([parameter(Mandatory = $True)][alias("Module")][string]$ModuleName)
	#following line changed at the recommendation of @andyjmorgan
	$LoadedModules = Get-Module |% { $_.Name.ToString() }
	#bug reported on 21-JAN-2013 by @schose 
	
	[string]$ModuleFound = ($LoadedModules -like "*$ModuleName*")
	If($ModuleFound -ne $ModuleName) 
	{
		$module = Import-Module -Name $ModuleName -PassThru -EA 0
		If($module -and $?)
		{
			# module imported properly
			Return $True
		}
		Else
		{
			# module import failed
			Return $False
		}
	}
	Else
	{
		#module already imported into current session
		Return $True
	}
}

#only check for the ActiveDirectory module if an InputFile was not entered
If([String]::IsNullOrEmpty($InputFile) -and !(Check-LoadedModule "ActiveDirectory"))
{
	Write-Host "Unable to run script, no ActiveDirectory module"
	Exit
}

Function ProcessComputer 
{
	Param([string]$TmpComputerName)
	
	If(Test-Connection -ComputerName $TmpComputerName -quiet -EA 0)
	{
		try
		{
			Write-Host "$(Get-Date): `tRetrieving Folder RedirectionGroup Policy event log entries"
			$Errors = Get-EventLog -logname application `
			-computername $TmpComputerName `
			-source "Microsoft-Windows-Folder Redirection" `
			-entrytype "Error" `
			-after $StartDate `
			-before $EndDate `
			-EA 0
		}
		
		catch
		{
			Write-Host "$(Get-Date): `tServer $($TmpComputerName) had error being accessed"
			Out-File -FilePath $Filename2 `
			-Append -InputObject "Server $($TmpComputerName) had error being accessed $(Get-Date)"
			Continue
		}
		
		If($? -and $Null -ne $Errors)
		{
			$Errors = $Errors | Select UserName | Sort UserName -Unique
			
			$Script:ErrorCount = ($Errors | measure-object).count
			If(!($Errors -is [Array]))
			{
				#force the singleton to an array
				[array]$Errors = $Errors
			}

			$Script:TotalErrorCount += $Script:ErrorCount
			
			Write-Host "$(Get-Date): `t$($Script:ErrorCount) Folder Redirection errors found"
			$Script:AllMatches += $Errors
			[array]$Script:AllMatches = $Script:AllMatches | Sort UserName -Unique
			
			$ErrorArrayCount = ($Script:AllMatches | measure-object).count
			Write-Host "$(Get-Date): `t`t$($ErrorArrayCount) total Folder Redirection errors found"
		}
		Else
		{
			Write-Host "$(Get-Date): `tNo Folder Redirection errors found"
		}
	}
	Else
	{
		Write-Host "`tComputer $($TmpComputerName) is not online"
		Out-File -FilePath $Filename2 -Append -InputObject "Computer $($TmpComputerName) was not online $(Get-Date)"
	}
}

$startTime = Get-Date

[string]$Script:FileName1 = "$($pwdpath)\FolderRedirectionErrors.txt"
[string]$Script:FileName2 = "$($pwdpath)\FROfflineServers.txt"
#make sure files contain the current date only
Out-File -FilePath $Script:FileName1 -InputObject (Get-Date)
Out-File -FilePath $Script:FileName2 -InputObject (Get-Date)

If(![String]::IsNullOrEmpty($ComputerName) -and [String]::IsNullOrEmpty($InputFile))
{
	#computer name but no input file
	Write-Host "$(Get-Date): Retrieving list of computers from Active Directory"
	$testname = "*$($ComputerName)*"
	If(![String]::IsNullOrEmpty($OrganizationalUnit))
	{
		$Computers = Get-AdComputer -filter {DNSHostName -like $testname} -SearchBase $OrganizationalUnit -SearchScope Subtree -properties DNSHostName, Name -EA 0 | Sort Name
	}
	Else
	{
		$Computers = Get-AdComputer -filter {DNSHostName -like $testname} -properties DNSHostName, Name -EA 0 | Sort Name
	}
}
ElseIf([String]::IsNullOrEmpty($ComputerName) -and ![String]::IsNullOrEmpty($InputFile))
{
	#input file but no computer name
	Write-Host "$(Get-Date): Retrieving list of computers from Input File"
	$Computers = Get-Content $InputFile
}
ElseIf(![String]::IsNullOrEmpty($ComputerName) -and ![String]::IsNullOrEmpty($InputFile))
{
	#both computer name and input file
	Write-Host "$(Get-Date): Retrieving list of computers from Input File"
	$testname = "*$($ComputerName)*"
	$Computers = Get-Content $InputFile | ? {$_ -like $testname}
}
Else
{
	Write-Host "$(Get-Date): Retrieving list of computers from Active Directory"
	If(![String]::IsNullOrEmpty($OrganizationalUnit))
	{
		$Computers = Get-AdComputer -filter {OperatingSystem -like "*server*"} -SearchBase $OrganizationalUnit -SearchScope Subtree -properties DNSHostName, Name -EA 0 | Sort Name
	}
	Else
	{
		$Computers = Get-AdComputer -filter {OperatingSystem -like "*server*"} -properties DNSHostName, Name -EA 0 | Sort Name
	}
}

If($? -and $Null -ne $Computers)
{
	If($Computers -is [array])
	{
		Write-Host "Found $($Computers.Count) servers to process"
	}
	Else
	{
		Write-Host "Found 1 server to process"
	}

	$Script:AllMatches = @()
	[int]$Script:TotalErrorCount = 0
	[int]$Script:ErrorCount = 0

	If(![String]::IsNullOrEmpty($InputFile))
	{
		ForEach($Computer in $Computers)
		{
			$TmpComputerName = $Computer
			Write-Host "Testing computer $($TmpComputerName)"
			ProcessComputer $TmpComputerName
		}
	}
	Else
	{
		ForEach($Computer in $Computers)
		{
			$TmpComputerName = $Computer.DNSHostName
			Write-Host "Testing computer $($TmpComputerName)"
			ProcessComputer $TmpComputerName
		}
	}

	$Script:AllMatches = $Script:AllMatches | Sort UserName -Unique
	
	Write-Host "$(Get-Date): Output Folder Redirection errors to file"
	If($Script:TotalErrorCount -gt 0)
	{
		Out-File -FilePath $Script:FileName1 -Append -InputObject $Script:AllMatches
	}
	Else
	{
		Out-File -FilePath $Script:FileName1 -Append -InputObject "No Folder Redirection errors were found"
	}

	If(Test-Path "$($Script:FileName2)")
	{
		Write-Host "$(Get-Date): $($Script:FileName2) is ready for use"
	}
	If(Test-Path "$($Script:FileName1)")
	{
		Write-Host "$(Get-Date): $($Script:FileName1) is ready for use"
	}

	Write-Host "$(Get-Date): Script started: $($StartTime)"
	Write-Host "$(Get-Date): Script ended: $(Get-Date)"
	$runtime = $(Get-Date) - $StartTime
	$Str = [string]::format("{0} days, {1} hours, {2} minutes, {3}.{4} seconds", `
		$runtime.Days, `
		$runtime.Hours, `
		$runtime.Minutes, `
		$runtime.Seconds,
		$runtime.Milliseconds)
	Write-Host "$(Get-Date): Elapsed time: $($Str)"
	$runtime = $Null
}
Else
{
	If(![String]::IsNullOrEmpty($ComputerName) -and [String]::IsNullOrEmpty($InputFile))
	{
		#computer name but no input file
		Write-Host "Unable to retrieve a list of computers from Active Directory"
	}
	ElseIf([String]::IsNullOrEmpty($ComputerName) -and ![String]::IsNullOrEmpty($InputFile))
	{
		#input file but no computer name
		Write-Host "Unable to retrieve a list of computers from the Input File $InputFile"
	}
	ElseIf(![String]::IsNullOrEmpty($ComputerName) -and ![String]::IsNullOrEmpty($InputFile))
	{
		#computer name and input file
		Write-Host "Unable to retrieve a list of matching computers from the Input File $InputFile"
	}
	Else
	{
		Write-Host "Unable to retrieve a list of computers from Active Directory"
	}
}

# SIG # Begin signature block
# MIIgCgYJKoZIhvcNAQcCoIIf+zCCH/cCAQExCzAJBgUrDgMCGgUAMGkGCisGAQQB
# gjcCAQSgWzBZMDQGCisGAQQBgjcCAR4wJgIDAQAABBAfzDtgWUsITrck0sYpfvNR
# AgEAAgEAAgEAAgEAAgEAMCEwCQYFKw4DAhoFAAQUBLHvJs6+H1CVkt/R69hjFdkI
# ki6gghtxMIIDtzCCAp+gAwIBAgIQDOfg5RfYRv6P5WD8G/AwOTANBgkqhkiG9w0B
# AQUFADBlMQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYD
# VQQLExB3d3cuZGlnaWNlcnQuY29tMSQwIgYDVQQDExtEaWdpQ2VydCBBc3N1cmVk
# IElEIFJvb3QgQ0EwHhcNMDYxMTEwMDAwMDAwWhcNMzExMTEwMDAwMDAwWjBlMQsw
# CQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3d3cu
# ZGlnaWNlcnQuY29tMSQwIgYDVQQDExtEaWdpQ2VydCBBc3N1cmVkIElEIFJvb3Qg
# Q0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCtDhXO5EOAXLGH87dg
# +XESpa7cJpSIqvTO9SA5KFhgDPiA2qkVlTJhPLWxKISKityfCgyDF3qPkKyK53lT
# XDGEKvYPmDI2dsze3Tyoou9q+yHyUmHfnyDXH+Kx2f4YZNISW1/5WBg1vEfNoTb5
# a3/UsDg+wRvDjDPZ2C8Y/igPs6eD1sNuRMBhNZYW/lmci3Zt1/GiSw0r/wty2p5g
# 0I6QNcZ4VYcgoc/lbQrISXwxmDNsIumH0DJaoroTghHtORedmTpyoeb6pNnVFzF1
# roV9Iq4/AUaG9ih5yLHa5FcXxH4cDrC0kqZWs72yl+2qp/C3xag/lRbQ/6GW6whf
# GHdPAgMBAAGjYzBhMA4GA1UdDwEB/wQEAwIBhjAPBgNVHRMBAf8EBTADAQH/MB0G
# A1UdDgQWBBRF66Kv9JLLgjEtUYunpyGd823IDzAfBgNVHSMEGDAWgBRF66Kv9JLL
# gjEtUYunpyGd823IDzANBgkqhkiG9w0BAQUFAAOCAQEAog683+Lt8ONyc3pklL/3
# cmbYMuRCdWKuh+vy1dneVrOfzM4UKLkNl2BcEkxY5NM9g0lFWJc1aRqoR+pWxnmr
# EthngYTffwk8lOa4JiwgvT2zKIn3X/8i4peEH+ll74fg38FnSbNd67IJKusm7Xi+
# fT8r87cmNW1fiQG2SVufAQWbqz0lwcy2f8Lxb4bG+mRo64EtlOtCt/qMHt1i8b5Q
# Z7dsvfPxH2sMNgcWfzd8qVttevESRmCD1ycEvkvOl77DZypoEd+A5wwzZr8TDRRu
# 838fYxAe+o0bJW1sj6W3YQGx0qMmoRBxna3iw/nDmVG3KwcIzi7mULKn+gpFL6Lw
# 8jCCBTAwggQYoAMCAQICEAQJGBtf1btmdVNDtW+VUAgwDQYJKoZIhvcNAQELBQAw
# ZTELMAkGA1UEBhMCVVMxFTATBgNVBAoTDERpZ2lDZXJ0IEluYzEZMBcGA1UECxMQ
# d3d3LmRpZ2ljZXJ0LmNvbTEkMCIGA1UEAxMbRGlnaUNlcnQgQXNzdXJlZCBJRCBS
# b290IENBMB4XDTEzMTAyMjEyMDAwMFoXDTI4MTAyMjEyMDAwMFowcjELMAkGA1UE
# BhMCVVMxFTATBgNVBAoTDERpZ2lDZXJ0IEluYzEZMBcGA1UECxMQd3d3LmRpZ2lj
# ZXJ0LmNvbTExMC8GA1UEAxMoRGlnaUNlcnQgU0hBMiBBc3N1cmVkIElEIENvZGUg
# U2lnbmluZyBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPjTsxx/
# DhGvZ3cH0wsxSRnP0PtFmbE620T1f+Wondsy13Hqdp0FLreP+pJDwKX5idQ3Gde2
# qvCchqXYJawOeSg6funRZ9PG+yknx9N7I5TkkSOWkHeC+aGEI2YSVDNQdLEoJrsk
# acLCUvIUZ4qJRdQtoaPpiCwgla4cSocI3wz14k1gGL6qxLKucDFmM3E+rHCiq85/
# 6XzLkqHlOzEcz+ryCuRXu0q16XTmK/5sy350OTYNkO/ktU6kqepqCquE86xnTrXE
# 94zRICUj6whkPlKWwfIPEvTFjg/BougsUfdzvL2FsWKDc0GCB+Q4i2pzINAPZHM8
# np+mM6n9Gd8lk9ECAwEAAaOCAc0wggHJMBIGA1UdEwEB/wQIMAYBAf8CAQAwDgYD
# VR0PAQH/BAQDAgGGMBMGA1UdJQQMMAoGCCsGAQUFBwMDMHkGCCsGAQUFBwEBBG0w
# azAkBggrBgEFBQcwAYYYaHR0cDovL29jc3AuZGlnaWNlcnQuY29tMEMGCCsGAQUF
# BzAChjdodHRwOi8vY2FjZXJ0cy5kaWdpY2VydC5jb20vRGlnaUNlcnRBc3N1cmVk
# SURSb290Q0EuY3J0MIGBBgNVHR8EejB4MDqgOKA2hjRodHRwOi8vY3JsNC5kaWdp
# Y2VydC5jb20vRGlnaUNlcnRBc3N1cmVkSURSb290Q0EuY3JsMDqgOKA2hjRodHRw
# Oi8vY3JsMy5kaWdpY2VydC5jb20vRGlnaUNlcnRBc3N1cmVkSURSb290Q0EuY3Js
# ME8GA1UdIARIMEYwOAYKYIZIAYb9bAACBDAqMCgGCCsGAQUFBwIBFhxodHRwczov
# L3d3dy5kaWdpY2VydC5jb20vQ1BTMAoGCGCGSAGG/WwDMB0GA1UdDgQWBBRaxLl7
# KgqjpepxA8Bg+S32ZXUOWDAfBgNVHSMEGDAWgBRF66Kv9JLLgjEtUYunpyGd823I
# DzANBgkqhkiG9w0BAQsFAAOCAQEAPuwNWiSz8yLRFcgsfCUpdqgdXRwtOhrE7zBh
# 134LYP3DPQ/Er4v97yrfIFU3sOH20ZJ1D1G0bqWOWuJeJIFOEKTuP3GOYw4TS63X
# X0R58zYUBor3nEZOXP+QsRsHDpEV+7qvtVHCjSSuJMbHJyqhKSgaOnEoAjwukaPA
# JRHinBRHoXpoaK+bp1wgXNlxsQyPu6j4xRJon89Ay0BEpRPw5mQMJQhCMrI2iiQC
# /i9yfhzXSUWW6Fkd6fp0ZGuy62ZD2rOwjNXpDd32ASDOmTFjPQgaGLOBm0/GkxAG
# /AeB+ova+YJJ92JuoVP6EpQYhS6SkepobEQysmah5xikmmRR7zCCBT8wggQnoAMC
# AQICEALKvIFdDaFKh3T2QAUcJiIwDQYJKoZIhvcNAQELBQAwcjELMAkGA1UEBhMC
# VVMxFTATBgNVBAoTDERpZ2lDZXJ0IEluYzEZMBcGA1UECxMQd3d3LmRpZ2ljZXJ0
# LmNvbTExMC8GA1UEAxMoRGlnaUNlcnQgU0hBMiBBc3N1cmVkIElEIENvZGUgU2ln
# bmluZyBDQTAeFw0xNjEwMTgwMDAwMDBaFw0xNzEwMjMxMjAwMDBaMHwxCzAJBgNV
# BAYTAlVTMQswCQYDVQQIEwJUTjESMBAGA1UEBxMJVHVsbGFob21hMSUwIwYDVQQK
# ExxDYXJsIFdlYnN0ZXIgQ29uc3VsdGluZywgTExDMSUwIwYDVQQDExxDYXJsIFdl
# YnN0ZXIgQ29uc3VsdGluZywgTExDMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIB
# CgKCAQEAqGo3KHWZmWVSao7Ur+ldBIwwM7v4tM7NQ3X3A9H7DqfGXSVvWvVj5zbc
# zX1yns9Qot1bnrTRLlnimIPJa+GieuEz7ON7jpzQjErmuzJz4HBEfbfAqoVuVmpy
# dsPpxfNqWMQt+0YqeEgYZqoF5mIXK2ACugsQz5e9SMWEsR9Z0s9FQyjEnIKuhQYq
# cLY7y85/CNsH4qgKNoHPfZ+LlPaWFfHCI7XIleLC2QHcLlEe760NDv163eXq6rkC
# tJroHqT4WKeXEEj14nhFNxSp/UUuk004/ju5Pb1gsgOYxkQ94BrixMW9zYghXX2H
# K3JzL8O56djKJuD8em8whmpXAmR6FQIDAQABo4IBxTCCAcEwHwYDVR0jBBgwFoAU
# WsS5eyoKo6XqcQPAYPkt9mV1DlgwHQYDVR0OBBYEFPC0K6tjLci4jiul81bZG+CS
# MocaMA4GA1UdDwEB/wQEAwIHgDATBgNVHSUEDDAKBggrBgEFBQcDAzB3BgNVHR8E
# cDBuMDWgM6Axhi9odHRwOi8vY3JsMy5kaWdpY2VydC5jb20vc2hhMi1hc3N1cmVk
# LWNzLWcxLmNybDA1oDOgMYYvaHR0cDovL2NybDQuZGlnaWNlcnQuY29tL3NoYTIt
# YXNzdXJlZC1jcy1nMS5jcmwwTAYDVR0gBEUwQzA3BglghkgBhv1sAwEwKjAoBggr
# BgEFBQcCARYcaHR0cHM6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzAIBgZngQwBBAEw
# gYQGCCsGAQUFBwEBBHgwdjAkBggrBgEFBQcwAYYYaHR0cDovL29jc3AuZGlnaWNl
# cnQuY29tME4GCCsGAQUFBzAChkJodHRwOi8vY2FjZXJ0cy5kaWdpY2VydC5jb20v
# RGlnaUNlcnRTSEEyQXNzdXJlZElEQ29kZVNpZ25pbmdDQS5jcnQwDAYDVR0TAQH/
# BAIwADANBgkqhkiG9w0BAQsFAAOCAQEAAwIwqMrUHX/2xnjs13V3ikCzJ+LkAMXu
# z4daOhkO5EdCkE8Cl9nnKtVGEVnC8v2xkUSgDWb9yAoGJfOx8oamS6IA3J1C+lND
# 8cKJwb70FAHzQV+Tyzmwm38VavUC0kc27iE5kfziUOU+UH/bZYwmeo1Z54SiooEB
# atp1RYmvbwE8ATyme/KmYkfbUkYlbfpP0aWGey33sKGiI8ZmWUC4PSDWQ+aXiAWv
# YZQXUiGQTWleWvmhlpSVATho62Db2KuE3hsR8v1wLY3s/WPs0OyhrBD80ExWiX/q
# HoQGTmaBGz0SczPU0sfro1gKghTUr96046UFQQjeybpebrrlMLwcGDCCBmowggVS
# oAMCAQICEAMBmgI6/1ixa9bV6uYX8GYwDQYJKoZIhvcNAQEFBQAwYjELMAkGA1UE
# BhMCVVMxFTATBgNVBAoTDERpZ2lDZXJ0IEluYzEZMBcGA1UECxMQd3d3LmRpZ2lj
# ZXJ0LmNvbTEhMB8GA1UEAxMYRGlnaUNlcnQgQXNzdXJlZCBJRCBDQS0xMB4XDTE0
# MTAyMjAwMDAwMFoXDTI0MTAyMjAwMDAwMFowRzELMAkGA1UEBhMCVVMxETAPBgNV
# BAoTCERpZ2lDZXJ0MSUwIwYDVQQDExxEaWdpQ2VydCBUaW1lc3RhbXAgUmVzcG9u
# ZGVyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAo2Rd/Hyz4II14OD2
# xirmSXU7zG7gU6mfH2RZ5nxrf2uMnVX4kuOe1VpjWwJJUNmDzm9m7t3LhelfpfnU
# h3SIRDsZyeX1kZ/GFDmsJOqoSyyRicxeKPRktlC39RKzc5YKZ6O+YZ+u8/0SeHUO
# plsU/UUjjoZEVX0YhgWMVYd5SEb3yg6Np95OX+Koti1ZAmGIYXIYaLm4fO7m5zQv
# MXeBMB+7NgGN7yfj95rwTDFkjePr+hmHqH7P7IwMNlt6wXq4eMfJBi5GEMiN6ARg
# 27xzdPpO2P6qQPGyznBGg+naQKFZOtkVCVeZVjCT88lhzNAIzGvsYkKRrALA76Tw
# iRGPdwIDAQABo4IDNTCCAzEwDgYDVR0PAQH/BAQDAgeAMAwGA1UdEwEB/wQCMAAw
# FgYDVR0lAQH/BAwwCgYIKwYBBQUHAwgwggG/BgNVHSAEggG2MIIBsjCCAaEGCWCG
# SAGG/WwHATCCAZIwKAYIKwYBBQUHAgEWHGh0dHBzOi8vd3d3LmRpZ2ljZXJ0LmNv
# bS9DUFMwggFkBggrBgEFBQcCAjCCAVYeggFSAEEAbgB5ACAAdQBzAGUAIABvAGYA
# IAB0AGgAaQBzACAAQwBlAHIAdABpAGYAaQBjAGEAdABlACAAYwBvAG4AcwB0AGkA
# dAB1AHQAZQBzACAAYQBjAGMAZQBwAHQAYQBuAGMAZQAgAG8AZgAgAHQAaABlACAA
# RABpAGcAaQBDAGUAcgB0ACAAQwBQAC8AQwBQAFMAIABhAG4AZAAgAHQAaABlACAA
# UgBlAGwAeQBpAG4AZwAgAFAAYQByAHQAeQAgAEEAZwByAGUAZQBtAGUAbgB0ACAA
# dwBoAGkAYwBoACAAbABpAG0AaQB0ACAAbABpAGEAYgBpAGwAaQB0AHkAIABhAG4A
# ZAAgAGEAcgBlACAAaQBuAGMAbwByAHAAbwByAGEAdABlAGQAIABoAGUAcgBlAGkA
# bgAgAGIAeQAgAHIAZQBmAGUAcgBlAG4AYwBlAC4wCwYJYIZIAYb9bAMVMB8GA1Ud
# IwQYMBaAFBUAEisTmLKZB+0e36K+Vw0rZwLNMB0GA1UdDgQWBBRhWk0ktkkynUoq
# eRqDS/QeicHKfTB9BgNVHR8EdjB0MDigNqA0hjJodHRwOi8vY3JsMy5kaWdpY2Vy
# dC5jb20vRGlnaUNlcnRBc3N1cmVkSURDQS0xLmNybDA4oDagNIYyaHR0cDovL2Ny
# bDQuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0QXNzdXJlZElEQ0EtMS5jcmwwdwYIKwYB
# BQUHAQEEazBpMCQGCCsGAQUFBzABhhhodHRwOi8vb2NzcC5kaWdpY2VydC5jb20w
# QQYIKwYBBQUHMAKGNWh0dHA6Ly9jYWNlcnRzLmRpZ2ljZXJ0LmNvbS9EaWdpQ2Vy
# dEFzc3VyZWRJRENBLTEuY3J0MA0GCSqGSIb3DQEBBQUAA4IBAQCdJX4bM02yJoFc
# m4bOIyAPgIfliP//sdRqLDHtOhcZcRfNqRu8WhY5AJ3jbITkWkD73gYBjDf6m7Gd
# JH7+IKRXrVu3mrBgJuppVyFdNC8fcbCDlBkFazWQEKB7l8f2P+fiEUGmvWLZ8Cc9
# OB0obzpSCfDscGLTYkuw4HOmksDTjjHYL+NtFxMG7uQDthSr849Dp3GdId0UyhVd
# kkHa+Q+B0Zl0DSbEDn8btfWg8cZ3BigV6diT5VUW8LsKqxzbXEgnZsijiwoc5ZXa
# rsQuWaBh3drzbaJh6YoLbewSGL33VVRAA5Ira8JRwgpIr7DUbuD0FAo6G+OPPcqv
# ao173NhEMIIGzTCCBbWgAwIBAgIQBv35A5YDreoACus/J7u6GzANBgkqhkiG9w0B
# AQUFADBlMQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYD
# VQQLExB3d3cuZGlnaWNlcnQuY29tMSQwIgYDVQQDExtEaWdpQ2VydCBBc3N1cmVk
# IElEIFJvb3QgQ0EwHhcNMDYxMTEwMDAwMDAwWhcNMjExMTEwMDAwMDAwWjBiMQsw
# CQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3d3cu
# ZGlnaWNlcnQuY29tMSEwHwYDVQQDExhEaWdpQ2VydCBBc3N1cmVkIElEIENBLTEw
# ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDogi2Z+crCQpWlgHNAcNKe
# VlRcqcTSQQaPyTP8TUWRXIGf7Syc+BZZ3561JBXCmLm0d0ncicQK2q/LXmvtrbBx
# MevPOkAMRk2T7It6NggDqww0/hhJgv7HxzFIgHweog+SDlDJxofrNj/YMMP/pvf7
# os1vcyP+rFYFkPAyIRaJxnCI+QWXfaPHQ90C6Ds97bFBo+0/vtuVSMTuHrPyvAwr
# mdDGXRJCgeGDboJzPyZLFJCuWWYKxI2+0s4Grq2Eb0iEm09AufFM8q+Y+/bOQF1c
# 9qjxL6/siSLyaxhlscFzrdfx2M8eCnRcQrhofrfVdwonVnwPYqQ/MhRglf0HBKIJ
# AgMBAAGjggN6MIIDdjAOBgNVHQ8BAf8EBAMCAYYwOwYDVR0lBDQwMgYIKwYBBQUH
# AwEGCCsGAQUFBwMCBggrBgEFBQcDAwYIKwYBBQUHAwQGCCsGAQUFBwMIMIIB0gYD
# VR0gBIIByTCCAcUwggG0BgpghkgBhv1sAAEEMIIBpDA6BggrBgEFBQcCARYuaHR0
# cDovL3d3dy5kaWdpY2VydC5jb20vc3NsLWNwcy1yZXBvc2l0b3J5Lmh0bTCCAWQG
# CCsGAQUFBwICMIIBVh6CAVIAQQBuAHkAIAB1AHMAZQAgAG8AZgAgAHQAaABpAHMA
# IABDAGUAcgB0AGkAZgBpAGMAYQB0AGUAIABjAG8AbgBzAHQAaQB0AHUAdABlAHMA
# IABhAGMAYwBlAHAAdABhAG4AYwBlACAAbwBmACAAdABoAGUAIABEAGkAZwBpAEMA
# ZQByAHQAIABDAFAALwBDAFAAUwAgAGEAbgBkACAAdABoAGUAIABSAGUAbAB5AGkA
# bgBnACAAUABhAHIAdAB5ACAAQQBnAHIAZQBlAG0AZQBuAHQAIAB3AGgAaQBjAGgA
# IABsAGkAbQBpAHQAIABsAGkAYQBiAGkAbABpAHQAeQAgAGEAbgBkACAAYQByAGUA
# IABpAG4AYwBvAHIAcABvAHIAYQB0AGUAZAAgAGgAZQByAGUAaQBuACAAYgB5ACAA
# cgBlAGYAZQByAGUAbgBjAGUALjALBglghkgBhv1sAxUwEgYDVR0TAQH/BAgwBgEB
# /wIBADB5BggrBgEFBQcBAQRtMGswJAYIKwYBBQUHMAGGGGh0dHA6Ly9vY3NwLmRp
# Z2ljZXJ0LmNvbTBDBggrBgEFBQcwAoY3aHR0cDovL2NhY2VydHMuZGlnaWNlcnQu
# Y29tL0RpZ2lDZXJ0QXNzdXJlZElEUm9vdENBLmNydDCBgQYDVR0fBHoweDA6oDig
# NoY0aHR0cDovL2NybDMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0QXNzdXJlZElEUm9v
# dENBLmNybDA6oDigNoY0aHR0cDovL2NybDQuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0
# QXNzdXJlZElEUm9vdENBLmNybDAdBgNVHQ4EFgQUFQASKxOYspkH7R7for5XDStn
# As0wHwYDVR0jBBgwFoAUReuir/SSy4IxLVGLp6chnfNtyA8wDQYJKoZIhvcNAQEF
# BQADggEBAEZQPsm3KCSnOB22WymvUs9S6TFHq1Zce9UNC0Gz7+x1H3Q48rJcYaKc
# lcNQ5IK5I9G6OoZyrTh4rHVdFxc0ckeFlFbR67s2hHfMJKXzBBlVqefj56tizfuL
# LZDCwNK1lL1eT7EF0g49GqkUW6aGMWKoqDPkmzmnxPXOHXh2lCVz5Cqrz5x2S+1f
# wksW5EtwTACJHvzFebxMElf+X+EevAJdqP77BzhPDcZdkbkPZ0XN1oPt55INjbFp
# jE/7WeAjD9KqrgB87pxCDs+R1ye3Fu4Pw718CqDuLAhVhSK46xgaTfwqIa1JMYNH
# lXdx3LEbS0scEJx3FMGdTy9alQgpECYxggQDMIID/wIBATCBhjByMQswCQYDVQQG
# EwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3d3cuZGlnaWNl
# cnQuY29tMTEwLwYDVQQDEyhEaWdpQ2VydCBTSEEyIEFzc3VyZWQgSUQgQ29kZSBT
# aWduaW5nIENBAhACyryBXQ2hSod09kAFHCYiMAkGBSsOAwIaBQCgQDAZBgkqhkiG
# 9w0BCQMxDAYKKwYBBAGCNwIBBDAjBgkqhkiG9w0BCQQxFgQUGixOC5varF3Exwp2
# siwKzX3ZqZgwDQYJKoZIhvcNAQEBBQAEggEAAlKyqtMKkoqTfDkpMNnncfqTFrHF
# /uHa3Lyqt+Io6LlyMVW0mqyWHQbDyrJKCcRYCxGJoIWX7/+xxeRVzKwFGcVJHGUr
# 1SGRkOBk4GSEq8N+91SsEEBbT+iTUJuyCKG4mwtfdBDa32ZE0XV8Z4S/bUNX6Ihq
# Fln8ErKnRRHk3yuWwA1aw11c/+G19TxxyB6wHgfS9tYkd8YdjPcWs8gaMF0d/AGs
# Hm7ZPDe+18cdAXXvy6jzaE/Sx38iXiS9HPZDRVqHLe0nJD9xC//+5y8nwMlE/xXk
# i7WY2qvR0MJtUI9oNnWNBcFNJlIKhryckFdmzVql51CZSFlO8mOmwWbpwaGCAg8w
# ggILBgkqhkiG9w0BCQYxggH8MIIB+AIBATB2MGIxCzAJBgNVBAYTAlVTMRUwEwYD
# VQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5jb20xITAf
# BgNVBAMTGERpZ2lDZXJ0IEFzc3VyZWQgSUQgQ0EtMQIQAwGaAjr/WLFr1tXq5hfw
# ZjAJBgUrDgMCGgUAoF0wGAYJKoZIhvcNAQkDMQsGCSqGSIb3DQEHATAcBgkqhkiG
# 9w0BCQUxDxcNMTcwNTE4MTQxNzA4WjAjBgkqhkiG9w0BCQQxFgQU+1gqZC7/kPBF
# eHYgDqpE8lDd8CEwDQYJKoZIhvcNAQEBBQAEggEAlUJn2YtAfnvFUVraOrJwOTc3
# wcVgIZky4win1ba/xNgGY215WN9/JBnGn86pglkOP7TASmhPVRlozaV5ztINfMC5
# kmtFK+Lh66qHb/6nup+mrHvmpnRhnyQcNxp/Jv4kR4A2OXE5YiRMFuAcMN/1YZP8
# H4Is3hhWSL68IVDkadXQKwF2TG4HjA9Vf6iPfJoVDE9V3BAHr9ucTtjnzmLMhFeo
# EFdxd2MuH07GDJC7fLzGaRzVznq6pZ3A5ZjezhwkftZSEnNJpI8mrXuqp2vjXx2S
# nYbaBeJSHBhYwX0wfvHPMyThKS0gV9ifIS2sKuW9IBrQ/xGMMnSxAx5mVqCyrQ==
# SIG # End signature block
