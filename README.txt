Required enable auditing logon/logoff by using Local Group Policy (gpedit.msc):
=============================================================================== 
- Computer Configuration/Windows Settings/Security Settings/Local Policies/Audit Policy
- There are two types of auditing that address logging on, they are Audit Logon Events and Audit Account Logon Events.
- Enable for errors

via command line
-list settings
  auditpol.exe /get /Category:*
  
-set option: 
  auditpol /set /subcategory:"Logon"  /failure:enabled
  subcategory in deutsch = "Anmelden"

Audit Policy Reset on Restart or every 16 hours
===============================================
There can be conficting setting in the policy configuration that cause the Login audit setting to be reset.
See this article on how to solve this problem. 
https://social.technet.microsoft.com/Forums/windowsserver/en-US/03cb345e-baf1-45b7-97e1-b3b7a9ebe119/audit-policy-reset-on-restart

#########################################
extract from this post
-------
Turns out the Local Security Policy thought it needed to apply audit policy settings even though every Advanced Audit Policy Configuration item was marked "Not Configured."  On reboot or when forcing local policy application with "GPUPDATE.EXE /force" all advanced audit policy was being wiped out.  this was confirmed by looking at the output from "GPRESULT.EXE /H ResultsBefore.htm."  This showed audit policy was being applied.

I needed to remove the files that Local Security Policy used to track audit policy.  Removing 
c:\Windows\security\audit\audit.csv, 
c:\Windows\System32\GroupPolicy\Machine\Microsoft\WindowsNT\Audit\audit.csv and 
c:\Windows\System32\GroupPolicy\gpt.ini 
then rebooting put Local Security Policy back into a state where it didn't think it needed to apply advanced audit policy.  Any changes using AUDITPOL.EXE or AuditSetSystemPolicy(...) then survived local policy application.  "GPRESULT /H ResultsAfter.htm" also confirmed that audit policy was not being  applied.

A couple of caveats regarding advanced audit policy...per Ned's blog post above, you CANNOT rely on the Local Security Policy MMC console to accurately reflect the active policy settings...use AUDITPOL.EXE.  Also once you set ANY advanced audit policy setting in Local Security Policy it will overwrite all advanced settings when the policy is applied.  AFAIK there is no way short of manually deleting the above files to tell Local Security Policy to forget advanced audit settings.
#########################################

Test script
===========
from command-line - CSCRIPT.EXE ts_block.vbs


ts_block.vbs - Blocks brute force Terminal Services login attempts
Copyright 2011 Wellbury LLC - See LICENSE for license information

Release 20110901 - Adapted from sshd_block release 20100120
Release 20120530 - Added MSI to distribution

For support, please contact Evan Anderson at Wellbury LLC
EAnderson@wellbury.com, (866) 569-9799, ext 801
<http://serverfault.com/users/7200/evan-anderson>

If you like this program please drop me an email. If you have any
ideas for enhancements or find a bug, I'd love to hear about that
too.


Overview
========
ts_block is a VBScript program that acts as a WMI event sink to receive 
events logged by Windows in response to invalid Terminal Services 
logons. It parses these log entries and acts upon them as follows: 

 - If the IP address attempts to logon with a username flagged as "block 
immediately" the IP address is blocked immediately. 

 - If the IP address attempts to logon with more frequently than is 
allowed in a given time period the IP address is blocked. 

The "block immediately" usernames and thresholds associated with repeated 
logon attempts are configurable in the "Configuration" section of the 
script. Default settings are as follows: 

	Block Immediately Usernames - administrator, root, guest
	Logon attempts allowed - 5 in 120 seconds (2 minutes)
	Duration of block - 300 seconds (5 minutes)

The configuration variables for these values are reasonably 
self-explanatory. Additional variables to enable/disable debugging and 
event log usage are also present and self-explanatory. Review the 
section "Registry Configuration Parameters" for information about 
configuring via the registry (which is useful for management via Group 
Policy). 

Four times per second IP addresses that have remained blocked for their 
assigned block duration are unblocked.

ts_block does not run under Windows 2000 Server (because I have no 
machines handy to test it on and because the Terminal Services logon 
type, 10, is not logged on Windows 2000 Server) and under Windows XP 
(because the remote IP address is not included in the event log 
message). 


Pre-Requisite Computer Configuration
====================================
For Windows Vista, 2008, 7, and 2008 R2 the "Advanced Firewall" is used 
to create inbound firewall rules blocking traffic from the blocked host. 
On these operating systems no special configuration of the registry or 
network adapters is necessary. 

Because Windows Server 2003 lacks sufficient features in its built-in 
firewall functionality a black-hole host route is used. Unfortunately, 
the "trick" commonly used on Linux to black-hole traffic with a route to 
"lo" (127.0.0.1) doesn't work on Windows. The "route" command will fail 
if the destination specified isn't local to one of machine's interfaces, 
as well. As such, there are two options for selecting the destination 
address used for the black-hole route. 

You may specify the black-hole destination IP address as REG_SZ value as 
described below in the "Registry Configuration Parameters" section. This 
address must be local to (in the same IP subnet as) one of the server 
computer's interfaces. It is recommended that you select an address that 
is unused in your network. This is my preferred method of installation 
because no device drivers need to be installed. 

Alternatively you may install a network interface with a static IP 
address assigned and no default gateway specified be present on the 
server computer. A physical hardware device is not necessary as the 
Microsoft Loopback Adapter serves the purposes of this application. 
Details about installing the Microsoft Loopback Adapter is available 
from: http://support.microsoft.com/kb/842561 

