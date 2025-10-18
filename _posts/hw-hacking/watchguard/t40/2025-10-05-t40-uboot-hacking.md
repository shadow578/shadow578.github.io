---
layout: post
title: cracking the T40's U-Boot
tags: watchguard-t40 hardware-hacking ghidra
---

with linux mostly running (and me procrastinating continuing work on the last ethernet PHY), there still was a little hangup for me:
on the T40, it's easy enough to get into U-Boot by simply removing the SSD.
but wouldn't it be nice to have a way to get into U-Boot without having to open the case?

# The Hidden Password Prompt

that's exactly what the engineers at WatchGuard thought too, so they added a hidden password prompt to the U-Boot build.
you can access it by pressing `CTRL-C` at the SysA / SysB prompt.

| ![Hidden Password Prompt](/assets/images/hw-hacking/watchguard/t40/uboot/hidden-password-prompt.png) |
| ---------------------------------------------------------------------------------------------------- |
| Hidden Password Prompt                                                                               |

sadly, the password isn't "WatchGuard!" this time around.
would've been funny though.

# The Elusive U-Boot Patch

on the [OpenWRT forum thread about the T40](https://forum.openwrt.org/t/installing-openwrt-on-watchguard-t40w/230048), there was some [discussion](https://forum.openwrt.org/t/installing-openwrt-on-watchguard-t40w/230048/12) about a possible patch to bypass the password.
this was posted by [neggles on the OpenWRT IRC](https://oftc.catirclogs.org/openwrt-devel/2022-04-26#30875010):

> 12:02 <neggles> stintel: I binpatched the u-boot on my T20 to get into it
> 12:02 <neggles> changed one cbz to cbnz

sadly, no further details were given on where to patch the binary.
so I set out to figure it out myself.
how hard could it be?

## Loading The U-Boot Binary Into Ghidra

this first step is where i hit my first issues, really making me reconsider if this was worth the effort.
dumping the U-Boot binary was easy enough using the working linux system, as the MTB partition is simply exposed as `/dev/mtd1` (even with a nice label "U-Boot").

that is where the easy part ends, though.

i spend forever trying to find the base address to load the binary at.
all this was made 100x harder by me being an idiot and choosing the wrong ARM architecture in Ghidra.

eventually, i stumbled upon [binbloom](https://github.com/quarkslab/binbloom), which is a really neat tool for analysing raw firmware binaries.
i had never heard of it before, but it worked really well for this purpose:

```bash
$ binbloom -e be mtd1_uboot.bin
[i] Selected big-endian architecture.
[i] File read (1048576 bytes)
[i] Endianness is BE
[i] 3318 strings indexed
[i] Found 31351 base addresses to test
[i] Base address seems to be 0x00f8b000 (not sure).
```

loading the binary at `0x00f8b000` in Ghidra looked really promising, so i think it's correct.

## Figuring Out The Password Check

before we can patch anything, we need to understand what is happening.
as a quick start, i decided to search for references to the header string we saw in the password prompt, and got to a function that likely handles drawing the prompt.
single caller is a function that looks suspiciously like it handles the bootdelay count down, as well as having a direct reference to the string "password>".
suspicious indeed.

cleaning stuff up (a lot!), here's some snippets of the relevant code (function's really long, so just the relevant parts):

```c
// (...)
  password_entered_ok_maybe = false;
  scrambled_correct_password = SCRAMBLED_CORRECT_PASSWORD;
  // (...)
  printf_ish("Hit any key to stop autoboot: %2d",bootdelay);
redraw_bootdelay_countdown_maybe:
  if (/* (...) */) {
    // (...)
    do {
      iVar3 = ControlCPressed_likely();
      if (iVar3 != 0) goto do_password_check;
      // (...)
    } while (ms_delay_counter < 1000);
    // (...)
  }
  // (...)
  if (password_entered_ok_maybe) {
    return;
  }
  // (...)

do_password_check:
  while (read_char != `\r`) {
    while( true ) {
      read_char = maybe_read_serial_char();
      // (...)

      // get_password_text_with_prompt displays the string as a prompt,
      // then reads input from serial input until ENTER is pressed.
      // input is stored in PASSWORD_INPUT global.
      pwdlen = get_password_text_with_prompt("password>");
      if (0 < pwdlen) {
        scramble_password(&PASSWORD_INPUT,pwdlen,scrambled_password);
        pwdlen = memcmp(&scrambled_correct_password,scrambled_password,0x14);
        if (pwdlen == 0) {
          password_entered_ok_maybe = true;
          goto boot_or_redraw;
        }
      }
      // (...)
    }
  }
  password_entered_ok_maybe = false;
// (...)
```

i think we have a winner here.
best part: the `if (pwdlen == 0)` check is implemented using a `CBZ` instruction, which is exactly what neggles mentioned in the IRC.
patching this to a `CBNZ` should make it so that any entered password (well, except for the correct one) is accepted.

to patch your binary the same way that neggles likely did, simply open the binary in a hex editor, search for the hex pattern `e32f0194c0000034`, and replace the last byte `34` with `35`.
the byte should be at offset `0x199C3` in /dev/mtd1, or `0x1199C3` in the whole flash dump.
note that these offsets are specific to the T40's U-Boot binary, and may differ in other devices or U-Boot versions.

after flashing the patched binary back, simply press `CTRL-C` at the boot selection, enter any password (cannot be empty), and you're dropped into U-Boot.

## Cracking the Password

nope, didn't do that.
though i'm curious, `scamble_password` is a rather long and annoying function to reverse.
maybe someone else wants to give it a try?
