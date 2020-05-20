+++
title = "terminews - a CLI based RSS reader"
date = "2019-07-26"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Golang"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

I recently got used to view informations using RSS. It allows me to ignore ads, distractions and to focus on the on the essential: having all the informations that I want in the same place. And it save a really consequent amount of time.

Furthermore, I love the terminal, I like to use it excessively and I prefer It over GUI applications. That's good because I've found a wonderful CLI application to read RSS: terminews.

Terminews is a Go written application that allows to save multiple RSS flux to allow reading directly from the terminal. The video below show how it works:

[![asciicast](https://asciinema.org/a/WKvIugMqbohNtxqCZHHPDcWRr.svg)](https://asciinema.org/a/WKvIugMqbohNtxqCZHHPDcWRr)

# How to install ?

## Using the binary

You can download the binary right here: https://github.com/antavelos/terminews/releases/download/v1.2.0/terminews

## With go

```sh
go install github.com/antavelos/terminews
```

n.b: at the time, terminews is only supported on linux like OS.

Happy reading !