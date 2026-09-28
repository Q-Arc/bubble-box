# Bubblebox

An approach to running Nix package manager on any Linux distro without needing sudo and keeping it all contained inside your $HOME

## Overview

This project provides a robust method for installing and managing Nix packages on Fedora Atomic distributions without modifying the base system. By leveraging Bubblewrap (`bwrap`) for containerization, we achieve:

- **System Integrity**: The immutable `/usr` partition remains untouched
- **User Isolation**: All Nix operations occur in a sandboxed environment
- **Host Integration**: Nix binaries seamlessly integrate with the host `$PATH`
- **Security**: Minimal privilege escalation with controlled filesystem access
- **Flexibility**: Full Nix ecosystem support including flakes and profiles

## Architecture

### Core Components

| Component | Purpose |
|-----------|---------|
| `nix-shell-env` | Interactive sandboxed shell for Nix operations |
| `nix-exec` | Transparent wrapper for executing individual Nix binaries |
| Bubblewrap | Lightweight sandboxing runtime |
| Nix Profile | Per-user package installation (no daemon required) |

### Security Model

The setup employs multiple security layers:
- Read-only binding of system directories (`/usr`, `/etc`, `/run`)
- Temporary filesystem root with selective bind mounts
- User-owned Nix store in `$HOME/.nix`
- Network sharing only when explicitly required
- No system-wide daemon (single-user mode)

## Prerequisites

- Bubblewrap (usually already present in most distros if you have flatpaks enabled as well)
- The location .local/bin being in $PATH
- 
That's it, I guess. 

## Installation & Setup

1. Create scripts & directories:
```
   mkdir -p ~/.nix ~/.local/bin
```
2. Save nix-shell-env & nix-exec, then make executable:
```
   chmod +x ~/.local/bin/nix-shell-env ~/.local/bin/nix-exec
```

4. Launch nix-shell-env and run single-user installer:
```
   curl --proto '=https' --tlsv1.2 -sSf -L https://nixos.org/nix/install | sh -s -- --no-daemon
```
5. Enable Flakes (inside nix-shell-env):
```
   mkdir -p ~/.config/nix
   echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf
```
## Workflow

Installing packages inside Nix:
```
   nix-shell-env
   nix profile install nixpkgs#<package_name>
   exit
```

Exposing binaries to host ($PATH):
```
   rm -f ~/.local/bin/<tool>
   ln -s ~/.local/bin/nix-exec ~/.local/bin/<tool>
```

Garbage collection / Cleaning disk space:

```
   nix-shell-env
   nix store gc
```

## Caveats 

The following is needed to be added to the .bashrc to make direnv work

```
if command -v direnv &>/dev/null && command -v bwrap &>/dev/null; then
  eval "$(direnv hook bash | sed "s|/nix/store/[^/]*/bin/direnv|$HOME/.local/bin/direnv|g")"
fi
```

And due to the way that the bubblewrap environments are created currently, things like eza always show you $HOME rathrr than the current work directory - $PWD. Working on it... 

And bash completions don't work currently... 
Have to look into where and how nix profile deals with bash completions to figure something out... 

## Practical Situation 

Have no idea how this would work for zsh, fish and other shells because I only have bash and have used bash.
Tested on Fedora Cosmic Atomic Spin though it should work on all Fedora Atomic offerings.
Works on the Universal Blue (uBlue) distros like Bazzite, Bluefin and Aurora. 
It should also work on OpenSUSE Aeon, Kalpa and MicroOS.
It should also work on the average mutable distro as well so basically normal Fedora, Ubuntu, Gentoo, etc. 
It should also work on NixOS as well but why do this.
Should have no issues with SELinux or AppArmor as well, at least I haven't encountered any in my testing so far.

## Future?

Probably to get bash completions working and to see how I can make it work with other package managers like UV, Cargo, etc because why not.

## Thank You to -

https://github.com/YaLTeR because this blog of his inspired this entire thing - 
https://bxt.rs/blog/easy-sandboxing-on-linux-with-bubblewrap/
bubblewrap itself and the people who came up with namespaces and eveyrthing else
