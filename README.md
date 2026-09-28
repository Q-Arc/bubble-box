# Bubblebox

> A user-space development environment powered by Bubblewrap.

**Bubblebox** is an approach to running the [Nix package manager](https://nixos.org/) on Linux without needing `sudo` or modifying the host system, while keeping the entire Nix installation inside your `$HOME`.

It started as an experiment on Fedora Atomic:

> *Can I get a proper Nix development environment without layering Nix onto an immutable system?*

Apparently, yes.

The basic idea is to use [Bubblewrap](https://github.com/containers/bubblewrap) to construct a temporary filesystem namespace, give Nix the filesystem layout it expects, and keep the persistent parts of the installation in the user's home directory.

The goal is simple:

**Keep the host clean. Put the weird stuff in the bubble.** 🫧

---

## Why?

Immutable Linux systems are great until you want to install some random development tool that expects to live in `/usr`, `/opt`, or somewhere else the operating system doesn't particularly want you touching.

Nix already solves a large part of this problem by providing its own package store and user profiles. The annoying part is getting Nix itself onto a system where modifying the base filesystem isn't desirable.

Bubblebox puts the two together:

```
┌──────────────────────────────────────────┐
│                 Linux Host               |
│                                          |
│  /usr       /etc       /run       /var   |
│   │           │          │          │    | 
│   └───────────┴───────┐──┴──────────┴────┐
│                       |                  │
│                 Bubblewrap               │
│                       |                  │
│  ┌────────────────────────────────────┐  │
│  │          Nix Environment           │  │
│  │                                    │  │
│  │  /nix  →  ~/.nix                   │  │
│  │  /home →  /var/home                │  │
│  │  temporary filesystem root         │  │
│  │                                    │  │
│  │  Nix packages / profiles / flakes  │  │
│  └────────────────────────────────────┘  │
│                                          │
└──────────────────────────────────────────┘
```

The host remains untouched while Nix gets the environment it needs.

---

## Architecture

Bubblebox currently consists of two small scripts:

| Component       | Purpose                                                      |
| --------------- | ------------------------------------------------------------ |
| `nix-shell-env` | Starts an interactive sandboxed shell for Nix operations     |
| `nix-exec`      | Transparently executes individual Nix binaries from the host |
| Bubblewrap      | Provides the sandbox and namespace machinery                 |
| Nix Profile     | Provides per-user package installation without a daemon      |

### Security model

The current environment uses several layers of filesystem isolation:

* `/usr`, `/etc`, and `/run` are exposed read-only.
* The sandbox starts with a temporary filesystem root.
* Only selected parts of the host filesystem are bind-mounted into the environment.
* The Nix store lives inside the user's home directory at `~/.nix`.
* The network is shared with the host where required.
* Nix runs in single-user mode without a system-wide daemon.
* No root privileges are required for the Nix installation itself.

This is not intended to be a replacement for a full container runtime or a security boundary for hostile workloads.

It is primarily a way of constructing a useful **user-space development environment** on systems where modifying the host filesystem is undesirable.

---

## Prerequisites

You need:

* [Bubblewrap](https://github.com/containers/bubblewrap)
* `~/.local/bin` somewhere in your `$PATH`
* A Linux system

That's it, I guess.

Bubblewrap is already present on many desktop Linux systems, particularly if you use Flatpaks.

---

## Installation

### 1. Create the required directories

```bash
mkdir -p ~/.nix ~/.local/bin
```

### 2. Install the scripts

Place `nix-shell-env` and `nix-exec` in:

```text
~/.local/bin/
```

Then make them executable:

```bash
chmod +x ~/.local/bin/nix-shell-env ~/.local/bin/nix-exec
```

### 3. Enter the environment

Launch the Bubblewrap environment:

```bash
nix-shell-env
```

Then install Nix in single-user mode:

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://nixos.org/nix/install | sh -s -- --no-daemon
```

> **Note:** As always with `curl | sh`, read the script before running it if you are uncomfortable executing remotely fetched code. The command above is the standard Nix installation method used by this setup.

### 4. Enable flakes

Inside `nix-shell-env`:

```bash
mkdir -p ~/.config/nix
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf
```

And that's the basic setup.

---

## Using Nix

Enter the environment:

```bash
nix-shell-env
```

Install packages through a Nix profile:

```bash
nix profile install nixpkgs#<package_name>
```

For example:

```bash
nix profile install nixpkgs#ripgrep
```

Then leave the environment:

```bash
exit
```

### Exposing binaries to the host

`nix-exec` can be used as a wrapper so that individual Nix-installed programs can be invoked directly from the host.

For example:

```bash
rm -f ~/.local/bin/rg
ln -s ~/.local/bin/nix-exec ~/.local/bin/rg
```

Now running:

```bash
rg
```

will cause the wrapper to execute the corresponding Nix profile binary inside the Bubblewrap environment.

The result is effectively:

```text
Host shell
    │
    ▼
~/.local/bin/rg
    │
    ▼
nix-exec
    │
    ▼
Bubblewrap
    │
    ▼
~/.nix-profile/bin/rg
```

The host gets the command.

Nix gets to keep doing Nix things.

Everyone is happy.

Probably.

---

## Garbage collection

Because the Nix store lives in the user's home directory, you can clean unused store paths with:

```bash
nix-shell-env
nix store gc
```

This is especially useful if you've been installing packages purely because you wondered what they were.

You know who you are.

---

## Caveats

There are currently a few rough edges.

### `direnv`

To make `direnv` work with the way Nix binaries are exposed, the following needs to be added to `.bashrc`:

```bash
if command -v direnv &>/dev/null && command -v bwrap &>/dev/null; then
    eval "$(direnv hook bash | sed "s|/nix/store/[^/]*/bin/direnv|$HOME/.local/bin/direnv|g")"
fi
```

Yes, this uses `sed` inside `eval`.

No, I am not proud of this.

Yes, I have a story about this.

### Bash completions

Bash completions do not currently work correctly.

This is something I still need to investigate, particularly how Nix profiles expose and manage completion files.

---

## Practical Situation

Bubblebox is currently developed and tested on **Fedora Atomic**, specifically the **Fedora COSMIC Atomic Spin**.

It should work across the broader Fedora Atomic family, including [Universal Blue](https://universal-blue.org/) distributions such as [Bazzite](https://bazzite.gg/), [Bluefin](https://projectbluefin.io/), and [Aurora](https://getaurora.dev/).

It should also be applicable to other immutable Linux distributions such as:

* openSUSE Aeon
* openSUSE Kalpa
* openSUSE MicroOS

Despite being designed with immutable systems in mind, there is nothing particularly Atomic-specific about the underlying approach.

It should also work on traditional mutable distributions such as:

* Fedora
* Ubuntu
* Gentoo
* and probably most other Linux distributions

It should even work on **NixOS**.

Why you would want to do this on NixOS is, of course, a question for the reader.

### Shells

The current setup is **Bash-only**.

I use Bash, and that is what I have tested. I have no idea yet how well this translates to Zsh, Fish, or other shells, particularly when shell initialization, completions, and environment hooks are involved.

Contributions and experimentation in this area are very welcome.

### SELinux and AppArmor

So far, I have not encountered problems with **SELinux** or **AppArmor** during testing.

That is not a guarantee that every possible security policy or configuration will behave identically. Linux security configurations vary, and this project has not been tested against every possible setup.

If you encounter something interesting, please open an issue.

---

## Future?

There is still plenty of unnecessary nonsense to build on top of this.

Naturally, I intend to do so.

Things I'd like to explore include:

* **Bash completions** — because apparently getting the actual environment working wasn't enough.
* **Other shells** — particularly Zsh and Fish.
* **Alternative package managers and development tools** — such as [uv](https://github.com/astral-sh/uv), Cargo, and whatever else seems fun.
* **Better host integration** — while keeping the core idea small and user-space focused.
* **More development workflows** — because once you have a bubble, the natural question becomes *what else can I put in it?*

The goal isn't to reinvent Nix, Bubblewrap, package managers, containers, or shells.

It's mostly to see how far this little pile of **Bubblewrap, namespaces, Nix, shell scripts, and questionable decisions** can go.

---

## Inspiration

This project would not exist without the work that came before it.

The original inspiration came from **Ivan Molodetskikh (YaLTeR)** and his article:

[Easy Sandboxing on Linux with Bubblewrap](https://bxt.rs/blog/easy-sandboxing-on-linux-with-bubblewrap/)

That article demonstrated how Bubblewrap can be used to construct a practical, low-friction sandbox from ordinary Linux filesystem primitives.

That was the starting point for experimenting with a similar approach for a user-space Nix environment.

A huge thank-you to YaLTeR for writing the article and showing how approachable Bubblewrap-based sandboxing can be.

---

## Built on the Work of Others

Bubblebox stands on top of several projects and technologies that do the difficult parts.

* [Bubblewrap](https://github.com/containers/bubblewrap) — provides the unprivileged sandboxing and namespace machinery used to construct the environment.
* [Nix](https://nixos.org/) — provides the package management, store, profiles, flakes, and development-environment machinery.
* [Linux namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html) — provide the kernel-level isolation primitives underlying technologies such as Bubblewrap.
* [Linux kernel](https://github.com/torvalds/linux) — provides the namespace, mount, and filesystem machinery underneath the entire stack. In particular, see [`fs/namespace.c`](https://github.com/torvalds/linux/blob/master/fs/namespace.c).

This project is an application of those technologies, not an attempt to re-implement them.

---

## Acknowledgements

Special thanks to:

* **Ivan Molodetskikh (YaLTeR)** — for the original Bubblewrap sandboxing article that inspired this project.
* **The Bubblewrap contributors** — for building and maintaining the sandboxing tool that makes this approach possible.
* **The Linux kernel contributors** — for the namespace and filesystem primitives underneath the entire stack.
* **The Nix community and contributors** — for building the package-management and development-environment machinery used here.
* **Everyone who has written documentation, examples, experiments, and strange little shell scripts that make Linux easier to understand.**

And, inevitably, the random people on the internet whose offhand comments somehow turn into several evenings of Linux experimentation.

---

## License

Bubblebox is licensed under the **MIT License**.

See [`LICENSE`](LICENSE) for the full text.

---

## Final Disclaimer

This project is an experiment that happens to be useful.

It has been tested on my setup. It has not been tested on every Linux distribution, shell, security policy, filesystem configuration, or combination of weird software that exists in the wild.

Read the scripts.

Understand what they do.

Especially the part involving `eval`.

And for the love of all that is holy, **be prudent with `sed`.** 🫧
