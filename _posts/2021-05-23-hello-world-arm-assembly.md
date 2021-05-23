---
layout: post
title: "\"Hello, world\" in ARM assembly"
tags: ARM assembly QEMU
categories: ARM
---

The mandatory example to try when learning any new language is a "Hello, world" program. So let's do one in ARM
assembly. You can find a similar one for C
[here](https://balau82.wordpress.com/2010/02/28/hello-world-for-bare-metal-arm-using-qemu/), including some useful
informat on compiling and linking that I've also used here.

ARM assembly is not difficult to learn, because the instruction set is limited: ARM stands for Advanced RISC Machine,
where RISC stands for Reduced Instruction Set Computer. You can find the reference manual
[here](https://developer.arm.com/documentation/dui0068/b/ARM-Instruction-Reference), and while the PDF is 350+ pages,
chapter 4 is the interesting bit (for now at least) and that is less than a 100 pages. There are plenty of tutorials,
quick references and cheat sheets to be found online, such as [this one from Azeria
Labs](https://azeria-labs.com/writing-arm-assembly-part-1/).

The typical example would be to write the prototypical program for execution in user-space on a operating system, i.e.
like a regular program. There is a good example on how to do that
[here](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part1.html). I'm more interested to do it on a bare-metal
system. However, since I don't have an ARM-based board readily available, I'll use [QEMU](https://www.qemu.org) to
emulate one. I'll use the [generic "virt" board](https://qemu-project.gitlab.io/qemu/system/arm/virt.html) that has a
single UART. You can actually also use QEMU to emulate an ARM processor in user space (`qemu-arm`).


## Basic program structure

The basic structure of source file will include the `.text` and `.data` sections:

```
.global _start

.text

_start:

halt:
    b halt

.data

message:
    .asciz "Hello, world!\n"

.end
```

This program will start at the `_start` label. The first instruction is `b halt`, which means to branch to the location
labelled with `halt`. Since this is an unconditional jump, this is an endless loop that we will use to end execution.

The data section contains the null-terminated string that we are going to print (the `.asciz` annotation results in a
`\0` added automatically at compile time).


## Printing the message

Basically, the idea is to do an equivalent of:

```
for (i = 0; i < length_of_message; i++) {
    print(message[i])
}
```

Hence, for our "Hello, world" program, we need three things:
 1. an address to write to, so we can see our message;
 2. the start address of our message;
 3. a loop counter to iterate over the message, one character at the time.

As said, this program is going to run on an embedded board, bare metal. No screen or terminals available for a `stdout`
to print to. Instead, we need to output via the UART. The base address of the UART of the "virt" board [can be found at
`0x9000000`]({% post_url 2021-05-18-qemu-arm-device-tree %}). The
[datasheet](https://developer.arm.com/documentation/ddi0183/g/programmers-model/summary-of-registers) tells us that the
data register is immediately at the base address, so that is the address we need to write to. We'll use register `r0`
for that. We'll use `r1` to store the start address of our message and `r2` for the loop counter:

```
ldr r0, =uart_base  /* data register is at the uart base address */
ldr r1, =message    /* pointer to first character */
ldr r2, =#0         /* loop counter */
```

We will use `r3` as an intermediate register. Then the loop is set up as follows:
 1. load the value at `message_address + loop_counter` (1 byte) in `r3`;
 2. compare the value in `r3` to zero;
 3. if `r3 == 0`, exit the loop; otherwise push the value in `r3` to the data register of the UART;
 4. increment the loop counter;
 5. go to step (1).

```
loop:
    ldrb r3, [r1, r2]   /* load byte from address r1 + offset r2 in r3 */
    cmp r3, #0          /* check for end of string */
    beq halt            /* stop when finished */
    str r3, [r0]        /* else, write byte from r3 to uart data register */
    add r2, r2, #1      /* increment counter */
    b loop              /* loop */
```

That's it. Here is the entire program:

```
.global _start

.text

.set uart_base, 0x09000000

_start:
    ldr r0, =uart_base  /* data register is at the uart base address */
    ldr r1, =message    /* pointer to first character */
    ldr r2, =#0         /* loop counter */

loop:
    ldrb r3, [r1, r2]   /* load byte from address r1 + offset r2 in r3 */
    cmp r3, #0          /* check for end of string */
    beq halt            /* stop when finished */
    str r3, [r0]        /* else, write byte from r3 to uart data register */
    add r2, r2, #1      /* increment counter */
    b loop              /* loop */

halt:
    b halt

.data

message:
    .asciz "Hello, world!\n"

.end
```

{:.message-info}
if this was a real ARM board, we would have to add some code to actually initialize the UART, setting the
*baud rate, etc. However, the "virt" board doesn't seem to really care about that: you can just write to the data
*register and it will be printed to `stdout` of your host operating system.


## Compiling and running

To compile, you need the appropriate cross-compiler toolchain. We have a "bare-metal" target and ARM [embedded
application binary interface (EABI)](https://en.wikipedia.org/wiki/Application_binary_interface). ARM provides a
GCC-based toolchain on their
[website](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads),
which is what I will be using here, but there are other options.

These are the commands you need, assuming you save the program as `hello.s`:

```
arm-none-eabi-as -o hello.o hello.s
arm-none-eabi-ld -Ttext=0x0 -o hello.elf hello.o
arm-none-eabi-objcopy -O binary hello.elf hello.bin
```

First the assembler is invoked to compile the source into an object file. Then we use the linker to make an executable
([ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)), linking the text section at `0x0`. Finally,
because we do not have an operating system, we convert the executable into a raw binary.

Ideally, this should be enough, but there is a caveat: how to tell the board to start executing our binary? If you read
through the help for [invocation of QEMU](https://qemu-project.gitlab.io/qemu/system/invocation.html), you find that
the use cases are geared towards booting kernel images. There is however an option to specify a ROM image, which is not
really clear from the documenation. [This
post](https://stackoverflow.com/questions/57461025/how-to-add-sd-flash-to-qemu-virt-machine) by [Peter
Maydell](https://translatedcode.wordpress.com/about/) (one of the developers for QEMU) was very helpful: you can
provide a flash ROM image via the `-drive` flag:

```
-drive if=pflash,format=raw,file=flash.img
```

Using `file=hello.bin` throws and error, because the "virt" board expects a 64MB image:

```
$ qemu-system-arm -machine virt -m 128M -nographic -drive if=pflash,format=raw,file=hello.bin
qemu-system-arm: device requires 67108864 bytes, block backend provides 66048 bytes
```

We can easily make an image of the appropriate size with `dd`:

```
dd if=/dev/zero of=flash.img bs=1m count=64
dd if=hello.bin of=flash.img conv=notrunc
```

The first command makes an image file of exactly 64 MB, filled with zeros. The second command puts our binary at the
beginning of that image. Now run it (press `Ctrl-a, x` to terminate):

```
$ qemu-system-arm -machine virt -m 128M -nographic -drive if=pflash,format=raw,file=flash.img
Hello, world!
```

[Here](https://gist.github.com/lcvisser/142be2a62ee1052673287fd26a9eca99) is a simple Makefile to tie it all together.
