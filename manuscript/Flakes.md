# Flakes

This book has leaned pretty heavily on the Nix command line's support for "flakes", but I've glossed over the details of how flakes work despite how much we've already been using them.  In this chapter I'll give a more patient breakdown of how flakes work and why we use them.

Most of what this chapter will cover is information that you can already find from other resources, like the [NixOS Wiki page on Flakes](https://nixos.wiki/wiki/Flakes#Flake_schema) or by running `nix help flake`.  However, I'll still try to explain flakes in my own words and also include my own commentary on "dos" and "don'ts" of using flakes.

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

In the above command `.#packages.x86_64-linux.default` uniquely identifies a "flake reference" and an attribute path, which are separated by a `#` character:

- The first half (the flake reference) specifies where a flake is located

  In our above example the flake reference is `.` (a shorthand for our current directory).

- The second half (the attribute path) specifies which output attribute to use

  In the above example, it is `packages.x86_64-linux.default` and `nix run` uses that output attribute path to select which executable to run.

Usually we don't want to write out something like `.#packages.x86_64-linux.default` when we use flakes, so flake-enabled Nix commands provide a few convenient shorthands that we can use to shorten the attribute path.

First off, many Nix commands will automatically expand part of the flake attribute path on your behalf.  For example, if you run:

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

You can even omit the attribute path, in which case it will default to an attribute path of `default`.  For example, if you run:

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

So far these examples have only used a flake reference of `.` (the current directory), but the ones we'll be focusing on in this book are:

- paths

  These can be relative paths (like `.` or `./utils` or `../bar`), home-anchored paths (like `~/workspace`), or absolute paths (like `/etc/nixos`).

- GitHub URIs

  These take the form `github:${OWNER}/${REPOSITORY}` or `github:${OWNER}/${REPOSITORY}/${REFERENCE}` (where `${REFERENCE}` can be a branch, tag, or revision).  Nix will take care of cloning the repository for you in a cached and temporary directory and (by default) look for a `flake.nix` file within the root directory of the repository.

- indirect URIs

  An indirect URI is one that refers to an entry in Nix's "flake registry".  If you run `nix registry list` you'll see a list of all your currently configured indirect URIs.

### Flake inputs

Normally the way flakes work is that you specify both inputs and outputs, like this:

```nix
{ inputs = {
    foo.url = "${FLAKE_REFERENCE}";
    bar.url = "${FLAKE_REFERENCE}";
  };

  outputs = { self, foo, bar }: {
    baz = …;
    qux = …;
  };
}
```

In the above example, `foo` and `bar` would be the flake inputs while `baz` and `qux` would be the flake outputs.  In other words, the sub-attributes nested underneath the `inputs` attribute are the flake inputs and the attributes generated by the `outputs` function are the flake outputs.

Notice how the `outputs` function takes input arguments which share the same name as the flake inputs because the flakes machinery resolves each input and then passes each resolved input as a function argument of the same name to the `outputs` function. To illustrate this, if you were to build the `baz` output of the above flake using:

```bash
$ nix build .#baz
```

… then that would sort of be like building this Nix pseudocode:

```nix
let
  flake = import ./flake.nix;

  foo = resolveFlakeURI flake.inputs.foo;

  bar = resolveFlakeURI flake.inputs.baz;

  self = flake.outputs { inherit self foo bar; };

in
  self.baz
```

… where `resolveFlakeURI` would be sort of like a function from an input's flake reference to the Nix code packaged by that flake reference.

However, if you were paying close attention you might have noticed that our original example flake does not have any `input`s:

```nix
{ outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
  };
}
```

… and the `outputs` function references a `nixpkgs` input which we never specified.  The reason this works is because flakes automatically convert missing inputs to "indirect" URIs that are resolved using Nix's flake registry.  In other words, it's as if we had written:

```nix
{ inputs = {
    nixpkgs.url = "nixpkgs";  # Example of an indirect flake reference
  };

  outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
  };
}
```

An indirect flake reference is resolved by doing a lookup in the flake registry, which you can query yourself like this:

```bash
$ nix registry list | grep nixpkgs
global flake:nixpkgs github:NixOS/nixpkgs/nixpkgs-unstable
```

… so we could have also written:

```nix
{ inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable";
  };

  outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
  };
}
```

… which would have produced the same result: both flake references will attempt to fetch the `nixpkgs-unstable` branch of the `nixpkgs` repository to resolve the `nixpkgs` flake input.

{blurb, class:information}
Throughout the rest of this chapter (and book) I'm going to try to make flake references as pure as possible, meaning:

- no indirect flake references

  In other words, instead of `nixpkgs` I'll use `github:NixOS/nixpkgs/23.05`.


- all GitHub flake references will include a tag

  In other words, I won't use a flake reference like `github:NixOS/nixpkgs`.

Neither of these precautions are strictly necessary when using flakes because flakes lock their dependencies using a `flake.lock` file which you can (and should) store in version control.  However, the inline flake examples in this book can't reasonably include the `flake.lock` file, so the above precautions ensure that the inline examples are more reproducible.

Also, it's a good idea to take these precautions anyway even if you can include the `flake.lock` file alongside your `flake.nix` file.  The more reproducible your flake references, the better you document how to regenerate or update your lock file.
{/blurb}

Suppose we wanted to use our own local `git` checkout of `nixpkgs` instead of a remote `nixpkgs` branch: we'd have to change the `nixpkgs` input to our flake to reference the path to our local repository (since paths are valid flake references), like this:

```nix
{ inputs = {
    nixpkgs.url = ~/repository/nixpkgs;
  };

  outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
  };
}
```

… and then we also need to build the flake using the `--impure` flag:

```bash
$ nix build --impure
```

Without the flag we'd get this error message:

```
error: the path '~/proj/nixpkgs' can not be resolved in pure mode
```

{blurb, class:information}
Notice that we're using flake references in two separate ways:

- on the command line

  e.g. `nix build "${FLAKE_REFERENCE}"`

- when specifying flake inputs

  e.g. `inputs.foo.url = "${FLAKE_REFERENCE}";`

However, they differ in two important ways:

- command line flake references never require the `--impure` flag

  In other words, `nix build ~/proj/nixpkgs#foo` is fine, but if you specify `~/proj/nixpkgs` as a flake references then you have to add the `--impure` on the command line.

- flake inputs can be specified as attribute sets instead of strings

  That means that instead of specifying a flake reference as a string you can specify it as structured attribute set.

To elaborate on the latter point, instead of specifying a `nixpkgs` input  like this:

```nix
{ inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs";
  };

  …
}
```

We could instead specify the same input like this:

```nix
{ inputs = {
    type = "github";

    owner = "NixOS";

    repo = "nixpkgs";
  };

  …
}
```

Throughout this book I'll consistently use the non-structured (string) representation for flake references to simplify things.
{/blurb}

### Flake outputs

We haven't yet covered what we actually *get* when we resolve a flake input.  For example, what Nix expression does a flake reference like `github:NixOS/nixpkgs/23.05` resolve to?

The answer is that a flake reference will resolve to the output attributes of the corresponding flake.  For a flake like `github:NixOS/nixpkgs/23.05` that means that Nix will:

- clone [the `nixpkgs` repository](https://github.com/NixOS/nixpkgs)

- check out [the `23.05` tag](https://github.com/NixOS/nixpkgs/tree/23.05) of that repository

- look for [a `flake.nix`](https://github.com/NixOS/nixpkgs/blob/23.05/flake.nix) file in the top-level directory of that repository

- resolve inputs for that flake

  There are none to resolve.

- look for [an `outputs` attribute](https://github.com/NixOS/nixpkgs/blob/23.05/flake.nix#L6), which will be a function

- computed the fixed-point of that function

- return that fixed-point as the result

  In this case the result would be an [attribute set](https://github.com/NixOS/nixpkgs/blob/23.05/flake.nix#L16-L74) containing five attributes: `lib`, `checks`, `htmlDocs`, `legacyPackages`, and `nixosModules`.

In other words, it would behave like this (non-flake-enabled) Nix code:

```nix
# nixpkgs.nix

let
  pkgs = import <nixpkgs> { };

  nixpkgs = pkgs.fetchFromGitHub {
    owner = "NixOS";
    repo = "nixpkgs";
    rev = "23.05";
    hash = "sha256-btHN1czJ6rzteeCuE/PNrdssqYD2nIA4w48miQAFloM=";
  };

  flake = import "${nixpkgs}/flake.nix";

  self = flake.outputs { inherit self; };

in
  self
```

… except that with flakes we wouldn't have to figure out what hash to use since that would be transparently managed for us by the flake machinery.

If you were to load the above file into the REPL:

```bash
$ nix repl --file nixpkgs.nix
```

… you would get the exact same result as if you had loaded the equivalent flake into the REPL:

```bash
$ nix repl github:NixOS/nixpkgs/23.05
```

In both cases the REPL would now have the `lib`, `checks`, `htmlDocs`, `legacyPackages`, and `nixosModules` attributes in scope since those are the attributes returned by the `outputs` function:

```bash
nix-repl> legacyPackages.x86_64-linux.hello
«derivation /nix/store/zjh5kllay6a2ws4w46267i97lrnyya9l-hello-2.12.1.drv»
```

This `legacyPackages.x86_64-linux.hello` attribute path is the same attribute path that our original flake output uses:

```nix
{ …

  outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
                                          # ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  };
}
```

There's actually one more thing you can do with a flake, which is to access the original path to the flake.  Flakes have an `outPath` attribute


### Platforms

All of the above examples hard-coded a single system (`x86_64-linux`), but usually you want to support building a package for multiple systems.  People typically use the `flake-utils` flake for this purpose, which you can use like this;

```nix
{ inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/23.05";

    flake-utils.url = "github:numtide/flake-utils/v1.0.0";
  };

  outputs = { flake-utils, nixpkgs, ... }:
    flake-utils.lib.eachDefaultSystem (system: {
      packages.default = nixpkgs.legacyPackages."${system}".hello;
    });
}
```

… and that is essentially the same thing as if we had written:

```nix
{ inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/23.05";

    flake-utils.url = "github:numtide/flake-utils/v1.0.0";
  };

  outputs = { nixpkgs, ... }: {
    packages.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.hello;
    packages.aarch64-linux.default = nixpkgs.legacyPackages.aarch64-linux.hello;
    packages.x86_64-darwin.default = nixpkgs.legacyPackages.x86_64-darwin.hello;
    packages.aarch64-darwin.default = nixpkgs.legacyPackages.aarch64-darwin.hello;
  };
}
```

[^1]: We'll cover the difference between `apps` and `packages` later in this chapter.
