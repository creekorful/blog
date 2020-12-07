+++ 
title = "How to expose Traefik 2.x dashboard securely on Docker Swarm"
date = "2020-01-12"
author = "Aloïs Micard"
authorTwitter = "" #do not include @ 
cover = ""
tags = ["Docker Swarm", "Dev Ops"]
keywords = ["", ""]
description = ""
showFullContent = false 
+++

This article is part of a series about Docker Swarm. For the first article please
check [here](https://blog.creekorful.com/how-to-install-traefik-2-docker-swarm/).

On this short tutorial you'll learn how to deploy securely the Traefik built-in dashboard with HTTPS support and basic
authentication system.

This article assume that you have a working Docker Swarm cluster with Traefik running with HTTPS support. If not you can
following [this article](https://blog.creekorful.com/how-to-install-traefik-2-docker-swarm/) to get started.

------

Traefik 2.0 has introduced a brand new dashboard app that allows a quick view on the configuration. This is useful to
view configured entrypoints, existing routers, services, ...

![Traefik dashboard](/img/traefik-dashboard.png)

# How to install it ?

## Enable the Dashboard and the API

Let's take the final docker compose file from
the [first tutorial](https://blog.creekorful.com/how-to-install-traefik-2-docker-swarm/) and add some instructions:

```yaml
version: '3'

services:
  reverse-proxy:
    image: traefik:v2.3.4
    command:
      # Docker swarm configuration
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-public"
      # Configure entrypoint
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      # SSL configuration
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencryptresolver.acme.email=user@domaine.com"
      - "--certificatesresolvers.letsencryptresolver.acme.storage=/letsencrypt/acme.json"
      # Global HTTP -> HTTPS
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      # Enable dashboard
      - "--api.dashboard=true"
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

------

```yaml
- "--api.dashboard=true"
```

Enable the Dashboard web interface & the Traefik API.

## Expose the dashboard securely

Now that you have enabled the API and the Dashboard you'll need to expose it. It can be done in multiple way, here we'll
choose to expose it via HTTPS using Traefik: a ***traefik-ception***.

The Traefik dashboard is available using a service called api@internal so all you have to do is to expose this service.
The following compose file will do it:

```yaml
version: '3'

services:
  reverse-proxy:
    image: traefik:v2.3.4
    command:
      # Docker swarm configuration
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-public"
      # Configure entrypoint
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      # SSL configuration
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencryptresolver.acme.email=user@domaine.com"
      - "--certificatesresolvers.letsencryptresolver.acme.storage=/letsencrypt/acme.json"
      # Global HTTP -> HTTPS
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      # Enable dashboard
      - "--api.dashboard=true"
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
      labels:
        - "traefik.enable=true"
        - "traefik.http.services.traefik.loadbalancer.server.port=888" # required by swarm but not used.
        - "traefik.http.routers.traefik.rule=Host(`traefik-ui.local`)"
        - "traefik.http.routers.traefik.entrypoints=websecure"
        - "traefik.http.routers.traefik.tls.certresolver=letsencryptresolver"
        - "traefik.http.routers.traefik.service=api@internal"
volumes:
  traefik-certificates:
networks:
  traefik-public:
    external: true
```

------

```yaml
- "traefik.http.routers.traefik.rule=Host(`traefik-ui.local`)"
- "traefik.http.routers.traefik.entrypoints=websecure"
- "traefik.http.routers.traefik.tls.certresolver=letsencryptresolver"
```

Configure the exposure of the Traefik dashboard on the **traefik-ui.local** domain name, using the websecure entrypoint
with the letsencryptresolver. If you want more information about how to configure these, just check
my [first blog post about Traefik](https://blog.creekorful.com/how-to-install-traefik-2-docker-swarm/).

```yaml
- "traefik.http.routers.traefik.service=api@internal"
```

Indicate to Traefik which service should be exposed.

And that's it ! Now Traefik should be available on traefik-ui.local.

## Adding a basic authentication system

If you intend to expose Traefik to the outside world, it is essential to add an authentication system otherwise everyone
can access your dashboard. Hopefully this can be done easily using Traefik built-in middleware system.

```yaml
version: '3'

services:
  reverse-proxy:
    image: traefik:v2.3.4
    command:
      # Docker swarm configuration
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-public"
      # Configure entrypoint
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      # SSL configuration
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencryptresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencryptresolver.acme.email=user@domaine.com"
      - "--certificatesresolvers.letsencryptresolver.acme.storage=/letsencrypt/acme.json"
      # Global HTTP -> HTTPS
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      # Enable dashboard
      - "--api.dashboard=true"
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
      labels:
        - "traefik.enable=true"
        - "traefik.http.services.traefik.loadbalancer.server.port=888" # required by swarm but not used.
        - "traefik.http.routers.traefik.rule=Host(`traefik-ui.local`)"
        - "traefik.http.routers.traefik.entrypoints=websecure"
        - "traefik.http.routers.traefik.tls.certresolver=letsencryptresolver"
        - "traefik.http.routers.traefik.service=api@internal"
        - "traefik.http.routers.traefik.middlewares=traefik-auth"
        - "traefik.http.middlewares.traefik-auth.basicauth.users=admin:$$apr1$$8EVjn/nj$$GiLUZqcbueTFeD23SuB6x0"
volumes:
  traefik-certificates:
networks:
  traefik-public:
    external: true
```

------

```yaml
- "traefik.http.middlewares.traefik-auth.basicauth.users=admin:$$apr1$$8EVjn/nj$$GiLUZqcbueTFeD23SuB6x0"
```

Create a middleware named *traefik-auth*, and define the basic auth users. The users are a comma separated list of the
follow format: username:password.

To generate a password you can use [htpasswd](https://httpd.apache.org/docs/2.4/programs/htpasswd.html) (here with sed
to escape the $ present in the hash):

```sh
creekorful@localhost ~$: echo $(htpasswd -nb admin admin) | sed -e s/\\$/\\$\\$/g
admin:$$apr1$$8EVjn/nj$$GiLUZqcbueTFeD23SuB6x0
```

------

```yaml
- "traefik.http.routers.traefik.middlewares=traefik-auth"
```

Finally tell Traefik to use the middleware named traefik-auth.

And that's it ! Re deploy your compose file and you should now have a running Traefik instance with the dashboard
exposed securely.

Happy hacking !
