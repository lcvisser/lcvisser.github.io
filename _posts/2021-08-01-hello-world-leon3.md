---
layout: post
title: "\"Hello, world\" on a LEON3"
tags: LEON3 FPGA Arty-A7
category: LEON3
---

In a [previous post]({% post_url 2021-05-31-leon3-processor-on-a-digilent-arty-a7 %}) I explained how to deploy a
softcore LEON3 processor on a [Digilent Arty-A7
35T](https://reference.digilentinc.com/programmable-logic/arty-a7/start) development board. Now it is time to run some
code on it.

{:.message-info}
This article assumes a Linux operating system. I have no idea if things will work on Windows or MacOS. As explained
earlier, I used a Virtualbox environment with CentOS 8 for working with the Gaisler tools and the Digilent board.


## Toolchain

To compile for the LEON3, you need the [LEON Bare-C Cross Compilation System
(BCC)](https://www.gaisler.com/index.php/products/operating-systems/bcc), which is provided by Gaisler for free. It is
based on GCC, so it easy to work with if you are used to those tools. BCC can be downloaded from
[here](https://www.gaisler.com/index.php/downloads/compilers?task=view&id=161).

{:.message-warning}
Make sure to download BCC 2.x. BCC 1.0.x has reached end-of-life.

Installing BCC2 is easy: as explained in the user manual, just extract it somewhere and add the folder with executables
to you `$PATH` environment variable, and the folder with manual pages to the `$MANPATH` environment variable. For
example, if you have extracted the toolchain to `/opt`, adding this to your `~/.bashrc` is all that is needed:

```sh
export PATH=/tools/bcc-2.2.0-gcc/bin:$PATH
export MANPATH=/tools/bcc-2.2.0-gcc/man:$MANPATH
```


## Hello, world

In the BCC2 archive there is an directory with examples. In it is a `README` file, a `Makefile`, and a number of
directories with various examples worked out. The folder `hello` contains our "hello, world" example, implemented in a
single file `hello.c`:

```c
#include <stdlib.h>
#include <stdio.h>

int main(void)
{
        printf("hello, world\n");

        return EXIT_SUCCESS;
}
```

Pretty straight forward. From the root of the examples directory (i.e., where the `Makefile` is), execute:

```sh
$ make CFLAGS="-mcpu=leon3 -msoft-float" hello.elf
```

This will produce the executable `hello.elf`. With the board connected, start GRMON in UART debug mode (`-u`):

```sh
grmon -digilent -u
```

Reset the processor and load the binary that was just compiled:

```sh
$ grmon -digilent -u
grmon3> reset
grmon3> load hello.elf
          40000000 .text             23.0kB /  23.0kB   [===============>] 100%
          40005C00 .rodata            128B              [===============>] 100%
          40005C80 .data              1.2kB /   1.2kB   [===============>] 100%
  Total size: 24.28kB (19.45kbit/s)
  Entry point 0x40000000
  Image /path/to/examples/hello.elf loaded
```

The program can be executed via the `run` command. Because GRMON was started in UART debug mode, the output is printed
to the GRMON console:

```
grmon3> run
hello, world

  Program exited normally
```


## Flashing to ROM

Instead of having to manually load and run our program, it can also be flashed to ROM. The [MKPROM]
(https://www.gaisler.com/index.php/products/boot-loaders/mkprom2) utility takes any executable and embeds it into a an
image that takes care of system initialization, then extracts the executable to RAM and runs it. It is basically a tool
that adds your program to a bootloader.

MKPROM2 can be downloaded from Gaisler's download page.

{:.message-warning}
MKPROM2 must be installed in `/opt`. There are some hard-coded paths in the utility that look for
things there. A symlink in `/opt` that points to the extracted archive works.


### Preparation

The MKPROM2 utility needs to know a few things of the board that it will be compiling for, in particular the frequency
it runs at and the baudrate of the UART. Both can be extracted via GRMON with the `info mkprom2` command:

```sh
$ grmon -digilent
...
grmon3> info mkprom2
  Mkprom2 switches:
  -leon3 -freq 83 -baud 38425
```

The reported frequency should (and it does) match the frequency that was specified when the FPGA bitstream was built.
The baudrate is estimated and the value reported here is not a standard baudrate. We need to use the nearest standard
rate, which is 38400.


### Compiling the example

The same examples folder used before has an `mkprom-hello` folder with a `Makefile` and a C file that we will be using.
All commands in this section must be executed in this folder.

The example LEON3 design for the Arty A7 does not have a floating point unit (not available in the GPL version of
GRLIB) and has single vector trapping model enabled. Combined with the parameters reported by GRMON, the compilation
command for the MKPROM example is:

```sh
$ make CFLAGS="-mcpu=leon3 -msoft-float -qsvt" MKPROMOPT="-leon3 -msoft-float -qsvt -freq 83 -baud 38400"
```

{:.message-warning}
The flags to MKPROM (`MKPROMOPT`) must match those for compilation (`CFLAGS`) of the executable itself!

The resulting executable has an entry point at `0x0`, as can be observed via `obj-dump`:

```sh
$ sparc-gaisler-elf-objdump -x hello.prom

hello.prom:     file format elf32-sparc
hello.prom
architecture: sparc, flags 0x00000012:
EXEC_P, HAS_SYMS
start address 0x00000000

Program Header:
    LOAD off    0x00000060 vaddr 0x00000000 paddr 0x00000000 align 2**5
         filesz 0x0000b940 memsz 0x0000b940 flags rwx

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000b940  00000000  00000000  00000060  2**5
...
```

In the `README` inside the design folder for the Digilent Arty-A7 (`designs/leon3-digilent-arty-a7/README.txt`) there
is however a very important message:

> Typically the lower part of the SPI flash device will hold the
> configuration bitstream for the FPGA. The SPIMCTRL core is configured
> with an offset value that will be added to the incoming AHB address
> before the address is propagated to the SPI flash device. The
> default offset is 0x00400000 (this value is set via xconfig and the
> constant is called CFG_SPIMCTRL_OFFSET). When the processor starts
> after power-up it will read address 0x0, this will be translated by
> SPIMCTRL to 0x00400000.

> SPIMCTRL can only add this offset to accesses made via the core's
> memory area. For accesses made via the register interface the offset
> must be taken into account. This means that if we want to program
> the Flash with an application which is linked to address 0x0 (our
> typical bootloader) then we need to add the offset 0x00400000 before
> programming the file with GRMON. We load the Flash with our application
> starting at 0x00400000 and SPIMCTRL will then translate accesses from
> AMBA address 0x0 + n to Flash address 0x00400000 + n.

You can verify this offset in the configuration code for the board support:

```sh
$ grep SPIMCTRL_OFFSET leon3-digilent-arty-a7/config.vhd
  constant CFG_SPIMCTRL_OFFSET : integer := 16#400000#;
```

To add the offset, we need to modify the Load Memory Address (LMA) of our binary. This can be easily done with the
following command:

```sh
$ sparc-gaisler-elf-objcopy --change-section-lma *+0x400000 hello.prom hello.spim
```

I changed the extension from `.prom` to `.spim` for clarity. Verify that the LMA address has changed:

```sh
$ sparc-gaisler-elf-objdump -x hello.spim

hello.spim:     file format elf32-sparc
hello.spim
architecture: sparc, flags 0x00000012:
EXEC_P, HAS_SYMS
start address 0x00000000

Program Header:
    LOAD off    0x00000060 vaddr 0x00000000 paddr 0x00400000 align 2**5
         filesz 0x0000b940 memsz 0x0000b940 flags rwx

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000b940  00000000  00400000  00000060  2**5
...
```


### Flashing to ROM

Start GRMON and let it detect the flash memory module before proceeding:

```sh
$ grmon -digilent -freq 83
...
grmon3> spim flash detect
  Got manufacturer ID 0x01 and device ID 0x2018
  Detected device: Cypress S25FL128S
```

The detection step is mandatory. Now load the offset binary to flash; it will be loaded at `0x40000`:

```sh
grmon3> spim flash load hello.spim
            400000 .text             46.3kB /  46.3kB   [===============>] 100%
  Total size: 46.30kB (820.42bit/s)
  Entry point 0x00000000
  Image /path/to/examples/mkprom-hello/hello.spim loaded
```

To verify that it was successful, one could run the verify command, but we can also just have a quick look at the first
four instructions (the address is the address that the processor sees, not the physical address!):

```sh
grmon3> disassemble 0 4
       0x00000000: a7580000  mov  %tbr, %l3
       0x00000004: a1480000  mov  %psr, %l0
       0x00000008: a60ceff0  and  %l3, 0xff0, %l3
       0x0000000c: a734e004  srl  %l3, 0x4, %l3
```

Compare it with the binary we just built:

```sh
$ sparc-gaisler-elf-objdump -D hello.prom | head -n 11

hello.prom:     file format elf32-sparc


Disassembly of section .text:

00000000 <_start_svt_real>:
       0:	a7 58 00 00 	rd  %tbr, %l3
       4:	a1 48 00 00 	rd  %psr, %l0
       8:	a6 0c ef f0 	and  %l3, 0xff0, %l3
       c:	a7 34 e0 04 	srl  %l3, 4, %l3
```

{:.message-warning}
Warning: ROM must be erased before writing. Use the SPIM `flash erase` command to erase memory, but be careful: without
arguments it erases the entire ROM, including the FPGA bitstream!


### Running the program

Open a second terminal and connect it to the UART:

```sh
$ screen /dev/ttyUSB1 38400
```

In the first terminal, in GRMON, instruct the processor run from address 0:

```sh
grmon3> run 0
  Program exited normally
```

On the second terminal, you should see something like this:

```sh
  MKPROM2 boot loader v2.0.67
  Copyright Cobham Gaisler AB - all rights reserved

  system clock   : 83.0 MHz
  baud rate      : 38425 baud
  prom           : 512 K, (15/15) ws (r/w)
  sram           : 2048 K, 1 bank(s), 3/3 ws (r/w)

  decompressing .text to 0x40000000
  decompressing .rodata to 0x4000f120
  decompressing .data to 0x4000f930

  starting hello.elf

--- EXAMPLE BEGIN ---

Hello World!

tick 1
tick 2
tick 3
tick 4
tick 5
tick 6
tick 7
tick 8
tick 9
tick 10

--- EXAMPLE END ---
```

The executable will be run every time the board is powered on, immediately after flash programming. You can verify this
by attaching a `screen` session to the serial device and press the reset button on the board.  Note that without GRMON
attached, you’ll see **two** `ttyUSBx` devices in `/dev`: the first is the JTAG bridge, the second is the UART. On my
computer (MacOS) I can do:

```sh
% screen /dev/tty.usbserial-210319B0C2D81 38400
```
