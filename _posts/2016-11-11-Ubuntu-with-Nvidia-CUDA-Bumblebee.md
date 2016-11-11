---
layout: post
title: Nvidia, CUDA and Bumblebee with Linux (Ubuntu) on Optimus laptop
excerpt: Configure Nvidia drivers, CUDA and Bumblebee with Ubuntu on an Optimus laptop.
comments: true
tags:
  - Nvidia
  - CUDA
  - Bumblebee
  - Ubuntu
  - Linux
  - Optimus
  - Nvidia Geforce
---

In this guide, I list the procedure to get proprietary [Nvidia driver], [CUDA] and [Bumblebee] working on [Dell 7559].

My aim is to get the benefits of power management provided by [Optimus] technology at the same time having the flexibility to use the Nvidia GPU features on demand without requiring to logout/login each time a GPU change is needed. In this configuration, integrated Intel GPU will be the primary display driver and Nvidia GPU will be powered up on need basis.

This guide is opinionated and much of the choices I make are based on my requirements below

1. Stability over cutting edge features.
2. Ubuntu LTS version supported for next couple years.
3. System update should be seamless using `apt-get upgrade`
4. System updates should not break the system, mainly the Xserver and CUDA install.
5. Use Proprietary Nvidia drivers.
6. Integrated Intel GPU to be primary display driver.
7. Nvidia GPU switched off by default.
8. Power up Nvidia GPU on demand, without requiring and configuration changes, logout or restarts.

### Select Ubuntu version ###

These steps are confirmed to work with [kernel v4.2]. I was not able to get this working on any other kernel, Xserver combination or LTS version other than [LTS 14.04.4].

[Download Ubuntu LTS 14.04.4] iso and [create a bootable USB].

### Install Ubuntu ###

You can follow the [guide to install Windows and Ubuntu in dual boot mode]. Make note of the below points and the GRUB settings discussed in the next section before you start the installation.

> Ensure point 4 *Tun OFF Fastboot (in Windows) and disable Secure boot (in BIOS)* is completed exactly as mentioned.

I had disabled Secure Boot in BIOS, but did not disable Fastboot in Windows. As a result I was having a hard time installing 14.04, and the installer would fail each time with the error `grub-efi-amd64-signed failed to install into /target/. Without GRUB boot loader, the installed system will not boot`.

> During installation I did not connect to the network/internet. This was to prevent any unwanted kernel or Xserver updates during installation.

> Additionally, I selected the option to install third party software.

### GRUB settings for Ubuntu Live USB ###

When you boot from the live USB, you will be presented with the GRUB screen allowing you to try Ubuntu before installing.

Select the `Install Ubuntu` option and press `e` which will open the menu edit screen. Add the following kernel parameters just before `quiet splash` in the line that starts with `linux /boot/vmlinuz-linux .....`

```
nomodeset i915.modeset=1
```

Press `F10` and complete the installation as per [guide to install Windows and Ubuntu in dual boot mode]

Once the installation is complete, you will be presented with the GRUB menu, follow the above procedure to edit the kernel parameters and boot into Ubuntu. Next we will make these kernel parameters persistent.

### Persist GRUB settings ###

To avoid updating the kernel options each time the system is booted, update below GRUB settings using `sudo vi /etc/default/grub`

```
GRUB_CMDLINE_LINUX_DEFAULT="nomodeset i915.modeset=1 quiet splash"
```

Optionally, for High DPI or Retina displays, GRUB boot menu options are not readable as the font size is too small. It can be fixed by the below two options

```
GRUB_GFXMODE=1280x1024
GRUB_GFXPAYLOAD_LINUX=keep
```

Optionally, set Windows as the default system to boot

```
GRUB_DEFAULT="Windows Boot Manager (on /dev/sda1)"
```

> Above label is specific to my laptop, replace accordingly with the Windows label for your system.

Update GRUB with these modified settings using the command `sudo update-grub2`

### Upgrade the system ###

