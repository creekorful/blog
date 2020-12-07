+++ 
title = "How to setup easily ELK on a Docker Swarm"
date = "2020-12-07"
author = "Aloïs Micard"
authorTwitter = "" #do not include @ 
cover = ""
tags = ["Docker Swarm", "Dev Ops"]
keywords = ["", ""]
description = "In this tutorial you'll see how to take advantage of the ELK stack to set up a centralized logging mechanism for your Swarm cluster"
showFullContent = false 
+++

In this tutorial you'll see how to set up easily an ELK (**E**lastic, **L**ogstash, **K**ibana) stack to have a
centralized logging solution for your Docker swarm cluster.

# Install the stack

Below you'll find the full stack to have a working ELK stack on your docker swarm.

```yaml
version: '3'

services:
  elasticsearch:
    image: elasticsearch:7.9.3
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
    volumes:
      - esdata:/usr/share/elasticsearch/data
  kibana:
    image: kibana:7.9.3
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"
  logstash:
    image: logstash:7.9.3
    ports:
      - "12201:12201/udp"
    depends_on:
      - elasticsearch
    deploy:
      mode: global
    volumes:
      - logstash-pipeline:/usr/share/logstash/pipeline/
volumes:
  esdata:
    driver: local
  logstash-pipeline:
    driver: local
```

I will break down the configuration part to explain what's going on:

---

```yaml
elasticsearch:
  image: elasticsearch:7.9.3
  environment:
    - discovery.type=single-node
    - ES_JAVA_OPTS=-Xms2g -Xmx2g
  volumes:
    - esdata:/usr/share/elasticsearch/data
```

First we need to create an elasticsearch container, nothing fancy:

```yaml
- discovery.type=single-node
```

We need to set the discovery to `single-node` to evade bootstrap checks. See [this](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html#single-node-discovery) for more information.

```yaml
- ES_JAVA_OPTS=-Xms2g -Xmx2g
```

Here we need to increase the minimum memory for the container.

---

```yaml
kibana:
  image: kibana:7.9.3
  depends_on:
    - elasticsearch
  ports:
    - "5601:5601"
```

This one is simple, we are creating a kibana container to view the logs. The container expose the port `5601` in order
for you to reach the dashboard. In a production context it will be better to expose kibana trough [Traefik](https://traefik.io/)
with a [Basic Auth](https://doc.traefik.io/traefik/middlewares/basicauth/) middleware for example.

The kibana container will automatically try to connect to an elasticsearch search at address `elasticsearch`. So as
long as the elasticsearch container is named `elasticsearch`, no further configuration is required to make it work.

---

```yaml
logstash:
  image: logstash:7.9.3
  ports:
    - "12201:12201/udp"
  depends_on:
    - elasticsearch
  deploy:
    mode: global
  volumes:
    - logstash-pipeline:/usr/share/logstash/pipeline/
```

Final part, the logstash container. It's the process who will collect the containers logs to aggregate them
and push them to ES.

```yaml
ports:
  - "12201:12201/udp"
```

The logstash process will expose a GELF UDP endpoint on port 12201, and the docker engine will push the logs to that
endpoint.

```yaml
deploy:
  mode: global
```

We will need to have one logstash agent running per node, so that container can push logs to it.

```yaml
volumes:
  - logstash-pipeline:/usr/share/logstash/pipeline/
```

We will need to setup logstash to listen on that port and forward the logs to ES. This will be done by creating
a pipeline, that we will put in this volume.

---

Now we will need to bring up this stack:

```shell
creekorful@host:~$ docker stack deploy logs -c logs.yaml
```

# Configure logstash

Now that we have our 3 containers up and alive, we will need to configure logstash. This is done by putting the following
file `logstash.conf` into the `logstash-pipeline` volume.

```conf
input {
   gelf {
   }
}

output {
	elasticsearch {
		hosts => "elasticsearch:9200"
	}
}
```

This configuration will set up an `UDP` (default) GELF endpoint on port `12201` (default), and output the logs
to elasticsearch host.

Now we will need to kill the container so that it restart with the new configuration.

# Configure Docker logging driver

Now that we have our containers up and running we need to indicate to Docker to push the logs to logstash.
This can be done by two approach:

## Global configuration

We can change the Docker default logging driver to that every container created will push the logs automatically
to the logstash container. This is done by editing the `/etc/docker/daemon.json` config file.

```json
{
  "log-driver": "gelf",
  "log-opts": {
    "gelf-address": "udp://localhost:12201"
  }
}
```

Note: you can see that we are submitting the logs to `udp://localhost:12201`, this is because the logs are submitted
trough the host network.

## Per stack configurations

If you prefer to configure only logging for some of your containers, this can be done individually on each stack
like this:

```yaml
logging:
  driver: gelf
  options:
    gelf-address: "udp://localhost:12201"
```

# Conclusion

Now you should have a running logging mechanism.

Long live Docker Swarm and Happy hacking !
