Lights Out
==========

This repository contains tools that allow getting software control of the webcam LED on ThinkPad X230 **without physical access to the laptop**.
These were created as a practical demonstration that malware can record video through the webcam without the LED indication.

This works via reflashing the webcam firmware over USB (the X230 webcam is connected over USB internally) to add a capability of arbitrarily controlling the LED.
This approach likely affects many other laptops, as connecting the webcam over USB and allowing to reflash its firmware is a common design pattern across laptop manufacturers.

See the ["Lights Out: Covertly turning off the ThinkPad webcam LED indicator"](https://docs.google.com/presentation/d/1NSS2frdiyRVr-5vIjAU-2wf_agzpdiMR1DvVhz2eDwc/edit?usp=sharing) talk ([pdf](https://powerofcommunity.net/poc2024/Andrey%20Konovalov,%20Lights%20Out%20-%20Covertly%20turning%20off%20the%20ThinkPad%20webcam%20LED%20indicator.pdf)) I gave at [POC 2024](https://powerofcommunity.net/) for the details: discovering a way to reflash the X230 webcam firmware, reverse engineering the firmware, adding an implant for LED control, and notes about the applicability of the approach to other laptops.

**Note: Reflashing the webcam firmware might brick the webcam, use these tools with caution**.


## Overview

The webcam used on ThinkPad X230 (and a few other laptops from the same era) is based on the Ricoh R5U8710 USB camera controller.
This controller stores a part of its firmware, the SROM part, on the SPI flash chip located on the webcam board.
The controller also allows reflashing the contents of the SPI chip over USB.

The LED on the X230 webcam board is connected to the GPIO B1 pin of the R5U8710 controller.
The GPIO B port is mapped to address `0x80` in the `XDATA` memory space of the 8051-based CPU inside R5U8710.
Thus, changing the value at that address changes the state of the LED.
This works regardless of whether the webcam is streaming video at the moment or not.

The tools provided in this repository allow flashing custom firmware with a USB-controlled so-called "universal implant" onto the SPI chip on the webcam board.
This implant allows writing controlled data to arbitrary addesses (within the `XDATA` memory space) and calling arbitrary addresses (within the `CODE` memory space; aliased with `XDATA` starting from offset `0xb000`).

The universal implant can be used for:

- Dynamically uploading a second-stage implant within the camera contoller memory and executing it (originally used for reverse engineering purposes);

- Directly controlling the webcam LED.

See the talk slides for more details.


## Tools

- [srom.py](srom.py) — reads and writes the SROM part of the firmware of a Ricoh R5U8710–based webcam over USB.

    Note: The webcam only loads the SROM firmware during its boot.
Thus, you will need to power cycle the laptop (full shutdown, not just reboot) for the updated firmware to get loaded;

- [patch_srom.py](patch_srom.py) — patches the SROM image from the FRU `63Y0248` webcam (not from the original X230 webcam) to add the universal implant.

    Note: This tool requires modification to work with the original X230 webcam SROM image.
However, the FRU `63Y0248` SROM image (optionally, with the implant added) can be flashed onto the original X230 webcam as well;

- [fetch.py](fetch.py) — fetches the contents of the `IRAM`, `XDATA`, or `CODE` memory space over USB via a second-stage implant that gets dynamically uploaded via the universal implant;

- [led.py](led.py) — turns the webcam LED on or off by overwriting the value at address `0x80` in `XDATA` via the universal implant.


## Memory dumps

- [srom/x230.bin](srom/x230.bin) — SROM contents of the original X230 webcam module (FRU unknown; `19N1L1NVRA0H` marking on the board);

- [srom/63Y0248.bin](srom/63Y0248.bin) — SROM contents of the FRU `63Y0248` webcam module;

- [code/63Y0248.bin](code/63Y0248.bin) — Contents of the `CODE` memory space leaked from the FRU `63Y0248` webcam module.

    Note: Boot ROM is below the offset `0xb000`, and it is identical to the Boot ROM on the original X230 webcam module.
