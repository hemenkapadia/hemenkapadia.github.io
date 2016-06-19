---
layout: post
title: Ubuntu with HighDPI or Retina displays
excerpt: Recommended configuration changes for Ubuntu to work with HighDPI or Retina displays. Without these changes the high resolution of Retina displays causes text to appear very small.
comments: true
---
I recently purchased a new laptop, [Dell 7559](http://www.dell.com/us/p/inspiron-15-7559-laptop/pd?oc=dncwpw5724s&model_id=inspiron-15-7559-laptop), and it came with an Ultra HD display also known as a High DPI or Retina display, and I am immensely impressed with it. The display looks good even on Windows, an OS I do not appreciate for its font rendering engine. So I installed Ubuntu on the machine, hoping to get a Mac like experience, but was shocked with the default experience as it did not cater very well to the High DPI display.

I have been using Linux for at least 20 years now and I am amazed that even now the installation does not fall short in throwing up challenges, which is what makes using Linux exciting. After all what is life without some challenge.

This blog post is a list of configurations I find useful to make Ubuntu display usable on Retina displays.

### Scaling of the Unity UI ###

This issue was observed in 14.04, 15.10 but seems to have been taken care of in 16.04. The window experience with the default scaling of 1 renders the text so small that it is a challenge to read anything. I notice that the scaling factor of 2 works best as compared to any setting between 1 and  2

To change the scaling, in Unity search for Display and open it. Drag the slider for `Scale for menu and title bars:` to 2 (or any other depending in your display).

### Scaling of Virtual Console fonts ###

While fixing my system to run [Nvidia + Bumblebee]({% post_url 2016-05-07-Ubuntu-with-Nvidia-Bumblebee %}) I had to work on the Virtual Console (Alt + Ctrl + F1..F6). The fonts appear too small to read here too. To fix that follow the below steps.

Open up a terminal (need not be a Virtual Terminal), and issue the command

`sudo dpkg-reconfigure console-setup`

A curses based application will start, select the configuration as below

```
Encoding - UTF-8
Character Set - . Combined - Latin; Salvic Cyrillic; Greek
Font for console - Terminus
Font Size - 16x32
```
So as to apply this configuration each time the machine start up we need to make below changes for systemctl (15.10 onwards) or init (14.04).

**For Systemctl**

Edit `/lib/systemd/system/console-setup.service` and in `[Service]` section add `ExecStart=/bin/setupcon` to the bottom.

**For Init**

Edit `/etc/init/console-setup.conf` and add `exec /bin/setupcon` to the bottom of the file.

Both these settings cause `/bin/setupcon` command to run each time on boot.

Restart the system and open a virtual console. This time you will see that the text fonts are scaled up.

If any time you need to start Ubuntu in recovery mode, Systemctl or Init will not apply the above settings. In that case execute the command `/bin/setupcon` and the settings will get applied.

### Mouse pointer size ###

This issue was experienced on 14.04, 15.10, but not experienced on 16.04. Post scaling while the windows were comfortable size, the mouse pointer still appeared very small. To fix it add the setting `Xcursor.size: 48` in file `/etc/X11/Xresources/x11-common`

### Display brightness adjustment ###

This issue is noticed in 14.04, 15.10 and 16.04 was well, but in 16.04 the brightness level seems to persist once changed manually. 15.10 and 14.04 did not persist the changes done manually so the following steps are needed.

> Note the path of the file mentioned below is dependent on your graphics card.

Fix is to add the below line above the `exit 0` line in `/etc/rc.local` (note above exit 0)

`echo 339 > /sys/class/backlight/intel_backlight/brightness`

To get the desired number (339 in the above case), adjust brightness to the desired level manually and issue comment to get the required number which you can then substitute in the above command.

`cat /sys/class/backlight/intel_backlight/brightness`

### Scaling of GRUB menu ###

Refer to the section "Persist GRUB Settings" in [Nvidia + Bumblebee]({% post_url 2016-05-07-Ubuntu-with-Nvidia-Bumblebee %}) which provides settings to improve the font scaling in GRUB menu.


Ok. That's it for now.
