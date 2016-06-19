---
layout: post
title: Nvidia + Bumblebee (Optimus) on Ubuntu
excerpt: Guide to configure Ubuntu to work with Nvidia drivers + Bumblebee on an Optimus laptop.
comments: true
---
This guide lists the procedure of installing Nvidia proprietary drivers on [Dell 7559](http://www.dell.com/us/p/inspiron-15-7559-laptop/pd?oc=dncwpw5724s&model_id=inspiron-15-7559-laptop) along with Bumblebee to get the benefits of power management provided by the Optimus technology at the same time having the flexibility to use the Nvidia GPU features without requiring to logout/login each time a configuration change is needed.

This guide is based on the capabilities I desired in my laptop, and much of the choice I made are guided by these principals.

1. Stable Ubuntu version, to be supported for next 3 years.
2. Stability over cutting edge features.
3. Integrated Intel card to be my primary display driver.
4. Use proprietary Nvidia drivers (vs Nouveau) to get maximum benefit of the GPU.
5. Nvidia GPU needs to be switched off when not in use, to save power.
6. Turn on Nvidia GPU on demand for specific applications, without requiring configuration changes, logout or restarts.

> From the Ubuntu LTS version perspective, I had option to choose 14.04 or 16.04. I tired to install Nvidia drivers on 16.04 but was never successful. After spending numerous hours figuring things out, I decided to go with version 14.04.4 which had an updated kernel and Xserver based on 15.10 as part of [LTS Hardware Enablement Stack](https://wiki.ubuntu.com/TrustyTahr/ReleaseNotes#LTS_Hardware_Enablement_Stack).

### Configure BIOS and Windows ###

I am assuming that you would be installing Ubuntu in Dual boot mode (along with Windows) and that Windows is booting in UEFI mode. As a result, we will need to install Ubuntu in UEFI mode only.

If you are not sure how to [install Windows and Ubuntu in dual boot mode](www.everydaylinuxuser.com/2013/09/install-ubuntu-linux-alongside-windows.html), refer to the linked guide. I would like to stress on point 4 *Disable Fastboot (in Windows) and Secure boot (in BIOS)*, as those steps are super critical to follow exactly as mentioned.

I had disabled Secure Boot in BIOS, but did not disable Fastboot in Windows. As a result I was having a hard time installing 14.04, and the installer would fail each time with the error `grub-efi-amd64-signed failed to install into /target/. Without GRUB boot loader, the installed system will not boot`.

### GRUB settings for Ubuntu Live USB ###

When you start installing using a live USB, you will be presented with the GRUB screen that lets you try Ubuntu without installing as well as install Ubuntu.

Select the `Install Ubuntu` option and press `e` which will open the menu edit screen. Add the following kernel parameters just before `quiet splash` in the line that starts with `linux /boot/vmlinuz-linux .....`

> for kernel 4.2 intel driver is i915, for kernel 4.4 it is i915_bpo. Replace accordingly.

```
nomodeset i915.modeset=1
```

Press `F10` and complete the installation as per [install Windows and Ubuntu in dual boot mode](www.everydaylinuxuser.com/2013/09/install-ubuntu-linux-alongside-windows.html)

Once the installation is complete, you will be presented with the GRUB menu, follow the above procedure to edit the kernel parameters and boot. We will make these kernel parameters permanent in the next step.

### Persist GRUB settings ###

To avoid updating the kernel options each time the system is booted, update the GRUB settings `sudo vi /etc/default/grub` and chnage as below

```
GRUB_CMDLINE_LINUX_DEFAULT="nomodeset i915.modeset=1 quiet splash"
```

For High DPI or Retina displays, GRUB boot menu options are not readable as the font size is too small. It can be fixed by the below two options

```
GRUB_GFXMODE=1280x1024
GRUB_GFXPAYLOAD_LINUX=keep
```

Set Windows as the default system to boot

> This setting specific to my laptop, replace accordingly with the Windows label for your
> system

```
GRUB_DEFAULT="Windows Boot Manager (on /dev/sda1)"
```

To apply the changes, issue the command `sudo update-grub2`

### Optional - HighDPI settings ###

This is an optional step, but highly recommended if you have a HighDPI or Retina display. Apply the HighDPI configuration discussed in [Configuring Ubuntu for HighDPI and Retina displays]({% post_url 2016-05-06-Configuring-Ubuntu-for-highDPI-and-Retina-displays %})

### Remove Nvidia and Nouveau, and disable services ###

* Restart the machine and ensure that all setttings applied so far are working as expected.
* Switch to Virtual Console (Atl + Ctrl + F1 .. F6)
* Stop the xserver `sudo service lightdm stop` (for init) or `sudo systemctl stop lightdm.service` (for systemctl)
* Remove any existing installs of nvidia, nouveau etc.

```
sudo apt-get remove --purge nvidia*
sudo apt-get remove --purge bumblebee*
sudo apt-get --purge remove xserver-xorg-video-nouveau*
```
* To blacklist nouveau add the following lines to `/etc/modprobe.d/blacklist.conf`

```
# Blacklisting nouveau to get nvidia running with bumblebee
blacklist nouveau
blacklist lbm-nouveau
alias nouveau off
alias lbm-nouveau off
options nouveau modeset=0
```

* Disable gpu-manager.service for both systemctl as well as init

```
sudo systemctl mask gpu-manager.service

sudo vi /etc/init/gpu-manager.conf

# Comment these start on settings ; GPU Manager ruins our work
#start on (starting lightdm
#          or starting kdm
#          or starting xdm
#          or starting lxdm)
task
exec gpu-manager --log /var/log/gpu-manager.log

```

* Create initramfs `sudo update-initramfs -u -k all`
* Reboot the system and ensure that xserver is running fine - i.e. GUI is working properly
* Verify that nouveau is successfully blacklisted. The command `lsmod | grep nouveau` should not return any lines with the word nouveau in it.
* Check if gpu-manager.service is disabled `sudo systemctl status gpu-manager.service` or `sudo service gpu-manager status`. If not, disable as mentioned above.

### Install  Nvidia and Bumblebee ###

* Switch to Virtual Console and disable lightdm `sudo service lightdm stop` (for init) or `sudo systemctl stop lightdm.service` (for systemctl). Everything below will happen in a Virtual Console and not the X terminal. Do not restart the machine unless you complete all the configuration steps mentioned below. Everything should happen in one shot, in a Virtual Console.

* Nvidia-352.63 is the most stable driver i have noticed. Let us see if that is the version that will get installed

```
sudo apt-get update
sudo apt-cache policy nvidia-352

# Output
hemen@hemen-Inspiron-7559:~$ sudo apt-cache policy nvidia-352

nvidia-352:
  Installed:
  Candidate: 352.63-0ubuntu0.14.04.1
  Version table:
 *** 352.63-0ubuntu0.14.04.1 0
        500 http://us.archive.ubuntu.com/ubuntu/ trusty-updates/restricted amd64 Packages
        500 http://security.ubuntu.com/ubuntu/ trusty-security/restricted amd64 Packages
        100 /var/lib/dpkg/status
```
Notice the Candidate, which says `nvidia-352.63`, great. Also it is available in standard Ubuntu repo, based on the link `us.archive.ubuntu.com`. So we are good to proceed.

* Install the required drivers

```
sudo apt-get install --no-install-recommends nvidia-352
sudo apt-get install libcuda1-352 nvidia-opencl-icd-352
sudo apt-get install --no-install-recommends bumblebee
sudo apt-get install bumblebee-nvidia primus
sudo apt-get install mesa-utils
sudo apt-get install xserver-xorg-video-intel-lts-wiley
```


### Bumblebee configuration ###

* Add the following two lines to the `/etc/modules-load.d/modules.conf` file

```
i915
bbswitch
```

* Blacklist nvidia from loading, its insertion and removal from the kernel will be handled by bbswitch. Add the following lines to `/etc/modprobe.d/bumblebee.conf`.

```
# 352
blacklist nvidia-352
blacklist nvidia-352-updates
blacklist nvidia-experimental-352
```

* Bumblebee configuration

```
sudo vi /etc/bumblebee/bumblebee.conf

## Configuration file for Bumblebee. Values should **not** be put between quotes
[bumblebeed]
VirtualDisplay=:8
KeepUnusedXServer=false
ServerGroup=bumblebee
TurnCardOffAtExit=false
NoEcoModeOverride=false
Driver=nvidia
XorgConfDir=/etc/bumblebee/xorg.conf.d

## Client options. Will take effect on the next optirun executed.
[optirun]
Bridge=primus
VGLTransport=proxy
PrimusLibraryPath=/usr/lib/x86_64-linux-gnu/primus:/usr/lib/i386-linux-gnu/primus
AllowFallbackToIGC=false

## Section with nvidia driver specific options, only parsed if Driver=nvidia
[driver-nvidia]
KernelDriver=nvidia-352
PMMethod=bbswitch
LibraryPath=/usr/lib/nvidia-352:/usr/lib32/nvidia-352
XorgModulePath=/usr/lib/nvidia-352/xorg,/usr/lib/xorg/modules
XorgConfFile=/etc/bumblebee/xorg.conf.nvidia

## Section with nouveau driver specific options, only parsed if Driver=nouveau
# [driver-nouveau]
# KernelDriver=nouveau
# PMMethod=auto
# XorgConfFile=/etc/bumblebee/xorg.conf.nouveau
```

The value of BusID in the below files would be different for your system. You can get the value for your system as shown below.

```
lspci | egrep -i "VGA|3D"

00:02.0 VGA compatible controller: Intel Corporation Device 191b (rev 06)
02:00.0 3D controller: NVIDIA Corporation GM107M [GeForce GTX 960M] (rev ff)


sudo vi /etc/bumblebee/xorg.conf.nvidia

Section "ServerLayout"
    Identifier  "Layout0"
    Option      "AutoAddDevices" "false"
    Option      "AutoAddGPU" "false"
EndSection

Section "Device"
    Identifier  "DiscreteNvidia"
    Driver      "nvidia"
    VendorName  "NVIDIA Corporation"
    BusID "PCI:02:00:0"
    Option "ProbeAllGpus" "false"
    Option "NoLogo" "true"
    Option "UseEDID" "false"
    Option "UseDisplayDevice" "none"
EndSection
```

```
sudo vi /etc/X11/xorg.conf

Section "ServerLayout"
  Identifier        "layout"
  Screen       0    "intel"
  Inactive          "nvidia"
EndSection


Section "Device"
  Identifier        "intel"
  Driver            "intel"
  BusID             "PCI:00:02:0"
  Option            "AccelMethod"  "SNA"
EndSection

Section "Screen"
  Identifier        "intel"
  Device            "intel"
EndSection


Section "Device"
  Identifier        "nvidia"
  Driver            "nvidia"
  BusID             "PCI:02:00:0"
  Option            "ConstrainCursor" "off"
EndSection

Section "Screen"
  Identifier        "nvidia"
  Device            "nvidia"
  Option            "AllowEmptyInitialConfiguration" "no"
  Option            "IgnoreDisplayDevices" "CRT"
EndSection
```

* Update alternatives

```
sudo update-alternatives --config x86_64-linux-gnu_gl_conf

There are 3 choices for the alternative x86_64-linux-gnu_gl_conf (providing /etc/ld.so.conf.d/x86_64-linux-gnu_GL.conf).

  Selection    Path                                       Priority   Status
------------------------------------------------------------
  0            /usr/lib/nvidia-352/ld.so.conf              8604      auto mode
  1            /usr/lib/nvidia-352-prime/ld.so.conf        8603      manual mode
  2            /usr/lib/nvidia-352/ld.so.conf              8604      manual mode
* 3            /usr/lib/x86_64-linux-gnu/mesa/ld.so.conf   500       manual mode

Press enter to keep the current choice[*], or type selection number:

Select the mesa option where available, and make the below configurations similarly

sudo update-alternatives --config x86_64-linux-gnu_egl_conf
sudo update-alternatives --config i386-linux-gnu_gl_conf
sudo update-alternatives --config i386-linux-gnu_egl_conf
sync
sudo ldconfig

sudo update-initramfs -u -k all
sudo usermod -a -G bumblebee $USER

```

* Start bumblebeed

```
sudo service bumblebeed start
```

### Testing ###

Run the following commands and ensure that you get appropriate response as mentioned here.

```
hemen@hemen-Inspiron-7559:~$ sudo service bumblebeed status
bumblebeed start/running, process 1327

hemen@hemen-Inspiron-7559:~$ cat /proc/acpi/bbswitch
0000:02:00.0 OFF

hemen@hemen-Inspiron-7559:~$ primusrun glxinfo | grep OpenGL
OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: GeForce GTX 960M/PCIe/SSE2
OpenGL core profile version string: 4.3.0 NVIDIA 352.63
OpenGL core profile shading language version string: 4.30 NVIDIA via Cg compiler
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile
OpenGL core profile extensions:
OpenGL version string: 4.5.0 NVIDIA 352.63
OpenGL shading language version string: 4.50 NVIDIA
OpenGL context flags: (none)
OpenGL profile mask: (none)
OpenGL extensions:

hemen@hemen-Inspiron-7559:~$ glxinfo | grep OpenGL
OpenGL vendor string: Intel Open Source Technology Center
OpenGL renderer string: Mesa DRI Intel(R) Skylake Halo GT2
OpenGL core profile version string: 3.3 (Core Profile) Mesa 11.0.2
OpenGL core profile shading language version string: 3.30
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile
OpenGL core profile extensions:
OpenGL version string: 3.0 Mesa 11.0.2
OpenGL shading language version string: 1.30
OpenGL context flags: (none)
OpenGL extensions:
```

### Restart ###

Restart your machine and ensure that X server, GUI as well as the tests mentioned above are working as expected.
