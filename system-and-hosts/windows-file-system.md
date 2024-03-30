# Windows File System

## Alternate Data Streams

Any file created on a drive which has been formatted using NTFS will have two different streams.

One stream is the *data* stream - this is the default stream and contains the data of the file. An example would be the text in a txt file.

There is a second stream called the *resource* stream. This stream stores meta data about the file. This might be data such as when the file was created and who it was created by.

## Abusing Data Streams

We can abuse this functionality by hiding malicious data in the *resource* stream of a legitimate file. This would hide the malicious data in a simple attempt to evade signature based anti-virus software.

A simple example to illustrate how this works is to use: `notepad test.txt:secret.txt` which will let us create a new txt file. In this case, we are working in the *resource data* stream so we can add some secret text. When we open this file with: `notepad test.txt` we will see that it is empty as we are looking in the *data* stream. We can edit it as usual. If we want to access the secret text in the resource stream, we can use: `notepad test.txt:secret.txt`

We can hide a *malicious file or script* inside a normal looking txt file using: `type PowerView.ps1 > accounts.txt:pv.ps1`

We can now add innocent data to `accounts.txt` if we want to and then transfer the file to a victim machine.

We can now execute the malicious binary or script. In this case we would use `. .\accounts.txt:pv.ps1` from within a powershell session. For an executable binary, we could create a *symbolic link* using an elevated cmd prompt: `mklink accounts.exe C:\Temp\accounts.txt:winpeas.exe` The symbolic link needs to be created in `C:\Windows\system32` and then we can just type `accounts` to execute the malicious binary.

---

> be extremely subtle even to the point of formlessness - be extremely mysterious even to the point of soundlessness - thereby you can be the director of your opponent's fate