After you have installed the Microsoft Loopback Adapter (or chosen an 
unused physical hardware NIC) specify a static IP address and no 
default gateway in the TCP/IP version 4 properties for the adapter. The 
IP address and subnet mask assigned to this adapter should not match any 
network in use in your enterprise and should be in the RFC 1918 space. 

The ts_block script will locate the adapter with no default gateway 
specified and use it as the destination for the black-hole route. 


Registry Configuration Parameters
=================================
The following configuration paramters are available under the registry 
path:  HKLM\Software\Policies\Wellbury LLC\ts_block

Parameter: BlockAttempts
Type: REG_DWORD
Explanation: The number of sequential failed logon attempts (with 
accounts that are not considered "block immediately" accounts) that will 
trigger a block. 

Parameter: BlockDuration
Type: REG_DWORD
Explanation: The duration, in seconds, of a block (either because of 
reaching the BlockAttempts threshhold or because of a "block 
immediately"). 

Parameter: BlockTimeout
Type: REG_DWORD
Explanation: The duration, in seconds, that must elapse between failed 
logon attempts to reset the count of failed logon attempts for a given 
IP address. 

Parameter: BlackholeIP
Type: REG_SZ
Explanation: The IP address used for the black-hole route (for Windows 
Server 2003). If not specified the default algorithm of selecting the IP 
address of a network interface with no default gateway specified will be 
used.  This setting is not used in Windows Server 2008 and later versions
of Windows.

A Group Policy Administrative Template (ADM) file is included with this 
distribution that is capable of setting these values. Deploying a GPO 
near the top of the domain with the BlockAttempts, BlockDuration, and 
BlockTimeout values specified and Site or OU-level GPOs with the 
BlackholeIP value specified (as this will vary based on the subnets 
where the server computers are located, and is only necessary for 
Windows Server 2003 machines) is recommended. 


Script Testing
==============
It is recommended that you copy the ts_block.vbs script to your desired 
location, modify the configuration parameters if you are unsatisfied 
with the defaults, and execute the script either. It is recommended that 
you execute the script using the CSCRIPT.EXE utility, but it is possible 
to execute the script using WSCRIPT.EXE via double-clicking on the 
script file in Windows Explorer. Be aware that, should debugging be 
enabled, execution is only effectively possible through CSCRIPT.EXE 
because message logging to pop-up dialogs will "stall" the script until 
the dialogs are dismissed. 

Test the functionality of the script by performing both invalid logons 
using both a "block immediately" account and attempting repeated logons 
with a valid or invalid account that is not in the "block immediately" 
list. Blocking and unblocking events will be logged in the Application 
event log. (It is recommended that you perform your tests via a protocol
that won't cause you to lose communication with the server computer
during testing or from an IP address that isn't going to be blocked.) 


Windows Service Installation
============================
A binary copy of the public domain "Non-Sucking Service Manager" (nssm, 
available from http://iain.cx/src/nssm/) is included with ts_block to 
facilitate installation as a Windows service. The Microsoft SRVANY tool 
may also be used to run ts_block as a Windows Service. 

If you choose to use nssm, copy the nssm.exe file to the location of 
your choice (in "%ProgramFiles%\ts_block", for example). 

After you are satisfied with the performance of the script in testing 
and have copied nssm.exe to the desired location, install the script as 
a Windows service using the following command-line (from the directory 
where nssm was installed): 

	nssm install ts_block %SystemRoot%\System32\cscript.exe 
		"\"%ProgramFiles%\ts_block\ts_block.vbs\""

The command is depicted as two lines above but should be entered on a 
single line. It is necessary to enter the "\" characters as depicted 
such that the resulting registry entry is surrounded by double quotes. 
This command will create a service set to start automatically. (If your 
ts_block.vbs is stored in a path w/o spaces then you don't need to go 
through those gyrations.) 

After installing the service start it and verify that it functions 
properly. 


External Dependencies
=====================
The following external programs are required to be in the PATH for the 
user context under which ssdh_block is executing: 

ROUTE.EXE - For black-hole routing blocked IP addresses under Windows XP

NETSH.EXE - For creating Advanced Firewall rules on Windows Vista and
  later versions of Windows

EVENTCREATE.EXE - For writing to the event log
  (only if event logging is enabled)


Performance and Security
========================
A simple and fairly unscientific stress test function is included in the 
script (and disabled by default). Testing with the parameters listed in 
the script (but with the BLOCK_DURATION decreased from the default to 60 
seconds) on a Windows Server 2003 SP2 x86 Stadard Edition machine 
resulted in peak memory usage of 6,780KB. As the blocked queue drained 
at the end of the test, the memory usage decreased slightly. On the face 
of it, it would appear that the script can handle at least thousands of 
unique IP addresses being blocked at a rate of one IP address every 10 
to 50ms with no major issues. 

Parameters passed to calls to external programs for creating Windows 
Event Log entries or altering IP routes are sanitized through a regular 
expression match (allowing only the characters 0-9, a-z, A-Z, and 
period, underscore, space, right-leaning slash, colon, and minus). 


Windows Installer Package (MSI)
===============================
A Windows Installer package (MSI) version of ts_block is included with 
this distribution (along with the WiX source file used to create the 
MSI). The MSI is self-contained (all necessary files are compressed and 
embedded within it) and can be used for automated deployment of ts_block 
as a service under NSSM. It has been tested using Group Policy Software
Installation Policy under Windows Server 2003, Windows Server 2008 R2,
and Windows 7. Uninstallation via Group Policy has also been tested.


Future Roadmap
==============
Exciting enhancement possibilities include:

 - Loading values for "block immediately" usernames from the registry.
