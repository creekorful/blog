+++
title = "Going serverless"
date = "2020-06-03"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Dev Ops"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

I used to manage a dozen VPS since many years: Zabbix, Gitlab/Gitlab CI, [private docker registry](https://blog.creekorful.com/2020/01/harbor-private-docker-registry/), 
production environment (3 nodes docker swarm cluster), database server (MariaDB & MongoDB), 
blog server (running [Ghost](https://ghost.org/)), logs collector ([Graylog](https://www.graylog.org/)), etc...

I was spending a consequent amount of money & time for all these VPS, and it was time to change.

---

# From Ghost to Hugo

One of the most important thing I run is this blog. It was previously running on [Ghost](https://ghost.org).

Ghost is a really powerful CMS, with membership support, built-in [WYSIWYG](https://en.wikipedia.org/wiki/WYSIWYG) markdown editor, etc...
It's a really complete solution, open source, 100% free if self-hosted.

## What's the problem then?

The biggest problem with Ghost, is that since it's a CMS, it's not static:

You need to have a running [Node.js](https://nodejs.org/en/) server, a proxy server such as [Nginx](https://www.nginx.com/),
a database server, etc... For me it was too much for something that can be a plain static website.
I was ready to lose their beautiful editor to save some money / time.

## The alternative

The alternative I have chosen is [Hugo](https://gohugo.io/).
Hugo is a free & open source static website generator. (i.e it parses markdown files and generate HTML)

It was a good alternative for me, since I already write all my blog posts in markdown. With Hugo no need to have
a database server or even a VPS with Node.JS support! All you need is a hosting provider that can serve static content.

[Netlify](https://www.netlify.com/) is a perfect fit.

Netlify is a free platform that allows deploy of static content directly from Git,
with custom domains, free HTTPS support & fast CDN.

Basically you write your blog posts on your computer and previewed them locally, 
once your done you push the changes to your Git repository, 
Netlify bots pull them each time you push, 
they run the ``hugo build`` command to generate the website, and finally publish the HTML/CSS/IMGs in their CDN.

All for **FREE** (check their [pricing](https://www.netlify.com/pricing/) for more details).

N.B: By default, Hugo will add the /posts/ prefix before all your blog posts. If like me you are migrating
and don't want to lose your page rank, you'll need to make sure the new blog post URL is the same as the
old one.

---

# What's next?

## Netlify all the way

I have decided to move all my static websites to Netlify, thus saving 2 VPS (Nginx & my custom load balancer).

## Lamdba?

For complex use case, such as running a Golang / Python API, I'll certainly use [AWS lambda](https://aws.amazon.com/lambda).

Their [pricing](https://aws.amazon.com/lambda/pricing/) is interesting:

> The AWS Lambda free usage tier includes 1M free requests per month and 400,000 GB-seconds of compute time per month.

And it's quite performant. (While you'll certainly need to re-design your API, since Lambda are designed
to hold a single responsibility. You'll need to split one API endpoint per Lambda function)

---

# Conclusion

For static websites (plain html, Angular / React application / etc...) I'll certainly go with Netlify.

For complex use case like API, I'll try to split my code into little functions that I'll deploy on AWS lambda,
and if I need storage I'll pick up [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) which is available on the free tier.

But if I was worried about my application data, or the source code being monitored / analyzed, I'll certainly move on
self hosted solution, like a manually provisioned Docker Swarm cluster running on OVH servers.

Happy hacking!