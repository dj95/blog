---
title: "Debugging neovim with nix shell"
date: 2021-10-09T12:05:02+02:00
draft: false
tags: ["neovim", "nix", "debugging"]
---

Since the introduction of the new lua api in neovim 0.5.0 there are plenty new vim plugins based on this api.
While testing some of the new plugins, my neovim exited with a `SIGSEGV` and sometime with a `SIGABRT` in specific workflows.
As these crashes are reproducible, it should be easy to debug the problem.
However, we need a debug build of neovim and, since I'm using macOS, the stack trace of neovim.

macOS offers an application called `Console`, which captures and stores all stack traces of crashed applications.
After opening the application, we should find the neovim crashes under the `Crash Reports` tab.

![crash report](/img/debug-nvim-with-nix/console.png)

Depending on the installation, these crash reports might already contain debug symbols which allows me to examine the root cause.
If this is not the case or you want to dig deeper and step through neovim with a debugger, to completely understand the root cause and not only the function in which the error occurs, you need a debug build of neovim.


Since I do not want to use the debug build as my daily driver and simplify the build process to quickly spin up a debug build in case another error occurs, I utilize [nix](://nixos.org).
Nix enables a declarative way to specify installed packages and allows me to override attributes in these packages.
In addition, with nix these packages can be accesses in a separate shell environment instead of installing them in the `$PATH` of the system.

For this shell, a `shell.nix` file is required, that specifies the required environment.
Hence I want to debug neovim, it contains the installation candidate for neovim.
However, the neovim package is extended and configured with an overlay.
This overlay overrides the version of neovim and enables the debugging build with `cmakeBuildType="debug"`.
The following code block depicts the `shell.nix` that can be used to debug neovim.


```nix
{ pkgs ? import <nixpkgs> { 
  overlays = [
    (self: super: {
      neovim-dev = (super.pkgs.neovim-unwrapped.override {}).overrideAttrs(oa:{
        # use the latest stable neovim
        version = "0.5.0";

        src = pkgs.fetchFromGitHub {
          owner = "neovim";
          repo = "neovim";
          rev = "v0.5.0";
          sha256 = "0aXo8f1YGEI8PkLDOSgxqQy7qK031R+eapCppUFy61E";
        };

        # use the debug type to enable debug symbols
        cmakeBuildType="debug";

        shellHook = ''
          export NVIM_PYTHON_LOG_LEVEL=DEBUG
          export NVIM_LOG_FILE=/tmp/log
        '';
      });
    })
  ];
}}:

pkgs.mkShell {
  nativeBuildInputs = [
    pkgs.neovim-dev
  ];
}
```

When `nix-shell` is exexuted in the same directory as the preceeding `shell.nix`, neovim will be built with the specified configuration.
To verify that nix-shell build the debug vesion, `nvim --version` should show `debug` as the build type.

```shell
$ nvim --version
NVIM v0.5.1
Build type: Release
LuaJIT 2.1.0-beta3
Compilation: 
Compiled by _nixbld1

$ nix-shell
# ... some build output

[nix-shell] $ nvim --version
NVIM v0.5.1
Build type: debug
LuaJIT 2.1.0-beta3
Compilation: 
Compiled by daniel
```

