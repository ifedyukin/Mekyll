---
layout: post
title: "Proving Grounds SPX: Privilege Escalation"
date: 2025-07-07
description: A privilege escalation case study of the SPX Proving Grounds machine.
tags:
  - ctf
  - privesc
  - linux
---

As I progress through [TJ Null's OSCP PWK V3 list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview), I wanted to cover a particular machine that had a very unique privilege escalation vulnerability. The machine in question is SPX. I'll skip over initial access for the sake of brevity and limiting spoilers. Speaking of spoilers, you've been warned!

# Identifying the Vulnerability

Once I had my initial footfold into the machine, and gained access to a local user account, my immediate thought was:

*what privileges do I have in this new user context?* 

After some manual enumeration, I eventually found the answer. A sudo command for the `make` binary:

![profiler sudo privileges](/assets/img/PG-SPX-PrivEsc_Sudo-Cmd.png)

## Research

Anytime I come across a specific Unix binary with special permissions (`sudo` in this case), I always consult the [GTFOBins](https://gtfobins.github.io/gtfobins/make/#sudo) GitHub Pages site.

Initially, it looked like I had an easy win here, but I do not have access to the `-s` and `--eval` flags shown in the example GTFOBins code snippet. The target's whitelisted command from the `sudo -l` output was `/usr/bin/make install -C /home/profiler/php-spx`:

![GTFOBin Research](/assets/img/PG-SPX-PrivEsc_Make-GTFOBin.png)

I'll be honest, I was stumped on this problem for a good hour or so. I decided to thoroughly enumerate the rest of the system. I was convinced I missed something somewhere. Eventually, I realized the `sudo` permission for this user was the obvious privilege escalation vulnerability. It was my only access to command execution as *root* on the target.

### Even More Research

Having used the `make` utility in the past, I knew it called on instructions specified in a `Makefile`. I searched for `Makefile` exploits/privilege escalation techniques. Googling this topic eventually led me to this [Medium](https://medium.com/@adamforsythebartlett/makefile-privilege-escalation-oscp-62ea2c666d23) article, written by one Adam Bartlett. He explains that `make` can be used to execute bash commands when the target directory is specified using the `-C` flag.

> The “make” command looks for a file called “Makefile”. If you’re looking for code execution, make sure you’re running “make” in the location where “Makefile” is executed, or specify the directory using the -C flag.

Here is the malicious syntax for bash execution provided in the article.

![Medium Article Example](/assets/img/PG-SPX-PrivEsc_Make-Medium-Blog.png)

# Crafting the Makefile

Now it's time to match the example to the target system. I always like to start by making a backup of a file I'm modifying. It's not a requirement by any means, but it's a good habit to have.

```
profiler@spx:/var/www/html$ cd ~/php-spx/
profiler@spx:~/php-spx$ cp Makefile ./Makefile.bak
```

There's many different bash techniques that can be used to spawn a *root* shell in this scenario. I chose to create a malicous copy of bash with an SUID bit. When a standard user executes this bash copy with a `-p` flag, it tells bash to not reset the effective user ID with my real user ID. In short, bash will run with the file owner's permissions (*root*).

![Malicious Makefile](/assets/img/PG-SPX-PrivEsc_New-Makefile.png)

# Escalating Privileges

With the malicious `Makefile` in place, it's simply a matter of executing the allowed `sudo` command.

![Sudo Make Exploit](/assets/img/PG-SPX-PrivEsc_Sudo-Make-Exploit.png)

Now I'll execute the new `/tmp/evil` bash binary, ensuring the `-p` flag is provided to spawn a *root* shell!

![Evil Bash Binary](/assets/img/PG-SPX-PrivEsc_Bash-Privilege-Escalation.png)

# Lessons Learned

The very specific `sudo` command was a great starting point for securing the system, but the true vulnerability lies in the specified directory's permissions, `/home/profiler/php-spx`. The standard user *profiler* had **write** access to the directory, and as a result, write access to the `Makefile` itself. If the directory prevented the *profiler* user from having write access, there would be no opportunity to escalate privileges.

This machine was an excellent reminder that the devil is always in the details, and file permissions matter!

![Lessons Learned](/assets/img/PG-SPX-PrivEsc_Lessons-Learned.png)