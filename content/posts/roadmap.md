---
title: "The Road Ahead"
date: "2026-03-30"
description: "Where Mere Linux is headed and how to follow along"
---

The package manager works. The recipe format works. CI builds packages for
both `x86_64` and `aarch64` and publishes them automatically. What's missing is
packages — enough of them to make a system you can actually use.

That's where the road heads next. We've sketched out three broad milestones
that lay the course for which packages will be added and in roughly what order.

## Three milestones

### 1. Bootable networked system

Goal: Mere boots on real hardware or a VM, initializes services with s6,
connects to the network, and builds its own packages.

Most of the toolchain and core libraries are already built. What remains is
the kernel, s6, a bootloader, device management, and the filesystem tools
needed to get from power-on to a shell prompt.

### 2. Graphical desktop

Goal: A Wayland compositor runs on Mere with a terminal emulator. You can log
in graphically and work in a terminal.

This means building the graphics stack — mesa, libdrm, libinput, wayland,
freetype, fontconfig, cairo, pango — and a compositor and terminal on top.

The first compositor is likely to be [river](https://github.com/riverwm/river)
and we'll play around with different compatible Window Managers.

### 3. Web browser

Firefox builds and runs on Mere.

Firefox pulls in GTK, dbus, audio, video codecs, NSS, Rust, and dozens of
supporting libraries. If Firefox runs, the distro is real.

## Following along

Each milestone lives on
[Codeberg](https://codeberg.org/merelinux/recipes/milestones). In the coming
days and weeks, we'll be building out the shape of those milestones through
individual package recipes being added and built. You'll be able to monitor
the progress through the repository activity there.

Every package built is also a working example of how to write a recipe. The
format is intentionally simple — reading a few existing recipes and the
[recipe specification](https://codeberg.org/merelinux/mere/src/branch/main/docs/design/recipe_spec.md)
should get you started. If you want to try building one locally, it would be
something like this:

```sh
mere key generate
mere dev build recipes/core/jq/recipe.kdl
mere dev import local
```

This fetches sources, enters a namespace, runs the build phases, and produces
a package. If the build fails, `mere` drops you into a debug shell inside the
namespace so you can poke around.

Stay tuned for more detailed developer workflow docs.
