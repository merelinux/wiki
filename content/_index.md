---
title: Mere Linux
---

Mere Linux is a lightweight Linux distribution built around
[musl libc](https://musl.libc.org/) and
[s6](https://skarnet.org/software/s6/) for system initialization and process
supervision.

The goal is a minimal, understandable system where packages are immutable,
upgrades are atomic, and multiple versions coexist cleanly — using plain
directories, symlinks, and user namespaces rather than heavy abstractions.

Mere uses its own [package manager](https://codeberg.org/merelinux/mere/)
written in Zig, [busybox](https://busybox.net) for core utilities, and
[llvm+clang](https://llvm.org) as the toolchain.

New to Mere? Take the [Tour of Mere](/tour/) to learn how the package manager
works.

Mere is under active development — the toolchain is solid but the package
manager is new, and we're still growing the package repository, docs, and
tooling. Here be dragons. You can follow along and contribute at
[codeberg.org/merelinux/recipes](https://codeberg.org/merelinux/recipes).
