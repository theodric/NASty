The Raspberry Pi kernel build process which pertains to `bullseye`-era (Debian 11) /boot layout, corresponding to 5.15.92+, is documented here:
https://web.archive.org/web/20211201095755/https://www.raspberrypi.com/documentation/computers/linux_kernel.html

Current documentation for `bookworm` (Debian 12) onward lives here:
https://www.raspberrypi.com/documentation/computers/linux_kernel.html

5.15.92+ runs fine with the new /boot/firmware layout and `trixie` (Debian 14) as of 2026-01-08

IMPORTANT NOTE:
The Pericom PCIe switch used in this build causes instability in an unpatched kernel! 
The GitHub issue tracking the changes is available here: https://github.com/raspberrypi/linux/issues/5352

I have merged the necessary changes (which were targeted at a 6.1-series kernel) into 5.15.92.
The modified files are available in this repo in [kernel-5.15.92-config-and-modified-files.tar](kernel-5.15.92-config-and-modified-files.tar).
I have also provided a validated kernel config in the same file.

On buildhost mac ACVM vm in qemu: 
``GRUB_CMDLINE_LINUX_DEFAULT="nosplash text biosdevname=0 net.ifnames=0 console=tty0 console=ttyAMA0,115200 earlyprintk=ttyAMA0,115200 consoleblank=0 systemd.show_status=true"``

Kernel changes:

General setup:
* Local version -> -v8+-sata
* Preemption Model -> No Forced Preemption (Server)

Device drivers:

* Serial ATA and Parallel ATA drivers (libata) -> * (build in) rather than M (module)
* AHCI SATA support -> * (build in) rather than M (module)
* Marvell SATA support -> * (build in) rather than M (module)

File systems:
* XFS -> * (build in) rather than M (module)
