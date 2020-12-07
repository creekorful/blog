+++
title = "How to install Traefik 2.x on a Docker Swarm"
date = "2019-10-21"
author = "Aloïs Micard"
authorTwitter = "" #do not include @ 
cover = ""
tags = ["Docker Swarm", "Dev Ops"]
keywords = ["", ""]
description = ""
showFullContent = false 
+++

I have recently migrated my production docker swarm from Traefik 1.7 to Traefik 2.0 and since I cannot found a good
tutorial I have decided to write one. So in this tutorial you'll learn how to deploy Traefik with HTTPS support on a
docker swarm.

Please note that I won't explain what Traefik is since it may needs his own article and I will focus on the deployment
and configuration. This tutorial will also assume that you have a working docker swarm.

This tutorial will be part of a series regarding Docker Swarm, I'll write other articles to explain how to expose
Traefik dashboard securely, deploy Portainer, etc...

# Install Traefik

Please note that Traefik will need to be deployed on a manager node on your swarm. You'll also need to make sure that
your firewall on this node is correctly setup to allow both port 80 and 443 (http / https) from outside. This is
important because Traefik will listen on these ports for incoming traffic.

The first thing before creating the config file is to create a docker swarm network that will be used by Traefik to
watch for services to expose. This can be done in one command:

```
docker network create --driver=overlay traefik-public
```

