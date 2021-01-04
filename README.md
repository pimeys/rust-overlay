# rust-overlay

![CI](https://github.com/oxalica/rust-overlay/workflows/CI/badge.svg)
![sync-channels](https://github.com/oxalica/rust-overlay/workflows/sync-channels/badge.svg)

*Pure and reproducible* overlay for binary distributed rust toolchains.
A compatible but better replacement for rust overlay of [mozilla/nixpkgs-mozilla][mozilla].

Hashes of toolchain components are pre-fetched (and compressed) in tree (`manifests` directory),
so the evaluation is *pure* and no need to have network (but [nixpkgs-mozilla][mozilla] does).
It also works well with [Nix Flakes](https://nixos.wiki/wiki/Flakes).

- The toolchain hashes are auto-updated daily using GitHub Actions.
- Current oldest supported version is stable 1.29.0 and beta/nightly 2018-09-13
  (which are randomly chosen).

## Use as a classic Nix overlay

You can put the code below into your `~/.config/nixpkgs/overlays.nix`.
```nix
[ (import (builtins.fetchTarball "https://github.com/oxalica/rust-overlay/archive/master.tar.gz")) ]
```
Then the provided attribute paths are available in nix command.
```bash
$ nix-env -iA rust-bin.stable.latest.rust # Do anything you like.
```

Alternatively, you can install it into nix channels.
```bash
$ nix-channel --add https://github.com/oxalica/rust-overlay/archive/master.tar.gz rust-overlay
$ nix-channel --update
```
And then feel free to use it anywhere like
`import <nixpkgs> { overlays = [ (import <rust-overlay>) ]; }` in your nix shell environment.

## Use with Nix Flakes

This repository already has flake support.

NOTE: **Only the output `overlay` is stable and preferred to be used in your flake.**
Other outputs like `packages` and `defaultPackage` are for human try and are subject to change.

For a quick play, just use `nix shell` to bring the latest stable rust toolchain into scope.
(All commands below requires preview version of Nix with flake support.)
```shell
$ nix shell github:oxalica/rust-overlay
$ rustc --version
rustc 1.49.0 (e1884a8e3 2020-12-29)
$ cargo --version
cargo 1.49.0 (d00d64df9 2020-12-05)
```

Here's an example of using it in nixos configuration.
```nix
{
  description = "My configuration";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    rust-overlay.url = "github:oxalica/rust-overlay";
  };

  outputs = { nixpkgs, rust-overlay, ... }: {
    nixosConfigurations = {
      hostname = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          ./configuration.nix # Your system configuration.
          ({ pkgs, ... }: {
            nixpkgs.overlays = [ rust-overlay.overlay ];
            environment.systemPackages = [ pkgs.rust-bin.stable.latest.rust ];
          })
        ];
      };
    };
  };
}
```

## Attributes provided by the overlay

```nix
{
  rust-bin = {
    # The default dist url for fetching.
    # Override it if you want to use a mirror server.
    distRoot = "https://static.rust-lang.org/dist";

    # Select a toolchain and aggregate components by rustup's `rust-toolchain` file format.
    # See: https://github.com/ebroto/rustup/blob/c2db7dac6b38c99538eec472db9d23d18f918409/README.md#the-toolchain-file
    fromRustupToolchain = { channel, components ? [], targets ? [] }: «derivation»;
    # Same as `fromRustupToolchain` but read from a `rust-toolchain` file (legacy one-line string or in TOML).
    fromRustupToolchainFile = rust-toolchain-file-path: «derivation»;

    stable = {
      # The latest stable toolchain.
      latest = {
        # Aggregate all default components. (recommended)
        rust = «derivation»;
        # Individial components.
        rustc = «derivation»;
        cargo = «derivation»;
        rust-std = «derivation»;
        # ... other components
      };
      "1.49.0" = { /* toolchain */ };
      "1.48.0" = { /* toolchain */ };
      # ... other versions.
    };

    beta = {
      # The latest beta toolchain.
      latest = { /* toolchain */ };
      "2021-01-01" = { /* toolchain */ };
      "2020-12-30" = { /* toolchain */ };
      # ... other versions.
    };

    nightly = {
      # The latest nightly toolchain.
      latest = { /* toolchain */ };
      "2020-12-31" = { /* toolchain */ };
      "2020-12-30" = { /* toolchain */ };
      # ... other versions.
    };

    # ... Some internal attributes omitted.
  };

  # These are for compatibility with nixpkgs-mozilla and
  # provide same toolchains as `rust-bin.*`.
  latest.rustChannels = /* ... */;
  rustChannelOf = /* ... */;
  rustChannelOfTargets = /* ... */;
  rustChannels = /* ... */;
}
```

Some examples (assume `nixpkgs` had the overlay applied):

- Latest stable/beta/nightly rust with all default components:
  `nixpkgs.rust-bin.{stable,beta,nightly}.latest.rust`
- A specific version of stable rust:
  `nixpkgs.rust-bin.stable."1.48.0".rust`
- A specific date of beta rust:
  `nixpkgs.rust-bin.nightly."2021-01-01".rust`
- A specific version of stable rust:
  `nixpkgs.rust-bin.stable."1.48.0".rust`
- A specific date of nightly rust:
  `nixpkgs.rust-bin.nightly."2020-12-31".rust`
- Latest stable rust with additional component `rust-src` and extra target
  `arm-unknown-linux-gnueabihf`:

  ```nix
  nixpkgs.rust-bin.stable.latest.rust.override {
    extensions = [ "rust-src" ];
    targets = [ "arm-unknown-linux-gnueabihf" ];
  }
  ```
- If you already have a [`rust-toolchain` file for rustup](https://github.com/ebroto/rustup/blob/c2db7dac6b38c99538eec472db9d23d18f918409/README.md#the-toolchain-file),
  you can simply use `fromRustupToolchainFile` to get the customized toolchain derivation.

  ```nix
  nixpkgs.rust-bin.fromRustupToolchainFile ./rust-toolchain
  ```

For more details, see also the source code of `./rust-overlay.nix`.

[mozilla]: https://github.com/mozilla/nixpkgs-mozilla
