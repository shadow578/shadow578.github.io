---
layout: post
title: Running Linux on a WatchGuard T40
tags: hardware-hacking linux
---

as if my obsession with watchguard firewalls wasn't bad enough already, i recently got my hands on a WatchGuard T40.
since i have no use for them as just an appliance, i decided it would be fun to get linux running on it.
oh, how wrong i was...

## Hardware Overview

the T40 is already discussed in the [OpenWRT forums](https://forum.openwrt.org/t/installing-openwrt-on-watchguard-t40w/230048), with the general consensus being that it'd make a great OpenWRT device, if only it was supported.
the hardware information claimed on the forums (as to whether they're correct, we'll see later) is as follows:

- Board: LS1043a QDS
  - [LS1043A](https://www.nxp.com/products/LS1043A) Layerscape SOC
- Storage: mSATA 16GB
- Bootloader: Custom U-Boot for WatchGuard, locked down to only boot from internal storage
- Stock device tree is available in [a post](https://forum.openwrt.org/t/installing-openwrt-on-watchguard-t40w/230048/12)
  - "it's pretty much the `LS1043A-RDB` one from upstream linux"

now, here's what i found after investigating my own T40:

- LS1043A CPU, as expected
- 4GB RAM, also expected
- 16GB M.2 SATA SSD (idk how they got mSATA...)
- 1 x Atheros AR8035 GbE PHY (WAN port)
- 1 x Marvell 88E1510 Quad GbE PHY (LAN1-4 ports)
- Bootloader is indeed a custom U-Boot, but easy to bypass (see below)

## Accessing the U-Boot Shell

while the U-Boot is indeed locked down and only allows booting from the internal SATA SSD, there's a little quirk:
if - for any reason - U-Boot fails to boot the OS, it will drop to the U-Boot shell.
if only we could make the boot fail...

one thing to make the boot fail is to simply remove the SSD.
boot it up without the SSD, and it will drop to the U-Boot shell.
from there, we can simply override the boot command to something else.
i chose to set it to `true`, which does nothing and drops me into a shell immediately.

funnily enough, the boot option for SysB is called `Recovery / Diagnostic Mode`.
a unprotected shell is a diagnostic mode, right?

```shell
env set wgBootSysB 'true'
env save
reset
```

from here on, the SSD can be reinserted, as we have a persistent way to drop into the U-Boot shell.

on the T20, i believe the boot device is a NAND flash chip soldered to the board.
thus, it's not really feasible to remove it.
however, maybe something similar to the technique used for hacking the Nintendo Wii (see: [Team Twiizers and the tweezer attack for the Wii](http://web.archive.org/web/20090505005003/www.atomicmpc.com.au/Tools/Print.aspx?CIID=102079)) could work here - i.e. shorting some pins on the NAND flash to make it temporarily unreadable.

## Initial Linux Boot

we have a U-Boot shell, so what now?
one way i previously tackled this was to build a custom buildroot distro, put it on a USB stick, and boot from that.
but that is kind of annoying, and also means i have to recompile every time i want to add some new software.

instead, i opted to boot [Arch Linux ARM](https://archlinuxarm.org) and the generic ARM64 image they provide.
this gives me a full linux distro, with a package manager and everything, without having to recompile anything myself.
well, except for a few caveats:

- while the generic ARM64 kernel image kinda works, it doesn't really have the driver support for the LS1043A. Things like the SATA controller or networking won't work.
- there is no device tree for the T40 included. luckily, we can just use the stock one - either extracted from the original firmware, or the one uploaded on the OpenWRT forums.
- the initramfs format is not compatible with U-Boot. we can fix this by converting it to a uImage format using `mkimage`.

to summarize, prepare a USB stick as follows:

- download the generic [Arch Linux ARM](https://archlinuxarm.org/platforms/armv8/generic) image
- extract to a usb stick using `bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt/my_usb_stick`
- convert initramfs using `mkimage -A arm64 -O linux -T ramdisk -C gzip -d /mnt/my_usb_stick/boot/initramfs-linux.img /mnt/my_usb_stick/boot/initramfs.uImage`.
- (compile and) copy the T40 device tree to `/mnt/my_usb_stick/boot/wg_t40.dtb`.

now, we can plug the usb stick into the T40, and boot from it using U-Boot:

```shell
# start and scan usb
usb start

# ensure correct load addresses
env set kernel_addr_r 0x80080000
env set fdt_addr_r 0x90000000
env set ramdisk_addr_r 0x94000000

# load kernel, dtb and initramfs from usb
load usb 0:1 ${kernel_addr_r} /boot/Image
load usb 0:1 ${fdt_addr_r} /boot/wg_t40.dtb
load usb 0:1 ${ramdisk_addr_r} /boot/initramfs.uImage

# ensure bootargs are correct. can theoretically be omitted
env set bootargs console=ttyS0,115200 earlycon=uart8250,mmio,0x21c0500 root=/dev/sda1 rw rootwait

# boot the kernel image
booti ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}
```

with that, linux should boot up kinda fine.
of course, without proper drivers and a outdated device tree, not much more can be done.
there's no networking, no sata, and everything is kinda limited.

but it does run linux!

## A Custom Device Tree

for the custom device tree, i started with the [work done by @neggles](https://github.com/neggles/openwrt/commit/0e2ba8ef96acc7a6c7fcb57319e89563decea333#diff-73fafdd684ce62f3c48c572d7966775d7a9ef262213ecfea4bd6b75c58bc678c).
i still think this was the right move, even tho it caused me a lot of headaches later on.
what didn't make things better was that at this point in time, i also used a custom compiled kernel, which of course had a different configuration than the one provided by Arch Linux ARM.

### A First Hiccup: OP-TEE

running a custom kernel and device tree, i thought i'd get at least as far as i did before.
but nope, it simply crashes early during boot.

| ![this is fine?](/assets/images/common/reactions/this-is-fine.jpg) |
| ------------------------------------------------------------------ |
| _this is fine?_                                                    |

```
[ 4.220318] optee: probing for conduit method.
[ 4.231807] Internal error: Oops - Undefined instruction: 0000000002000000 [#1] SMP
[ 4.239559] Modules linked in:
[ 4.242619] CPU: 2 UID: 0 PID: 1 Comm: swapper/0 Not tainted 6.17.0-rc6-00266-gcd89d487374c #1 PREEMPT
[ 4.257512] pstate: 80000005 (Nzcv daif -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[ 4.264481] pc : __arm_smccc_smc+0x4/0x30
[ 4.268504] lr : optee_smccc_smc+0x1c/0x2c
[ 4.272605] sp : ffff800081aabac0
[ 4.275917] x29: ffff800081aabad0 x28: ffff8000815c9068 x27: ffff8000814f00b0
[ 4.283069] x26: ffff8000819c2000 x25: ffff000800100810 x24: 0000000000000000
[ 4.290220] x23: ffff000800100800 x22: 0000000000000000 x21: ffff800081aabb28
[ 4.297370] x20: ffff80008198f4c0 x19: ffff800080cdd550 x18: 0000000000000006
[ 4.304519] x17: ffff80008194dfc0 x16: 0000000014d92057 x15: ffff800081aab500
[ 4.311668] x14: ffffffffffffffff x13: 0000000000000038 x12: 0101010101010101
[ 4.318819] x11: 7f7f7f7f7f7f7f7f x10: 00007fff7e1fa464 x9 : 0000000000000018
[ 4.325969] x8 : ffff800081aabb28 x7 : 0000000000000000 x6 : 0000000000000000
[ 4.333119] x5 : 0000000000000000 x4 : 0000000000000000 x3 : 0000000000000000
[ 4.340267] x2 : 0000000000000000 x1 : 0000000000000000 x0 : 00000000bf00ff01
[ 4.347418] Call trace:
[ 4.349862] __arm_smccc_smc+0x4/0x30 (P)
[ 4.353881] optee_probe+0xd4/0x970
[ 4.357373] platform_probe+0x5c/0x98
[ 4.361044] really_probe+0xbc/0x29c
[ 4.364624] __driver_probe_device+0x78/0x12c
[ 4.368987] driver_probe_device+0xd8/0x15c
[ 4.373176] __driver_attach+0x90/0x19c
[ 4.377017] bus_for_each_dev+0x7c/0xe0
[ 4.380857] driver_attach+0x24/0x30
[ 4.384436] bus_add_driver+0xe4/0x208
[ 4.388188] driver_register+0x5c/0x124
[ 4.392028] __platform_driver_register+0x24/0x30
[ 4.396740] optee_smc_abi_register+0x1c/0x28
[ 4.401102] optee_core_init+0x28/0x68
[ 4.404858] do_one_initcall+0x80/0x1c8
[ 4.408699] kernel_init_freeable+0x204/0x2e0
[ 4.413066] kernel_init+0x20/0x1d8
[ 4.416565] ret_from_fork+0x10/0x20
[ 4.420148] Code: d53cd045 d53cd042 d53cd043 d503245f (d4000003)
[ 4.426247] ---[ end trace 0000000000000000 ]---
[ 4.430910] Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b
[ 4.438576] SMP: stopping secondary CPUs
[ 4.442507] Kernel Offset: disabled
[ 4.445993] CPU features: 0x000000,00080000,20002000,0400421b
[ 4.451742] Memory Limit: none
[ 4.454797] ---[ end Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b ]--
```

what the heck is `optee`?
this is were my inexperience with linux, especially on ARM, got really apparent to me.
i spent forever on this, trying to figure out what the heck is going on.

well, turns out that linux expected to be booted by a trusted execution environment (TEE), essentially secure boot for ARM.
but u-boot does not do that.

because the way that OP-TEE in linux works, it calls into the TEE using a special SMC instruction.
without the TEE actually there, and the CPU not configured for it, this instruction is undefined instead, and the kernel panics.

so, why is OP-TEE even enabled?
well, the culprit is in the base device tree fragment for LS1043a, defined by the linux kernel (`fsl-ls1043a.dtsi`):

```dts
/ {
  // (...)
  firmware {
  	optee {
  		compatible = "linaro,optee-tz";
  	  	method = "smc";
  	};
  };
};
```

this basically tells the kernel that there's a TEE available, and it should use it.
removing this node makes the kernel boot just fine, and is easily done by adding the following to our custom device tree:

```dts
/delete-node/ &{/firmware/optee};
```

with that, the kernel at least boots without panicking.
however, we still not back to where we were before, as now usb enumeration doesn't work anymore.

### The Pain of USB Enumeration Issues

normally, usb enumeration not working wouldn't be that much of a dealbreaker.
sure, i'd like to have usb working, but it's not essential.

except in this case, it is.
with SATA still not working, the place i put my root filesystem was on a usb stick.
and that usb stick is not being detected at all.
so, no usb = no root filesystem = no linux.

```
:: mounting '/dev/sdb1' on real root
mount: /new_root: fsconfig() failed: /dev/sdb1: Can't lookup blockdev.
       dmesg(1) may have more information after failed mount system call.
ERROR: Failed to mount '/dev/sdb1' on real root
You are now being dropped into an emergency shell.
```

so what's going on here?
the LS1043a has tree build-in usb controller, all of which are based on the Synopsys DesignWare Core 3 USB controller IP (`snps,dwc3`).
this controller supports both USB2 and USB3, is configurable as host or device, and is handled by the `USB_DWC3` linux kernel driver.

all of that is _completely_ irrelevant here, as the issue was something completely unrelated to the usb controller itself.
this took me forever to figure out, at least two weeks of research and trial-and-error.
nothing worked, until i randomly decided to remove the `dma-coherent` property from the `soc` node in the device tree.
no idea why, but that made usb enumeration work again.

the watchguard dts also doesn't have that property, so maybe this is a bug in the upstream dts file?
i don't know, but at least it works now.

in the custom device tree, we add the following and continue - confused as to why it works:

```dts
&soc {
    /delete-property/ dma-coherent;
};
```

there's one good thing about this whole ordeal:
i now am a lot more comfortable with device trees, decompiled and in source form.

## Custom Linux Kernel Configuration

until now, i simply used the ARM64 defconfig of linux.
with that, lots of things don't work, things as simple as usb network adapters.

to fix this, i extracted the kernel config from the Arch Linux ARM image, and used that as a base for my own custom kernel config.
with this, i have a sensible base to start from, and can now enable things as needed.
first on that list is the SATA controller, so i can access the internal SSD.

### The SATA Controller

this is super simple.
the device tree already has the correct node for the SATA controller, and it's not even broken.
we simply need to enable the QORIQ AHCI kernel driver with `CONFIG_AHCI_QORIQ=y`, and the SATA controller gets detected.

Note: QORIQ is NXP's marketing name for some of their SoCs, including the LS1043a.

### Thermals

it'd be nice to have thermal monitoring, but by default the sensors are not picked up.
for this, we again simply need to enable the QORIQ thermal driver with `CONFIG_QORIQ_THERMAL=y`.
with this, the thermal sensors are detected and available in `/sys/class/thermal/`.

### Networking I: Quad PHY (LAN1-4)

remember the OpenWRT forum post from earlier, the one claiming the T40 is basically an LS1043A-RDB?
well, at least when it comes to four of the five ethernet ports, this is true.

on both the RDB as well as the T40, four of the ethernet ports are handled by a Quad Ethernet PHY - in the case of the T40, a Marvell 88E1510.
simply copying over the ethernet-related parts of the RDB's device tree over to our custom T40 device tree, replacing the mdio stuff done by neggles, makes these four ports work just fine.

of course, we also need to enable the corresponding kernel drivers for Freescale's FMan (`CONFIG_FSL_FMAN=y`), but that's it.
four of five ethernet ports working, not bad.

### Networking II: Atheros PHY (WAN0)

this i still haven't figured out yet.
this part will be updated once i do.

---

note: this is a WIP post, documenting my process as i go along.
see the [wip github repo](https://github.com/shadow578/linux/tree/v6.16_wg-t40) for the current state of things.
