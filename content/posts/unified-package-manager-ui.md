+++
title = "An unified package manager user interface"
date = "2020-11-08"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["My Projects", "Rust"]
keywords = ["", ""]
description = "Funny experiment to make a unified package manager user interface"
showFullContent = false
+++

I've made a little fun experiment this weekend by trying to make a unified package manager user interface.
The idea was to design the simplest package manager UI possible. And I've come up with something that I really like.

I've named the project 'x' (I am tired in finding meaningful name derive from Greek god or anything...). 
Here's how it works:

The program read a sequence of operations to apply. Each operation is prefixed by a token indicate what the operation
kind (install, update or remove).

- **+** is used to install a program
- **-** is used to remove a program
- **^** is used to update a program

# Examples

## Install a program

Let's say we want to install vim:

```
$ x +vim
```

## Remove a program

Let's say we want to remove neofetch:

```
$ x -neofetch
```

## Update a program

Let's say we want to update gcc:

```
$ x ^gcc
```

What if we want to update all packages?

This is sufficient:

```
$ x ^
```

## Multiple operations

What if we want to perform multiple operations?

```
$ x +vim -neofetch ^gcc
```

Yes, this works!

# The implementation

The idea was looking great, so I've decided to implement it (first in C, then refactoring in Rust).

X is available on MacOS, Windows, and Linux, and simply wrap an underlying package manager (APT, Brew, Chocolatey).

The source code is available [on Github](https://github.com/creekorful/x).

Happy hacking!