During installation we did not connect the system to internet. This is the time to update the system. Setup the internet connection.

We will select the best Ubuntu update server based on our location,
* Select Software and Updates.
* In the tab "Ubuntu Software" select Other in the dropdown for Download Server.
* Select your country name and then select best server.
* Select the server identified as best and close.

With the best Ubuntu update server selected, open a terminal and enter the below commands to update the system

```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
sudo apt-get autoremove
```
> In the upgrade step, when prompted to select the GRUB file, select to keep the local version currently installed.


### Optional - HighDPI settings ###

This is an optional step, but highly recommended if you have a HighDPI or Retina display. Apply the HighDPI configuration discussed in [Configuring Ubuntu for HighDPI and Retina displays]

### Restart and Test ###

Restart the machine and test that it is working as expected, especially the Xserver and graphical display.

### Remove Nvidia and Nouveau, and disable services ###

Now we start the process to get Nvidia drivers installed on the system, but before we do that, we need to remove all possible conflicting drivers

Switch to Virtual Console (Atl + Ctrl + F1). You can have 6 virtual consoles F1 to F6.

Stop the Xserver and remove any existing installs of Nvidia, nouveau etc.

```
sudo service lightdm stop
sudo apt-get remove --purge nvidia*
sudo apt-get remove --purge bumblebee*
sudo apt-get --purge remove xserver-xorg-video-nouveau*
```

Blacklist nouveau, add the following lines to `/etc/modprobe.d/blacklist.conf`

```
# Blacklisting nouveau to get nvidia running with bumblebee
blacklist nouveau
blacklist lbm-nouveau
alias nouveau off
alias lbm-nouveau off
options nouveau modeset=0
```

Disable gpu-manager.service, as it has a tendency to overwrite xorg.conf and result in black screen on boot.

```
sudo vi /etc/init/gpu-manager.conf

# Comment these start on settings ; GPU Manager ruins our work
#start on (starting lightdm
#          or starting kdm
#          or starting xdm
#          or starting lxdm)
task
exec gpu-manager --log /var/log/gpu-manager.log
```

Create initramfs `sudo update-initramfs -u -k all`

### Restart and Test ###

Reboot the system and ensure that xserver is running fine - i.e. GUI is working properly

Verify that nouveau is successfully blacklisted. The command `lsmod | grep nouveau` should not return any lines with the word nouveau in it.

Ensure gpu-manager.service is disabled `sudo service gpu-manager status`. If not, disable as mentioned above.

### Download CUDA toolkit ###

We will install Nvidia drivers as part of the CUDA toolkit installation.

From [CUDA Toolkit download] page, download the `deb(network)` file for Linux, x86_64, Ubuntu, 14.04 to your `$HOME` directory. The filename, as of this writing, is `cuda-repo-ubuntu1404_8.0.44-1_amd64.deb`.

### Install CUDA toolkit and Nvidia drivers ###

CUDA Toolkit installation can be done by installing any of the [CUDA Toolkit Meta Packages].

We will install the `cuda-8-0` meta package to install both the CUDA Toolkit and the Nvidia driver and ensure that they will need to be explicitly upgraded when needed. The `cuda` meta package can get updated as part of system upgrade and may break our working Xserver and other configuration, which we do not want.

The below steps are to be performed in Virtual Console (Ctrl + Alt + F1), as it requires stopping the graphical interface.

Switch to Virtual Console and disable lightdm. `sudo service lightdm stop`

Now we install CUDA toolkit

```
sudo apt-get install linux-headers-$(uname -r)
sudo apt-get install mesa-utils
sudo dpkg -i $HOME/cuda-repo-ubuntu1404_8.0.44-1_amd64.deb
sudo apt-get update
sudo apt-get install cuda-8-0
sudo apt-get autoremove
```
In `autoremove` step a whole lot of needless xservers are removed. Ensure `xserver-xorg-video-intel-lts-wiley` is in the list.  

### Restart and Test ###

