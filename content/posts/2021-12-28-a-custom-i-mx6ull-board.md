---
template: post
title: A Custom i.MX6ULL Board
slug: catfood-custom-imx6ull-board
socialImage: /media/catfood_layout.png
draft: false
date: 2021-12-28T02:25:16.649Z
description: As a vehicle for learning OrCAD, I built a custom embedded Linux
  board using NXP's i.MX6ULL processor. The board, called CATFOOD, has the
  i.MX6, DDR3, NAND flash, an Ethernet PHY, and an SD card. It presents a serial
  console over the USB-C connector, which also supplies power.
category: Hardware
tags:
  - Hardware
  - PCB
  - Linux
---
As a vehicle for learning OrCAD, I built a custom embedded Linux board using NXP's i.MX6ULL processor. The board is called CATFOOD (I was more excited to get started than I was to come up with a name). It has the i.MX6, DDR3, NAND flash, an Ethernet PHY, and an SD card. It presents a serial console over the USB-C connector, which also supplies power.

![CATFOOD layout screenshot](/media/catfood_layout.png "CATFOOD layout screenshot")

This was my first embedded Linux design, as well as my first design incorporating DDR or Ethernet. The design is heavily based on NXP's reference implementation. I also used Seeed Studio's i.MX6 module as a reference (it is itself a stripped-down version of the NXP design).

![A half-populated PCB.](/media/catfood_paste.jpg "Half-populated CATFOOD")

I had the 4-layer board and a stencil manufactured using a low-cost PCB prototyping service and assembled it at home.

## Issues/Bring-up

### DDR

There was at first an issue booting. I traced this down to a problem with the DDR by interrogating the device over the built-in USB bootloader. See <https://github.com/boundarydevices/imx_usb_loader> as well as NXP's DDR calibration tool. After checking the DDR voltage reference and a few clock termination passives I reflowed the memory chip, which solved the issue. If I had to guess (and I do, because I don't have an X-ray machine), I didn't get enough heat on that part the first time I reflowed it.

![Assembled PCB](/media/catfood_assembled.jpg "Assembled PCB")

### Ethernet

There were a few driver problems with the Ethernet. This particular part (LAN8700) was my most significant deviation from the reference schematic so it's no surprise that the board init and device tree needed some modifications.

* I needed to have u-boot reset the PHY (using the reset GPIO) early in the board initialization. I also set up the ENET clock (provided by the i.MX6 to the LAN8700) in the board init.
* The reference device tree has two ENET modules, with the (shared) MDIO bus configured in the second. This is usually fine even when only one is used, but the part I used has the second ENET module disabled via fuse[^1].
* I removed the tempmon node in the device tree. It doesn't exist in my part and so Linux kept attempting to do deferred probes of this device. I'm still not clear on why, but the first attempted tempmon probe after Linux initializes the PHY would cause a hang.

Finally, I [generated](https://www.hellion.org.uk/cgi-bin/randmac.pl?scope=local&type=unicast) a valid (unicast, locally-administered) MAC address and burned it into the fuses using u-boot:

```
# This programs ENET1 MAC address: F6:07:37:F1:EC:7a
fuse prog 4 3 0xf607
fuse prog 4 2 0x37f1ec7a
```

![Photo of a monitor with a running CATFOOD in the foreground. The monitor shows a working ping to iank.org](/media/catfood_ping.jpg "Working Ethernet")

## Lessons Learned

I'm happy with the way this came out. It incorporated a lot of new elements for me and they all ended up working. Some disorganized thoughts:

* I'm stressing the limits of my usual paste stencil alignment method (scrap boards and masking tape). Next time I'm going to try putting some alignment holes in the board and stencil.
* The reference design includes a large number of pull-up/down resistors for configuring bootstrap pins. I kept these on my board, knowing they were unnecessary but maybe useful. After learning a bit about the platform I shouldn't need to include them again. All that's needed is access to a couple of the BOOT_MODE pins, combined with the default behavior and the built-in USB loader.
* There's some other elements of NXP's design, mostly around power and reset, that I think I understand well enough now to simplify or drop.
* This was the first time I did stencil paste application / hot air reflow on both sides of the board, and I was worried about either having components fall off or not preheating the board/ground planes enough and having tombstoning. I set my preheater to "100C" and didn't have any issues.
* When placing small passives I should give them a little tap (into the paste) and use a wide nozzle from relatively far to prevent parts from blowing away.
* I designed a 3d-printed enclosure, which I don't typically do. I need to pay more attention to connector stickout past the board edge.

[^1]: Both ENET modules have instances of the MDIO registers, and it normally doesn't matter which ENET is used to access the bus. But with ENET2 fused, accessing the MDIO registers from that module hung the device.
