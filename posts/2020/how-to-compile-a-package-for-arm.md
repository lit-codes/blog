---
title: "How to compile a package for ARM"
date: 2020-04-28
tags: [How-to]
---

In this post, we will use `Qemu multiarch` inside Docker to compile source
codes for `ARM` processors.

> If you want to know how to run an `ARM64` Debian inside Docker skip to [Qemu
multiarch](#qemu-multiarch)

## ARM architecture

<a href="https://en.wikipedia.org/wiki/ARM_architecture" target="_blank">ARM</a>
refers to a large group of CPU architectures designed
for embedded systems and low-cost computing. There are many variations of ARM
instruction code and unfortunately, they are not very compatible with each
other.  It's important to identify the target architecture otherwise the
built files may not be compatible with the target machine.

For this post, I am going to compile for `arm64` (aka `aarch64` as GCC calls
it). This architecture is used on `AWS ARM instances` which are available for a
cheaper price than the `X86_64` ones.

Let's check the architecture of a Debian machine running on `aarch64`:

```bash{outputLines: 2,4}
uname -a
Linux hostname [version] #1 SMP Debian 4.9.210-1 [date] aarch64 GNU/Linux
dpkg --print-architecture
arm64
```

As you can see `arm64` and `aarch64` are used almost interchangeably.

## Building using Docker

There are two ways we can go about building a source file for a different
architecture of your current machine (e.g. your machine runs on an `X86_64` CPU
and you want to compile for `arm64`).

One way is `cross-compiling` inside a machine with a different architecture
using `aarch64-linux-gnu-gcc` which is a GCC cross-compiler for `arm64`
architecture.

The other (and less complicated) way is to make the package inside a Virtual
Machine. I will use `Docker` with `Qemu multiarch` which acts similar to a VM
and allows running Docker images built for other architectures including
`aarch64`.

### Qemu multiarch

First, we will need to enable
<a href="https://github.com/multiarch/qemu-user-static" target="_blank">qemu-user-static</a>
which allows us to run the Docker image built for ARM.

**Note:** This setup uses `binfmt`, read more:
<a href="https://en.wikipedia.org/wiki/Binfmt_misc" target="_blank">binfmt_misc</a>

```bash
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

This will setup the multiarch files and now we can run Docker images that are
built for different architectures seamlessly.

### Debian for aarch64

After enabling mutliarch, we can simply run the Debian image built for ARM:

```bash{outputLines: 2}
docker run -it arm64v8/debian:stretch
root@3b853bce5181:/#
```

> The Docker image for
<a href="https://hub.docker.com/r/aarch64/debian" target="_blank">aarch64</a>
is officially deprecated in favor of
<a href="https://hub.docker.com/r/arm64v8/debian/" target="_blank">arch64v8</a>
which has support for broader variants of the architecture.

Now we can see the architecture is shown to be ARM:

```bash{outputLines: 2,4}
uname -a
Linux 3b853bce5181 5.3.0-46-generic #38-Ubuntu SMP [date] aarch64 GNU/Linux
dpkg --print-architecture
arm64
```

That's it, we should now be able to download the desired source code and build
it for `aarch64` inside that container.

For this post we will stick to a very simple program written in C.

### Building a simple helloworld in C

Just for the sake of the demonstration, let's write a simple hello world `C` and
compile it for ARM.

```c:title=helloworld.c
#include <stdio.h>
int main() {
   printf("Hello, World!\n");
   return 0;
}
```

First we will need to install `gcc` to compile our source code:

```bash
apt update && apt install gcc file -y
```

Then we can compile our source code and run it:

```bash{outputLines: 3}
gcc -o helloworld helloworld.c
./helloworld
Hello, World!
```

We can also use the `file` command to tell us about the binary format of the
executable we just built:

```bash{outputLines: 2}
file ./helloworld
helloworld: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV) ...
```

It's correctly built for ARM :tada:.

One little trick we can do with GCC is that we can see the assembler code,
therefore we can see what the instruction set looks like:

```bash
gcc -o helloworld.asm -S helloworld.c
```

Contents of `helloworld.asm` will look like this:

```nasm
	.arch armv8-a
	.file	"helloworld.c"
	.section	.rodata
	.align	3
.LC0:
	.string	"Hello, World!"
	.text
	.align	2
	.global	main
	.type	main, %function
main:
	stp	x29, x30, [sp, -16]!
	add	x29, sp, 0
	adrp	x0, .LC0
	add	x0, x0, :lo12:.LC0
	bl	puts
	mov	w0, 0
	ldp	x29, x30, [sp], 16
	ret
	.size	main, .-main
	.ident	"GCC: (Debian 6.3.0-18+deb9u1) 6.3.0 20170516"
	.section	.note.GNU-stack,"",@progbits
```

### Raspberry Pi

This method can be used to build applications that run on other ARM processors.
For example on Raspberry Pi devices, instead of `aarch64` and `arm64`, you
would expect something similar to `armv7l` and `armhf`.

```bash{outputLines: 2,4}
uname -a
Linux raspberrypi 4.19.97-v7l+ #1294 SMP [date] armv7l GNU/Linux
dpkg --print-architecture
armhf
```
