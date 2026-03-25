---
title: "One File"
date: "2026-03-25"
description: "What if your lockfile and your package list were the same file?"
---

Every package manager I've used draws a line between what you ask for and what
you get.

npm has `package.json` and `package-lock.json`. Cargo has `Cargo.toml` and
`Cargo.lock`. Nix has `flake.nix` and `flake.lock`. The pattern is universal:
one file for intent, another for resolution. You write the first. The tool
writes the second. They're different formats with different rules, and you need
both to reproduce a build.

This split exists for good reasons. Intent is loose — "give me vim" — and
resolution is precise — "give me vim 9.1.0, release 1, with this exact
content hash." Those feel like different things, so they get different files.

When I was designing the package manager for Mere, I kept coming back to a
question: do they actually need to be different?

## The gradient

In Mere, a profile is described by a `profile.kdl` file. Here's the simplest
possible one:

```kdl
profile {
    package "make"
}
```

That's a statement of intent: I want `make`. When you apply this to a profile,
Mere resolves it — finds the latest version, fetches it, installs it to the
content-addressed store, and writes a realized profile:

```kdl
profile {
    schema-version 2
    created-at 1774407107
    profile-name "lyra-test"
    package "make" version="4.4.1" release=3 \
        content-hash="9426fcd11c446be26544b763d6d649186ec9fd98142204a79d582dc6409ecead"
}
```

That output is the resolution. It records exactly what was installed, down to
the content hash of the package in the store.

Here's the thing: that output is also valid input.

Feed it back into `mere profile apply` on another machine, and you get the
exact same package. The content hash drives the lookup — Mere checks the local
store first, fetches by hash if it's missing. No resolver needed. No ambiguity.

There's no separate lockfile because the resolved form *is* the same format as
the input. The tool reads both equally. What changes is precision:

- **Name only** → "give me the latest"
- **Name + version** → "pin to this version"
- **Name + version + content hash** → "reproduce exactly this"

Users never need to write a content hash. They appear only in the output. But
because the output is valid input, you get exact reproduction for free just by
keeping the file around.

## One format, three uses

This isn't just a system management trick. The same `profile.kdl` format
drives three different workflows.

**System profiles.** `mere install vim` reads the current system generation,
adds vim, resolves, and writes a new generation with full metadata. `mere
profile apply profile.kdl` does the same thing from a file. Both produce the
same artifact — a realized profile with content hashes.

**Rollback.** Every generation is a realized `profile.kdl`. Rolling back is
activating an older one. The format doesn't change between generations — it's
profiles all the way down.

**Dev environments.** Drop a `profile.kdl` in a project directory and run
`mere shell`:

```kdl
profile {
    package "busybox"
    package "make"
}
```

```
$ mere shell -- make --version
  building shell profile from profile.kdl
  ...
GNU Make 4.4.1
Built for x86_64-pc-linux-musl
```

Mere reads the file, builds an isolated profile, and drops you into a
namespace with exactly those packages. It's the same resolution path as system
profiles — same format, same store, same content hashes. The realized output
lives in the profile directory, so you can inspect exactly what your shell
environment contained.

Want to pin your dev environment? Copy the realized profile back into your
project and commit it. Now everyone on the team gets the same tools at the same
versions, verified by content hash. No extra tooling, no separate lock
command. The file you already have is the lockfile.

## Why not two files?

The split-file approach works. I've used it for years. But it has friction.

Lockfiles are opaque. `package-lock.json` is thousands of lines of
machine-generated JSON that nobody reads. `flake.lock` is better but still a
separate artifact you have to remember to commit. The intent file and the lock
file can drift — you update one and forget to regenerate the other.

With a single format, there's nothing to drift. The file is either loose
(names, maybe versions) or precise (names, versions, hashes). You choose your
level of precision by including or omitting fields. The system meets you where
you are.

There's also a conceptual simplicity I find appealing. A generation's
`profile.kdl` is a complete, self-contained description of a system state. You
can read it. You can diff two generations. You can copy it to another machine.
It's not a computed artifact that requires tooling to interpret — it's a list
of packages with their identities.

## The loop

The part that satisfies me most is the closed loop.

You write a loose profile. The system resolves it and writes a precise one.
That precise one is valid input. Feeding it back in skips resolution entirely
— the content hashes go straight to the store. The output of that second
application is identical (same hashes, same packages). The loop is stable.

This means a realized profile is a reproducibility artifact. Not because the
build system is pure or the inputs are hermetically sealed, but because the
content hash is a statement about the artifact itself. You're not saying "these
inputs should produce the same output." You're saying "give me this exact
output." It's a simpler contract.

---

The source is at [codeberg.org/merelinux/mere](https://codeberg.org/merelinux/mere).
Mere is pre-release — the package manager is functional and manages a real
system, but the package set is still growing. If this kind of design interests
you, I'd welcome feedback.
