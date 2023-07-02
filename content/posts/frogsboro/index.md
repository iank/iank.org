---
template: post
title: "FROGSBORO: SAM9X60 SiP Embedded Linux"
slug: frogsboro-embedded-linux-board-sam9x60-sip
featured_image: /media/frogsboro_sq.jpg
draft: false
date: 2022-01-08T04:44:49.895Z
description: A custom embedded Linux board based on the SAM9X60 SiP.
category: Hardware
tags:
  - hardware
  - pcb
  - linux
  - sam9x60
---
Another custom embedded Linux board. I recently did a free evaluation of Altium Designer and built this as a way to try it out. The design is based on the SAM9X60 SiP and has a USB device port, two USB host ports, a microSD card, and a low profile 40-pin expansion connector (following the Raspberry Pi pinout) which breaks out GPIOs and various other peripherals. It's called FROGSBORO, which I guess is a moderate improvement over [CATFOOD](https://iank.org/posts/catfood-custom-imx6ull-board).

Design files: [Schematic (pdf)](/media/frogsboro_schematic_v1.0.2.pdf), [BOM (pdf)](/media/frogsboro_bom_v1.0.2.pdf), [Gerbers (zip)](/media/frogsboro_gbr_v1.0.2.zip)

**Update 2022-04-17**: Added a section on the stencil alignment fixture.

{{< figure src="frogsboro_top_complete.jpg" caption="Completed PCB, top view." alt="The board measures 67mm by 21mm, has a mini USB-B connector sticking out over one end and two vertical USB-A connectors on the other." >}}

{{< figure src="frogsboro_bottom_complete_side.jpg" caption="Completed PCB, bottom view." alt="There is a microSD card connector on one end and a 40-pin low-profile board-to-board connector nearby." >}}

## Layout

{{< figure src="frogsboro_layout.png" caption="" alt="A screenshot of some of the gerber layers superimposed upon each other." >}}

I used a six layer stackup from a low-cost fab. It measures 67mm by 21mm. A MIC2800 PMIC provides the three necessary voltage rails. An oscillator and a crystal provide the core clocks, there's a DDR voltage reference, and the various connectors have ESD protection. (This is the first spin of the board; the 1.0.2 version number reflects some back and forth with the fab). The total cost for PCBs, stencils, and components was around $200.

## Assembly

### Stencil Alignment Fixture

