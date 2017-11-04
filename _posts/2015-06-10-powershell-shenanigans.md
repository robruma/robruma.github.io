---
id: 19
title: PowerShell Shenanigans
date: 2015-06-10T23:36:29+00:00
author: robruma
layout: post
guid: http://invisiblerobots.org/?p=19
permalink: /2015/06/10/powershell-shenanigans/
categories:
  - whaa??
---
Yeah, that&#8217;s right, I said [PowerShell!](http://en.wikipedia.org/wiki/Windows_PowerShell) The Schizophrenic scripting &#8220;language&#8221; that reminded me of Java, Perl, Bash, VB script and Windows command line all smashed together.
  
I&#8217;ve never coded any PowerShell before, but when my team needed some automation done on a Windows application running in ec2, I was willing to give it a shot. I was taking on the challenge of writing a script to run from a Windows scheduled task that read an entry from an Oracle RDS and executed a batch file depending on the data the table contained.
  
Not having any experience with PowerShell, I resorted to Google for some research. In about a day, I had a working script that did everything I needed it to do without the over-complication I was initially expecting. PowerShell has what seems like dozens of cmdlets for each version iteration but once I was able to figure out the fundamentals, it was nearly the same as writing a Bash script.
  
I&#8217;m a fan of using the right tool for the job and while PowerShell isn&#8217;t my first choice for automation, it made a lot of sense in this situation. If I were to write the same script in Python, it would have added an unnecessary layer of complexity to the instance build automation.
  
Here&#8217;s a few cool things I learned during this adventure:
  
Create a securestring:

<pre>$securestring = Read-Host "Password: " -AsSecureString
$key = foreach ($i in 1..32) { Get-Random -Minimum 0 -Maximum 255 }
Set-Content .\key.txt $key
$securestring | ConvertFrom-SecureString -Key $key | Set-Content .\securestring.txt</pre>

Decrypt that securestring:

<pre>$key = (Get-Content .\key.txt)
$securestring = (Get-Content .\securestring.txt)
$plaintext = $securestring | ConvertTo-SecureString -Key $key | ForEach-Object {[Runtime.InteropServices.Marshal]::PtrToStringAuto(\[Runtime.InteropServices.Marshal]::SecureStringToBSTR($_))}
Write-Output $plaintext</pre>

Connect to an Oracle instance:

<pre># Load Oracle Provider
[Reflection.Assembly]::LoadWithPartialName("System.Data.OracleClient")

# SQL commands
$select_cmd = "select ${column} from ${table} where NAME_OF_COLUMN='${name_of_column}' and NAME_OF_OTHER_COLUMN='${name_of_other_column}'"
$update_cmd = "update ${table} set ${column}='${column_value}' where NAME_OF_COLUMN='${name_of_column}' and NAME_OF_OTHER_COLUMN='${name_of_other_column}'"

# Create connection
$connection = New-Object DATA.OracleClient.OracleConnection("Data Source=${datasource};User Id=${userid};Password=${plaintext}")
$connection.Open()
$value = [System.String]
$value = (new-Object DATA.OracleClient.OracleCommand($select\_cmd,$connection)).ExecuteScalar()
$connection.Close()</pre>