+++
title = "Supervisor: a host management solution"
date = "2019-09-03"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["My Projects"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

Back in 2015 I used to have a lot of devices connected to my LAN. There was my Minecraft server running on a Blade, my Plex server running on an old computer, my gaming computer, my laptop, a Promox single node running on another Blade and several Raspberry Pi used to monitor temperature, sensor activity, performing a DynDNS like synchronization, etc... Some of the devices were running continuously (such as the Pies because they consume a few amount of electricity) and some others were running periodically when I needed them.

The biggest problem with such a setup is that you need physical access to the hosts to power them up, and it was something I cannot do every time. 

But fortunately there was a solution: [Wake-On-Lan](https://en.wikipedia.org/wiki/Wake-on-LAN)

# Wake-On-Lan

Wake On Lan is an Ethernet standard that allows a computer to be turned on by a network message knows as the "Magic Packet". The magic packet is composed of 6 bytes of 255 (hex: FF FF FF FF FF FF) followed by sixteen repetitions of the target computer's MAC address.

![WOL packet layout](/img/wol-packet.png)

Once received, the message is directly processed by the target computer's network card that will wake up the computer if the packet is well formed.

Wake-On-Lan has some limitations too:

- You cannot awake host that are connected by WiFi: most 802.11 wireless interfaces does not maintain a link when running in low power states.
- Wake-On-Lan only operate within the same network by default: this is because the packets are broadcasted through the entire network, and broadcast packets are generally not routed. This limitation increase security in return.

The first problem wasn't a big deal, most of my devices were connected using ethernet wire. 
But the second one obliged me to develop an interface to allow sort of remote Wake-On-Lan capability.

# Supervisor principle

This is why supervisor was designed: to create an easy interface to manage, awake, shut-down my local hosts.

![Supervisor layout](/img/supervisor-layout.png)

Here's the big picture of the solution. Supervisor run on a dedicated Pi and will expose a web interface to interact with the hosts connected in my LAN. Supervisor act more or less as a Wake-On-Lan gateway.

# The Software

Supervisor is written using PHP and Python. The choice was made because it run easily on unix-like system, has a low resources requirement and is well designed to build web applications. Plus I was quite accustomed with them. The solution is composed of two sub-programs:

- The web interface (PHP): this is the exposed service. It allows the user to login into the application, view the hosts in the local network, add new host, edit existing host, delete existing host, start/boot a host, shutdown a host.
- A daemon (Python): this is the internal service, it reads the MySQL database to get existing hosts, then will perform a test to check if the host is alive.

![Supervisor architecture](/img/supervisor-architecture.png)

Note: I know the architecture sucks, I was very bad at this. The separation of concerns is not respected at all. Please don't follow this architecture if you want to make something clean and maintainable.

## The web interface

The web interface is written in old PHP5. It mainly consist of several PHP scripts outputting HTML, with a bit of [Bootstrap](https://getbootstrap.com/) for the design and a bit of Javascript with jQuery to enhance user experience. The persistence layer is done with PDO.

The Wake-On-Lan part is as simple as that:

```php
$reply = $db->query('SELECT MAC_ADDRESS FROM hosts WHERE ID=' . $db->quote($_GET['id']));
exec('wakeonlan ' . $reply->fetch()[0]); 
```

## The daemon

The daemon is written in Python 3. It is scheduled using cron each 5 minutes, but can be directly schedule using the web interface (if you want to force refresh the status). The principle of operation is really simple:

![Supervisor daemon](/img/supervisor-daemon.png)

As you can see in the diagram, the daemon simply ping the target host and wait for a reply. Please note that it will work in my case because I have enabled reply from ICMP in my computers. But in real life it will fail if the computer has disabled response from ICMP.

So the daemon is sending a ping to the target host, wait for a reply (with a timeout) and if there is a positive reply the daemon will update the refresh date to now and set the status to UP.

```python
def is_host_up(host_address):
    response = os.system("ping -c 1 -w2 " + host_address + " > /dev/null 2>&1")
    return response == 0
```

The command above use the fact that if the ICMP echo request fails, the command [will return exit code 0](https://linux.die.net/man/8/ping).

# The later features

The first version of Supervisor took one day to go to production. It was easy to get everything working and so I started to add new features.

## Pushbullet integration

The first thing I wanted to do was to integrate my solution with [Pushbullet](https://www.pushbullet.com/). For those who don't know Pushbullet is a cross device notification platform (actually is more than that but that's not the subject). I have created a private notification channel and refactored my daemon script to send a notification each time a host goes up/down. It allowed me to notice that sometimes my brothers used my gaming computer without my permission. 🙄

## Service monitoring

The last feature I wanted to add was service monitoring. Like I've said in the beginning, I have a lot of services running on my hosts (Plex, Gitlab, Proxmox, ...) and I wanted to monitor them. What I mean by monitoring ? I wanted to be informed if a service goes down so I can react. This was done by adding a Service notion. First by adding new screen to the web interface to view/add/edit/delete service from a host. The service contains a port and a description. Later the python script were refactored to execute a simple [netcat](https://en.wikipedia.org/wiki/Netcat) command on the host IP suffixed by the service port. And if the netcat command timeout and return a wrong exit code the daemon will send a Pushbullet notification.

# The future

Currently the source code of Supervisor is not available. This is wanted because the codebase and the architecture is a real mess (I admit it at least). But I really want to rework all of this to provider a robust service, with easy setup to allow anyone to install the solution, and API oriented, with an angular application as frontend.

Maybe in the future I'll commit something. Who knows...

Happy hacking! 