---
title: "A New Package Manager"
date: "2026-03-17"
description: "Status update and new package manager announcement"
---

Mere Linux has been quiet for a few years.

That wasn't accidental.

It started with dissatisfaction with `pacman`.

`pacman` is a great package manager, and it serves the Arch Linux community very
well. But three things kept gnawing at me:

1. A build system based on Bash. The scripts that drive packaging can be slow,
   fragile, and difficult to extend with code that is testable and portable.
2. True build isolation required external tools. I had already patched makepkg
   to avoid the problematic `fakeroot` utility and instead used Docker
   containers. This worked well, but it still required a fair amount of glue.
3. Officially, package signing requires a GPG setup. I have never been a fan of
   GPG and wanted to use something lighter and more modern. I had actually
   patched `pacman` to use `asignify`, but that patch was
   rejected upstream, and reasonably so. It also wasn't a direction I felt
   fully aligned with.

So I paused.

That pause turned into a deeper exploration of what I actually wanted from a
package manager, and eventually, that path led to Nix.

## Down the Nix Rabbit Hole

Nix immediately caught my attention. It takes build isolation seriously, makes
package outputs immutable, and gives every build result a unique identity based
on its inputs. The result is a system where multiple versions of software can
coexist safely, upgrades are predictable, and rollbacks are trivial.

After spending years dealing with the limitations of traditional package
managers, those ideas were deeply appealing.

But as I spent more time with Nix, I also began to understand the tradeoffs it
makes to achieve those goals.

One of its core ambitions is functional purity: builds should have no side
effects and depend only on explicitly declared inputs. That's a powerful idea.
But it also means the system has to work very hard to prevent accidental
dependencies on the surrounding environment.

That effort shows up in several places. The Nix language is powerful, but has a
steep learning curve. The system relies on embedding absolute dependency paths
into binaries, which requires compiler and linker wrappers to ensure builds
only see Nix-provided paths, and scripts often require rewritten shebangs.

These are clever solutions. They work.

But they introduce machinery whose primary purpose is enforcing purity.

Over time, I realized this was the core design choice: Nix accepts significant
complexity to guarantee strong purity and reproducibility.

What I felt for a long time, but couldn't quite articulate until a friend put
it into words, was this:

> _Simplicity is its own kind of purity._

For me, that sentence reframes everything.

If a system is simple enough to read, inspect, and understand, that
transparency becomes its own form of integrity. Not because it prevents every
mistake, but because it makes mistakes visible and recoverable.

That distinction led to a question: can we keep what makes Nix compelling
(e.g., build isolation, immutability, atomic installs, multiple versions, strong
identity) while staying close enough to traditional Unix that we don't have
to fight it?

Today I'm publishing `mere`, a package manager that explores that path.

## A Mere Walkthrough

At its core, `mere` is built around a few simple ideas. Many are inspired by
Nix, but the mechanisms are intentionally different.

Packages are stored as immutable objects in a content-addressed store. Each
package is described by a signed manifest that records its identity and
dependencies. System state is expressed through profiles that reference those
immutable objects, allowing installs, upgrades, and rollbacks to happen
atomically without overwriting files.

Let’s walk through the main pieces.

### The Content-Addressed Store

Similar to Nix, the foundation of the system is a content-addressed package
store located at `/mere/store`. When a package is built, its output is hashed
and stored under a path derived from that hash. Because the identity of each
object is derived from its contents, store objects are immutable: changing the
contents necessarily produces a new object. This property gives every package
artifact a strong, unambiguous identity and prevents packages from overwriting
one another. Even without enforcing strict functional purity, this model still
provides strong artifact identity, immutability, and the ability for multiple
package versions to coexist safely.

What this looks like in practice:

