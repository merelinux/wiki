---
title: Getting Started
---

The easiest way to get started experimenting with Mere Linux is  through a
Docker container.

```sh
docker run -it mere/base
```

The image size is quite small:

```sh
$ docker image ls mere/base
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
mere/base    latest    5ce2dc13e8ce   4 months ago   4.55MB
```

From here, you can use the `pacman` command to begin exploring and installing
other packages.

The process for installing Mere Linux to a machine is still fairly manual, but
we've tried to make it simple. There are three main methods for installation.

- [Using the Mere Linux ISO Image](iso/)
- [Using an existing Linux system](existing-system/)
- [Using the Mere Linux raw disk image](raw-disk/)
