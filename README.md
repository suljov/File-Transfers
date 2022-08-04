# File transfers cheat sheet and more 

## Table Of Content

- [File transfers](#File-transfers)
- [Windows File Transfer Methods](#Windows-File-Transfer-Methods)
  - [Introduction](#Introduction)
  - [Download Operations](#Download-Operations)
  - [PowerShell Base64 Encode and Decode](#PowerShell-Base64-Encode-and-Decode)
  - [PowerShell Web Downloads](#PowerShell-Web-Downloads)
- [Linux File Transfer Methods](#Linux-File-Transfer-Methods)
- [Transfering Files with Code](#Transfering-Files-with-Code)
- [Miscellaneous File Transfer Methods](#Miscellaneous-File-Transfer-Methods)
- [Protected File Transfers](#Protected-File-Transfers)
- [Catching Files over HTTP and HTTPS](#Catching-Files-over-HTTP-and-HTTPS)
- [Living off The Land](#Living-off-The-Land)
- [Detection](#Detection)
- [Evading Detection](#Evading-Detection)




![image](https://user-images.githubusercontent.com/24814781/182864196-0d4ea001-1ea6-49e4-b9ac-042d1a1b320c.png)




### File transfers
There are many situations when transferring files to or from a target system is necessary. Let's imagine the following scenario:

#### Setting the Stage
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

### Windows File Transfer Methods

#### Introduction
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

#### Download Operations

We have access to the machine MS02, and we need to download a file from our Pwnbox machine. Let's see how we can accomplish this using multiple File Download methods.

![image](https://user-images.githubusercontent.com/24814781/182867651-286106d2-e74d-497f-b752-3a5bf1898308.png)

#### PowerShell Base64 Encode and Decode
Depending on the file size we want to transfer, we can use different methods that do not require network communication. If we have access to a terminal, we can encode a file to a base64 string, copy its contents from the terminal and perform the reverse operation, decoding the file in the original content. Let's see how we can do this with PowerShell.

An essential step in using this method is to ensure the file you encode and decode is correct. We can use md5sum,
```
https://man7.org/linux/man-pages/man1/md5sum.1.html
```
a program that calculates and verifies 128-bit MD5 checksums. The MD5 hash functions as a compact digital fingerprint of a file, meaning a file should have the same MD5 hash everywhere. Let's attempt to transfer a sample ssh key. It can be anything else, from our Pwnbox to the Windows target.

#### Check SSH Key MD5 Hash
```
Suljov@htb[/htb]$ md5sum id_rsa

4e301756a07ded0a2dd6953abf015278  id_rsa
```
#### Encode SSH Key to Base64
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

#### Confirming the MD5 Hashes Match
```
PS C:\htb> Get-FileHash C:\Users\Public\id_rsa -Algorithm md5

Algorithm       Hash                                                                   Path
---------       ----                                                                   ----
MD5             4E301756A07DED0A2DD6953ABF015278                                       C:\Users\Public\id_rsa
```
Note: While this method is convinient, it's not always possible to use. Windows Command Line utility (cmd.exe) has a maximum length of the string that you can use at the command prompt of 8191 characters. Also webshell may fail if you attempt to send large strings. 

While this method is convenient, it's not always possible to use. Windows Command Line utility (cmd.exe) has a maximum string length of 8,191 characters. Also, a web shell may error if you attempt to send extremely large strings.

#### PowerShell Web Downloads

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


### Linux File Transfer Methods


### Transfering Files with Code


### Miscellaneous File Transfer Methods


### Protected File Transfers


### Catching Files over HTTP and HTTPS


### Living off The Land


### Detection


### Evading Detection