```sh
$ ls -l /mere/store | tail -3
dr-xr-xr-x    4 root     root          4096 Feb  3 00:37 fa885783bcd1493203b2e691ffa8402101814953acc3dee16a888f56ed0b27f4-execline-2.9.4.0
dr-xr-xr-x    4 jeremy   jeremy        4096 Mar  4 19:45 fcd990e9111fc8a831f7961513d0a1ab2625a011fd0cf582ab8fe252457bb7ea-libcxx-21.1.8
dr-xr-xr-x    5 root     root          4096 Mar  4 19:45 fdc5a786422f9e8f2406f003ded699fc36cedb4c79cccf97b4390ad56511b538-musl-1.2.5
```

Notice that all store directories are installed non-writable. You can also see
that `libcxx` was installed by an unprivileged user, while the other two are
owned by root. I'll explain more about that distinction later. For now, let's
look at profiles.

### Profiles

Packages in the store are immutable artifacts, but the system still needs a way
to define which packages should actually be visible. In mere, that role is
handled by profiles. A profile represents a realized view of a set of packages.

Instead of installing files directly into shared locations, the system
constructs a profile that references the packages that should be present. In
practice, a profile is simply a directory tree made up of symlinks pointing
into the store. This allows multiple profiles to exist simultaneously, without
packages overwriting one another. It is what enables custom user views of
installed packages and makes rolling back system changes straightforward.

Here's a small view of a profile:

```sh
$ ls -l /mere/profiles/system/gen-12/
drwxr-xr-x    2 root     root          4096 Mar  4 19:38 bin
drwxr-xr-x    4 root     root          4096 Mar  4 19:38 etc-defaults
-rw-r--r--    1 root     root          1054 Mar  4 19:38 manifest.json
drwxr-xr-x    2 root     root          4096 Mar  4 19:38 sbin
drwxr-xr-x    4 root     root          4096 Mar  4 19:38 usr

$ ls -l /mere/profiles/system/gen-12/usr/
drwxr-xr-x    2 root     root          4096 Mar  4 19:38 bin
drwxr-xr-x    5 root     root          4096 Mar  4 19:38 share

$ ls -l /mere/profiles/system/gen-12/usr/bin/
lrwxrwxrwx    1 root     root           103 Mar  4 19:38 awk -> /mere/store/d14e3cf6ec2f0b84a437bda195736fba07276f8b73aa2cc888eb2dbd1eec50d8-busybox-1.37.0/usr/bin/awk
lrwxrwxrwx    1 root     root           103 Mar  4 19:38 env -> /mere/store/d14e3cf6ec2f0b84a437bda195736fba07276f8b73aa2cc888eb2dbd1eec50d8-busybox-1.37.0/usr/bin/env
lrwxrwxrwx    1 root     root           111 Mar  4 19:38 mkinitramfs -> /mere/store/d14e3cf6ec2f0b84a437bda195736fba07276f8b73aa2cc888eb2dbd1eec50d8-busybox-1.37.0/usr/bin/mkinitramfs
lrwxrwxrwx    1 root     root           100 Mar  4 19:38 patch -> /mere/store/883ea055547c160921d36f485fc8e0f75254c058ca85e75a4ff1283bef454efa-patch-2.8/usr/bin/patch
```

From this you can see that a profile just contains a partial view of system
root directories. When you drill down to individual files, they are simply
symlinks pointing to their real locations in the store.

If you're wondering about the `etc-defaults` directory, we'll come back to that
in a bit. For now, let's talk about how system profiles are activated.

### System Activation

Profiles represent realized views of packages, but the system still needs a way
to decide which profile is currently active. With `mere`, this is handled
through a very simple mechanism: a single symlink.

The active system profile is referenced by `/mere/profiles/system/current`.
Activating a new system configuration simply means creating a new generation
(a numerically incrementing directory in the profile such as gen-17),
and updating that symlink to point to it. Because profiles are just symlink
trees pointing into the immutable store, switching generations does not modify
any package artifacts. The system simply begins referencing a different set of
packages.

In practice it looks like this:

