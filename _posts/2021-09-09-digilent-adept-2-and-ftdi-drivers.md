---
layout: post
title: "Digilent Adept 2 and FTDI drivers"
tags: Digilent Arty-A7
category: miscellaneous
---

This issue seems to come back to me every time I provision a virtual machine for working with Digilent boards: I
install `libftdi` and `libusb`, and then download and install the [Digilent Adept
2](https://digilent.com/reference/software/adept/start) runtime and utilities and think I'm done. But every time:

```sh
$ dadutil enum
No devices found
```

Frustration follows, until I remember: Digilent Adept (and Xilinx Vivado as well) requires `libftd2xx`. This library is
[closed source and therefore not available via package managers](https://elinux.org/Libftdi_vs_FTD2XX). Digilent ships
a version of this library inside the runtime archive (in subfolder `ftdi.drivers-<version>`), or you can download the
newest version from [FTDI](https://ftdichip.com/drivers/d2xx-drivers/).

Unfortunately, the Adept 2 runtime installation does not install this library automatically, nor does it ask you to do
so. Hence, it is easy to end up with an incomplete installation. [This forum
post](https://forum.digilentinc.com/topic/20277-devices-cannot-be-found-with-digilent-utilities/?do=findComment&comment=57708)
provides a way to check if your installation is complete: use `ldconfig` to see what libraries are known. The output
should look like this:

```sh
$ /sbin/ldconfig -p | grep ftd2xx
    libftd2xx.so (libc6,x86-64) => /usr/lib64/libftd2xx.so
    libdftd2xx.so.1 (libc6,x86-64) => /usr/lib64/digilent/adept/libdftd2xx.so.1
    libdftd2xx.so (libc6,x86-64) => /usr/lib64/digilent/adept/libdftd2xx.so
```

If `libftd2xx` is not installed the first line will be missing; the libraries in the `digilent` folder are not complete
implementations, as is explained in the linked forum post.

Now that I wrote it down, I hope I don't forget again.
