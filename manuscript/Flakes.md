# Flakes

This book has leaned pretty heavily on the Nix command line's support for "flakes", but I've glossed over the details of how flakes work despite how much we've already been using them.  In this chapter I'll give a more patient breakdown of how flakes work and why we use them.

## Motivation

You can think of flakes as the package manager for Nix.  In other words, if we use Nix to build and distribute packages written in other programming languages (e.g.  Go, Haskell, Python), then flakes are how we "build" and distribute Nix packages.

Here are some example Nix packages that are distributed as flakes:

- `nixpkgs`

  This is the most widely used Nix package of all.  Nixpkgs is a giant `git` repository [hosted on GitHub](https://github.com/NixOS/nixpkgs) containing the vast majority of software packaged for Nix.  Nixpkgs also includes several important helper functions that you'll need for building even the simplest of packages, so you pretty much can't get anything done in Nix without using Nixpkgs to some degree.

- `flake-utils`

  This is a Nix package containing useful utilities for creating flakes and is itself distributed as a flake.

- `sops-nix`

  This is a flake we just used in the previous chapter to securely distribute secrets.

All three of the above packages provide reusable Nix code that we might want to incorporate into downstream Nix projects.  Flakes provide a way for us to depend on and integrate Nix packages like these into our own projects.

## Flakes, step-by-step

We can build a better intuition for how flakes work by starting from the simplest possible flake you can write:

```nix
# ./flake.nix

{ outputs = { nixpkgs, ... }: {
    # Replace *BOTH* occurrences of `x86_64-linux` with your machine's system
    #
    # If you don't know your system then keep reading
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
  };
}
```

You can then build and run that flake with this command:

```bash
$ nix run
Hello, world!
```

### Flake references

We could have also run the above command as:

```bash
$ nix run .
```

… or like this:

```bash
$ nix run '.#default'
```

… or in this fully qualified form:

```bash
$ # Replace x86_64-linux with your machine's system
$ nix run '.#packages.x86_64-linux.default'
```

In the above command `.#packages.x86_64-linux.default` is a "flake reference", which has two halves separated by a `#` character:

- a flake URI

  The first half of a flake reference specifies where a flake is located.  In our above example the flake URI is `.` (a shorthand for our current directory).

- a flake attribute path

  The second half of a flake reference specifies the output attribute path we want to use.  In the above example, it is `packages.x86_64-linux.default` and `nix run` uses that output attribute path to select which executable to run.

Usually we don't want to write out a fully qualified flake reference like `.#packages.x86_64-linux.default` when we use flakes, so flake-enabled Nix commands provide a few convenient shorthands that we can use to shorten flake references.

First, many Nix commands will automatically expand part of the flake attribute path on your behalf.  For example, if you run:

```bash
$ nix run '.#default'
```

… then `nix run` will attempt to expand `.#default` to a fully qualified attribute path of `.#apps."${system}".default`[^1] and if the flake does not have that output attribute path then `nix run` will fall back to a fully qualified attribute path of `.#packages."${system}".default`.

Different Nix commands will expand the attribute path differently.  For example:

- `nix build` and `nix eval` expand `foo` to `packages."${system}".foo`

- `nix run` expands `foo` to `apps."${system}".foo`

  … and falls back to `packages."${system}".foo` if that's missing

- `nix develop` expands `foo` to `devShells."${system}".foo`

  … and falls back to `packages."${system}".foo` if that's missing

- `nixos-rebuild`'s `--flake` option expands `foo` to `nixosConfigurations.foo`

- `nix repl` will not expand attribute paths at all

In each case the `"${system}"` in the expanded attribute path corresponds to your current system, which you can query using this command:

```bash
$ nix eval --impure --expr 'builtins.currentSystem'
```

You can even omit the attribute path, in which case it will default to `default`.  For example, if you run:

```bash
$ nix run .
```

… then `nix run` will expand `.` to `.#default` (which will in turn expand to `.#packages.${system}.default` for our flake).

Furthermore, you can omit the flake reference, which will default to `.`, so if you run:

```bash
$ nix run
```

… then that expands to a flake reference of `.` (which will then continue to expand according to the above rules).

### Flake URIs

So far these examples have only used a flake URI of `.` (the current directory), but flakes support many types of URIs:

- paths

  These can be relative paths (like `.` or `./utils` or `../bar`), home-anchored paths (like `~/workspace`), or absolute paths (like `/etc/nixos`)[^2].

#### TODO

- Talk about why we define output attributes that are not `packages`


### Platforms

This first minimal example highlights one important aspect of flakes: selecting the appropriate system for building our code.  Normally we don't explicitly write out the system when authoring a flake and we instead typically use the `flake-utils` package to manage the system for us, like this:

```nix
{ outputs = { flake-utils, nixpkgs, ... }:
    flake-utils.lib.eachDefaultSystem (system: {
      packages.default = nixpkgs.legacyPackages."${system}".hello;
    });
}
```

… and that essentially is the same thing as if we had written:

```nix
{ outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
    packages.aarch64-linux.default = nixpkgs.legacyPackages.aarch64-linux.hello;
    packages.x86_64-darwin.default = nixpkgs.legacyPackages.x86_64-darwin.hello;
    packages.aarch64-darwin.default = nixpkgs.legacyPackages.aarch64-darwin.hello;
  };
}
```

However, it might not be clear why we need to write (or generate) code that specifies the system twice.  In other words: why do we specify the system on both the left-hand and right-hand side of the equals sign?

The system on the left-hand side of the equals sign is used by the command-line interface to select which output attribute of our flake to build.  By default, if we don't specify

```nix
{ outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
  };
}
```

The above flake creates a `hello` package

## Anatomy of a flake

Most of what this section will cover is information that you can already find from other resources, like the [NixOS Wiki page on Flakes](https://nixos.wiki/wiki/Flakes#Flake_schema) or by running `nix help flake`.  However, I'll still try to explain flakes in my own words and also include my own commentary on "dos" and "don'ts" of using flakes.

[^1]: We'll cover the difference between `apps` and `packages` later in this chapter.
