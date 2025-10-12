---
layout: post
title: Overclocking the WatchGuard T40
tags: hardware-hacking overclocking
---

with linux mostly running (and me still procrastinating continuing further work on it), i figured i'd try and solve my last hang-up with the T40.
the T40's SOC is a NXP QorIQ LS1043A, and that supports up to 1.6GHz.
however, the T40 is clocked at only 1.0GHz.

only when springing for the T45, which is mostly the same device but clocked higher, do you get the 1.6GHz clockspeed.
that's unfair!
i paid for the full LS1043A, so i want to use all of it.

# How The CPU Speed Is Set

linux has a cpu frequency driver for the LS1043A (specifically, all QorIQ SoCs).
however, that can only scale **down** from the maximum frequency.
setting that maximum frequency is done even before the bootloader is run, by something appropriately named the "Pre-Boot Loader" (PBL), which loads something called a "Reset Configuration Word" (RCW) from memory on boot.

## Something Familiar

i was already familiar with the concept of an RCW, as the HC32F460 microcontroller (that i ported [Arduino](https://github.com/shadow578/framework-arduino-hc32f46x/) and Marlin to) uses something similar.
it just so happens that NXP made their boot ROM much more flexible, allowing not just a fixed register to be loaded, but instead allowing writes to _any_ register.
these writes are called "Pree-Boot Initialization" (PBI) commands, and are stored in a binary data structure at the start of the memory device configured by some GPIOs.

the structure of these commands is super simple, it's just a bite count field (up to 64 bytes), a target address (at least 24 bits of it), and the data itself.
everything the PBL does, from the RCW to doing a CRC on the PBI data, is done through these commands.

as for why NXP choose to do it this way, has a fairly clever reason.
simply by changing the PBI data, you can do all sorts of stuff, but importantly: you can fix errata simply by changing the SDK source code that generates the PBI data.
this isn't just in theory either: NXP has fixed two PCIe-related errata ([A-009859](https://github.com/nxp-qoriq/rcw/blob/devel/ls1043aqds/a009859.rcw) and [A-009929](https://github.com/nxp-qoriq/rcw/blob/devel/ls1043aqds/a009929.rcw)) in the LS1043A this way, simply by writing magic values to otherwise undocumented registers.
all-in-all, pretty cool.

# Understanding and Parsing the PBI Data

to change the CPU clock speed, we need to change part of the PBI (specifically, the RCW) that sets up the PLLs for the CPU core cluster.
however, the PBI includes a CRC-32 checksum at the end, so we can't just change the data and expect it to work.

to make this whole process easier, and to understand how everything works, i decided to write a small tool to parse and print the PBI data, as well as calculating the CRC-32 checksum needed.
that shouldn't be too hard, especially since NXP provides a comprehensive [reference manual](https://www.nxp.com/products/LS1043A) for the LS1043A, which includes everything we need to know about.

as for why i decided to go this overboard, well... let's just say that it'll help us for what i plan to do later.

| ![it's a suprise tool that'll help us later](/assets/images/common/reactions/its-a-suprise-tool-later.png) |
| ---------------------------------------------------------------------------------------------------------- |
| _it's a suprise tool that'll help us later_                                                                |

if you're interested, i published the code of the tool on [GitHub](https://github.com/shadow578/QorIQ_PBI_dump).
just note that it's far from perfect, and very much not meant for production use.

for that, you should probably use [NXP's rcw tool](https://github.com/nxp-qoriq/rcw) instead.
i only found this after writing my own tool, so... oh well.
luckily (at least so i can tell myself that i didn't waste my time), NXP's tool isn't able to handle the weird stuff that WatchGuard does, so mine is still useful.

## The Weird Stuff WatchGuard Does

WatchGuard does something really weird - and frankly incredibly stupid - in their PBI.
to cut an incredibly long, and frustrating, story short: they for some reason decided to use the PBI to switch the QuadSPI peripheral's endianness from 64-bit little-endian to 32-bit big-endian.
now, you could excuse this if this was some default setting of the SoC, but no: you can freely choose the on-reset configuration for this using strapping pins.
the engineers at WatchGuard just... decided to do this for some reason.

| ![WatchGuard deciding on data endianness](/assets/images/common/reactions/the-office-parkour.gif) |
| ------------------------------------------------------------------------------------------------- |
| _WatchGuard deciding on data endianness_                                                          |

what that meant for me is that i had to handle the behaviour of the QuadSPI peripheral in my PBI parser, **and** handle when the PBI write switches the endianness.
figuring all that out was a huge pain, and took way too long.
but i eventually did it, and now my PBI parser can handle the T40's PBI data.
take a look:

<details>
<summary>The Output is pretty long, so click here to expand it</summary>

<pre>
-- System Info --
 Flash dump: ../rcwmod/spiflash.20251012.1000mhz.bin
 Initial QSPI endianess: QWORD_LITTLE_ENDIAN
 Assumed SOC Model: LS1043A

-- PBI Parser Messages ---
ALTCBAR set to 00002200 by PBI write
ALTCBAR set to 00000300 by PBI write
QSPI Endianess set to DWORD_LITTLE_ENDIAN due to QUADSPI_MCR write

-- RAW PBI Frames --
WRITE @ DCFG_CCSR_RCWSR0 (0x01EE0100): 0610000a0a0000000000000000000000455800020000001240044000c10020000000000000000000000000000003fffe200045040418320a0000009600000001 (CCSR)
WRITE @ SCFG_QSPI_CFG (0x0157015C): 40100000 (CCSR)
WRITE @ SCFG_SCRATCHRW0 (0x01570600): 00000000 (CCSR)
WRITE @ SCFG_SCRATCHRW1 (0x01570604): 40100000 (CCSR)
WRITE @ LNDSSCR1 (0x01EA08DC): 00502880 (CCSR)
WRITE @ CCI_UNKN_BARRIER_DISABLE (0x01570178): 0000e010 (CCSR)
WRITE @ CCI_400_Control_Override (0x01180000): 00000008 (CCSR)
WRITE @ SCFG_USB_REFCLK_SELCR1 (0x01570418): 0000009e (CCSR)
WRITE @ SCFG_USB_REFCLK_SELCR2 (0x0157041C): 0000009e (CCSR)
WRITE @ SCFG_USB_REFCLK_SELCR3 (0x01570420): 0000009e (CCSR)
WRITE @ DCFG_CCSR_RSTRQMR1 (0x01EE00C0): 00004400 (CCSR)
WRITE @ SCFG_ALTCBAR (0x01570158): 00002200 (CCSR)
WRITE @ PEX_OUTBOUND_WRITE_HANG_ERRATUM (0x22008040): 00000001 (ACS)
WRITE @ LNDSSCR1 (0x01EA08DC): 00502880 (CCSR)
WRITE @ SCFG_ALTCBAR (0x01570158): 00000300 (CCSR)
WRITE @ PEX1_GEN3_RElATED_OFF_ERRATUM (0x03400890): 01000100 (ACS)
WRITE @ PEX2_GEN3_RElATED_OFF_ERRATUM (0x03500890): 01000100 (ACS)
WRITE @ PEX3_GEN3_RElATED_OFF_ERRATUM (0x03600890): 01000100 (ACS)
WRITE @ QUADSPI_MCR (0x01550000): 000f400c (CCSR)
COMMAND @ PBI_CRC: 0x06DE05A9 (current) == 0x06DE05A9 (calculated) -> PASS

-- RCW Fields --
             SYS_PLL_CFG: 0x0    | 0      | 0b0                | OK
             SYS_PLL_RAT: 0x3    | 3      | 0b11               | 3:1
             MEM_PLL_CFG: 0x0    | 0      | 0b0                | OK
             MEM_PLL_RAT: 0x10   | 16     | 0b10000            | 16:1
            CGA_PLL1_CFG: 0x0    | 0      | 0b0                | OK
            CGA_PLL1_RAT: 0xA    | 10     | 0b1010             | 10:1
            CGA_PLL2_CFG: 0x0    | 0      | 0b0                | OK
            CGA_PLL2_RAT: 0xA    | 10     | 0b1010             | 10:1
              C1_PLL_SEL: 0x0    | 0      | 0b0                | CGA_PLL1 /1
           SRDS_PRTCL_S1: 0x4558 | 17752  | 0b100010101011000  |
             FM1_MAC_RAT: 0x1    | 1      | 0b1                |
 SRDS_PLL_REF_CLK_SEL_S1: 0x0    | 0      | 0b0                |
              HDLC1_MODE: 0x0    | 0      | 0b0                |
              HDLC2_MODE: 0x0    | 0      | 0b0                |
          SRDS_PLL_PD_S1: 0x0    | 0      | 0b0                |
            SRDS_DIV_PEX: 0x0    | 0      | 0b0                |
          DDR_REFCLK_SEL: 0x1    | 1      | 0b1                |
         LYNX_REFCLK_SEL: 0x0    | 0      | 0b0                |
           DDR_FDBK_MULT: 0x2    | 2      | 0b10               |
                 PBI_SRC: 0x4    | 4      | 0b100              | QSPI
                 BOOT_HO: 0x0    | 0      | 0b0                | All cores expect core 0 in hold off
                   SB_EN: 0x0    | 0      | 0b0                | Secure Boot Disabled
                IFC_MODE: 0x44   | 68     | 0b1000100          |
      HWA_CGA_M1_CLK_SEL: 0x6    | 6      | 0b110              | Async mode, Cluster Group A PLL 2 /3 is clock
                DRAM_LAT: 0x1    | 1      | 0b1                | 8-8-8 or higher latency DRAMs
                DDR_RATE: 0x0    | 0      | 0b0                |
                DDR_RSV0: 0x0    | 0      | 0b0                |
             SYS_PLL_SPD: 0x1    | 1      | 0b1                |
             MEM_PLL_SPD: 0x0    | 0      | 0b0                |
            CGA_PLL1_SPD: 0x0    | 0      | 0b0                |
            CGA_PLL2_SPD: 0x0    | 0      | 0b0                |
            HOST_AGT_PEX: 0x0    | 0      | 0b0                |
                 GP_INFO: 0x0    | 0      | 0b0                |
                UART_EXT: 0x0    | 0      | 0b0                | See UART_BASE
                 IRQ_EXT: 0x0    | 0      | 0b0                |
                 SPI_EXT: 0x0    | 0      | 0b0                | See SPI_BASE
                SDHC_EXT: 0x0    | 0      | 0b0                | See SDHC_BASE
               UART_BASE: 0x7    | 7      | 0b111              | { UART1_SOUT, UART1_SIN, UART3_SOUT,UART3_SIN, UART2_SOUT, UART2_SIN, UART4_SOUT, UART4_SIN }
                  ASLEEP: 0x1    | 1      | 0b1                |
                     RTC: 0x1    | 1      | 0b1                |
               SDHC_BASE: 0x1    | 1      | 0b1                | GPIO2[4:9]
                 IRQ_OUT: 0x1    | 1      | 0b1                |
                IRQ_BASE: 0x1FF  | 511    | 0b111111111        |
                SPI_BASE: 0x2    | 2      | 0b10               | GPIO2[0:3]
           IFC_GRP_A_EXT: 0x1    | 1      | 0b1                |
           IFC_GRP_D_EXT: 0x0    | 0      | 0b0                |
          IFC_GRP_E1_EXT: 0x0    | 0      | 0b0                | See IFC_GRP_E1_BASE
           IFC_GRP_F_EXT: 0x1    | 1      | 0b1                |
           IFC_GRP_G_EXT: 0x0    | 0      | 0b0                |
         IFC_GRP_E1_BASE: 0x1    | 1      | 0b1                | GPIO2[10:12]
          IFC_GRP_D_BASE: 0x1    | 1      | 0b1                |
          IFC_GRP_A_BASE: 0x1    | 1      | 0b1                |
             IFC_A_22_24: 0x0    | 0      | 0b0                |
                     EC1: 0x0    | 0      | 0b0                | RGMII1
                     EC2: 0x1    | 1      | 0b1                | GPIO3, GPIO3[19:23]
                     EM1: 0x0    | 0      | 0b0                | MDC/MDIO (EM1)
                     EM2: 0x0    | 0      | 0b0                | MDC/MDIO (EM2)
              EMI2_DMODE: 0x1    | 1      | 0b1                |
              EMI2_CMODE: 0x1    | 1      | 0b1                |
             USB_DRVVBUS: 0x0    | 0      | 0b0                | USB_DRVVBUS
            USB_PWRFAULT: 0x0    | 0      | 0b0                | USB_PWRFAULT
               TVDD_VSEL: 0x1    | 1      | 0b1                | 2.5V
               DVDD_VSEL: 0x2    | 2      | 0b10               | 3.3V
          QE_CLK_OVRRIDE: 0x0    | 0      | 0b0                |
              EMI1_DMODE: 0x1    | 1      | 0b1                |
               EVDD_VSEL: 0x0    | 0      | 0b0                | 1.8V
               IIC2_BASE: 0x0    | 0      | 0b0                | OK
              EMI1_CMODE: 0x1    | 1      | 0b1                |
                IIC2_EXT: 0x2    | 2      | 0b10               | GPIO4_2, GPIO4_3
             SYSCLK_FREQ: 0x258  | 600    | 0b1001011000       | 100.000 MHz (100000200 Hz)
      HWA_CGA_M2_CLK_SEL: 0x1    | 1      | 0b1                | Async mode, Cluster Group A PLL 2 /1 is clock

-- Effective Clocks --
 SYSCLK:                       100.00 MHz
 System (Bus):                 300.00 MHz
 CGA (Cores):                  1000.00 MHz
 MEM (Memory):                 1600.00 MHz
 HWA_CGA_M1 (FMAN):            500.00 MHz
 HWA_CGA_M2 (eSDHC & QuadSPI): 1000.00 MHz

-- Erratum Workarounds --
 Erratum A-009859 workaround: Yes
 Erratum A-009929 workaround: Yes

-- CRC Results --
 CRC Frame Present: Yes
 CRC Offset:     0x00DC
 In-File CRC:    0x06DE05A9
 Calculated CRC: 0x06DE05A9
 CRC Valid?:     Yes
</pre>

</details>

# Changing The CPU Clock Speed

changing the CPU clock is fairly simple, once you have a way to actually create a valid PBI.
the SoC runs on a single 100MHz reference clock (SYSCLK), and the CPU clock is derived from that using a PLL.
the PLL for the CPU (and actually everything else) is configured using the `CGA_PLL1_RAT` (Cluster Group A PLL 1 Ratio) field in the RCW.
well, actually there is a second PLL (`CGA_PLL2_RAT`), and you can freely choose which one to use for the CPU cores, but that's not important right now.

there's some other things to keep in mind, mainly since the `CGA_PLLn` clock is also used for the hardware accceleration engines (FMAN = Frame Manager) and the eSDHC and QuadSPI peripherals.
the clocks of those can be derived from either of the `CGA_PLLn` PLLs.
luckily, in the T40's RCW, they use PLL2 while the CPU cores use PLL1, so we can change the CPU clock without affecting those peripherals.

other than that, there's also the option to divide the PLL clock by either 1 or 2 depending on the `C1_PLL_SEL` field, but we'll simply leave it at PLL1 /1.

with that, we can start overlocking the CPU as follows:

- set `CGA_PLL1_RAT` to the desired multiplier (5 - 40). the field is at offset 0x0C in the SPI flash dump. For 1.0GHz, it's set to 0xA (10:1), for 1.6GHz, set it to 0x10 (16:1).
- run `pbidump` to check the changes, and to calculate the new CRC-32 checksum.
- write the new CRC-32 checksum at offset 0xDC in the SPI flash dump.
- run `pbidump` again to verify that everything is correct.
- flash the modified image to the T40's SPI flash.
- cross fingers and boot.

do all that, and you should be greeted by u-boot reporting 1600MHz CPU clock:

```
U-Boot 2018.09 (Dec 05 2019 - 10:13:47 -0800)

SoC:  LS1043AE Rev1.1 (0x87920011)
Clock Configuration:
       CPU0(A53):1600 MHz  CPU1(A53):1600 MHz  CPU2(A53):1600 MHz
       CPU3(A53):1600 MHz
       Bus:      300  MHz  DDR:      1600 MT/s  FMAN:     500  MHz
Reset Configuration Word (RCW):
       00000000: 06100010 0a000000 00000000 00000000
       00000010: 45580002 00000012 40044000 c1002000
       00000020: 00000000 00000000 00000000 0003fffe
       00000030: 20004504 0418320a 00000096 00000001
Model: LS1043A QDS Board - T40/T20
Board: LS1043AQDS, boot from vBank: 0
```

success!

## Benchmarking

to see how much of a difference this makes, i threw together a quick test using both [coremark](https://github.com/eembc/coremark) and [coremark-pro](https://github.com/eembc/coremark-pro).
the results are really promising, showing almost linear scaling with the increased clock speed.

| Run           | CoreMark    | CoreMark-Pro (SingleCore) | CoreMark-Pro (MultiCore) |
| ------------- | ----------- | ------------------------- | ------------------------ |
| T40 @ 1.0 GHz | 3276 (+0%)  | 686 (+0%)                 | 2178 (+0%)               |
| T40 @ 1.6 GHz | 5268 (+61%) | 1097 (+60%)               | 3326 (+53%)              |

<details>
<summary>Click for raw benchmark results</summary>

T40 @ 1.0GHz:

<pre>
-- CPU Freq Config --
Setting cpu: 0
Setting cpu: 1
Setting cpu: 2
Setting cpu: 3
analyzing CPU 0:
  driver: qoriq_cpufreq
  CPUs which run at the same hardware frequency: 0 1 2 3
  CPUs which need to have their frequency coordinated by software: 0 1 2 3
  maximum transition latency: 41 ns
  hardware limits: 500 MHz - 1000 MHz
  available frequency steps:  1000 MHz, 500 MHz
  available cpufreq governors: conservative ondemand userspace powersave performance schedutil
  current policy: frequency should be within 500 MHz and 1000 MHz.
                  The governor "performance" may decide which speed to use
                  within this range.
  current CPU frequency: 1000 MHz (asserted by call to hardware)

-- CoreMark --
2K performance run parameters for coremark.
CoreMark Size    : 666
Total ticks      : 12209
Total time (secs): 12.209000
Iterations/Sec   : 3276.271603
Iterations       : 40000
Compiler version : GCC14.2.1 20250207
Compiler flags   : -O2 -DPERFORMANCE_RUN=1  -lrt
Memory location  : Please put data memory location here
                        (e.g. code in flash, data on heap etc)
seedcrc          : 0xe9f5
[0]crclist       : 0xe714
[0]crcmatrix     : 0x1fd7
[0]crcstate      : 0x8e3a
[0]crcfinal      : 0x25b5
Correct operation validated. See README.md for run and reporting rules.
CoreMark 1.0 : 3276.271603 / GCC14.2.1 20250207 -O2 -DPERFORMANCE_RUN=1  -lrt / Heap

-- CoreMark-Pro --
WORKLOAD RESULTS TABLE

                                                 MultiCore SingleCore
Workload Name                                     (iter/s)   (iter/s)    Scaling
----------------------------------------------- ---------- ---------- ----------
cjpeg-rose7-preset                                  113.64      29.33       3.87
core                                                  0.81       0.20       4.05
linear_alg-mid-100x100-sp                            39.18      10.19       3.84
loops-all-mid-10k-sp                                  1.27       0.45       2.82
nnet_test                                             2.68       0.80       3.35
parser-125k                                          18.78       6.90       2.72
radix2-big-64k                                      123.11      72.61       1.70
sha-test                                            192.31      57.47       3.35
zip-test                                             57.14      15.38       3.72

MARK RESULTS TABLE

Mark Name                                        MultiCore SingleCore    Scaling
----------------------------------------------- ---------- ---------- ----------
CoreMark-PRO                                       2177.64     686.01       3.17
</pre>

T40 @ 1.6GHz:

<pre>
-- CPU Freq Config --
Setting cpu: 0
Setting cpu: 1
Setting cpu: 2
Setting cpu: 3
analyzing CPU 0:
  driver: qoriq_cpufreq
  CPUs which run at the same hardware frequency: 0 1 2 3
  CPUs which need to have their frequency coordinated by software: 0 1 2 3
  maximum transition latency: 41 ns
  hardware limits: 500 MHz - 1.60 GHz
  available frequency steps:  1.60 GHz, 1000 MHz, 800 MHz, 500 MHz
  available cpufreq governors: conservative ondemand userspace powersave performance schedutil
  current policy: frequency should be within 500 MHz and 1.60 GHz.
                  The governor "performance" may decide which speed to use
                  within this range.
  current CPU frequency: 1.60 GHz (asserted by call to hardware)
-- CoreMark --
2K performance run parameters for coremark.
CoreMark Size    : 666
Total ticks      : 18981
Total time (secs): 18.981000
Iterations/Sec   : 5268.426321
Iterations       : 100000
Compiler version : GCC14.2.1 20250207
Compiler flags   : -O2 -DPERFORMANCE_RUN=1  -lrt
Memory location  : Please put data memory location here
                        (e.g. code in flash, data on heap etc)
seedcrc          : 0xe9f5
[0]crclist       : 0xe714
[0]crcmatrix     : 0x1fd7
[0]crcstate      : 0x8e3a
[0]crcfinal      : 0xd340
Correct operation validated. See README.md for run and reporting rules.
CoreMark 1.0 : 5268.426321 / GCC14.2.1 20250207 -O2 -DPERFORMANCE_RUN=1  -lrt / Heap
-- CoreMark-Pro --
WORKLOAD RESULTS TABLE

                                                 MultiCore SingleCore
Workload Name                                     (iter/s)   (iter/s)    Scaling
----------------------------------------------- ---------- ---------- ----------
cjpeg-rose7-preset                                  181.82      47.17       3.85
core                                                  1.31       0.33       3.97
linear_alg-mid-100x100-sp                            62.97      16.40       3.84
loops-all-mid-10k-sp                                  1.84       0.70       2.63
nnet_test                                             4.30       1.29       3.33
parser-125k                                          28.99      10.99       2.64
radix2-big-64k                                      147.43     113.48       1.30
sha-test                                            312.50      92.59       3.38
zip-test                                             88.89      24.39       3.64

MARK RESULTS TABLE

Mark Name                                        MultiCore SingleCore    Scaling
----------------------------------------------- ---------- ---------- ----------
CoreMark-PRO                                       3325.58    1096.57       3.03
</pre>

</details>

# Conclusion

this was really fun to do, and i'm super happy that it worked out.
to quote a [famous Scientist](https://www.youtube.com/watch?v=Y6ljFaKRTrI) (?):

> This was a triumph  
> I'm making a note here, huge success  
> It's hard to overstate  
> My satisfaction
>
> - GLaDOS
