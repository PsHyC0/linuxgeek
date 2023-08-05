---
title: "ddcutils"
date: 2023-08-01T12:01:22-07:00
draft: false
author: "John Holbrook"
tags: [ "linux","display","desktop" ]
image: img/monitors.jpg
---
**Note:** This write-up was originally posted May 27, 2021. Some instructions may have changed since then. Let me know if you run into any problems.


I found it frustrating constantly having to hit the buttons on one or more of my three monitors to change inputs between my work laptop, home PC and other devices. Figured there's gotta be a better way. I wanted a menu allowing me to switch things up via a keyboard shortcut. Would also be great to power on and off monitors if possible.

After some Googling, I came across the mention of ddc (DDC/CI (Display Data Channel Command Interface) tools to switch monitor inputs and other monitor related tasks from command line.

Originally tried ddccontrol but it is outdated and possibly a dead project (last update 2006) and requires this library file for details on monitors. Didn't like that. Then I found ddcutil which is being actively developed and doesn't require stupid library files.

Whats ddcutil you ask?

*ddcutil is a Linux program for managing monitor settings, such as brightness, color levels, and input source. Generally speaking, any setting that can be changed by pressing buttons on the monitor can be modified by ddcutil.*

This looks like exactly what I need.

I attempted installing ddcutil from Debian packages via Opensuse Build Services (OBS) but it required some weird pci.ds package but that would break pciutils on MX Linux. Went to the source which is also much newer. Prerequisites are on the site if you run into any problems.

**Note:** These instructions were on MX Linux. Moved to Garuda/Arch and simply installed the package available there when I made the switch.

```
$ git clone https://github.com/rockowitz/ddcutil.git
$ cd ddcutil
```
SUDO to root
```
# ./autogen.sh
# ./configure
# make
# make install
```
Now that its installed let's query the monitors. I used the --nousb option as I don't have USB monitors. The monitors are hooked up in order from left to right. 1,2,3.

```
❯ ddcutil --nousb detect
Display 1
   I2C bus:  /dev/i2c-4
   EDID synopsis:
      Mfg id:               DELL
      Model:                DELL P2719H
      Product code:         453434
      Serial number:        XXXXXXXXX
      Binary serial number: 0 (0x00000000)
      Manufacture year:     2016,  Week: 30
   VCP version:         2.2

Display 2
   I2C bus:  /dev/i2c-5
   EDID synopsis:
      Mfg id:               DELL
      Model:                DELL S3220DGF
      Product code:         53492
      Serial number:        6P7N4W2
      Binary serial number: XXXXXXXXXXXX
      Manufacture year:     2020,  Week: 15
   VCP version:         2.1

Display 3
   I2C bus:  /dev/i2c-6
   EDID synopsis:
      Mfg id:               DELL
      Model:                DELL P2719H
      Product code:         453434
      Serial number:        XXXXXXXXXXXXX
      Binary serial number: 0 (0x00000000)
      Manufacture year:     2016,  Week: 30
   VCP version:         2.2
```

Let's examine the capabilities of my monitors. Will look at monitor 3 (far right)

```
$ ddcutil capabilities --display 3 --verbose  
Produces a lot of information but I think what I want is the Input Source

Feature: 60 (Input Source)
      Values:
         01: VGA-1
         03: DVI-1
         0f: DisplayPort-1
```

To test switching the monitor via the physical buttons and see the difference.  While connected to Display Port

```
❯ ddcutil --display 3 getvcp 60
VCP code 0x60 (Input Source                  ): DisplayPort-1 (sl=0x0f)
After switching to DVI

❯ ddcutil --display 3 getvcp 60
VCP code 0x60 (Input Source                  ): DVI-1 (sl=0x03)
So I know DP1 is 0x0f and DVI is 0x03.  That is actually shown above too. Sweet.
```

This switches monitors back and forth without touching the monitor

```
❯ ddcutil --display 3 setvcp 60 0x03
❯ ddcutil --display 3 setvcp 60 0x0f
```

Checking my Center monitor the numbers are a little different:

```
$ ddcutil capabilities --display 1   

<CUT>
Feature: 60 (Input Source)
      Values:
         0f: DisplayPort-1
         11: HDMI-1
         12: HDMI-2
```

Create 6 scripts to change the two monitors inputs and 2 scripts to power off and on the monitors. Again...probably a cleaner way to do this but it works for me.

I'll use a simple naming convention of LEFT_* or RIGHT_* with the different inputs. e.g. LEFT_DP.sh, LEFT_DVI.sh and LEFT_VGA.sh.

```
❯ cat LEFT_DVI.sh
#!/bin/sh
ddcutil --display 1 setvcp 60 0x03
❯ cat LEFT_DP.sh
#!/bin/sh
ddcutil --display 1 setvcp 60 0x0f
❯ cat LEFT_VGA.sh
#!/bin/sh
ddcutil --display 1 setvcp 60 0x01
```

Do the same for RIGHT monitor.

Create two scripts for powering off and on the monitors.  You'll notice that I can turn off all 3 monitors and only turn on 2. That's because I need to be able to see menu to turn on all the monitors. So I simply hit the button on my left monitor and then use rofi menu to power on the middle and right one. Hope that makes ense.

Lets look at Feature D6 (ddcutil capabilities --display # --verbose)

Feature: D6 (Power mode)
Values (unparsed): 01 02 03 04 05
Values (  parsed):
01: DPM: On,  DPMS: Off
02: DPM: Off, DPMS: Standby
03: DPM: Off, DPMS: Suspend
04: DPM: Off, DPMS: Off
05: Write only value to turn off display
For Center Monitor

Feature: D6 (Power mode)
Values (unparsed): 01 04 05
Values (  parsed):01: DPM: On,  DPMS: Off
04: DPM: Off, DPMS: Off
05: Write only value to turn off display

```❯ cat monitors_on.sh
#!/bin/sh
ddcutil --nousb --display 2 setvcp D6 0x01
ddcutil --nousb --display 3 setvcp D6 0x01
❯ cat monitors_off.sh
#!/bin/sh
ddcutil --nousb --display 1 setvcp D6 0x04
ddcutil --nousb --display 2 setvcp D6 0x04
ddcutil --nousb --display 3 setvcp D6 0x04
```
You can try each script above and make sure it works for you.

To make things easier (and prettier) I created a custom menu with rofi to do this:

$ sudo apt install rofi
Created ~/bin/screen_menu.py
Note: The script is a little messy and there's probably a more "correct" and cleaner way to do this. Let me know if you have any feedback. For my second (Center Dell 32") I don't currently have a need to switch inputs as it always stays on my main PC via Display Port. Change $USERNAME to your username.

```
#!/usr/bin/python3

# This script will allow me to switch one at a time or both at a time screens my Dell 27" left and right monitors inputs and turn off all monitors and turn on the center and right one.

# Utilizing DDCUTILS - https://www.ddcutil.com

# Notes
# Dell Main Monitor - Display 2
# 	Inputs
#         0f: DisplayPort-1 
#         11: HDMI-1 
#         12: HDMI-2

# Right Dell Monitor - Display 3
# Left Dell Monitor - Display 1
#	Inputs
#        01: VGA-1
#        03: DVI-1 
#        0f: DisplayPort-1

import rofi_menu

class Display1(rofi_menu.Menu):
        prompt = "Screen1 Inputs"
        items = [
                rofi_menu.BackItem(),
                rofi_menu.ShellItem("VGA", "/home/$USERNAME/bin/LEFT_VGA.sh"),
                rofi_menu.ShellItem("DisplayPort", "/home/$USERNAME/bin/LEFT_DP.sh"),
                rofi_menu.ShellItem("DVI", "/home/$USERNAME/bin/LEFT_DVI.sh"),
        ]

class Display3(rofi_menu.Menu):
        prompt = "Screen3 Inputs"
        items = [
                rofi_menu.BackItem(),
                rofi_menu.ShellItem("VGA", "/home/$USERNAME/bin/RIGHT_VGA.sh"),
                rofi_menu.ShellItem("DisplayPort", "/home/$USERNAME/bin/RIGHT_DP.sh"),
                rofi_menu.ShellItem("DVI", "/home/$USERNAME/bin/RIGHT_DVI.sh"),
        ]

class Power(rofi_menu.Menu):
        prompt = "Monitor Power"
        items = [
                rofi_menu.BackItem(),
                rofi_menu.ShellItem("On", "/home/$USERNAME/bin/monitors_on.sh"),
                rofi_menu.ShellItem("Off", "/home/$USERNAME/bin/monitors_off.sh"),
        ]


class MainMenu(rofi_menu.Menu):
    prompt = "Menu"
    items = [
	rofi_menu.BackItem(),
        rofi_menu.NestedMenu("Screen1 Inputs >", Display1()),
        rofi_menu.NestedMenu("Screen3 Inputs >", Display3()),
        rofi_menu.NestedMenu("Monitor Power  >", Power()),
	    ]


if __name__ == "__main__":
    rofi_menu.run(MainMenu())
```
To test you can run:

```
rofi -modi mymenu:/home/$USERNAME/bin/screen_menu.py -show displaymenu 
```
I'm using I3wm so I'll map a keyboard shortcut to bring up this particular rofi menu. I already have $mod+d to bring up rofi's main menu.
```
$ nano ~/.config/i3/config   
```
Add the following line (all one line). Change $USERNAME to your username obviously

```
bindsym $mod+Ctrl+d exec rofi -modi mymenu:/home/$USERNAME/bin/screen_menu.py -show mymenu
```

Hope this helps. Let me know if you have any feedback or comments.



