+++
title = "Weekend Project: xbps-command-not-found"
date = "2026-01-18"
+++

This weekend, I wrote a small script to suggest installation of XBPS packages
in interactive shell sessions when a command isn't found.

<!-- more -->

If you've ever used Debian or Ubuntu, you've possibly seen
[something similar](https://salsa.debian.org/jak/command-not-found) from your
shell. There are also similar programs for other distros, like
[Gentoo](https://github.com/Nowa-Ammerlaan/command-not-found-gentoo), and
semi-distro-agnostic like [pay-respects](https://codeberg.org/iff/pay-respects).

However, none of these existing implementations really had the right feel to me.
There's no real reason this needs to be written in Rust or Python, or use a
whole database (Debian please!). All we need is a little shell... and it can be
POSIX to boot.

[Since a little under a year ago](https://github.com/void-linux/void-packages/pull/53522),
XBPS packages have been given virtual package names in the format `cmd:<name>`
for binaries (and alternatives) they provide. This means you can search the
repositories with a command like:

```console
$ xbps-query -Rp provides -s cmd:awk
busybox-1.34.1_12: cmd:awk-0_1 (https://repo-fastly.voidlinux.org/current)
busybox-core-1.34.1_12: cmd:awk-0_1 (https://repo-fastly.voidlinux.org/current)
busybox-huge-1.34.1_12: cmd:awk-0_1 (https://repo-fastly.voidlinux.org/current)
gawk-5.3.2_1: cmd:awk-0_1 (https://repo-fastly.voidlinux.org/current)
mawk-1.3.4.20250131_2: cmd:awk-0_1 (https://repo-fastly.voidlinux.org/current)
nawk-20251225_1: cmd:awk-0_1 (https://repo-fastly.voidlinux.org/current)
```

and it will show every package that could provide the command `awk`!

You can also use this when installing packages[^1]:

```console
# xbps-install cmd:zola

Name Action    Version           New version            Download size
zola install   -                 0.22.0_1               -
```

I've leveraged this behaviour for `xbps-command-not-found`. In `bash`, `zsh`,
`fish`, and `nushell`, it is possible to run something when the user tries to
run a non-existent command. In interactive shells, `xbps-command-not-found`
 searches the XBPS repositories for `cmd:` provides based on the command
attempted. If it finds potential providers for the command, it prints a message
like this:

```console
$ vim
The command "vim" was not found, but is available from the following packages:
    - gvim
    - neovim
    - vim
    - vim-x11
and can be installed with xbps-install.
```

If you'd like to give it a try, you can install `xbps-command-not-found` now
from XBPS on Void Linux. To configure it for your shell, read the
[manual page](https://man.voidlinux.org/xbps-command-not-found.7).

The source code is available
[here](https://codeberg.org/classabbyamp/xbps-command-not-found).

[^1]: Though the behaviour when there are multiple providers isn't perfect. It's
a [work-in-progress](https://github.com/void-linux/xbps/pull/634).