I'm experimenting with a new alignment fixture for applying paste. In the past I've used scrap PCBs to frame the board and a masking tape hinge on one edge of the stencil [(Video from JLCPCB](https://www.youtube.com/watch?v=uXvXwzQf1gU), [Tutorial from Sparkfun](https://www.sparkfun.com/tutorials/58)). This has worked well enough for me. It takes some time to line everything up and tape it down, but once set up it's surprisingly repeatable. It's stressful to set up for just a few boards though, and double-sided assembly is a pain since the setup needs to be elevated to apply paste to the second side.

{{< figure src="frogsboro_blank.jpg" caption="The bare PCBs (top and bottom), with the breakaway rails still attached." alt="Two blank PCBs side-by-side. The PCBs have rails on either side that are attached by thin tabs. The tabs have mouse bites drilled into them and can be broken away." >}}

On this board I added breakaway rails with mouse bites[^1]. The rails each contain two stencil alignment holes, which are just a non-plated through hole and a corresponding footprint on the paste layer. Since it's not typical[^2] to have paste over holes I included a note to the fab to clarify that this was intentional. I then milled a fixture from a block of common pine.

{{< figure src="frogsboro_stencil_fixture_crosssection.png" caption="Stencil alignment figure" alt="Top: Stencil alignment fixture, milled from a block of pine. It has a recess for the board, cavities for components, and alignment holes. Inset left: Stencil alignment cross section in CAD program, showing the board (w/ top side components already assembled) sitting flush with the surface of the stencil fixture. Inset right: Screenshot of toolpaths from CAM program." >}}

The board sits in a recess so that the surface is flush with the wood, the stencil can sit on top of both, and gauge pins are used to fix everything in place. There is also a cavity to allow clearance for the top side components when applying paste to the bottom side.

{{< figure src="frogsboro_fixture_paste_application.jpg" caption="Paste application" alt="Top: Stencil alignment fixture with board, stencil, and gauge pins inserted. Paste has been wiped across the stencil. Bottom: Stencil and gauge pins have been removed, revealing the board sitting in the fixture with paste applied." >}}

There's obviously a different sort of setup time involved in the CAD/CAM and milling the fixture. In the end though it was pleasant to use, the alignment felt solid, and the resulting paste application was crisp and consistent. I won't use this for every board, but for double-sided boards and/or boards with small feature sizes it's a nice tool to have.

### Component Placement and Reflow

{{< figure src="frogsboro_top_ics_placed.jpg" caption="IC placement" alt="The PCB clamped onto a preheater, with some integrated circuits placed." >}}

Once paste was applied I used my normal method for assembly. I use my preheater (turned off) to clamp the board while I place components..

{{< figure src="frogsboro_top_reflowed.jpg" caption="Top after reflow" alt="The board (top view) with all of the top components placed and reflowed." >}}

... and then reflow with the preheater around "300&deg;C"[^3] and very low flow rate hot air at 350&deg;C.

{{< figure src="frogsboro_bottom_reflowed.jpg" caption="Bottom after reflow" alt="The board (bottom view) with all of the bottom components placed and reflowed." >}}

For the bottom side, I keep the preheater lower ("100&deg;C") and use the same hot air settings. Through hole parts are the final step. They need thermal relief as I don't have the board preheated when soldering these.

## Software

![A screenshot of a terminal running minicom. It's showing a serial console to the FROGSBORO board, and the output of uname -a and /proc/cpuinfo](/media/frogsboro_console.png "FROGSBORO serial console")

I didn't have to do much here. I was able to boot [a demo image form the sam9x60-ek evaluation kit](https://www.linux4sam.org/bin/view/Linux4SAM/Sam9x60EKMainPage#Demo_archives). I used linux4sam-buildroot-sam9x60ek-headless-2021.10.img. The device tree should ideally be modified to remove peripherals that aren't present (e.g., Ethernet), but leaving it as-is doesn't cause a problem.

I used an FTDI cable to access the UART for first boot. I then modified the inittab to load [g_serial](https://www.kernel.org/doc/Documentation/usb/gadget_serial.txt) and start getty on ttyGS0. This lets me use the SAM9X60 USB device port, which is also powering the device, as a serial console.

**Update 2022-11-10**: I've since built a [yocto layer](https://github.com/iank/meta-frogsboro) for this board. See [frogsboro-firmware](https://github.com/iank/frogsboro-firmware)

## Thoughts

* This was my first six layer board. The extra routing layers were nice, but in this case less useful than they could be. At this budget I'm stuck with (relatively large) through vias meaning some areas can quickly become Swiss cheese on every layer.
* I like this PMIC. Absolutely happy to use one part to provide all of the voltage rails when I can get away with it.
* The fab I use has a coarse silkscreen print resolution. This isn't the first board where I've had to delete some reference designators to make others fit. But the resulting silkscreen is still crowded to the point of ambiguity. I may start leaving designators off for most passives by default and only including the ones that I think may be frequently referenced.
* In the future it would be better to place the mouse bites recessed back from the board edge so that the broken-off pieces don't protrude.

[^1]: I used 0.5mm diameter non-plated holes, with 0.75mm from center to center and 0.5mm clearance from the outside hole edges to board edge. There is no copper on any layer on the rails.

[^2]: Aside from [paste-in-hole](https://www.airborn.com/resources/connector-encyclopedia/paste-in-hole), which I think is still fairly uncommon.

[^3]: This and other temperature settings I'll mention for this preheater only loosely correspond to the actual temperature the board reaches. It's an IR heating element an inch or so underneath the board and the control system, such as it is, controls the element temperature, not a temperature probe on the board.
