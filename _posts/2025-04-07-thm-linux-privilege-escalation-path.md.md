---
layout: post
title: THM Linux Privilege Escalation - PATH
subtitle: Using the PATH setting to escalate privileges
date: 2025-04-07
tags:
  - pentest
  - tryhackme
  - priv-esc
  - linux
  - PATH
cover-img: ""
thumbnail-img: /assets/img/linux-priv-esc.png
comments: true
mathjax: true
---

One vector to escalate privileges is to make use of our low-priv user's PATH, when we can write to a directory in the PATH. If we can find some binary or script that executes as `root` that makes use of PATH (rather than using absolute paths), we might be able to add some malicious code there will execute with root privileges if we can run the binary / script.

To answer the first question of this task, we first find writable directories.

# Writable Directories

We start by seeing the directories off `/` that might seem promising as so-called an 'unusual' writable directory.

![writables level 1](/assets/img/lin-priv-esc/writable-level1-path.png)

Hmmmm... We know `karen` has no `/home/karen/` folder, so what there in `/home`???

Let's do another find!

![writable directory](/assets/img/lin-priv-esc/writable-level2-path.png)

So `/home/murdoch/`, not `karen`'s home directory (which doesn't exist), is writable and looks like an unusual location. It's actually the answer to question 1.

Why don't we `cd` over there and take a look inside:

![/home/murdoch](/assets/img/lin-priv-esc/contents-murdoch-path.png)

There's a binary in there with the `s` bit set! We can execute it and it will run as `root`, which is one of the things we need for this attack vector.

This `thm.py` looks interesting...

![thm.py](/assets/img/lin-priv-esc/thm-py-path.png)

Aha, so this Python3 script just execute a command `thm`, without an absolute path, so the OS will check the PATH to find `thm` to run.

Let's just try running `./test` to see what happens:

![thm not found](thm-not-found-path.png)

So, what `./test` does is call `./thm.py` which itself tries to run `thm` (using PATH).

Let's check our PATH!

![PATH](/assets/img/lin-priv-esc/path-path.png)

Any of those directories writable? 

![writable in PATH?](/assets/img/lin-priv-esc/writable-in-path-path.png)

No luck!!

Let's just add `/home/murdoch` to the PATH since we know that's writable!

![path updated](/assets/img/lin-priv-esc/path-updated-path.png)

Alright! Looking good so far. The next step is to create a file called `thm` in `/home/murchoch` that does what we need our elevated privileged to do.

In this case we don't need a `root` shell, it's enough to `cat` the contents of our flag (in `/home/matt`):

![flag capture](/assets/img/lin-priv-esc/flag-path.png)

We could have also done `echo "/bin/bash" > thm` and the same technique would have yielded us a `root` shell!

# Key Learnings

- 🧩 **SUID + PATH = Privilege Escalation Potential**  
    If a SUID binary or script uses a relative command (like `thm` instead of `/usr/bin/thm`), the OS will use `$PATH` to resolve it — and that's exploitable.
    
- ✍️ **Writable Directories in `$PATH` Are Dangerous**  
    If a low-priv user can write to any directory in `$PATH`, they can plant a malicious binary that runs with elevated privileges.
    
- 🔍 **Always Inspect the Binary’s Behavior**  
    
- 📁 **Watch for Unusual Writable Locations**  
    Directories like `/home/otheruser`, `/opt`, or even `/tmp` (if added to `$PATH`) can be potential launchpads for this attack.
    
- 🧪 **Test and Tweak the Environment**  
    If no writable directories are in `$PATH`, try adding one  (e.g., `export PATH=/writable/dir:$PATH`) _before_ executing the vulnerable binary.
    
- 🏁 **Exploit Doesn't Have to Be a Shell**  
    You can escalate by doing anything that requires elevated rights — reading root-owned files, copying binaries, etc. A shell is just one of many options.