This will create an [overlay network](https://docs.docker.com/network/overlay/) named 'traefik-public' on the swarm.

Now we are going to create the docker compose file to deploy Traefik.

```yaml
version: '3'

services:
  reverse-proxy:
    image: traefik:v2.0.2
    command:
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-public"
      - "--entrypoints.web.address=:80"
    ports:
      - 80:80
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-public
    deploy:
      placement:
        constraints:
          - node.role == manager

networks:
  traefik-public:
    external: true
```

This is the minimal amount of config needed to deploy a working Traefik instance.

----

Let me explain in details what's going on:

```yaml
- "--providers.docker.endpoint=unix:///var/run/docker.sock"
```

Here we are telling to Traefik to listen to the unix docker socket.

```yaml
- "--providers.docker.swarmMode=true"
```

Enable swarm mode support by setting swarmMode to true.

```yaml
- "--providers.docker.exposedbydefault=false"
```

Little security: tell Traefik to not expose container by default: only expose container that are explicitly enabled (
using label traefik.enabled).

```yaml
- "--providers.docker.network=traefik-public"
```

Tell Traefik to dial with exposed containers using traefik-public network. (the one created earlier)

```yaml
- "--entrypoints.web.address=:80"
```

Create an entrypoint named **web** exposed on port 80. This entrypoint will be used to indicate on which port your
application should be exposed.

This configuration is enough to get started. You can deploy Traefik using the following command:

```sh
docker stack deploy traefik -c traefik.yaml
```

Great. Now we will deploy something.

# Deploy and expose a hello-world container

Now it's time to deploy something on your swarm to test the configuration. For this example we are going to deploy
tutum/hello-world a little container with an apache service that display an "Hello World" page.

```yaml
version: '3'
services:
  helloworld:
    image: tutum/hello-world:latest
    networks:
      - traefik-public
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.helloworld.rule=Host(`helloworld.local`)"
        - "traefik.http.routers.helloworld.entrypoints=web"
        - "traefik.http.services.helloworld.loadbalancer.server.port=80"
networks:
  traefik-public:
    external: true
```

The labels sections is read by Traefik to get the configuration of the container and to create the needed components to
expose it.

---

```yaml
- "traefik.enable=true"
```

This flag tells Traefik to expose the container, needed because we explicitly disable container auto exposition with
providers.docker.exposedbydefault.

```yaml
- "traefik.http.routers.helloworld.rule=Host(`localhost`)"
```

Create a Host Matching rule. Here we tell Traefik to redirect all traffic coming with host localhost to this container.

```yaml
- "traefik.http.routers.helloworld.entrypoints=web"
```

Tells Traefik that this container will be exposed using the web entrypoint. (the name correspond with the entrypoint
previously created in Traefik).

```yaml
- "traefik.http.services.helloworld.loadbalancer.server.port=80"
```

Indicate to Traefik that the container expose the port 80 internally. This is needed by Traefik to act as a proxy.

---

Now if you access the page at http://localhost you'll be redirect to Traefik that will proxify the content of the
helloworld container.

# Add HTTPS support

If you have followed this tutorial carefully you should now have a working Traefik instance and a helloworld service
running and accessible on http://localhost. This is working but not secure: you should always use HTTPS when possible.
Thankfully this can be done easily in Traefik.

In this tutorial we are going to use the [HTTP challenge](https://letsencrypt.org/docs/challenge-types/) to
automatically generate a Letsencrypt certificate.

I won't go in the details to explain how the HTTP-01 challenge work, but basically all you have to do is to add/update
the A record of your DNS zone to point to your docker swarm manager IP address. (where Traefik is exposed).

```yaml
version: '3'

services:
  reverse-proxy:
    image: traefik:v2.0.2
    command:
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-public"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencryptresolver.acme.email=user@domaine.com"
      - "--certificatesresolvers.letsencryptresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - 80:80
      - 443:443
    volumes:
      # To persist certificates
      - traefik-certificates:/letsencrypt
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-public
    deploy:
      placement:
        constraints:
          - node.role == manager
volumes:
  traefik-certificates:
networks:
  traefik-public:
    external: true
```

---

I will explain the new parts:

```
- "--entrypoints.websecure.address=:443"
```

Create an entrypoint named **websecure** exposed on port **443** (https). This entrypoint will be used to indicate on
which port your application should be exposed.

```yaml
- "--certificatesresolvers.letsencryptresolver.acme.httpchallenge=true"
```

Tell Traefik that the challenge named **letsencryptresolver** will use the http challenge.

```yaml
- "--certificatesresolvers.letsencryptresolver.acme.httpchallenge.entrypoint=web"
```

We are going to expose the http challenge over http (obvious since we have no certificate yet)

```yaml
- "--certificatesresolvers.letsencryptresolver.acme.email=user@domaine.com"
```

The email used to create a letsencrypt account

```yaml
- "--certificatesresolvers.letsencryptresolver.acme.storage=/letsencrypt/acme.json"
```

Where do Traefik will persist the certificate. This location should be bound to
a [volume](https://docs.docker.com/storage/volumes/) to be persisted between container restart. Here we are saving to
/letsencrypt/ directory, who is mounted as volume traefik-certificates (see traefik.yaml for details)

## Update hello-world to use HTTPS config

Now we are going to update the hello-world deployment configuration file in order to expose it using HTTPS.

```yaml
version: '3'
services:
  helloworld:
    image: tutum/hello-world:latest
    networks:
      - traefik-public
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.helloworld.rule=Host(`helloworld.local`)"
        - "traefik.http.routers.helloworld.entrypoints=websecure"
        - "traefik.http.routers.helloworld.tls.certresolver=letsencryptresolver"
        - "traefik.http.services.helloworld.loadbalancer.server.port=80"
networks:
  traefik-public:
    external: true
```

---

There is only two things to change:

```yaml
- "traefik.http.routers.helloworld.entrypoints=websecure"
```

To indicate to Traefik that we want to expose our container using the previously declared websecure entrypoint.

```yaml
- "traefik.http.routers.helloworld.tls.certresolver=letsencryptresolver"
```

To specify which certificate resolver we wanna use. Here we are using the letsencryptresolver declared before.

---

Easy right? Once the stack is redeployed Traefik will then ask Letsencrypt to generate a SSL certificate for the domain
helloworld.local and Traefik will use it when exposing your application.

N.B: Of course this example won't work since you cannot proove that you own the helloworld.local domain. (.local is a
reserved TLD used for local area network)

## Bonus: Create an automatic HTTPS redirect

If you want to redirect all HTTP traffic to HTTPS it can be done by easily by using a Middleware. Just add the following
labels to to the Traefik configuration file.

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.services.traefik.loadbalancer.server.port=888" # required by swarm but not used.
  - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)"
  - "traefik.http.routers.http-catchall.entrypoints=web"
  - "traefik.http.routers.http-catchall.middlewares=redirect-to-https@docker"
  - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
```

It will create a router named *http-catchall* that will intercept all HTTP request (using the hostregexp) and will
forward it to the router named redirect-to-https. This router will perform a redirection to the HTTPS scheme.

---

Here's the final stack to deploy a Traefik instance with HTTPS support and HTTP -> HTTPS global redirection

```yaml
version: '3'

services:
  reverse-proxy:
    image: traefik:v2.0.2
    command:
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-public"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencryptresolver.acme.email=user@domaine.com"
      - "--certificatesresolvers.letsencryptresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - 80:80
      - 443:443
    volumes:
      - traefik-certificates:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-public
    deploy:
      placement:
        constraints:
          - node.role == manager
      labels:
        - "traefik.enable=true"
        - "traefik.http.services.traefik.loadbalancer.server.port=888" # required by swarm but not used.
        - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)"
        - "traefik.http.routers.http-catchall.entrypoints=web"
        - "traefik.http.routers.http-catchall.middlewares=redirect-to-https@docker"
        - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
volumes:
  traefik-certificates:
networks:
  traefik-public:
    external: true
```

Long live Docker Swarm and Happy hacking !
