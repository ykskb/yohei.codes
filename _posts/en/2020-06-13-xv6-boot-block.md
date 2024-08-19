---
layout: default
title: "xv6 image: How exactly is the boot sector created and loaded?" 
lang: en
image:
    path: /assets/images/dax86-xv6.png
---

# xv6 image: How exactly is the boot sector created and loaded?

One of the difficulties I faced in the development of an x86 simulator, dax86 was the end-to-end understanding of the OS image structure. This article talks about the creation of the booting block and how it is formed into the OS image as well as how hardware loads and executes it.

## Makefile

Makefile is where we start obviously. It has all the setup for creating different binaries and linked objects required to compose the OS image. As you can see below, `xv6.img` is requiring `bootblock` and `kernel` and executing `dd` command to write them. This part has some key information for locating different files, so I'll come back to it in the later part of this article.

```sh
xv6.img: bootblock kernel
	dd if=/dev/zero of=xv6.img count=10000
	dd if=bootblock of=xv6.img conv=notrunc
	dd if=kernel of=xv6.img seek=1 conv=notrunc
```

## Bootblock Creation

From the codes below, we can see `bootblock` is requiring `bootasm.S` which is a small asm mainly for entering into the protected mode and `bootmain.c` which copies kernel from disk to memory and jumps to it.

```sh
bootblock: bootasm.S bootmain.c
	$(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
	$(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
	$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o
	$(OBJDUMP) -S bootblock.o > bootblock.asm
	$(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
	./sign.pl bootblock
```

Here I'd like to step back a bit and think of how a machine starts from the very beginning. Traditional computer with BIOS executes POST (power-on self test) once the power is on, followed by some configuration for MP from option ROM. After that it tries to find this signature of `0xAA55` at the last two bytes of the first sector of a bootdisk.

Below is the result of running `xxd` command on the OS image. We can observe the signature at the address `0x1FE` which is 2 bytes before the end of the sector (512 bytes),  

```sh
$ xxd xv6.img
...
000001a0: 1518 0001 008d 65f4 5b5e 5f5d c300 0000  ......e.[^_]....
000001b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001f0: 0000 0000 0000 0000 0000 0000 0000 55aa  ..............U.
...
```

The last line of bootblock section of Makefile shown above: `./sign.pl bootblock` is actually creating this signature as below:

```perl
$buf .= "\0" x (510-$n);
$buf .= "\x55\xAA";

open(SIG, ">$ARGV[0]") || die "open >$ARGV[0]: $!";
print SIG $buf;
close SIG;
```

So we have traced until BIOS finds the bootable signature which is called master boot record (MBR). After this, BIOS copies this sector into memory typically at `0x7C00` and starts execution from there, which is the reason we see linker commands as `$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o`. Here `text` secion is specified to be `0x7C00` so that the base address for instructions would match the addresses after BIOS loads the sector.

## Bootloading

Single sector of data loaded into memory is never enough to include the kernel codes of the OS. Its job is to load more data of the kernel instructions from disk, and it's left to OS developer to make it work. In xv6, the kernel block is located and loaded from the second sector of the OS image. It took me some time to figure this out as tracing the whole flow involved IO communication with disk.

In the code below, we can see the `offset` is sector number with one added to the parameter:

bootmain.c

```c
void
readseg(uchar* pa, uint count, uint offset)
{
  uchar* epa;

  epa = pa + count;

  // Round down to sector boundary.
  pa -= offset % SECTSIZE;

  // Translate from bytes to sectors; kernel starts at sector 1.
  offset = (offset / SECTSIZE) + 1;

  // If this is too slow, we could read lots of sectors at a time.
  // We'd write more to memory than asked, but it doesn't matter --
  // we load in increasing order.
  for(; pa < epa; pa += SECTSIZE, offset++)
    readsect(pa, offset);
}
```

On the OS image creation, it's specified as below:

Makefile

```sh
dd if=kernel of=xv6.img seek=1 conv=notrun
```

Here, `seek=1` is the key. `seek=n` is the option of `dd` command to speficy the number of blocks to be skipped. As the default block size of `dd` is 512 bytes which is a sector, we can say `seek=1` is for skipping a sector which is for the boot block. Subsequently as showen in the `bootmain.c` code above, xv6 skips the first sector when it loads the kernel codes into its memory.