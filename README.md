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

`bubblebox` is currently developed and tested on **Fedora Atomic**, specifically the **Fedora COSMIC Atomic Spin**.

It should work across the broader Fedora Atomic family, including **Universal Blue** distributions such as [Bazzite](https://github.com/ublue-os/bazzite), [Bluefin](https://github.com/ublue-os/bluefin), and [Aurora](https://github.com/ublue-os/aurora).

It should also be applicable to other immutable Linux distributions such as **openSUSE Aeon, Kalpa, and MicroOS**.

Despite being designed with immutable systems in mind, there is nothing particularly Atomic-specific about the underlying approach. It should also work on traditional mutable distributions such as **Fedora, Ubuntu, Gentoo**, and others.

It should even work on **NixOS**.

Why you would want to do this on NixOS? 

### Shells

The current setup is **Bash-only**.

I use Bash, and that is what I have tested. I have no idea yet how well this translates to **Zsh, Fish, or other shells**, particularly where shell initialization, completions, and environment hooks are concerned.

Contributions and experimentation in this area are very welcome.

### Security

The setup has so far been tested without encountering issues related to **SELinux or AppArmor**.

That said, security policies and configurations can vary significantly between distributions and installations, so this should not be interpreted as a guarantee that every SELinux or AppArmor configuration will behave identically.

If you run into something interesting, please open an issue.


## Future?

There is still plenty of unnecessary nonsense to build on top of this. Naturally, I intend to do so.

Some things I'd like to explore:

* **Bash completions** — because I don't wanna layer anything other than [keyd](https://github.com/rvaiya/keyd) on my system
* **Other shells** — particularly Zsh and Fish.
* **Alternative package managers and development tools** — such as [uv](https://github.com/astral-sh/uv), Cargo, and others.
* **Better integration** — while keeping the core idea small and user-space focused.

It's mostly to see how far this little pile of Bubblewrap, namespaces, Nix, shell scripts, and questionable decisions can go.


## Inspiration

This project would not exist without the work that came before it.

The original inspiration came from  Ivan Molodetskikh aka [YaLTeR](https://github.com/YaLTeR) and his article:

[Easy Sandboxing on Linux with Bubblewrap](https://bxt.rs/blog/easy-sandboxing-on-linux-with-bubblewrap/)

That article demonstrated how Bubblewrap can be used to construct a practical, low-friction sandbox from ordinary Linux filesystem primitives. The ideas in that article were the starting point for experimenting with a similar approach for a user-space Nix environment.

A huge thank-you to YaLTeR for writing that article and showing how approachable Bubblewrap-based sandboxing can be.

## Built on the work of others

bubblebox stands on top of several projects and technologies that do the difficult parts:

[bubblewrap](https://github.com/containers/bubblewrap) — provides the unprivileged sandboxing and namespace machinery used to construct the environment.
[Nix](https://nixos.org/) — provides the package management, store, profiles, and development environment machinery.
Linux namespaces — provide the kernel-level isolation primitives on which container and sandbox technologies such as Bubblewrap are built.
[Linux kernel](https://github.com/torvalds/linux) — namespaces, mount namespaces and the filesystem machinery underneath them.
See [`fs/namespace.c`](https://github.com/torvalds/linux/blob/master/fs/namespace.c).

This project is an application of those technologies, not an attempt to re-implement them.

Acknowledgements

Special thanks to:

Ivan Molodetskikh (YaLTeR) — for the original Bubblewrap sandboxing article that inspired this project.
The Bubblewrap contributors — for building and maintaining the sandboxing tool that makes this approach possible.
The Linux kernel and its contributors — for the namespace and filesystem primitives underneath the entire stack.
The Nix community and contributors — for building the package-management and development-environment machinery used here.
Everyone who has written documentation, examples, experiments, and strange little shell scripts that make Linux easier to understand.
