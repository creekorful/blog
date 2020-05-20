+++
title = "Docker swarm is not dead! (yet)"
date = "2020-02-29"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Docker Swarm", "Dev Ops"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

I have written an article on the provisioning of a Docker Swarm cluster from scratch ([you can read it here](https://blog.creekorful.com/how-to-provision-a-secure-docker-swarm-cluster-from-scratch)) and I have received a lot of comments stating that docker swarm is dead and that I should be moving to Kubernetes instead.

# What happened to docker?

For those who were not aware, [Mirantis](https://www.mirantis.com/) (a cloud provider) has bought [Docker enterprise](https://www.docker.com/products/docker-enterprise) in nov. 2019. Just after that, Mirantis has written a [blog post](https://www.mirantis.com/blog/mirantis-acquires-docker-enterprise-platform-business/) to announce the news:

> Q: What About Docker Swarm?
>
> A: The primary orchestrator going  forward is Kubernetes. Mirantis is committed to providing an excellent  experience to all Docker Enterprise platform customers and currently  expects to support Swarm for at least two years, depending on customer  input into the roadmap. Mirantis is also evaluating options for making  the transition to Kubernetes easier for Swarm users.

That's it, Docker swarm have only two years to live, before being left without commercial support. Please note that this doesn't mean that Swarm will be removed, since it's built-in Docker engine. It only means that there will be no commercial support or involving from Mirantis after these two years.

Hopefully things has changed recently.

# What about now?

Mirantis has published [another blog post](https://www.mirantis.com/blog/mirantis-will-continue-to-support-and-develop-docker-swarm/) in the beginning of the week:

> Here at Mirantis, we’re excited to announce our continued support for Docker Swarm, while also investing in new features requested by customers.

Apparently, a lot of docker entreprise customers has requested support and involvement from Mirantis, who has decided to continue commercial support as well as develop new features.

- Swarm Jobs: a new service mode enabling run-and-done workloads on a Swarm cluster.
- Cluster Volume Support: distributed persistent volumes.

That's a very exciting news: we will continue to have an easier Kubernetes alternative, an alternative that is better suited for simple workflow and test laboratories. 

Happy hacking!