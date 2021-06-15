---
layout: post
title:  "MacPorts and Mac OS Big Sur"
tag: Mac OS
category: miscellaneous
---

Since Mac OS Big Sur, Mac OS is not longer Mac OS X, but Mac OS 11 (Mac OS XI?): I'm currently running version 11.4.

The major version increment from 10 to 11 caused (or was foreseen to cause) all kinds of issues, so Apple decided to
build in a compatibility feature that makes Big Sur report as version 10.16.x instead of 11.x. In the shell you can
control this behaviour via an environment variable `SYSTEM_VERSION_COMPAT`: setting it to `1` and Big Sur reports
itself as 10.16.x; setting it to `0`` or undefined and it reports `11.x`.

See [this excellent post](https://eclecticlight.co/2020/08/13/macos-version-numbering-isnt-so-simple/) for details and
background information.

If you use [MacPorts](https://www.macports.org), you'll occassionally run into this issue when some port fails to
install with an error along the lines of:

> "<portname> requires macOS 11.0.0 or later, you have macOS 10.16.0."

Simply exporting `SYSTEM_VERSION_COMPAT` will not help, because a) you typicall install ports as root user and b) under
the hood MacPorts is spawning all kinds of processes that you don't have control over.

Finally I found a solution, tucked away in a [comment by a guy named Colin Bitterfield in a MacPorts bug
report](https://trac.macports.org/ticket/61736#comment:3) , that I'll quote here:

> Edit /etc/sudoers file on line 24 (add this line)
```sh
Defaults        env_keep += "SYSTEM_VERSION_COMPAT"
```
> Edit /opt/local/etc/macports/macports.conf Line 168 (add this line)
```sh
extra_env       SYSTEM_VERSION_COMPAT
```
> Edit your local profile (.profile, .bashrc, etc) and the following environment variable setting (export as proper for
> your shell)
```sh
SYSTEM_VERSION_COMPAT=0
export SYSTEM_VERSION_COMPAT
```

One caveat is that it appears that on every MacOS update (also the minor updates and patches) the `sudoers` file gets
reset to its default and you need to add the `Defaults` line again, so keep that in mind.

Of course, on major updates you'll need to get a new MacPorts version also, and you may need to edit `macports.conf`
again as well.

I hope this gets fixed somehow, but I guess that depends on package maintainers and/or upstream.
