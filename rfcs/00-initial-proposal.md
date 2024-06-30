# Initial Proposal for `flake-parts-cli`

## Background

`flake-parts` modularizes project `flake.nix` ([example](https://github.com/srid/haskell-template/blob/master/flake.nix)) sufficiently enough to make it almost look like YAML, due to the way the module system works. Developers set "options", with config implementation taken care of by upstream modules ([such as these](https://community.flake.parts/modules)).

Can we go one step further and substantially improve the DX by allowing developers to achieve typical Nix uses cases without having to write Nix? 

## Approach

To achieve this, we propose a CLI that takes as input some YAML or TOML (file formats familiar to many people) and "adjust" (if not auto-generate) the top-level `flake.nix`.

## Spec & Implementation

While the details of the spec & implementation are to be worked out, here's a good starting point:

Let's say our CLI generates an initial `flake.nix` with "placeholder" comments:

```nix
{
  inputs = {
    # START inputs managed by flake-parts cli
    <generate whatever we want here>
    # END inputs managed by flake-parts cli
  }:
  outputs = inputs: inputs.flake-parts.mkFlake {
    inherit inputs;
    # A toml file is easier to manage
    specialArgs.flake-parts-toml = ./nix/flake-parts.toml;
    modules = [
       # and this module could translate the toml into module definitions
      inputs.flake-parts-cli.flakeModule
    ];
  };
}
```

This flake points to a TOML file that, for a Haskell project, may look like this:

```toml
# ./nix/flake-parts.toml
# flake-parts TOML

[flake.registry]
url = "https://github.com/juspay/flake-registry"

[flake.inputs]
nixpkgs.url = "nixpkgs"
haskell-flake.url = "haskell-flake"
euler-hs.url = "github:juspay/euler-hs"
euler-hs.follow-auto = ["nixpkgs", "haskell-flake"]

[flake.parts]
imports = "haskell-flake"

[flake.parts.haskellProjects.default]
projectRoot = "."
[flake.parts.haskellProjects.default.settings]
euler-hs.check = false
[flake.parts.haskellProjects.default.defaults.settings.local]
buildFromSdist = false
haddock = true

[flake.outputs.packages]
default = "${self'.packages.hello-world}"

[flake.outputs.devShells.default]
packages = ["${pkgs.haskellPackages.ghcid}", "${pkgs.node}"]
```

Now, the user can simply edit the TOML file (`./nix/flake-parts.toml`) to make any changes, without writing a line of Nix. The `flake-parts-cli` app will support commands for "adding" or "removing" modules, which automatically will modify the `inputs` part of the flake. Developers can then configure their new modules in the TOML file. The CLI can also wrap the various Nix commands like `build`, `develop` and so forth, whilst even being ambitious enough to provide custom subcommands to be specified by upstream modules (for e.g., [services-flake](https://github.com/juspay/services-flake) can provide custom subcommands to add/remove services).

Developers can write custom modules and have them imported in top-level flake for those (rare) advanced use-cases; thus they don't have to move away from being able to use the full power of Nix.