Reboot the system and ensure that xserver is running fine - i.e. GUI is working properly

> Caveat - At this step, the first time I restart the machine, lightdm login screen appeared twice. After the second login, I was at the desktop screen. I restarted the machine once again. That resolved the issue.

Ensure lsmod shows both nvidia and i915 drivers loaded in the kernel.

```
hemen@hemen-Inspiron-7559:~$ lsmod | grep nvidia
nvidia_uvm            729088  0
nvidia_modeset        749568  2
nvidia              10223616  64 nvidia_modeset,nvidia_uvm
drm                   360448  8 i915,drm_kms_helper,nvidia

hemen@hemen-Inspiron-7559:~$ lsmod | grep i915
i915                 1130496  3
drm_kms_helper        126976  1 i915
drm                   360448  8 i915,drm_kms_helper,nvidia
i2c_algo_bit           16384  1 i915
video                  40960  3 i915,dell_wmi,dell_laptop

hemen@hemen-Inspiron-7559:~$ glxinfo | grep OpenGL
OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: GeForce GTX 960M/PCIe/SSE2
OpenGL core profile version string: 4.3.0 NVIDIA 361.93.02
OpenGL core profile shading language version string: 4.30 NVIDIA via Cg compiler
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile
OpenGL core profile extensions:
OpenGL version string: 4.5.0 NVIDIA 361.93.02
OpenGL shading language version string: 4.50 NVIDIA
OpenGL context flags: (none)
OpenGL profile mask: (none)
OpenGL extensions:

hemen@hemen-Inspiron-7559:~$ glxgears -info
Running synchronized to the vertical refresh.  The framerate should be
approximately the same as the monitor refresh rate.
GL_RENDERER   = GeForce GTX 960M/PCIe/SSE2
GL_VERSION    = 4.5.0 NVIDIA 361.93.02
GL_VENDOR     = NVIDIA Corporation
GL_EXTENSIONS =  .... DELETED LOT OF TEXT HERE ...
67692 frames in 5.0 seconds = 13537.328 FPS
68586 frames in 5.0 seconds = 13717.179 FPS
69667 frames in 5.0 seconds = 13932.193 FPS
69390 frames in 5.0 seconds = 13877.986 FPS

hemen@hemen-Inspiron-7559:~$ cat /proc/driver/nvidia/version
NVRM version: NVIDIA UNIX x86_64 Kernel Module  361.93.02  Wed Sep 21 16:32:29 PDT 2016
GCC version:  gcc version 4.8.4 (Ubuntu 4.8.4-2ubuntu1~14.04.3)

hemen@hemen-Inspiron-7559:~$ ls -l /dev/nvi*
crw-rw-rw- 1 root root 195,   0 Oct  4 22:23 /dev/nvidia0
crw-rw-rw- 1 root root 195, 255 Oct  4 22:23 /dev/nvidiactl
crw-rw-rw- 1 root root 195, 254 Oct  4 22:23 /dev/nvidia-modeset
crw-rw-rw- 1 root root 244,   0 Oct  4 22:23 /dev/nvidia-uvm
hemen@hemen-Inspiron-7559:~$

```
At this step we have ensured that Nvidia and i915 drivers can both coexist in the kernel. This is an important checkpoint to ensure that we are on the right track so far. The output of `glxinfo` shows that Nvidia is currently the main display driver.


### Install Bumblebee ###

Next we install [Bumblebee] to get the ability to power up Nvidia GPU on demand basis. We will also set Intel as the primary display driver and ensure the Nvidia GPU is powered OFF by default.

All steps below need to be performed in Virtual Console, and Xserver should not be running. Perfom all the steps in one shot, till you reach the restart and test step below.

Switch to virtual console `(Alt + Ctrl + F1)` and stop lightdm.

```
sudo service lightdm stop
sudo apt-get install --no-install-recommends bumblebee
sudo apt-get install bumblebee-nvidia primus

```

### Bumblebee configuration ###

Add the following two lines to `/etc/modules`

