---
layout: post
author: Breno Cesar
title:  "[The MacGyver toolbox] Windows 10 network interface monitoring"
subtitle: "You need to monitoring your network traffic but you love terminal interface ? No problem ! Here is the solllution that you already have on your windows machine"
categories: powershell
date: 2020-12-29 18:00:00
---
# #A BIG DISCLAMER - WHY "***The MacGyver toolbox***" ?
<br>
During the covid-19 world pandemic, a lot of our human work force was obligated to stay at home, and as well, it cause a big economy contraction.
<br>

>"The pandemic is expected to plunge most countries into recession in 2020, with per capita income contracting in the largest fraction of countries globally since 1870."(...)
>-- <cite>[World Bank](https://www.worldbank.org/en/news/feature/2020/06/08/the-global-economic-outlook-during-the-covid-19-pandemic-a-changed-world)</cite>

<br>
Then as a professionals from engeneering department, i knew that more than enever before, i needed to do and deploy more, with a less(or any) resources.
<br>
Sometimes, we have a lot of tools avaliable, but we dont use it for a lot of reasons, and sometimes, we spend money and resource to deploy some function/facility that was already avaliable on our systems, but in another format.
<br>
That's why i created this series of "MacGyver tooolbox", if you are from the 80's you know what im talking about, never underestimate the power from a paperclip, bubblegum and silvertape.
<br>
And specialy, its always fun to learn something new, and create something unespected.
<br>
<img src="https://media.tenor.com/images/30886b35b5eec92afd171e70ea71f649/tenor.gif">
<br>
Enjoy !!
<br>

# #END OF BIG DISCLAMER

# __ main __
So...you aways wanted to monitor you windows interfaces traffic, but you came from a linux world, and you are used to do it so with a terminal interface ? You problem its over, meet the cmdlet [Get-NetAdapter](https://docs.microsoft.com/en-us/powershell/module/netadapter/get-netadapter?view=win10-ps) and [Get-NetAdapterStatistics](https://docs.microsoft.com/en-us/powershell/module/netadapter/get-netadapterstatistics?view=win10-ps) from module [NetAdapter](https://docs.microsoft.com/en-us/powershell/module/netadapter/?view=win10-ps) in windows 10 powershell 5.1
<br>
This module its already installed in my windows 10 with powershell 5.1 by default, if it is not on your distribution , you can install it, just open your powershell as administrator, and execute the command bellow:
```powershell
PS C:\WINDOWS\system32> Install-Module -Name NetAdapter
```
Press "A" to accept all terms...after that installation, you will be able to execute theses cmdlet commands:
<br>

```powershell
PS C:\WINDOWS\system32> Get-NetAdapter

Name                      InterfaceDescription                    ifIndex Status       MacAddress             LinkSpeed
----                      --------------------                    ------- ------       ----------             ---------
Ethernet 3                VirtualBox Host-Only Ethernet Adapter        17 Up           0A-00-27-00-00-11         1 Gbps
Ethernet                  Intel(R) Ethernet Connection (6) I21...      12 Disconnected 88-6F-D4-FF-37-BF          0 bps
vEthernet (WSL)           Hyper-V Virtual Ethernet Adapter             62 Up           00-15-5D-30-43-7C        10 Gbps
Wi-Fi                     Intel(R) Wireless-AC 9560 160MHz              6 Up           5C-CD-5B-10-C0-8E     144.4 Mbps
Ethernet 2                Cisco AnyConnect Secure Mobility Cli...       3 Disabled     00-05-9A-3C-7A-00      10.0 Mbps


PS C:\WINDOWS\system32> Get-NetAdapterStatistics

Name                             ReceivedBytes ReceivedUnicastPackets       SentBytes SentUnicastPackets
----                             ------------- ----------------------       --------- ------------------
Ethernet 3                                   0                      0               0                  0
Ethernet                                     0                      0               0                  0
vEthernet (WSL)                      137032651                  73436        64972669              84576
Wi-Fi                                548825411                 437865        29951278             124685
```
<br>
Observe that those cmdlet will bring all the interfaces avaliable on your windows, but it will bring the data as a "BYTE" format, as a telecom engeneer, i need this data in "BIT" format, so, lets create a script where:
<br>
1. Select the interface to monitoring
<br>
2. Collect the interface received and transmited bytes
<br>
3. Transform the collected data to bit format
<br>
4. Calculate the DELTA and print the result
<br>

If you dont want to read and understant all this article, you can download the script [here](https://github.com/brenocrs/WinPrintTraffic) on my github.


## 0. A quick and fast guide to powershell
<br>
As a windows common user, you wil note that every file that have a extenssion, windows will associate it to a program, for powershell script, is the .ps1 extenssion.
So:
<br>
1. Create a file with .ps this extenssion using your text editor.
<br>
2. Right and click on it.
<br>
If you see the option "execute with powershell", it meens that your system its ready to execute scripts with powershell.
<br>

Probably what i sayed it's wrong because is based on my user perception, so you can do your own [google search](http://letmegooglethat.com/?q=how+to+create+scripts+in+powershell) about it ok ?
<br>
## 1. Select the interface
<br>

With our file created, we need to make a something to get the interface id from the cmdlet [Get-NetAdapter](https://docs.microsoft.com/en-us/powershell/module/netadapter/get-netadapter?view=win10-ps), and pass that information to [Get-NetAdapterStatistics](https://docs.microsoft.com/en-us/powershell/module/netadapter/get-netadapterstatistics?view=win10-ps) to get the network traffic statistcs.
<br>
## 1.1 Getting the network interfaces on windows
```powershell
$ADAPTER = (Get-NetAdapter | Select-Object @('InterfaceDescription','ifIndex'))
```
## 1.2 Printing the network interfaces
```powershell
$ADAPTER | Format-Table | Write-Output
```
## 1.3 Prompting for user to select the network interfaces
```powershell
$NETWORK_CARD = (Read-Host -Prompt "Input the ifIndex for monitoring ")
```
## 1.4 Getting the selected network interface description
```powershell
$NETWORK_CARD_DESCRIPTION = (Get-NetAdapter -ifIndex $NETWORK_CARD | select-object -exp InterfaceDescription)
```
<br>

## 2. Collect the interface data

<br>
For that , is just to create a new variables that we pass to then the result from "NETWORK_CARD" variable that we created before:

```powershell
$ADAPTER = (Get-NetAdapter | Select-Object @('InterfaceDescription','ifIndex'))
$ADAPTER | Format-Table | Write-Output
$NETWORK_CARD = (Read-Host -Prompt "Input the ifIndex for monitoring ")
$NETWORK_CARD_DESCRIPTION = (Get-NetAdapter -ifIndex $NETWORK_CARD | select-object -exp InterfaceDescription)
# Collecting the data vvv
$ACTUAL_RX = (Get-NetAdapter -ifIndex $NETWORK_CARD | Get-NetAdapterStatistics | select-object -exp ReceivedBytes)
$ACTUAL_TX = (Get-NetAdapter -ifIndex $NETWORK_CARD | Get-NetAdapterStatistics | select-object -exp SentBytes)
# Collecting the data ^^^
```

<br>
Note that we filter and register the result from received and transmited in different variables
<br>

## 3. Transform the collected data to bit format

<br>
For that we multiplicate by 8 and divide by 1MB.
<br>

```powershell
$ADAPTER = (Get-NetAdapter | Select-Object @('InterfaceDescription','ifIndex'))
$ADAPTER | Format-Table | Write-Output
$NETWORK_CARD = (Read-Host -Prompt "Input the ifIndex for monitoring ")
$NETWORK_CARD_DESCRIPTION = (Get-NetAdapter -ifIndex $NETWORK_CARD | select-object -exp InterfaceDescription)
# Transforming the data vvv
$ACTUAL_RX = ((Get-NetAdapter -ifIndex $NETWORK_CARD | Get-NetAdapterStatistics | select-object -exp ReceivedBytes)*8)/1MB
$ACTUAL_TX = ((Get-NetAdapter -ifIndex $NETWORK_CARD | Get-NetAdapterStatistics | select-object -exp SentBytes)*8)/1MB
# Transforming the data ^^^
```

<br>

## 4. Calculate the DELTA and print the result

<br>
As you can see, the data ploted by  [Get-NetAdapterStatistics](https://docs.microsoft.com/en-us/powershell/module/netadapter/get-netadapterstatistics?view=win10-ps) is a cumulative information, so we need to calculate a delta:
<br>

```powershell
$PRINTRXBITS = $ACTUAL_RX - $LAST_RX
$PRINTTXBITS = $ACTUAL_TX - $LAST_TX
```

<br>
But how to do that if we didn't collected the latest data from RX/TX ? Are we going to back in time ?
<br>
<img src="https://i.pinimg.com/originals/9f/4f/c5/9f4fc5a57eab8d92cdeb070f0381fc64.gif">
<br>
No, not yet...
<br>
For that, we need to create a loop, where:
<br>
1. We collect the first measurement
<br>
2. Calculate the delta
<br>
3. print the value and
<br>
4. Save the collected data in another variable for the next delta calculation
<br>
Then:

```powershell
$ADAPTER = (Get-NetAdapter | Select-Object @('InterfaceDescription','ifIndex'))
$ADAPTER | Format-Table | Write-Output
$NETWORK_CARD = (Read-Host -Prompt "Input the ifIndex for monitoring ")
$NETWORK_CARD_DESCRIPTION = (Get-NetAdapter -ifIndex $NETWORK_CARD | select-object -exp InterfaceDescription)
while($true)
{
	$ACTUAL_RX = ((Get-NetAdapter -ifIndex $NETWORK_CARD | Get-NetAdapterStatistics | select-object -exp ReceivedBytes)*8)/1MB
	$ACTUAL_TX = ((Get-NetAdapter -ifIndex $NETWORK_CARD  | Get-NetAdapterStatistics | select-object -exp SentBytes)*8)/1MB
# Calculating the delta vvv
	$PRINTRXBITS = $ACTUAL_RX - $LAST_RX
	$PRINTTXBITS = $ACTUAL_TX - $LAST_TX
# Calculating the delta ^^^
# Printing the output vvv
	Write-Output "$($NETWORK_CARD_DESCRIPTION) -> RX: $("{0:n2}" -f $PRINTRXBITS) Mbps TX:$("{0:n2}" -f $PRINTTXBITS) Mbps" | Format-Table
# Printing the output ^^
# Saving the data for next delta calculation vvv
	$LAST_RX = $ACTUAL_RX
	$LAST_TX = $ACTUAL_TX
# Saving the data for next delta calculation ^^^
	sleep 0.7
}
```

<br>
If you execute the script you will notice that the first print will provide incorrect data.
<br>

```powershell
InterfaceDescription                                                             ifIndex
--------------------                                                             -------
VirtualBox Host-Only Ethernet Adapter                                                 17
Intel(R) Ethernet Connection (6) I219-LM                                              12
Hyper-V Virtual Ethernet Adapter                                                      62
Intel(R) Wireless-AC 9560 160MHz                                                       6
Cisco AnyConnect Secure Mobility Client Virtual Miniport Adapter for Windows x64       3


Input the ifIndex for monitoring : 6
Intel(R) Wireless-AC 9560 160MHz -> RX: 20.636,13 Mbps TX:1.384,06 Mbps
Intel(R) Wireless-AC 9560 160MHz -> RX: 0,05 Mbps TX:0,02 Mbps
```

<br>
This happen because there is no data on "LAST_RX" and "LAST_RX" variables, so let's patch that:
<br>

```powershell
# Setting initial Variables to zero
$LAST_RX = "0"
$LAST_TX = "0"
$ACTUAL_RX = "0"
$ACTUAL_TX = "0"
# Setting a variable to indicate the first execution
$FIRST = "True"
$ADAPTER = (Get-NetAdapter | Select-Object @('InterfaceDescription','ifIndex'))
$ADAPTER | Format-Table | Write-Output
$NETWORK_CARD = (Read-Host -Prompt "Input the ifIndex for monitoring ")
$NETWORK_CARD_DESCRIPTION = (Get-NetAdapter -ifIndex $NETWORK_CARD | select-object -exp InterfaceDescription)
while($true)
{
	$ACTUAL_RX = ((Get-NetAdapter -ifIndex $NETWORK_CARD | Get-NetAdapterStatistics | select-object -exp ReceivedBytes)*8)/1MB
	$ACTUAL_TX = ((Get-NetAdapter -ifIndex $NETWORK_CARD  | Get-NetAdapterStatistics | select-object -exp SentBytes)*8)/1MB
	$PRINTRXBITS = $ACTUAL_RX - $LAST_RX
	$PRINTTXBITS = $ACTUAL_TX - $LAST_TX
# IF statement: if is the first RUN, print empty, if not print the delta calculation
	if ($FIRST -eq "True"){
		Write-Output "$($NETWORK_CARD_DESCRIPTION) -> RX: 0 Mbps RX: 0 Mbps"
		}
		else {
			Write-Output "$($NETWORK_CARD_DESCRIPTION) -> RX: $("{0:n2}" -f $PRINTRXBITS) Mbps TX:$("{0:n2}" -f $PRINTTXBITS) Mbps" | Format-Table	
			}
	$LAST_RX = $ACTUAL_RX
	$LAST_TX = $ACTUAL_TX
	# Setting the Next run as false
	$FIRST = "False"
	sleep 0.7
}
```

<br>
So now, we have a script to monitoring throught terminal the network traffic:
<br>

```
InterfaceDescription                                                             ifIndex
--------------------                                                             -------
VirtualBox Host-Only Ethernet Adapter                                                 17
Intel(R) Ethernet Connection (6) I219-LM                                              12
Hyper-V Virtual Ethernet Adapter                                                      62
Intel(R) Wireless-AC 9560 160MHz                                                       6
Cisco AnyConnect Secure Mobility Client Virtual Miniport Adapter for Windows x64       3


Input the ifIndex for monitoring : 6
Intel(R) Wireless-AC 9560 160MHz -> RX: 0 Mbps RX: 0 Mbps
Intel(R) Wireless-AC 9560 160MHz -> RX: 50,79 Mbps TX:1,44 Mbps
Intel(R) Wireless-AC 9560 160MHz -> RX: 63,02 Mbps TX:1,68 Mbps
Intel(R) Wireless-AC 9560 160MHz -> RX: 70,59 Mbps TX:0,89 Mbps
Intel(R) Wireless-AC 9560 160MHz -> RX: 0,36 Mbps TX:12,28 Mbps
Intel(R) Wireless-AC 9560 160MHz -> RX: 0,33 Mbps TX:12,38 Mbps
Intel(R) Wireless-AC 9560 160MHz -> RX: 0,31 Mbps TX:11,11 Mbps
```

<br>

## 5. Paperclip+Bubblegum+Silvertape

<br>
Here are some of the sources that i use to create this project:

Getting network data and statistics: [link](https://devops-collective-inc.gitbook.io/windows-powershell-networking-guide/getting-network-statistics)

Formating the data output: [link](https://devblogs.microsoft.com/scripting/powertip-formatting-numeric-output-using-powershell/)

Official cmdlet documentation:
[Get-NetAdapter](https://docs.microsoft.com/en-us/powershell/module/netadapter/get-netadapter?view=win10-ps), [Get-NetAdapterStatistics](https://docs.microsoft.com/en-us/powershell/module/netadapter/get-netadapterstatistics?view=win10-ps) and [NetAdapter](https://docs.microsoft.com/en-us/powershell/module/netadapter/?view=win10-ps) module

<br>

~~[]'s~~ (its not a good idea a hug in these times...sorry)

<br>
_\\// (Live long and prosper)
