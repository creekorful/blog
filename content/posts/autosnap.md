+++
title = "Autosnap"
date = "2020-08-27"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["My Projects", "Rust", "Packaging"]
keywords = ["", ""]
description = "Automatically make Snap package from source code."
showFullContent = false
+++

As my dev workstation is running Ubuntu, I have recently started using [Snap](https://snapcraft.io/)
to install most of my applications.

Snap is an interesting packaging approach since it allows applications publisher to release new versions
directly without having to involve distribution maintainers. This reduces the delay between application development
and end users deployment.

Another interesting aspect of Snap is that they are self-contained and running in a sandbox with limited access to the
host system. This isolation improves security and allows multiples version of the same snap to be installed at the
same time.

The packaging of Snap applications is really simple and is done with a single file `snapcraft.yaml`.

---

Here's as example the configuration file of [osync](https://github.com/creekorful/osync):

```yaml
name: osync
base: core18
version: git
summary: Tool to synchronize in a optimized way a lot of files to a FTP server.
description: |
  Osync is a Rust written tool designed to upload huge amount of files
  to a remote FTP server, in an efficient manner.
license: GPL-3.0

grade: stable
confinement: strict

parts:
  osync:
    plugin: rust
    source: .
    build-packages:
      - libc6-dev

apps:
  osync:
    command: bin/osync
    plugs:
      - home
      - removable-media
      - network
```

As you can see, Snap packaging is quite straightforward and simple, but I think we can improve the experience.
That's what I've tried with [autosnap](https://github.com/creekorful/autosnap).

# Autosnap

Autosnap allows automatic snap packaging easily in a fashion way:

```sh
$ autosnap https://github.com/creekorful/polonium.git
2020-08-27 07:23:32,400 INFO  [autosnap] Starting packaging of https://github.com/creekorful/polonium.git
2020-08-27 07:23:33,617 INFO  [autosnap] Successfully packaged polonium!
2020-08-27 07:23:33,617 INFO  [autosnap] The snapcraft file is stored at /home/creekorful/Documents/polonium/snapcraft.yaml
2020-08-27 07:23:33,618 INFO  [autosnap] Please fix any TODO in the file and run `cd /home/creekorful/Documents/polonium && snapcraft`
```

Depending on the language that will be packaged, autosnap will be able to detect the license,
the build packages, the version, etc... and therefore ease the life of package maintainer.

Current languages supported by Autosnap are: [Go, Rust] (new languages will be supported soon).

The source code of autosnap is available [here](https://github.com/creekorful/autosnap).

Happy hacking!
