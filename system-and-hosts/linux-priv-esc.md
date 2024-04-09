# Linux Privilege Escalation

## Cron Jobs

Linux implements task scheduling via a utility called cron. This is used by administrators to automate repetitive tasks such as backing up directories on a specified schedule.

Cron can run applications, scripts and other commands. These cron jobs can be configured to run on varying time schedules using the cron utility.

We can create *user specific* cron jobs or *system wide* cron jobs. When we land on a target machine as a low-level user, we will want to know more about the cron jobs which are scheduled to run (if there are any). We will not be able to see the files which contain the user specific cron jobs because they are only accessible to the root user. These files are kept in `/var/spool/cron/crontabs` and `/var/cron/tabs`

There is also a file called `crontab` which can be found at `/etc/crontab` - this is a configuration file which is used by the cron utility to store and track the established system wide cron jobs. We can access this in various ways. We will see the cron jobs which have been established as `system wide`.

>[!NOTE]
>For a look at how we can exploit cronjobs, you could read over [my writeup of cronos](https://github.com/puzz00/cronos-htb) which is a machine on [htb](https://www.hackthebox.com/)

### Enumeration

As mentioned above, there are different ways we can search for cron jobs. We will be able to see the cron jobs which are set up *system wide* using: `cat /etc/crontab` We can also try: `ls -al /etc/cron*` We can also check `systemctl list-timers -all`

We will be able to see any cronjobs which have been created as the user we have compromised the machine as by looking at the crontab editor for their *user specific* cron jobs using `crontab -e`

A problem here is that we will only see the cronjobs which have been set up system wide for any users or ones which have been created specifically for the user we have compromised the machine as. For privilege escalation purposes we really want to know about cron jobs which have been scheduled to run as the root user. The root user may well have created system wide cron jobs to run as root, but they can also create user specific cronjobs and if they have done so we will not be able to see the configuration file which contains them as a low-level user. We will also not be able to see user specific cronjobs which have been set up as other users on the system.

To solve this problem, we can utilise the `pspy` tool to look for all cronjobs - we can download the correct version based on the compromised machine's architecture and then transfer it to the victim machine. The small versions can be tried if the static versions are too large - they are indicated with an `s` at the end of their name such as `pspy64s` They might not work so it is best to try the static versions first. We will need to add executable permissions to the binary using `chmod +x pspy64` before executing it with `./pspy64` The htb box called [cronos](https://www.hackthebox.com/) is good for practising exploiting cron jobs for priv esc. In my [writeup](https://puzz00.github.io/cronos-htb) of it I show the *pspy* tool in action.

We could also use an automated linux enumeration script such as `linenum.sh` or `linpeas.sh` as these will also enumerate cronjobs.

We can add some arguments to the *pspy64* command such as the `-pf` flag which prints commands and file system events along with `-i 1000` which specifies checks to be made every one second.

Once `pspy64` is up and running, we just need to watch the processes and keep an eye out for any tasks which appear to be scheduled to run using cron.

As well as looking at running processes, we can enumerate the compromised machine to look for unusual scripts or files which might be executed as cronjobs. We will look for `tar` archives, too, as these might be using a cronjob along with a `*` which can be exploited to elevate our privileges. We can unzip and unarchive `tar.gz` files using: `tar -zxvf monitor.tar.gz`

It is worthwhile looking for files which we can write to using: `find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null` since if one of these files has been incorrectly set up to run as a cron job under the privileges of the root user we can either overwrite the original file with malicious code or append malicious code to it.

### Cron File Overwrites

This is the most simple and probably most found cronjob vulnerability. Essentially, we find a cronjob which runs a file or script as the root user - a file which we can write to as the current user. We then overwrite the file or script with a malicious command or append a malicious command to it. There are various malicious commands we can use such as opening a reverse shell, adding the lower privileged user which we have access to the victim machine as into the `sudoer` file or making a copy of `/bin/bash` which has the suid bit set.

To open a netcat shell on the localhost which will be running as root, we can use: `printf '#! /bin/bash\nnc -e /bin/bash 127.0.0.1 1234' > file.sh`

We can try to add a low privileged user to the sudoers group using: `echo 'echo "billybob ALL=NOPASSWD:ALL" >> /etc/sudoers' >> /usr/local/bin/file.sh` We can try `sudo -l` after the cronjob has executed the now malicious `file.sh` and if this works we can try to switch to the root user with `sudo su`

To use the cronjob to make a copy of `/bin/bash` which has the suid bit set, we can use: `echo 'cp /bin/bash /tmp/bash;chmod +s /tmp/bash' >> /usr/local/bin/file.sh` After the cronjob has executed the now malicious `file.sh` script as root, we can try to gain a root shell using: `/tmp/bash -p`

Whatever we are doing to files on the system, it is worthwhile to try to append our commands to the end of them and or to make a backup copy of the original file so we can clean things up and reinstate the original functionality of the file once we have completed our testing.

### Cron Wildcards

It might be that we find the `*` wildcard character being used in a cronjob. A good example is when somebody has scheduled `tar` to make backup archives of directories and specified that they want all the files in the source directory to be archived: `tar czf /tmp/backups/backup.tar.gz /home/billybob/*`

The above command will archive all of the files in the `/home/billybob` directory and zip them using `gzip` before placing the zipped tar archive file into `/tmp/backups` The problem is the use of the wildcard character - we can exploit this if we have access to `/home/billybob` and the tar command is running as a cronjob under root.

With tar we can execute commands as well as create and zip archives. A way to do this is to use the `--checkpoint=1` command. This is a switch for tar and will force it to stop archiving. It will then look for a command to use, which can be specified using `--checkpoint-action=exec=sh file.sh` In this case, tar will execute the command specified by the `--checkpoint-action` switch which is to execute a malicious shell script which we have created. The contents of the shell script could do the same as already mentioned above - open a reverse shell, add a low privileged user to the sudoers group or create a version of bash which has the suid bit set.

In order to get tar to execute the above commands, we just need to use them as filenames for empty files in the directory which the cronjob is archiving. When tar reads the filenames, it will take them as switches and therefore execute the command specified. We can create these malicious filenames using: `touch "/home/billybob/--checkpoint=1"` and `touch "/home/user/billybob/checkpoint-action=exec=sh file.sh"`

### Cron Path Hijacking

When we look at the crontab using `cat /etc/crontab` we will see the `PATH` environment variable which cron uses to look for binaries in cronjobs. If we see a cronjob which does not have an absolute path specified, we can have a look for it in the first part of the `PATH` env variable. If it is not there, and we can write to that directory, we can hijack the `PATH` by creating a malicious shell script in that directory with the same name as the binary being executed by the cronjob. If need be - for example we do not have write permisssions on the first directory - we can check the next directory in the same way.

>[!NOTE]
>For extra info on how scheduled tasks work on linux systems, you could take a :eyes: at my work on [cron jobs explained](https://github.com/zigzaga00/linux-notes/blob/main/linux-notes.md#automating-and-scheduling-tasks-using-cron-jobs)

## SUDO Shell Escapes

On a linux system, there are different types of user. There is a root user who has the highest privileges and therefore complete control of the system. This user is given the user id of 0. There are regular users who do not have the same privs as the root user, but they can be given elevated privs on a temporary basis using the `sudo` command

The `sudo` command stands for super user do and lets a user execute a command with elevated privileges on a temporary basis. It usually requires authentication with the user's own password.

We can allow a user to use `sudo` by adding them to the `/etc/sudoers` file. We do not edit this with text editors directly because if we mess it up it could lead to big problems. We therefore use `visudo` to edit it. This will check for syntax errors before saving the changes.

To add sudo privileges for specific users or groups to `/etc/sudoers` we can use a command such as: `mmouse ALL=(ALL:ALL) ALL` This command relates to the user `mmouse`. We can use a `%` prefix to specify a group like so: `%mmouse ALL=(ALL:ALL) ALL`

>[!NOTE]
>The syntax used above is explained in the next section which is about `/etc/sudoers`

The *sudo group* can use `sudo` - this illustrates how we can use group memberships to more finely control privileges. We could add `mmouse` to the *sudo group* like so: `usermod -aG sudo mmouse`

>[!TIP]
>When enumerating a box, keep your :eyes: open for the members of the `wheel` group as this is another name for the `sudo` group

If we have full sudo rights, we can open an elevated shell session using: `sudo -s` or `sudo /bin/bash` We can kill a sudo session using `sudo -k`

>[!NOTE]
>A sudo session will time out after fifteen minutes by default

We can switch into different users without needing to specify their password using: `sudo -u dduck /bin/bash` or `sudo -u dduck -s`

>[!IMPORTANT]
>We do not have to start a bash session as the user - *we can specify any command* to run as them

### /etc/sudoers

We can break down the entry: `mmouse ALL=(ALL:ALL) ALL` like so...

The *first* field specifies the *user* which the rule should be applied to.

The *second* field specifies the *hostname* that the rule applies to - usually this is specified as `ALL` for *all* hosts.

The *third* field specifies *the users mmouse can sudo into*.

The *fourth* field specifies *the groups mmouse can sudo into*.

The *fifth* field specifies the *command or commands which can be executed* using sudo

We could add `NOPASSWD` to the entry which means that the user will not need to enter their password when they want to elevate their privileges to do whatever has been specified in the rest of the entry. We could do this using: `mmouse ALL(ALL:ALL) NOPASSWD: ALL`

If we omit the `(ALL:ALL)` *user* and *group* specification then the user will only be able to sudo into the root user.

>[!TIP]
>We can specify specific commands in the entry like so: `dduck ALL=(ALL:ALL) NOPASSWD: /usr/bin/find`

### Finding SUDO Shell Escapes

One of the first things we can do when we land on a linux box is to run `sudo -l` which will list any commands which the current user can run as the root user *via the sudo command*.

### Exploiting SUDO Shell Escapes

If we see that the current user can run commands as the root user using `sudo` then it makes sense for us to check if there is a way to abuse this to escape the shell and open a root shell. A good resource for this is [gtfobins](https://gtfobins.github.io) which has lots of information regarding shell escapes for different binaries.

It may well be that we find a custom shell script or binary which can be executed as the root user via the `sudo` command. If this is the case, it is a good idea to have a look at the script or binary in order to understand how it works as it might be we can exploit it to gain a root shell. We could also check file permissions and the path as we might be able to hijack the path or replace or edit the script.