```sh
$ ls -l /mere/profiles/system/
lrwxrwxrwx    1 root     root             6 Mar  8 19:16 current -> gen-16
drwxr-xr-x    6 root     root          4096 Mar  4 18:53 gen-11
drwxr-xr-x    6 root     root          4096 Mar  4 19:38 gen-12
drwxr-xr-x    6 root     root          4096 Mar  4 21:58 gen-13
drwxr-xr-x    6 root     root          4096 Mar  4 21:59 gen-14
drwxr-xr-x    7 root     root          4096 Mar  4 22:00 gen-15
drwxr-xr-x    7 root     root          4096 Mar  8 19:16 gen-16
```

Switching the system to a new generation simply updates the current symlink to
point to a different profile. Because this operation is atomic, upgrades and
rollbacks are equally simple: activating a new system or reverting to an older
one both involve the same operation.

On a native `mere` system, activating a system profile exposes its binaries and
libs to the whole system because the core root directories themselves are
symlinks into `/mere/profiles/system/current`, like so:

```sh
$ ls -l /
lrwxrwxrwx    1 root     root            33 Mar  8 18:46 bin -> /mere/profiles/system/current/bin
drwxr-xr-x    2 root     root          4096 Sep 15  2021 boot
drwxr-xr-x    8 root     root          3120 Mar  2 13:09 dev
drwxr-xr-x   14 root     root          4096 Mar  8 20:59 etc
drwxr-xr-x    3 root     root          4096 Jun 15  2023 home
lrwxrwxrwx    1 root     root            33 Mar  8 18:47 lib -> /mere/profiles/system/current/lib
drwxr-xr-x   10 root     root          4096 Mar  4 18:22 mere
dr-xr-xr-x  222 root     root             0 Mar  2 13:09 proc
drwxr-x---    9 root     root          4096 Mar  8 19:16 root
drwxr-xr-x    8 root     root           220 Mar  2 13:09 run
lrwxrwxrwx    1 root     root            34 Mar  8 18:47 sbin -> /mere/profiles/system/current/sbin
drwxr-xr-x    2 root     root          4096 Sep 15  2021 srv
dr-xr-xr-x   13 root     root             0 Mar  2 13:09 sys
drwxrwxrwt    9 root     root         57344 Mar  8 23:13 tmp
drwxr-xr-x   10 root     root          4096 Mar  8 18:49 usr
drwxr-xr-x    9 root     root          4096 Mar 29  2024 var

$ ls -l /usr
lrwxrwxrwx    1 root     root            37 Mar  8 18:48 bin -> /mere/profiles/system/current/usr/bin
lrwxrwxrwx    1 root     root            37 Mar  8 18:48 include -> /mere/profiles/system/current/usr/include
lrwxrwxrwx    1 root     root            37 Mar  8 18:48 lib -> /mere/profiles/system/current/usr/lib
lrwxrwxrwx    1 root     root            41 Mar  8 18:49 libexec -> /mere/profiles/system/current/usr/libexec
drwxr-xr-x    7 root     root          4096 Apr 16  2025 local
lrwxrwxrwx    1 root     root            38 Mar  8 18:49 sbin -> /mere/profiles/system/current/usr/sbin
lrwxrwxrwx    1 root     root            37 Mar  8 18:48 share -> /mere/profiles/system/current/usr/share
```

With this structure, activating a new system simply changes which profile the
root filesystem paths point to. The package artifacts themselves remain untouched,
keeping the activation process straightforward while still providing reliable
upgrades and easy rollbacks.

Let's look now at how packages themselves are described.

### Package Manifests

Packages in `mere` are described by signed manifests. A manifest is a small piece
of metadata that records the identity of a package and binds it to its content.

Each manifest includes the package name, version, release, architecture, the
content hash of the package payload, and a creation timestamp. Because the
manifest is signed, it provides a verifiable statement about exactly what the
package is and when it was built.

The manifest is stored inside the package archive at `.mere/manifest.v1` (with
its signature at `.mere/manifest.v1.sig`), but is excluded from the content hash
computation. The content hash recorded in the manifest must match the hash of
the package payload placed in `/mere/store`, ensuring that the package being
installed is exactly the one that was signed.