```
i915
bbswitch
```
Create a softlink `/etc/modules-load.d/modules.conf` to `/etc/modules`

```
cd /etc/modules-load.d
sudo ln -s ../modules modules.conf
```

Blacklist nvidia from loading, its insertion and removal from the kernel will be handled by [bbswitch]. Add the following lines to `/etc/modprobe.d/bumblebee.conf`.

```
# 361
blacklist nvidia-361
blacklist nvidia-361-updates
blacklist nvidia-experimental-361
blacklist nvidia_361
blacklist nvidia_361_uvm
blacklist nvidia_361_modeset
alias nvidia nvidia_361
alias nvidia-uvm nvidia_361_uvm
alias nvidia-modeset nvidia_361_modeset
```

The value of BusID to be used in the below configuration files may be different on your system. You can get those details using `lspci` command. Make note of the numbers to the extreme left for both Intel and Nvidia controllers.

```
lspci | egrep -i "VGA|3D"

00:02.0 VGA compatible controller: Intel Corporation Device 191b (rev 06)
02:00.0 3D controller: NVIDIA Corporation GM107M [GeForce GTX 960M] (rev ff)
```

Update Bumblebee and Xserver configuration files as shown below

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
KernelDriver=nvidia_361
PMMethod=bbswitch
LibraryPath=/usr/lib/nvidia-361:/usr/lib32/nvidia-361
XorgModulePath=/usr/lib/nvidia-361/xorg,/usr/lib/xorg/modules
XorgConfFile=/etc/bumblebee/xorg.conf.nvidia

## Section with nouveau driver specific options, only parsed if Driver=nouveau
# [driver-nouveau]
# KernelDriver=nouveau
# PMMethod=auto
# XorgConfFile=/etc/bumblebee/xorg.conf.nouveau
```

```
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
    BusID       "PCI:02:00:0"
    Option      "ProbeAllGpus" "false"
    Option      "NoLogo" "true"
    Option      "UseEDID" "false"
    Option      "UseDisplayDevice" "none"
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
  Option            "AllowEmptyInitialConfiguration" "on"
  Option            "IgnoreDisplayDevices" "CRT"
EndSection
```
### Update alternatives ###

Update alternatives to ensure proper library paths are configured.

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
```
Select the mesa option where available, if not available, keep existing setting.

As done above for `x86_64-linux-gnu_gl_conf` update alternatives for below settings. For the i386 options we do not have mesa showing in the list. So keep as is to `/usr/lib/nvidia-361/alt_ld.so.conf`

```
sudo update-alternatives --config x86_64-linux-gnu_egl_conf
sudo update-alternatives --config i386-linux-gnu_gl_conf
sudo update-alternatives --config i386-linux-gnu_egl_conf
sync
sudo ldconfig
```
Perform required miscellaneous tasks by executing the commands mentioned below

```
sudo update-initramfs -u -k all
sudo usermod -a -G bumblebee $USER
sudo apt-get remove nvidia-prime
```
That's it. You are done with all the required configurations. Now we will test if everything is working as expected.


### Restart and Test ###

Restart the machine and ensure that the Xserver / graphical display is working correctly.

Check GPU modules loaded into the kernel. Nvidia should not be loaded, [Bumblebee] will load and unload it on demand. Intel should be loaded.

```
hemen@hemen-Inspiron-7559:~$ lsmod | grep nvidia
hemen@hemen-Inspiron-7559:~$ lsmod | grep i915
i915                 1130496  82
drm_kms_helper        126976  1 i915
drm                   360448  5 i915,drm_kms_helper
video                  40960  3 i915,dell_wmi,dell_laptop
i2c_algo_bit           16384  1 i915
```
Ensure Nvidia GPU is powered OFF by default and Intel is default display / OpenGL driver.

