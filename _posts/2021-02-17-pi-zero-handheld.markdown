---
layout: post
title:  "Building my own retro game handheld"
description: 3d printed Raspberry Pi Zero W Retropie Handheld
date:   2020-03-23 12:00:00 +0200
---

So, after going through the thingiverse library looking for cool prints, I thought one of the coolest was [Adafruit's Pi GRRL Zero](). I liked the shape, it is compact but big enough
to have a confortable grip. Other cool projects like [this tiny handheld](). It look super cool but too tiny!.

[pi ggrl zero foto][tiny gameboy project photo] add links to thingibverse

The only .. mention screen size 

Since the Pi GRRL has a whole guide on how to build the handheld and some assebling videos, and even some software and script, it was also a wise choice in terms on the experience I
have with electronics.
 
Start with installing retro pie 
next connect screen and activate overlay with drivers

This example sort of work with that wiring
 https://sudomod.com/forum/viewtopic.php?f=22&t=2312&start=260

changing the cmake with diferent parameters
https://github.com/juj/fbcp-ili9341 
  this paremeter changed from the previuous example did the trick 
  
  -DILI9341=ON: If you are running on any other generic ILI9341 display, or on Waveshare32b display that is standalone and not on the FreeplayTech CM3/Zero device, pass this flag.

  review what i have in the config.txt
  añlso adding it to the local.rc

  en adafruit hay un tutoirial para twikear retropie mejor fps

  soldar patas. protoboard 

  Instalar y setear botones en GPIO
  Instalar https://learn.adafruit.com/retro-gaming-with-raspberry-pi/adding-controls-software
  configure only A B and Dpad comment the rest 

´´´bash
# Sample configuration file for retrogame.
# Really minimal syntax, typically two elements per line w/space delimiter:
# 1) a key name (from keyTable.h; shortened from /usr/include/linux/input.h).
# 2) a GPIO pin number; when grounded, will simulate corresponding keypress.
# Uses Broadcom pin numbers for GPIO.
# If first element is GND, the corresponding pin (or pins, multiple can be
# given) is a LOW-level output; an extra ground pin for connecting buttons.
# A '#' character indicates a comment to end-of-line.
# File can be edited "live," no need to restart retrogame!

# Here's a pin configuration for the PiGRRL Zero project:

LEFT        # Joypad left
RIGHT     19  # Joypad right
DOWN      16  # Joypad down
UP        26  # Joypad up
Z         20  # 'A' button
X         21  # 'B' button
#A         16  # 'X' button
#S         19  # 'Y' button
Q         20  # Left shoulder button
W         21  # Right shoulder button
ESC       17  # Exit ROM; PiTFT Button 1
LEFTCTRL  22  # 'Select' button; PiTFT Button 2
ENTER     23  # 'Start' button; PiTFT Button 3
4         27  # PiTFT Button 4

# For configurations with few buttons (e.g. Cupcade), a key can be followed
# by multiple pin numbers.  When those pins are all held for a few seconds,
# this will generate the corresponding keypress (e.g. ESC to exit ROM).
# Only ONE such combo is supported within the file though; later entries
# will override earlier.
´´´
foto prototipo con la proto y la pantalla

Fusion 360 modeling, link to adafruit noe tutorials
Download existing parts from grabCAD
Model the parts I made myself, PCB + buttons

Exterior, Dpad and butons, thickness remixes from Noe's PiGRRL

Cable managment tricky

M3 Screws using heat to screw and create the threads
Didnt left in the corners to put the screws so only two in weird places were possible so it does not close well
I wanted all the Rpi ports available, so it had to be in that corner.

First design needed a 1mm more of space to leave space for the buttons to be pressed. the original one the buttons were pressed when closing the case.

Power switch did not fit
I had to add the lip for closing around the whole case and not only at the sides.

Couldn't figure out a way to make a snap fit with the space available.

I made some spacers for the 6mm screws and the TFT

Had to glue some pcb parts as they move up with the cables pressure and pressed the button against the lid.



Then improve performance and tweak for gaming!! firts with the official PIGRRL zero performance section

AUDIO VIA HDMI OK

test on tv and record

set hotkey and load save L R