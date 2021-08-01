---
layout: post
title: "LEON3 processor on a Digilent Arty-A7"
tags: LEON3 FPGA Arty-A7
category: LEON3
---

The LEON3 processor is a space-grade processor. Its VHDL sources are open source, which means you can have your own
space-grade processor running on a FPGA board, at home.


## What is the LEON processor?

The LEON processor is a 32-bit CPU fully compatible with the SPARC-V8 instruction set from Sun Microsystems. What makes
this processor interesting (for me at least), is that it is designed by the European Space Agency (ESA) for use in
space missions. While development is currently no longer done by ESA themselves, the LEON processors are still [used in
ESA
missions](http://www.esa.int/Enabling_Support/Space_Engineering_Technology/Onboard_Computers_and_Data_Handling/Microprocessors).
The latest version is the LEON5. [Wikipedia](https://en.wikipedia.org/wiki/LEON) has much more information for further
reading.

The LEON processors and supporting software are currently developed by [Cobham Gaisler](https://www.gaisler.com), but
the VHDL sources, build tools and software libraries are available under a GPL license, I guess as a result of ESA
funding much of their development with public money. This means that you can have your own spacecraft processor at
home! Well, a softcore version of one at least.

This article covers the necessary steps to synthesize a LEON3 core on a Digilent Arty-A7.

While the latest version of the LEON processor series is the LEON5, I choose the LEON3. It can be configured to fit on
fairly small (read: cheap) FPGAs and because it has been around for about 20 years already, there is a lot of
documentation and tools available for it.


### Choosing an FPGA board

GRLIB, which contains the VHDL sources and supporting scripts for the LEON processors and IP cores, can be downloaded
from Gaisler's [download page](https://www.gaisler.com/index.php/downloads/leongrlib). On that page, you'll find an
Excel sheet that provdes some estimates on the required FPGA sizing (number of LUTs, RAM, etc.) for a number of
configuration options. You can use this to choose an FPGA board, but easier is to download the library and look under
the `/designs/` directory. In there you'll find all the FPGA boards that are supported "out of the box" by GRLIB, in the
sense that they provide a CPU configuration that will fit in that particular FPGA. They also provide the required
constraint files and other parameters that are needed to get something useful running on that board.

Gaisler supports some boards with pre-built bitstreams and a bit more documentation. These are an oorder of magnitude
more expensive: for example, the [Xilinx
XCKU-105](https://www.gaisler.com/index.php/products/processors/leon-examples/leon-xcku) costs about US$ 3000. which is
quite a bit more that I'm willing to spend on a pet project. Gaisler also sells evaluation boards for their own
(radiation-hardened) processors, but those are definitely outside my budget.

From the boards supported by GRLIB, I choose the [Digilent Arty-A7
35T](https://reference.digilentinc.com/programmable-logic/arty-a7/start), mostly because it is cheap (about US$
130/€ 110) yet sufficient for some first steps with the LEON3.


## Preparation

We'll need to set up some things before we can use GRLIB and the Arty board.

{:.message-info}
This article assumes a Linux operating system. Windows should work too, but I haven't tried myself. I am on MacOS and
use VirtualBox with CentOS 8. Docker does not work on MacOS, since HyperKit does not allow USB forwarding, which makes
it impossible to program the board.

Install `libusb`, `libftdi`, `libXtst`, `tcl`, `tk` and `ncurses` via your preferred way (package manager or whatever
your prefer). GRLIB expects `libncurses.so.5` and `libform.so.5`, but you can symlink to `*.so.6`.

You'll also need the usual developer tools, like `make`, `autoconf`, etc.; on CentOS I just did a groupinstall of
"Development Tools".

If you want to use Vivado for bitstream synthesis, you also need an X server. If you are running in a virtual machine
or SSH-ing into a remote system, you can install `xauth` and `xorg-x11-apps` and run `xeyes` to see if your X
forwarding works.


### Digilent Adept runtime and utilities

First install the [Digilent Adept runtime and
utilities](https://reference.digilentinc.com/reference/software/adept/start). Both come with an install script that
must be run as root user.

After installation, plug in the board and run:

```sh
$ dadutil enum
Found 1 device(s)

Device: Arty
    Device Transport Type: 00020001 (USB)
    Product Name:          Digilent Arty A7-35T
    User Name:             Arty
    Serial Number:         210319B0C2D8
```

If this utility doesn't show your board, fix that first. If you are running in a virtual machine, make sure to have
USB3 enabled (xHCI). Also make sure that your user has rights to access serial devices; on CentOS you need to be added
to the `dialout` group.


### Vivado

You'll need something to generate a bitstream and program the FPGA. A full list of supported software can be found in
the [GRLIB User Manual](https://www.gaisler.com/products/grlib/grlib.pdf). I only have some limited experience with
Xilinx Vivado, and fortunately the Arty-A7 is [fully supported by the free WebPACK
edition](https://www.xilinx.com/products/design-tools/vivado/vivado-webpack.html#architecture), so I opted for that. I
used version 2020.2 for writing this article.

Before running the installer, set the `LC_ALL` and `LANG` environment variables (put this also in `.bashrc` or
similar):

```sh
export LC_ALL=C.UTF-8
export LANG=C.UTF-8
```

On CentOS 8 I had to create symlink `libtinfo.so.5`, since CentOS came with a newer `libtinfo.so.6`.

You'll need an account to download and install Vivado. Xilinx provides only a [single generic
installer](https://www.xilinx.com/support/download.html), and the installer will prompt you for your account details in
order to verify that you have a license for what you want to install. Hence, make sure to only install the WebPACK
components. In order to save space, select only Artix-7 support.

You can install Vivado in any location. You don't need root access to run the main installer, but I'd recommend to
install it in a location that is read-only for normal users, to avoid accidentally breaking things. Vivado comes with a
small bash script to set environment variables (such as adding `vivado` to `PATH`); you need to source it:

```sh
$ . /path/to/Vivado/2020.2/settings64.sh"
```

After finishing the main installer, you need to also install the Xilinx cable drivers. This must be done as root (`su`
will not work!):

```
$ sudo su
# /path/to/Vivado/2020.2/data/xicom/cable_drivers/lin64/install_script/install_drivers/install_drivers
```

{:.message-info}
I'm running from a VirtualBox virtual machine and have to boot the machine with the board already plugged in, otherwise
the cable drivers don't see the board for some odd reason. Maybe this is fixed in future versions of either VirtualBox
or Vivado.


### GRLIB and GRMON

You can download GRLIB from Gaisler's [download page](https://www.gaisler.com/index.php/downloads/leongrlib). It is a
single compressed archive that you only need to extract somewhere. I recommend putting it in a read-only location, so
that you don't mess it up.

You will also need GRMON, their debug tool. It too can be [downloaded from their
website](https://www.gaisler.com/index.php/downloads/debug-tools). It is free for personal use. Extract the archive and
add the `bin64` and `lib64` paths to your `PATH` and `LD_LIBRARY_PATH` variables.


## Building the LEON3 example design

GRLIB comes with a designs for a large number of boards. They reside in the `designs` directory. To get started, copy
the entire `leon3-digilent-arty-a7` directory to a workspace directory (careful: there is an identically named
directory under `boards`!). In the top-level Makefile, modify the `GRLIB` variable to point to the root of the
extracted GRLIB archive and then run:

```sh
$ make scripts
```

This will create a number of files and directories with support files for various toolchains, including Vivado. Open
the file `vivado/leon3mp_vivado.tcl` with a text editor and find the line

```
#upgrade_up [getips mig]
```

It should be near the end of the file. Uncomment it by removing the `#`.

{:.message-info}
"MIG" refers to Memory Interface Generator and is a Xilinx IP core. You need to build it only once, so after the first
run you can comment the line in the TCL script out again.

Now, start the build and synthesis process:

```sh
$ make vivado
```

This takes about 15 minutes on a reasonably up to date machine. Alternatively, run `make vivado-launch` for an
interactive session (in that case, do **not** uncomment the line in the TCL script, as it will immediately try to build
the MIG core on startup; instead, do the build from the context menu (look for "Upgrade IP...")).

{:.message-warning}
Vivado requires 3.5 GB of RAM with the Arty-A7, more than the 3 GB that Xilinx
[reports](https://www.xilinx.com/products/design-tools/vivado/memory.html). My headless CentOS virtual machine needs 5
GB of RAM to synthesize the default design. If you run from the Vivado GUI, you'll need more.


## Flashing the bitstream

When the build is complete, you can flash it to the board. You have to options:

1. Program the FPGA core only; the bitstream is not stored in ROM and thus "gone" after a power cycle.
2. Put the bitstream in ROM; the bitstream is automatically loaded into the core when power is turned on.

For the first option:
- Make sure that the MODE jumper (JP1) on the board is **open**.
- Execute `make vivprog`

For the second option:
- Make sure that the MODE jumper (JP1) on the board is **closed**.
- Execute `make vivrom`

After a few minutes, the LED labeled DONE will light up. Press the RESET button to start the LEON3 core. LED 4, 5 and 7
should light up.

Now, use GRMON to connect to the core (the `-freq` flag is should not be necessary, but sometimes GRMON detects the
frequency incorrectly):

```sh
$ grmon -digilent -freq 83
  GRMON debug monitor v3.2.11.1 64-bit eval version

  Copyright (C) 2021 Cobham Gaisler - All rights reserved.
  For latest updates, go to http://www.gaisler.com/
  Comments or bug-reports to support@gaisler.com

  This eval version will expire on 20/07/2021

JTAG chain (1): xc7a35t
  GRLIB build version: 4261
  Detected frequency:  83.0 MHz

  Component                            Vendor
  LEON3 SPARC V8 Processor             Cobham Gaisler
  JTAG Debug Link                      Cobham Gaisler
  GR Ethernet MAC                      Cobham Gaisler
  SPI Memory Controller                Cobham Gaisler
  AHB/APB Bridge                       Cobham Gaisler
  LEON3 Debug Support Unit             Cobham Gaisler
  Xilinx MIG Controller                Cobham Gaisler
  Generic UART                         Cobham Gaisler
  Multi-processor Interrupt Ctrl.      Cobham Gaisler
  Modular Timer Unit                   Cobham Gaisler
  AMBA Wrapper for OC I2C-master       Cobham Gaisler
  SPI Controller                       Cobham Gaisler
  General Purpose I/O port             Cobham Gaisler
  General Purpose I/O port             Cobham Gaisler
  General Purpose I/O port             Cobham Gaisler

  Use command 'info sys' to print a detailed report of attached cores

grmon3>
```

Our softcore LEON3 is ready! Next time we'll run some code on it.