```
hemen@hemen-Inspiron-7559:~$ cat /proc/acpi/bbswitch
0000:02:00.0 OFF
hemen@hemen-Inspiron-7559:~$ glxinfo | grep OpenGL
OpenGL vendor string: Intel Open Source Technology Center
OpenGL renderer string: Mesa DRI Intel(R) Skylake Halo GT2
OpenGL version string: 3.0 Mesa 11.0.2
OpenGL shading language version string: 1.30
OpenGL context flags: (none)
OpenGL extensions:
```
Check [Bumblebee] is able to switch to Nvidia GPU on demand. Note the `OpenGL vendor string` in the output below which is different than the one above.

```
hemen@hemen-Inspiron-7559:~$ optirun -vv glxinfo | grep OpenGL
[  146.851152] [DEBUG]Reading file: /etc/bumblebee/bumblebee.conf
[  146.851729] [INFO]Configured driver: nvidia
[  146.852116] [DEBUG]optirun version 3.2.1 starting...
[  146.852165] [DEBUG]Active configuration:
[  146.852183] [DEBUG] bumblebeed config file: /etc/bumblebee/bumblebee.conf
[  146.852200] [DEBUG] X display: :8
[  146.852217] [DEBUG] LD_LIBRARY_PATH: /usr/lib/nvidia-361:/usr/lib32/nvidia-361
[  146.852258] [DEBUG] Socket path: /var/run/bumblebee.socket
[  146.852296] [DEBUG] Accel/display bridge: primus
[  146.852328] [DEBUG] VGL Compression: proxy
[  146.852345] [DEBUG] VGLrun extra options:
[  146.852361] [DEBUG] Primus LD Path: /usr/lib/x86_64-linux-gnu/primus:/usr/lib/i386-linux-gnu/primus
[  148.118143] [INFO]Response: Yes. X is active.

[  148.118160] [INFO]Running application using primus.
[  148.118277] [DEBUG]Process glxinfo started, PID 2508.
OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: GeForce GTX 960M/PCIe/SSE2
OpenGL core profile version string: 4.3.0 NVIDIA 361.93.02
OpenGL core profile shading language version string: 4.30 NVIDIA via Cg compiler
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile
OpenGL core profile extensions:
OpenGL version string: 4.5.0 NVIDIA 361.93.02
OpenGL shading language version string: 4.50 NVIDIA
OpenGL context flags: (none)
OpenGL profile mask: (none)
OpenGL extensions:
[  148.223795] [DEBUG]SIGCHILD received, but wait failed with No child processes
[  148.223836] [DEBUG]Socket closed.
[  148.223846] [DEBUG]Killing all remaining processes
```

Check the working of `bbswitch` using `glxgears`. Open an additional terminal and check the initial state of Nvidia GPU.

```
hemen@hemen-Inspiron-7559:~$ cat /proc/acpi/bbswitch
0000:02:00.0 OFF
```

In the original terminal, execute `optirun -vv glxgears -info`. While `glxgears` is still running, come back to the other terminal and check the status of bbswitch and modules loaded in the kernel. Note that Nvidia GPU is powered ON and nvidia modules are loaded in the kernel.

```
hemen@hemen-Inspiron-7559:~$ cat /proc/acpi/bbswitch
0000:02:00.0 ON
hemen@hemen-Inspiron-7559:~$ lsmod | grep nvidia
nvidia              10223616  52
drm                   360448  10 i915,drm_kms_helper,nvidia
```

Terminate `glxgears` and check again.

```
hemen@hemen-Inspiron-7559:~$ cat /proc/acpi/bbswitch
0000:02:00.0 OFF
hemen@hemen-Inspiron-7559:~$ lsmod | grep nvidia
hemen@hemen-Inspiron-7559:~$
```

At this point, we have Nvidia drivers, Bumblebee and Xserver configured properly.

Restart two to three times and perform these same testing steps on each restart. This is to ensure that our configuration is persistent.

If all is OK after multiple restarts, then we proceed to configure CUDA.

### Configuring CUDA ###

Edit `~/.bashrc` and setup/update the `$PATH` and `$LD_LIBRARY_PATH` environment variables as shown below