Dependencies are automatically discovered during the build process by analyzing
ELF metadata (`DT_NEEDED` entries, `PT_INTERP`), script shebangs, and binary
linkage. This dependency information is stored in the repository database
rather than in the package manifest itself. The repository database is
responsible for identifying what packages exist, what libraries or binaries
they provide and what run-time dependencies each package requires.

Together, the manifest and the content-addressed store give every package a
strong and verifiable identity. The store guarantees the integrity of the
artifact itself, while the signed manifest binds that artifact to its declared
identity.

### Building Packages

`mere` takes an opinionated view of the build process. Isolation is important
to prevent leakage of the host system into the build artifact, but again, the
goal here isn't full purity. It is control over which libraries and tools are
found at build time without diving too far into toolchain modification. Linux
namespaces are a great match here.

`mere` starts by reading a `recipe.kdl` file. [KDL](https://kdl.dev/) was
chosen as the recipe's base language for its simplicity and readability. A
basic recipe might look like this:

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

After reading the recipe, `mere` fetches sources, installs build dependencies,
and begins executing ordered build phases inside a user namespace. The store
directory has been set up in mode `1777` so normal users can install their own
trusted packages there for builds and personal profiles, but before an existing
item in the store can be activated in the system profile, a full verification
and migration of ownership is performed, keeping system activation as a trust
boundary. This explains the non-root user in the output we showed earlier.

Each build phase is cached on completion which means that it is simple to
restart a build at any point. For example, if a `build` phase succeeds but
there's an error with the `install` phase, the `build` output will be restored
from cache and the `install` will begin again the next time `mere dev build` is
run for that recipe.

Finally, on full build completion, dependencies and provisions are automatically
discovered by ELF headers and similar markers, files are deduplicated, and split
packages are produced based on the specific files or file patterns listed in
each `package` section.


### etc and etc-defaults

Configuration files are a perennial source of friction in package management.
In `mere`, the rule is simple: `/etc` belongs to the user. Packages provide
defaults, but never overwrite what's already there.

Instead, when a package is built, any files it places in `etc/` are
automatically relocated to `etc-defaults/` during the packaging phase. This
happens transparently, the recipe author writes a normal `install` phase that
puts configuration files in `etc/`, and the build system moves them aside
before the package is archived. The result is that every package in the store
carries its default configuration under `etc-defaults/`, but never claims
ownership of `/etc` itself.

When a new system generation is activated, `mere` walks the `etc-defaults/`
tree of every package in the profile and compares each file against what
currently exists in `/etc`. There are three possible outcomes:

- **Missing**: the file doesn't exist in `/etc` yet, so the default is copied
  into place. This is the common case on a fresh install or when a new package
  introduces a new configuration file.
- **Identical**: the file in `/etc` matches the default exactly. Nothing to do.
- **Different**: the file in `/etc` has been modified. `mere` leaves it
  untouched and records the drift.

That last case is the important one. When a package ships a new default and the
administrator has customized the existing file, `mere` does not overwrite it,
prompt about it, or silently drop a `.new` file next to it. It simply notes
that the two have diverged.

The `mere etc` command gives the administrator tools to inspect and resolve
that drift on their own terms:

```sh
# Show which /etc files have drifted from the active defaults
$ mere etc status
Active system /etc status: 1 differing, 0 missing, 12 unchanged
  differing: /etc/myapp/config.conf <- myapp

# See what changed
$ mere etc diff /etc/myapp/config.conf

# Accept the new default (backs up the current file to .old)
$ mere etc apply /etc/myapp/config.conf
```

If two packages try to provide the same `/etc` path, activation fails with a
clear error rather than silently picking a winner. This keeps the mapping
between packages and configuration files unambiguous.

The result is a system where configuration defaults flow from packages, but
the administrator's changes are never at risk. Drift is visible and
recoverable, not hidden or overwritten.

## Too Long but Hopefully You Still Read

The source code is now available:

- [Main repo](https://codeberg.org/merelinux/mere)
- [Mirror](https://github.com/jhuntwork/mere)

The foundation is solid and working. From here, I'll be building out packages,
improving documentation, and making it easier to install and use.

If this direction resonates with you, I'd love feedback.

Thanks for reading.
