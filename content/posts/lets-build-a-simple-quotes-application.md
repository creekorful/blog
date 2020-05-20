+++
title = "Let's build a simple quotes application"
date = "2020-01-10"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["My Projects", "Golang"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

One night I was feeling inspired and decided to read again all my favorites quotes on Google Keep while listening to music. And suddenly an idea just popped into my head: *why not make a little mobile friendly application to view my quotes properly?*

And that's where it started...

# The idea

The idea was to build a simple quotes application where the user can view the quotes. I didn't wanted to built something complex or innovative, I just wanted to *build quickly something clean and working*.

![Demo of the UI](/img/quote-app-demo.gif)

----

And the challenge was a success. It took me few hours to have the final version up and running in production.

Wanna test the application ? You can access it just [right here](https://quotes.creekorful.fr) !

# Technical details

The solution is API oriented, meaning that somebody can develop an Android / iOS application later on without having to write so much code. The application only need to dial with the API to gather the existing quotes.

## The website

The first thing to do was to build the fully complete website/application using mocked (fake) data. This approach is called **Front End driven development**: You focus first on having the application up and running using fake data. Then you design the API contract based on your application needs. And finally you start implementing the API using the contract previously designed. It eliminate a lot of problems since you know what your front needs, and what your contract will be.

The website application is written using Angular 8. The design is done using the wonderful [Nebular library](https://akveo.github.io/nebular), I wanted to build something with it and stop using [Angular Material](https://material.angular.io) for every Angular project i'm making.

## The API

The API is written in Go. Persistence is done using MongoDB and routing with Echo. Thanks to these frameworks, it was really easy to build everything.

## Pagination & infinite scrolling

When the application was fully implemented, I have started to import all my quotes from Google Keep using a little python script. However as the number of quotes started to increase heavily, I have realized that I cannot load them all on application startup, it would be too heavy. I needed something clever. **A pagination system**.

The idea is simple, the number of quotes returned by the API will be limited, and the caller will have the responsibility to load them page by page using query parameters to identify the current page.

```
> GET /quotes?pagination-page=1&pagination-size=10

X-Pagination-Page=1
X-Pagination-Size=10
X-Pagination-Count=230

[{...}, {...}, ...]
```

----

The *pagination-page* parameter is used to identify which page the API should return. The *pagination-size* parameter is used to indicate the number of result per page.

In addition to these parameter the API will return 3 headers related to the pagination: *X-Pagination-Page* to indicate the current page, *X-Pagination-Size* to indicate the number of quotes per page and *X-Pagination-Count* to indicate the total numbers of quotes, useful to determinate if more quotes are available.

The infinite scrolling is implement easily using a dedicated nebular component named [NbInfiniteListDirective](https://akveo.github.io/nebular/docs/components/infinite-list/overview).

# Conclusion

This little project was really fun to build. I have made the most basic implementation which satisfy my requirements, however a lot of enhancements / optimizations can be done. For those who may be interested to contribute or check the source code, it's available on Github: [here for the API](https://github.com/creekorful/quotes-api), and [here for the UI](https://github.com/creekorful/quotes-ui).

# What's next ?

The concept of solving a project under a little amount of time, while open sourcing the solution was very entertaining. So entertaining that I wanted to built a community around the principle. More details (and a blog post) to come...

Happy hacking !