```
export PATH="$PATH:/usr/local/cuda-8.0/bin"
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/cuda-8.0/lib"
```

### Install and Build CUDA samples ###

Install CUDA samples in home directory and build the deviceQuery example.

```
hemen@hemen-Inspiron-7559:~$ cd /usr/local/cuda-8.0/bin
hemen@hemen-Inspiron-7559:/usr/local/cuda-8.0/bin$ ./cuda-install-samples-8.0.sh ~
hemen@hemen-Inspiron-7559:/usr/local/cuda-8.0/bin$ cd ~
hemen@hemen-Inspiron-7559:~$ cd NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery

hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ make
/usr/local/cuda-8.0/bin/nvcc -ccbin g++ -I../../common/inc  -m64    -gencode arch=compute_20,code=sm_20 -gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_60,code=compute_60 -o deviceQuery.o -c deviceQuery.cpp
nvcc warning : The 'compute_20', 'sm_20', and 'sm_21' architectures are deprecated, and may be removed in a future release (Use -Wno-deprecated-gpu-targets to suppress warning).
/usr/local/cuda-8.0/bin/nvcc -ccbin g++   -m64      -gencode arch=compute_20,code=sm_20 -gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_60,code=compute_60 -o deviceQuery deviceQuery.o
nvcc warning : The 'compute_20', 'sm_20', and 'sm_21' architectures are deprecated, and may be removed in a future release (Use -Wno-deprecated-gpu-targets to suppress warning).
mkdir -p ../../bin/x86_64/linux/release
cp deviceQuery ../../bin/x86_64/linux/release

hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ ls -l
total 628
-rwxrwxr-x 1 hemen hemen 582882 Oct  5 00:08 deviceQuery
-rw-r--r-- 1 hemen hemen  12174 Oct  5 00:07 deviceQuery.cpp
-rw-rw-r-- 1 hemen hemen  21128 Oct  5 00:08 deviceQuery.o
-rw-r--r-- 1 hemen hemen   9075 Oct  5 00:07 Makefile
-rw-r--r-- 1 hemen hemen   1737 Oct  5 00:07 NsightEclipse.xml
-rw-r--r-- 1 hemen hemen    168 Oct  5 00:07 readme.txt
```
This should create deviceQuery executable as shown above.

### Execute CUDA example ###

Executing the CUDA example is the tricky part, and involves the following steps in order

1. Power ON Nvidia GPU using `bbswitch`
2. Load nvidia and nvidia_uvm kernel modules manually
3. Use optirun to execute the required CUDA executable. This is required to set proper library paths.
4. Unload the nvidia and nvidia_uvm modules
5. Power OFF the Nvidia GPU using `bbswitch`

This complete process is outlined below

Initial State

```
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ lsmod | grep nvidia
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ ls -l /dev/nvid*
crw-rw-rw- 1 root root 195, 254 Oct  4 23:46 /dev/nvidia-modeset
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ cat /proc/acpi/bbswitch
0000:02:00.0 OFF
```

Power ON Nvidia GPU and load required kernel modules manually

```
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ sudo tee /proc/acpi/bbswitch <<< ON
ON
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ cat /proc/acpi/bbswitch
0000:02:00.0 ON
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ sudo modprobe nvidia_361
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ sudo modprobe nvidia_361_uvm
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ lsmod | grep nvidia
nvidia_uvm            729088  0
nvidia              10223616  1 nvidia_uvm
drm                   360448  6 i915,drm_kms_helper,nvidia
```

Our system is now in a state where we can execute CUDA programs. Optirun is required, or else we get the below error message if not used.

