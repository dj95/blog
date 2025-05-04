---
title: "Debugging another SIGABRT in neovim"
date: 2022-05-22T12:05:02+02:00
draft: false
tags: ["neovim", "nix", "debugging"]
---

In the last blog article I described a way how to debug SIGABRT crashes in neovim with the help of a nix shell.
It helped me to find out the cause of the crashes in neovim 0.5.0 and an already existing issue in the github project, which already provided a solution.
Unfortunately, since a few weeks, I experience crashes in neovim again.
Most of the time when using telescope with the preview, neovim randomly crashes.
However, when running `:checkhealth`, the crash occurs in the same way.

In order to debug the issue, the first approach would be to check the `Console` application since I'm using macOS.
The stacktrace from `Console` should provide enough information to find the root cause of the crashes.
Unlike the last error that cause SIGABRT crashes, the stacktrace does not useful information, as the following screenshot depicts.

![crash report](/img/debugging-another-sigabrt-in-neovim/console.png)

This requires another approach to find out more debug information.
For this, I've created another `shell.nix` with the current version of neovim and a debugging build of the current released version and the latest nightly version.
Since the error also occurs in the nightly version, debugging in it is more useful since it simplifies contributing a fix to the neovim project.
The following `shell.nix` will compile the current nightly on macOS.

```nix
{ pkgs ? import <nixpkgs> { 
  overlays = [
    (self: super: {
      neovim-dev = (super.pkgs.neovim-unwrapped.override {}).overrideAttrs(oa:{
        # use the latest stable neovim
        version = "master";

        src = pkgs.fetchFromGitHub {
          owner = "neovim";
          repo = "neovim";
          rev = "master";
          sha256 = "YfPM6yjKWu4RPAtLefggWC0ZOGgbcPsk6VT5M3D41B0=";
        };

        # use the debug type to enable debug symbols
        cmakeBuildType="debug";

        buildInputs = super.pkgs.neovim-unwrapped.buildInputs ++ [
          pkgs.darwin.apple_sdk.frameworks.CoreServices
        ];

        shellHook = ''
          export NVIM_PYTHON_LOG_LEVEL=DEBUG
          export NVIM_LOG_FILE=/tmp/log
        '';
      });
    })
  ];
}}:

pkgs.mkShell {
  nativeBuildInputs = with pkgs; [
    neovim-dev
  ];
}
```

Starting neovim with `nvim -u NONE` for starting with an empty config and running the healthcheck will result in the SIGABRT again.
Since the debugging build does not provide a stacktrace with more information, we need to use `lldb`, a debugger on macOS, to further find the cause.
Therefore we need to start the process with `lldb nvim`.
This starts `lldb` with `nvim` as target and provides an interactive shell.

Starting neovim with `run` (or `r` if you're lazy like me) will execute neovim and we are able to reproduce the crash again.
After neovim crashes, `lldb` provides short trace where the program actually crashed.
Since we want to analyze the complete stacktrace, we need to run `bt`, which provides a complete stacktrace, that is depicted in the following screenshot.

![backtrace](/img/debugging-another-sigabrt-in-neovim/bt.png)

Analyzing the stacktrace, we are able to examine the last function in neovim, that was executed: `nvim'tslua_add_language`.
This function can be found in a file called `treesitter.c` and is responsible for the treesitter integration.
Due to the name of the function and its content, we can examine, that it loads configured and already existing parsers.

When we further anlyze the stacktrace and watch out for functions that do not belong to the standard libraries, we are able to find the `markdown.so'_GLOBAL__sub_I_scanner.c`.
It provides a hint to the markdown treesitter parser, which actually consists of a `scanner.cc` file.

In order to test, if the parser causes the crashes, we uninstall it with `:TSUninstall markdown` and the run `:checkhealth` agian.
Suprisingly, neovim does not cash anymore!
As a quick fix, we can now remove the parser from the configuration and neovim should (hopefully) run stable again.
