The Raspberry Pi kernel build process is documented here:
https://www.raspberrypi.com/documentation/computers/linux_kernel.html


On buildhost mac ACVM vm in qemu: 
``GRUB_CMDLINE_LINUX_DEFAULT="nosplash text biosdevname=0 net.ifnames=0 console=tty0 console=ttyAMA0,115200 earlyprintk=ttyAMA0,115200 consoleblank=0 systemd.show_status=true"``

Kernel changes:

General setup:
* Local version -> -v8-sata
* Preemption Model -> No Forced Preemption (Server)

Device drivers:

* Serial ATA and Parallel ATA drivers (libata) -> * (build in) rather than M (module)
* AHCI SATA support -> * (build in) rather than M (module)
* Marvell SATA support -> * (build in) rather than M (module)

File systems:
* XFS -> * (build in) rather than M (module)