```
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ ./deviceQuery
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

cudaGetDeviceCount returned 35
-> CUDA driver version is insufficient for CUDA runtime version
Result = FAIL

# Notice the difference when using optirun to execute deviceQuery


hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ optirun ./deviceQuery
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "GeForce GTX 960M"
  CUDA Driver Version / Runtime Version          8.0 / 8.0
  CUDA Capability Major/Minor version number:    5.0
  Total amount of global memory:                 4046 MBytes (4242014208 bytes)
  ( 5) Multiprocessors, (128) CUDA Cores/MP:     640 CUDA Cores
  GPU Max Clock rate:                            1176 MHz (1.18 GHz)
  Memory Clock rate:                             2505 Mhz
  Memory Bus Width:                              128-bit
  L2 Cache Size:                                 2097152 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(65536), 2D=(65536, 65536), 3D=(4096, 4096, 4096)
  Maximum Layered 1D Texture Size, (num) layers  1D=(16384), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(16384, 16384), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
  Run time limit on kernels:                     Yes
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 2 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 8.0, CUDA Runtime Version = 8.0, NumDevs = 1, Device0 = GeForce GTX 960M
Result = PASS
```

Once done, unload the modules (in order as shown below) and then power OFF the GPU

```
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ sudo rmmod nvidia_uvm
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ sudo rmmod nvidia
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ lsmod | grep nvidia
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ cat /proc/acpi/bbswitch
0000:02:00.0 ON
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ sudo tee /proc/acpi/bbswitch <<< OFF
OFF
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$ cat /proc/acpi/bbswitch
0000:02:00.0 OFF
hemen@hemen-Inspiron-7559:~/NVIDIA_CUDA-8.0_Samples/1_Utilities/deviceQuery$
```

### Restart and Test ###

Restart your machine and ensure that X server, GUI as well as all the tests mentioned above are working as expected.

### Caution with System Update ###

Since you are on Ubuntu LTS 14.04.4 you will occasionally get a message to upgrade to the latest LTS version 14.04.5 or 16.04.

> Do not automatically upgrade as it will upgrade the kernel and Xserver and will break everything.

However we do want to keep our system updated for any security issues and updates available to the installed applications. It is possible and recommended to use `apt-get` to get the same done. I have done this multiple times and it is perfectly safe.

```
sudo apt-get update
sudo apt-get upgrade
```
I am not sure if I ran a `dist-upgrade`, so not sure if it works or not. If you see the upgrade changing kernel version to v4.4 or higher then there is a possibility that the display will not work post upgrade, so check what packages are being upgraded with `dist-upgrade` before you proceed.


### That's it ###

So, that it. I spent countless hours figuring this out. Hope this helps you. If it does drop a comment below. If you find something missing or erroneous, drop a comment in that case too.


[Nvidia driver]: http://www.nvidia.com/Download/index.aspx?lang=en-us
[CUDA]: https://developer.nvidia.com/cuda-zone
[Bumblebee]: https://wiki.archlinux.org/index.php/bumblebee
[Dell 7559]: http://www.dell.com/en-us/shop/productdetails/inspiron-15-7559-laptop/dncwpw5722b
[Optimus]: http://www.nvidia.com/object/optimus_technology.html
[kernel v4.2]: https://wiki.ubuntu.com/TrustyTahr/ReleaseNotes/14.04.4#Updated_Packages
[LTS 14.04.4]: https://wiki.ubuntu.com/TrustyTahr/ReleaseNotes/14.04.4
[Download Ubuntu LTS 14.04.4]: http://releases.ubuntu.com/14.04.4/
[create a bootable USB]: https://www.ubuntu.com/download/desktop/create-a-usb-stick-on-windows
[guide to install Windows and Ubuntu in dual boot mode]: http://www.everydaylinuxuser.com/2013/09/install-ubuntu-linux-alongside-windows.html
[Configuring Ubuntu for HighDPI and Retina displays]: {% post_url 2016-05-06-Configuring-Ubuntu-for-highDPI-and-Retina-displays %}
[CUDA Toolkit download]: https://developer.nvidia.com/cuda-downloads
[CUDA Toolkit Meta Packages]: http://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#package-manager-metas
[bbswitch]: https://github.com/Bumblebee-Project/bbswitch
