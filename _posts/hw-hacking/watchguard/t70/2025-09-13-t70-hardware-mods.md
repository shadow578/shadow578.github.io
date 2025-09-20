---
layout: post
title: (Somewhat) Cursed WatchGuard T55 / T70 Hardware Mods
tags: hardware-hacking
---

with both of my watchguards fully jailbroken and behaving (mostly) like a normal linux box, but still waiting on some smaller details before actually utilizing them, i decided to poke around the hardware a bit more.

my motivation on this was straight-forward: both of my units have some unpopulated components on the pcb, so i wanted to see if i could get them working.
also, there's the issue with the switch ports beign disabled by default, as the original os configured them in software...

## T55 & T70: Enabling the switch ports

this one's an already well known mod, but i figured i'd document it here anyway.

the switch chip on both the t55 and t70 is, by default, set to be configured by the main cpu via an mdio interface (connected to the backplane NIC's mdio port).
only issue is, you'd need a custom kernel module to do that, and there isn't one available outside of the original firmware.

luckily, by simply removing a single resistor (R607 on both the T55 and T70), you can force the switch into "dumb" mode, where it just acts like a regular unmanaged switch.
credits to [@konus on the OpenWRT forums](https://forum.openwrt.org/t/watchguard-t70-hw-discovery/155544/2) for figuring this out.

| ![removing R607 makes the switch dumb](/assets/images/hw-hacking/watchguard/t70/hw-mod/switch-unmanaged-mode.png) |
| ----------------------------------------------------------------------------------------------------------------- |
| removing R607 makes the switch dumb                                                                               |

## T55: Adding a WiFi module

besides the T55, watchguard also offers a T55-W model, which has WiFi capabilities - but is otherwise identical to the regular T55.
to save costs, watchguard simply removed the mini-PCIe slot and the associated components from the regular T55 model.
nothing we can't fix, though!

to restore the mini-PCIe slot, we simply need to add the inline capacitors of the pcie lanes (C271 and C272), as well as some in-line resistors for mpcie control lines (R8547 and R8551).
oh, and of course, the mini-PCIe slot itself.

annoyingly, watchguard also omits the power supply components for the mini-PCIe slot.
but we can simply steal power from the switch's power rail (@ TP61), as both use 3.3V, and a mini-PCIe card probably won't draw that much power anyway.

| ![Not pretty, but it works](/assets/images/hw-hacking/watchguard/t70/hw-mod/t55-mpcie.png) |
| ------------------------------------------------------------------------------------------ |
| Not pretty, but it works                                                                   |

## T70: Adding HDMI Output

both the T55 and T70 have an unpopulated HDMI port near in the bottom left corner of the PCB.
however, curiously, on the T70 they did **not** omit the associated components to actually make use of it.

meaning, we can simply solder a HDMI port onto the board, and get video output!

| ![HDMI port soldered onto the T70](/assets/images/hw-hacking/watchguard/t70/hw-mod/t70-hdmi.png) |
| ------------------------------------------------------------------------------------------------ |
| HDMI port soldered onto the T70                                                                  |

with this alone, we still don't get any output, as the iGPU is disabled by default in the BIOS.
however, since we have full BIOS access, we can simply enable it there, and voilà - we have video output!

now, what to do with it ...

### Windows 11 on the T70 ?!

with a video output, there's nothing stopping us from installing Windows 11 on the T70.
surely some random firewall appliance is a supported platform, right Microsoft?

installation and usage went pretty effortlessly (albeit quite slow, as theres only 2GB of RAM and the rest of the hardware is not exactly high-end either).

only quirk: for some reason, the OEM logo embedded into the BIOS is really messed up.

| ![Booting Windows 11 on the T70](/assets/images/hw-hacking/watchguard/t70/hw-mod/t70-windows-boot.png) |
| --------------------------------------------------------------------------------------------------- |
| Booting Windows 11 on the T70                                                                       |
