---
title: "Tour of Mere"
weight: 1
---

`mere` is a package manager for [Mere Linux](https://merelinux.org) that
combines ideas from Nix — content-addressed storage, immutable packages,
atomic upgrades and rollbacks — with the simplicity of traditional Unix
conventions.

This page walks through the main concepts and shows what they look like in
practice.

## The Store

Every package lives in `/mere/store` under a path derived from the hash of its
contents. Because identity comes from content, store objects are immutable:
changing anything produces a new hash and a new path.

```
$ ls /mere/store | tail -5
d14e3cf6…-busybox-1.37.0
fa885783…-execline-2.9.4.0
fcd990e9…-libcxx-21.1.8
fdc5a786…-musl-1.2.5
883ea055…-patch-2.8
```

Multiple versions of the same package coexist safely — they're just different
directories. Nothing is ever overwritten.

## Profiles

A profile is a directory of symlinks that points into the store. It defines
which packages are visible without copying or moving anything.

```
$ ls -l /mere/profiles/system/gen-12/usr/bin/
awk   -> /mere/store/d14e3cf6…-busybox-1.37.0/usr/bin/awk
env   -> /mere/store/d14e3cf6…-busybox-1.37.0/usr/bin/env
patch -> /mere/store/883ea055…-patch-2.8/usr/bin/patch
```

Each time you install or remove a package, a new generation is created. The
active generation is a single symlink:

```
$ ls -l /mere/profiles/system/
current -> gen-16
gen-11/
gen-12/
gen-13/
…
gen-16/
```

Rolling back is just pointing `current` at an older generation. The operation
is atomic — the system is never in a half-updated state.

## System Activation

On a Mere Linux system, the root filesystem directories are themselves
symlinks into the active profile:

```
$ ls -l /
bin  -> /mere/profiles/system/current/bin
lib  -> /mere/profiles/system/current/lib
sbin -> /mere/profiles/system/current/sbin
…

$ ls -l /usr
bin     -> /mere/profiles/system/current/usr/bin
lib     -> /mere/profiles/system/current/usr/lib
include -> /mere/profiles/system/current/usr/include
share   -> /mere/profiles/system/current/usr/share
…
```

Activating a new system generation updates which profile these paths resolve
to. The package artifacts in the store are never touched.

## Recipes

Packages are defined in KDL recipe files. A recipe declares metadata, sources,
and build/install phases:

```kdl
recipe {
    name "make"
    version "4.4.1"
    release 3
    description "A tool to control the generation of executables"
    url "http://www.gnu.org/software/make/"
    licenses "GPL3"
    archs "aarch64" "x86_64"
    depends "busybox" \
            "linux-headers" \
            "llvm" \
            "make" \
            "mimalloc-dev" \
            "musl-dev"
}

source "http://mirror.rit.edu/gnu/make/make-${recipe.version}.tar.gz" {
    blake3 "a7d8aee97b7e9a525ef561afa84eea0d929f246e3aafa420231c0602151cf9eb"
}

build {
    script r#"
        LDFLAGS="$LDFLAGS -static -Wl,-static" \
            ./configure --prefix=/usr \
            --disable-load
        make V=1
    "#
}

install {
    script r#"
        make DESTDIR="${DESTDIR}" install
    "#
}

package "make" {
    files "usr/bin/" \
          "usr/share/man/man1/"
}
```

Builds run inside Linux user namespaces for isolation. Each phase is cached
independently — if `build` succeeds but `install` fails, the next run restarts
from `install` without rebuilding.

Dependencies are discovered automatically from ELF metadata and shebangs
rather than declared manually at the runtime level.

## Configuration Management

`/etc` belongs to the user. Packages never overwrite files there.

When a package includes configuration files, the build system automatically
relocates them from `etc/` to `etc-defaults/`. On system activation, mere
compares defaults against what's in `/etc`:

- **Missing** — the default is copied into place
- **Identical** — nothing to do
- **Different** — left untouched, drift is recorded

The `mere etc` command lets you inspect and resolve drift:

```
$ mere etc status
Active system /etc status: 1 differing, 0 missing, 12 unchanged
  differing: /etc/myapp/config.conf <- myapp

$ mere etc diff /etc/myapp/config.conf

$ mere etc apply /etc/myapp/config.conf
```

If two packages try to provide the same `/etc` path, activation fails with a
clear error rather than silently picking a winner.

## User Packages

The store directory is mode `1777`, so unprivileged users can install packages
for their own builds and personal profiles without root access.

```
$ ls -l /mere/store | grep jeremy
drwxr-xr-x  jeremy jeremy  fcd990e9…-libcxx-21.1.8
```

Before a user-installed package can be activated in the *system* profile, mere
performs a full verification and migrates ownership. System activation is the
trust boundary — the store is open, but the system profile is not.

## Shell Environments

`mere shell` lets you enter an isolated environment based on any profile.
It uses Linux user namespaces — no root required, no containers.

```
$ mere shell -p myprofile
```

Inside the shell, you see a filesystem assembled from the selected profile:
`/bin`, `/lib`, `/usr` etc. are read-only bind mounts from the profile's
symlink tree into the store. `/home` and `/var` are passed through from the
host. `/mere` is mounted read-write so you can install packages from inside
the session.

`/etc` gets an overlayfs layer — you see the host's `/etc` but any changes
are isolated to the session and discarded on exit. If overlayfs isn't
available, `--no-etc-overlay` falls back to a read-only bind mount.

You can also run a single command without entering an interactive shell:

```
$ mere shell -p build -- make -j4
```

The profile can also be set via the `MERE_SHELL_PROFILE` environment
variable, which is useful for scripting or setting a default in your shell
rc file.

Because each shell session is scoped to a profile, you can have different
environments for different tasks — a minimal profile for runtime, a full
toolchain profile for development — without them interfering with each other
or with the host system.

## Signed Manifests

Every package carries a signed manifest at `.mere/manifest.v1` that records
its name, version, architecture, content hash, and build timestamp. The
content hash is computed over the package payload excluding the manifest
itself, so the signature binds the declared identity to the exact artifact in
the store.

## Links

- [Source code](https://codeberg.org/merelinux/mere)
- [Mirror](https://github.com/jhuntwork/mere)
- [Background — A New Package Manager](/posts/new-pm/)
