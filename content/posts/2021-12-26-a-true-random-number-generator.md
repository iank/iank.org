---
template: post
title: A True Random Number Generator
slug: a-true-random-number-generator
featured_image: /media/trng_final_assembled2.png
draft: false
date: 2014-06-06T22:09:08.626Z
description: A true random number generator project. A PN junction in avalanche
  breakdown provides the noise source. The noise is amplified, digitized, and
  conditioned. An LPC1343 provides a USB interface.
category: Hardware
tags:
  - PCB
  - hardware
  - entropy
---
This page describes the implementation of (Yet Another) avalanche noise hardware random number generator. This is a device which has been implemented many times [e.g. [Rob Seward](http://robseward.com/itp/adv_tech/random_generator/), [Aaron Logue](http://www.cryogenius.com/hardware/isarng/)], including some commercial offerings [e.g. [Entropy Key](http://www.entropykey.co.uk/), [TrueRNG](http://ubld.it/products/truerng-hardware-random-number-generator/)]. This project does not do anything better than existing work, but I learned a lot.

## Theory

This True Random Number Generator (TRNG) is based on avalanche breakdown noise in a [PN junction](http://en.wikipedia.org/wiki/P%E2%80%93n_junction). A PN junction is a semiconductor structure which forms a diode, allowing (ideally) electric current to flow in only one direction. Electric fields internal to the structure support a potential gradient allowing current to flow easily in one direction but creating a potential barrier for electrons driven in the reverse direction.

If strongly reverse-biased, however, electrons which overcome that potential barrier can have sufficient energy to cause **impact ionization**, leading to a multiplication effect[^1]. An energetic electron impacting a silicon atom in the lattice can knock another electron out of the valence band (forming another electron-hole pair). In the presence of a strong reverse-biased electric field (needed to initiate this process in the first place), the liberated electron will accelerate and the process can continue, creating something like a sustained chain reaction.

Laying aside quantum effects and considering electrons in our PN junction to act as a Newtonian gas, avalanche breakdown is formally a [chaotic](http://en.wikipedia.org/wiki/Chaos_theory) process: It depends upon strongly nonlinear interactions between a large number of elements and exhibits topological mixing (the charge carriers mix throughout the structure and effects are not localized). Chaos means that this system, even if theoretically deterministic (I believe it is not), is highly sensitive to a tremendously large and unknowable set of initial conditions, rendering it unpredictable.

## Prototype

The prototype is based upon a design by [Will Ware](http://web.jfet.org/hw-rng.html). The physical random source is a reverse-biased PN junction (actually two terminals of a [bipolar junction transistor](http://en.wikipedia.org/wiki/Bipolar_junction_transistor)). Part of a transistor is used, rather than a diode, because diodes are typically designed with either very high breakdown voltage (as in the case of rectifying diodes) or low breakdown voltages but with minimal avalanche noise (e.g. zener diodes which are meant to be used in breakdown). A schematic is shown below.

![TRNG prototype schematic](/media/trng_proto_sch.png "TRNG prototype schematic")

\
The PN junction formed by the base and emitter of T1 is reverse-biased by an 18V source (realized by two 9V batteries). The collector is left floating. The output is amplified by common-emitter amplifier T3 and coupled through C1 to another common-emitter stage (T2).

I constructed this prototype on perfboard and sampled the output using the ADC on-board an [Arduino Uno](http://arduino.cc/en/Main/arduinoBoardUno). After correcting the output [distribution](http://en.wikipedia.org/wiki/Probability_distribution) using [Von Neumann's algorithm](http://en.wikipedia.org/wiki/Hardware_random_number_generator#Software_whitening), the effective data rate was around 1000 random bits per second, and the device passed the [diehard](http://stat.fsu.edu/pub/diehard/) suite of statistical tests.

With a working prototype, I set out to create an integrated PCB, increase the data rate, and replace the Arduino with a USB interface.

## Iteration 1: PCB 1, power supply

The first iteration adds an 18V supply based on the Texas Instruments TPS61040 DC/DC boost converter designed for LCD and LED display lighting applications. The [datasheet](http://www.ti.com/lit/ds/slvs413f/slvs413f.pdf) contains a reference design (Figure 16, pg. 14) for an 18V output from a 5V supply. This eliminates the 9V batteries and allows the design to be powered from USB.

A PCB was created (using the free version of [CadSoft EAGLE®](http://www.cadsoftusa.com/)) containing the power supply, physics package, and amplifiers. I used 0603 passives and replaced the through-hole 2N3904 transistor from the prototype with the MMBT3904, a surface-mounted equivalent.

![Schematic for PCB 1](/media/trng1_sch.png "Schematic for PCB 1")

![Board layout for PCB 1](/media/trng1_brd.png "Board layout for PCB 1")

The PCB was manufactured by [Seeed Studio Fusion](https://www.seeedstudio.com/service/), a low-cost PCB service. It was assembled and tested with the Arduino.

## Iteration 2: PCB 2, ADC and LPC1343

Next, I needed a faster interface. After some research I chose to use the [LPC1343](http://www.nxp.com/documents/data_sheet/LPC1311_13_42_43.pdf), a 32-bit ARM Cortex-M3 microprocessor which has a native USB interface and runs at up to 72 MHz. Olimex offers a [LPC1343 dev board](https://www.olimex.com/Products/ARM/NXP/LPC-P1343/) with published schematics. Using the LPC1343's on-board ADC, I was able to get about a 40x speedup over the Arduino.

I adapted reference USB mass storage code for the LPC1343 and created a virtual mass storage interface for the device (this is not ideal). When inserted, the TRNG enumerates as a mass storage device (e.g. a USB thumbdrive) of arbitrary capacity. It would appear to, e.g. Linux, as a block device. Writes do nothing and return success. Reads return blocks of random data sampled from the noise circuit.

Next, I found the [Intersil HI5767/2CBZ](http://www.intersil.com/content/intersil/en/products/data-converters/high-speed-a-d-converters/a-d-converters/HI5767.html), a fast (up to 20 megasamples per second) dedicated 10-bit ADC in a [SOIC-28](http://en.wikipedia.org/wiki/Small_Outline_Integrated_Circuit) package. Driving the ADC at the LPC1343's clock frequency (12 MHz) led to an overall speedup of 400x vs the prototype.

I created a SSOIC-28 breakout board using a [PCB from Adafruit](http://www.adafruit.com/products/1208) and built a test circuit for the ADC on perfboard (pictured below), following a reference schematic in the [HI5767 datasheet](http://www.intersil.com/content/dam/Intersil/documents/hi57/hi5767.pdf) (Figure 19, pg. 13).

The HI5767 has an input dynamic range of no more than 1Vp-p, and I found that removing the second amplifier from the amplifier chain in my physics package reduced the output to within this level. I was able to hack the PCB (pictured below) to achieve this, saving me considerable time.

![ADC test rig](/media/trng_adc_test.png "ADC test rig")

![PCB 1 hack](/media/trng_pcb1_hack.png "PCB 1 hack")

The test setup, shown below, consists of the Arduino Uno (now providing only a 5V power supply!), the LPC1343 (providing a USB interface), The physics package PCB #1 from above (hacked for 1V output), and my ADC test circuit. It is a mess.

![Messy test setup](/media/trng_mess1.png "Messy test setup")

## Iteration 3: Integration

Finally, I created a single integrated PCB. I attempted to follow some basic rules of mixed-signal design (single sided PCB with large ground pour on reverse, isolating analog and digital circuitry, short clock traces, etc) to avoid contaminating the analog noise source with radiated [high-frequency components](http://mathworld.wolfram.com/FourierSeriesSquareWave.html) of the clock or digital outputs. It seems ironic to take measures to protect a noise signal from interference, but we must avoid introducing predictable patterns.

The final schematic, board layout, and images of the completed device are shown below. The final device, after moving the whitening logic to firmware (for completeness sake, but at a significant speed expense), achieved 9 kB/sec random data.

![Final schematic](/media/trng_final_sch.png "Final schematic")

![Final PCB layout](/media/trng_final_brd.png "Final PCB layout")

![Final assembly](/media/trng_final_assembled2.png "Final assembly")

## Conclusion

This design, like many others in its class (including some commercial offerings) is flawed and should not be used by anyone. It is not a differential design and is easily influenced by external fields. Unlike some commercial products, it has no [tampering detection](http://www.entropykey.co.uk/tech/) or countermeasures, leaving it vulnerable to manipulation.

I am done iterating and there are loose ends I do not intend to clean up. In particular, the USB mass storage class is a strange way to interface this device. 

## Links/misc

* [**Slides** from my talk](/media/NCSULUG_ik_trng_slides.pdf) at the [NCSU Linux Users' Group](http://lug.ncsu.edu/)
* **[Test output](/media/2014-07-18_dieharder_test.txt)** from the [dieharder test suite](http://www.phy.duke.edu/~rgb/General/dieharder.php)
* While not explicitly referenced, I found these links interesting or useful
  * <http://holdenc.altervista.org/avalanche/>
  * <https://code.google.com/p/avr-hardware-random-number-generation/wiki/AvalancheNoise>
  * <http://www.random.org/analysis/>
  * <http://www.cs.berkeley.edu/~daw/papers/ddj-netscape.html>
  * <http://gamesbyemail.com/News/DiceOMatic>
  * <http://boallen.com/random-numbers.html>

[^1]: McIntyre, R. J. "Multiplication noise in uniform avalanche diodes." Electron Devices, IEEE Transactions on 13.1 (1966): 164-168.
