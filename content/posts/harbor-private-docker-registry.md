+++
title = "Harbor: your own private docker registry"
date = "2020-01-18"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Docker Swarm", "Dev Ops"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

Since I have containerized my whole develoment workflow, from testing to production, I needed a docker registry to centralize my private images and ensure their deployment. I didn't wanted to use [Docker Hub](https://hub.docker.com/) or [Github Packages](https://github.com/features/packages) because the images would be publicly available. Therefore I have started searching for existing private registry providers...

# What's a docker registry again?

In a nutshell, a docker registry is a server used to upload (push) & download (pull) docker images. The most known docker registry is certainly [Docker Hub](https://hub.docker.com/).

It's something **vital** when migrating to a docker based production environment such as Kubernetes or Docker Swarm, because these tools will need a registry to pull images and create container from.

# The existing registry providers

There is a lot of existing registry providers. I'm gonna list the ones I have found, and why I didn't choose them.

- Google Container Registry : the product was very interesting, plus I have never experimented the google cloud platform and having the opportunity to do so seemed cool. However the pricing was the big no for me. Google charge based on image size (0.026$/GB/month), and on network usage. At first it could seems cheap, but it does not scale well.
- Canister : at first the service looks promising (clean web ui, 20 free private repository), but after a little look on the website which is not updated since 2016, the dead sub-reddit (no news since 1year), it was a big no for me.
- Digital Ocean Container Registry : the product is not live at the moment and only on beta phase. I haven't look at it in details.

# Going self hosted

Since I haven't found a registry provider that satisfy my requirements (and because I have a **LOT** of VPS ready to be used) I have decided to check for a self hosted solution.

However, going self hosted may not be the best solution for everybody: you may lower your costs but it comes with some **non trivial** drawbacks :

- You have to set up backup policies: redundancy is NOT an option. If your private registry end up failing and you loose all your images your gonna end with troubles.
- Everything's harder in the self hosted way: you'll need to provision a server, install and update needed tools, configure firewall, backups, wake up at night to fix problems when needed, etc... It's really a job.

If you are not afraid at this point, it means that just like me, you are a self host enthusiast with the minimal skills required to set up everything. (Or maybe you're crazy?).

Ready? Let's jump into the self host adventure !

## Docker default registry

The first solution I've found is the [default docker registry](https://docs.docker.com/registry/). It's the registry written by the Docker team and it's open source. (Source code available [here](https://github.com/docker/distribution))

At first glance, the docker registry looked like a good solution: open source, written by the team behind docker, a lot of online tutorials, really easy to use (work out of the box) ... But after a bit of testing, I have seen the product limitations.

Don't get me wrong: I think that docker registry is a really good solution but it doesn't fit well for production environment, for 2 big reasons:

- The registry is not RBAC capable: you cannot create a role to push image and a role to pull image. Every person with access to your registry could pull & push images as he wants. This feature lack is the biggest no for my use case.
- The registry was shipped with builtin Letsencrypt support, meaning that it's capable of generate & update an SSL certificate for your registry. The SSL support is a really important part if your registry is exposed to the outside world. I have opened a [PR](https://github.com/docker/distribution/issues/3041) on the repository, but there's no news at the moment.

## Harbor to the rescue

Since the docker default registry does not satisfy my requirements, I've continue my research, until I found [Harbor](https://goharbor.io/).

Harbor is an open source, cloud native docker registry. Features include: RBAC security, SSL support, image replication across multiple instances, remote storage (azure, s3, gcs, ...) everything's you'll need ! And the app is fully configurable using a simple yaml file.

### System requirements

| Resource  | Minimum  | Recommended  |
|---|---|---|
|CPU   | 2 CPU  | 4 CPU  |
| Mem  | 4 GB  | 8 GB  |
| Disk  | 40 GB  | 160 GB  |

### How to install

To keep this point short, I'll assume that you have a special dedicated server already provisioned with Docker. For the tutorial I'll use the local filesystem as storage for the images. For production environment I would recommend using a cloud based storage instead, it will add extra latency but in case of server crash your data will be safe.

Harbor is really easy to install since you'll only need to have the docker engine on the host machine. The wonderful harbor installer will automatically download & run all harbor needed services as docker container (docker-ception). Better process isolation & no more hours wasted for updating the dependencies, etc... Everything is done using a simple docker command.

The first things to do is to download the [latest version](https://github.com/goharbor/harbor/releases/latest) of the Harbor online installer from Github. Once downloaded just untar the archive.

The last step now is to configure Harbor. This is done by editing the harbor.yml file located in the folder you've just extract.

```yaml
hostname: registry.mydomaine.com

https:
  port: 443
  certificate: /var/certificates/registry.mydomaine.com.crt
  private_key: /var/certificates/registry.mydomaine.com.key

harbor_admin_password: Harbor12345

database:
  password: <long random string>
  max_idle_conns: 50
  max_open_conns: 100

data_volume: /mnt/data-volume

notification:
  webhook_job_max_retry: 10

log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor

_version: <version>
```

Let's break the config file into the different parts.

```yaml
hostname: registry.mydomaine.com
```

This is the hostname you'll use to dial with your registry.

```yaml
https:
  port: 443
  certificate: /var/certs/registry.mydomaine.com.crt
  private_key: /var/certs/registry.mydomaine.com.key
```

The HTTPS configuration for the registry, this will be used for both dashboard access and the registry API. I am using [Certbot](https://certbot.eff.org/) to manage my certificate generation & renewal automatically. I'm not gonna explain the setup in details since there is already a bunch of good article for it.

```yaml
harbor_admin_password: Harbor12345
```

The default password for the admin user. Please note that after first dashboard login you'll be forced to change it.

```yaml
database:
  password: <long random string>
  max_idle_conns: 50
  max_open_conns: 100
```

The database configuration. At the moment Harbor use Postgresql. You'll need to generate a strong random password for it. The other values here are the default ones.

```yaml
data_volume: /mnt/data-volume
```

The storage used to save the images. This is the important part. Here i'm saving the images on a separate remote disk so that in case of crash the images will be saved no matter what. If you want to use another driver or configure cloud storage more details are available [here](https://github.com/goharbor/harbor/blob/master/docs/1.10/install_config/configure_yml_file.md).

```yaml
notification:
  webhook_job_max_retry: 10
```

The maximum number of retries for web hook jobs.

```yaml
log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
```

Log configuration. Here i'm logging every message with level >= info. Everything will be logged on the /var/log/harbor folder, and log file will be rotated once the file size exceed 200Mb. The last 50 log files will be kept.

```yaml
_version: <version>
```

Please **DON'T** change this line. This is an indicator for Harbor.

Everything is be ready now. Harbor can be installed with a single command

```sh
sudo ./install.sh
```

**Boom !** Harbor should be available at https://registry.mydomaine.com you can login into the web dashboard using admin and the password you've generated before. If it's not the case check the log file for more details.

Using the web dashboard you can create group, user, setup their rights, everything is explained [here](https://github.com/goharbor/harbor/blob/master/docs/1.10/administration/managing_users/rbac.md).

# Conclusion

At the end of this tutorial you should have a running Harbor instance, and you can now host your private images. Please refer to the [original documentation](https://github.com/goharbor/harbor/blob/master/docs/1.10/index.md) for more tweaks.

If you have any questions / suggestions feel free to drop a comment, I'd love to have some feedback.

Happy hacking !