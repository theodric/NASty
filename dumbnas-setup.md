\#How to set up your dumb Pi CM4 NAS with an OS, rebuild and replace the kernel, and then reconfigure it to boot from the SATA disk.

####Note
*A Pi CM4 Lite would be better suited to this project, but mine uses a Pi CM4 with an eMMC because it's literally 2023 and you either buy whatever you can get your hands on or else you'll have nothing at all.*

###Preface

I will assume you know how to build a kernel, or can follow the Raspberry Pi Foundation's instructions successfully, and are comfortable with setting up disk partitions from the command line. If not, it's best if you don't attempt this, because day-to-day maintenance of the system will assume a comparable level of capability.

I assume you already have your SATA disk connected to the CM4 via a supported PCIe SATA controller, and have confirmed that it works out of the box after the system boots. If not, do that first: as of Q4 2022, all supported SATA controllers work out of the box-- just not as root devices at boot time. None of the below will have any effect if your device isn't supported or functional.



###Initial setup
1. Download a Raspberry Pi 64-bit OS image (Lite will likely suffice unless you intend to use the graphics capability of the Pi for something).

2. If you're using a CM4 with an eMMC, to access the eMMC from your computer and flash it with software, you will need to set the jumper [as shown in figure one](figure1.png), power it on, plug a microUSB cable from the CM4 I/O board into your computer, and run the 'rpiboot' tool to load the USB disk bootloader.

3. The 'rpiboot' USB disk bootloader tool referenced in the above step can be downloaded and compiled from the sources available here: https://github.com/raspberrypi/usbboot. It builds fine on an M1 Mac running Big Sur 10.7.4.

4. Use the Raspberry Pi imaging tool to flash the image downloaded in step 1 to the SD card or eMMC, because it offers the ability to easily configure several basic system settings including unprivileged user at flash time. Or don't, I'm not your mom. Be difficult, if you want.

5. If you're using a CM4 with an eMMC, power the system off and remove the jumper set in step 2. Otherwise, just insert the SD card to the I/O board's socket.

6. If you're using a CM4 with an eMMC, disconnect either end of the MicroUSB cable, because leaving it connected it will disable the onboard USB 2.0 ports which you'll need if you decide to attach a local keyboard or something.

7. Boot the system and log in - preferably over SSH.

###Building a kernel

We will need to build a kernel that has in-built support for the SATA card (and optionally for XFS, if you choose to use XFS for your root filesystem).

You will probably find it preferable to build (and likely cross-compile) the kernel on another, more performant system. For example, I build my kernel in an 8-core/8GB RAM Debian VM running inside the free 'ACVM' hypervisor manager on my M1 MacBook Air. A full build there requires about 9 minutes, but would likely take 2 hours or more on the Pi4. If you have no better option, or are just an incredibly obstinate person, you can build on the Pi itself.

Review the steps at https://www.raspberrypi.com/documentation/computers/linux_kernel.html paying special attention to the subtle differences between 32-bit and 64-bit kernel builds.
Install the required packages to configure and build the kernel on your buildhost.
I have a strong preference for 'nconfig' over menuconfig/config. You'll need to install ncurses support libraries to use it.

I diverge from the recommended workflows for installing the kernel, finding it simpler to execute the initial ``make -j`nproc` Image.gz modules dtbs`` step on the VM on my Mac as if I were building directly on the Pi, then .tar up the entire build environment, scp it over to the Pi, unpack it, and execute the remaining `make modules_install` and DBT installation steps locally. Do whatever you prefer, and don't tell me about it, because I don't need to know.

I highly recommend installing the new kernel using a different filename, e.g. `kernel-sata.img`, so you still have a backup kernel to fall back to in the event something goes wrong in the below steps.

Within the kernel config tool, I make the following tweaks to adjust the kernel to my requirements:

*General setup:*

`Local version -> -v8+-sata`

`Preemption Model -> No Forced Preemption (Server)`

*Device drivers:*

`Serial ATA and Parallel ATA drivers (libata) -> * (build in) rather than M (module)`

`AHCI SATA support -> * (build in) rather than M (module)`

`Marvell SATA support -> * (build in) rather than M (module)`

*File systems:*

`XFS -> * (build in) rather than M (module)`

If you like, you can [download my premade config file from this repo](config-5.15.92-v8+-sata). Import it in the kernel configurator, or simply move it to the build directory and rename it to '.config', then carry on with the remaining build steps.

###Testing the new kernel:

Assuming you followed my advice and specified a different filename for the new kernel, once you have built and installed the kernel, change to the /boot partition and make a backup of the stock config.txt to config.stock, then modify the config.txt by adding an additional stanza specifying the new kernel's filename:

`## 2023-0207 - added SATA kernal boot param`

`kernel=kernel-sata.img`

If it all goes to hell, you can plug the SD card into your computer (or rpiboot the eMMC again to access it) and restore the config.stock to config.txt, then boot the stock kernel again.

###SATA device setup:

Given that you already confirmed at the outset that your SATA device is working, we can move on to getting it set up for use as our new root filesystem device, and then cloning the SD/eMMC rootfs to it.

Prepare your SATA disk. It will require at least one partition formatted in either ext4 (standard) or XFS (if you're special, and built the module into the new kernel), which will be the new /, and optionally a second to serve as a swap partition.
You should not create a /boot partition on the SATA disk. The system will not boot from SATA, it will boot from either the SD card or eMMC, then mount the / partition on SATA. This is a limitation of the Pi4's firmware.

###Filesystem cloning:

1. Make a temporary mount directory named `/sataroot` and mount the ext4/xfs partition on the SATA disk there
2. `cd /`
3. `sudo rsync -aAxXHv / --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"},"/sataroot"} /sataroot`
4. `cd /sataroot`
5. `mkdir dev proc sys tmp run mnt media`

Config file changes:

We need to adjust two additional parameters to permit booting from the SATA disk:

1. change `/sataroot/etc/fstab` to use the UUID of the new root filesystem on SATA
	
	*(Definitely don't use /dev/sdX device names - connected USB disks are initialized before SATA on the Pi!)*

2. change `/boot/cmdline.txt` to use the partUUID of the new root filesystem on SATA

###Final test:

Once you're confident that you've populated the correct values in the files, issue a 'reboot' and watch the system come up on its new SATA rootfs!

Remember that from now on, **YOU** are responsible for building and installing kernel updates: if you take a stock kernel update from Raspberry Pi OS, it will not boot, because it doesn't have SATA rootfs support built in.
