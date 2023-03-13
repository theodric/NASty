# dumbnas

## Goal: build a power-efficient, silent, moderately performant, medium-capacity, semi-affordable NAS

## Equipment for testing:
#### Client: ThinkPad X230
* custom Core i7-3615QE 2.3GHz quad-core CPU
* 16GB RAM
* Crucial MX500 1TB SATA SSD
* Arch Linux
* kernel `6.1.8-arch1-1 PREEMPT_DYNAMIC x86_64`

#### Raspberry Pi NAS
* Jonsbo N1 5-bay SATA NAS case
* 120W max PicoPSU
* 12V 3A power brick
* Pi CM4 8GB RAM + 32GB eMMC
* CM4 I/O board
* Pericom 2-slot PCIe switch
* ASM1166 6-port SATA controller (4x, plugged into PCIe switch, 1x slot) - Kernel driver in use: ahci
* FL1100 USB 3 controller (1x, plugged into PCIe switch, 1x slot, not used) - Kernel driver in use: xhci_hcd
* 4x Samsung QVO 8TB SATA SSD
* 1x Crucial MX500 500GB SATA SSD

``root@nas2:~# lspci``
``00:00.0 PCI bridge: Broadcom Inc. and subsidiaries BCM2711 PCIe Bridge (rev 20)``
``01:00.0 PCI bridge: Pericom Semiconductor PI7C9X2G304 EL/SL PCIe2 3-Port/4-Lane Packet Switch (rev 05)``
``02:01.0 PCI bridge: Pericom Semiconductor PI7C9X2G304 EL/SL PCIe2 3-Port/4-Lane Packet Switch (rev 05)``
``02:02.0 PCI bridge: Pericom Semiconductor PI7C9X2G304 EL/SL PCIe2 3-Port/4-Lane Packet Switch (rev 05)``
``03:00.0 SATA controller: ASMedia Technology Inc. ASM1166 Serial ATA Controller (rev 02)``
``04:00.0 USB controller: Fresco Logic FL1100 USB 3.0 Host Controller (rev 10)``

Debian Testing aarch64 - Raspberry Pi OS "Lite" 64-bit - 2023-Feb-07 updates

Custom kernel with preemptibility disabled; SATA and XFS support built in: "5.15.92-v8-sata+ aarch64"

Pi CM4 boots from eMMC, then mounts root on Crucial MX500 SATA SSD.

Network: direct wired connection between Pi CM4 I/O board Ethernet and ThinkPad X230 onboard Ethernet
1Gbps uplink, averaging 935Mb/sec in iPerf3 testing.

--

### Methodology:

All server-side configs are the Debian defaults.

All filesystems are XFS.

No NFS tuning was performed.

Bulk read and write speeds are both obtained with dd, starting by creating a 32GB zero file via a
16MB write chunk size, then reading it back with the same parameters:

Write to file (local and NFS tests): `dd if=/dev/zero of=testfile bs=16M count=2048 status=progress`

Read from remote file to local file (NFS tests): `dd if=/nfs/testfile of=/localfile bs=16M status=progress`

Read from local file, dump to /dev/null (local tests): `dd if=testfile of=/dev/null bs=16M status=progress`

IOPS performance was determined using fio:

`fio --randrepeat=1 --ioengine=libaio --direct=1 --gtod_reduce=1 --name=fiotest --filename=testfio --bs=4k --iodepth=64 --size=8G --readwrite=randrw --rwmixread=75`

100% CPU load was induced using stress-ng

`stress-ng --cpu 5 --timeout 20`

--

### Benchmarks:

#### Power consumption:

* Idle power consumption with 5 SSDs is 17W
* Power consumption during 4-disk writes is 24W
* Power consumption during 5-disk writes with CPU at 100% using 'stress-ng' is 26W

#### Local:

Read & Write - 32GB file @ 16MB chunk

1x Crucial MX500 500GB [control]

> 32GB read: 406MB/sec
> 
> 32GB write: 404MB/sec

1x 8TB QVO:

> 32GB read: 418MB/sec
> 
> 32GB write: 196MB/sec

4x 8TB QVO, md RAID0

> 32GB read: 395MB/sec
> 
> 32GB write: 381MB/sec


#### IOPS

1x Crucial MX500 500GB (underprovisioned to 256GB) [control]

> read=94.2MB/sec @ 24.1k IOPS
> 
> write=31.9MB/sec @ 8156 IOPS

1x 8TB QVO:

> read=92.5MB/s @ 23.7k IOPS
> 
> write=31.3MB/sec @ 8005 IOPS

4x 8TB QVO, md RAID0:

> read=84MB/s @ 21.7k IOPS
> 
> write=28MB/sec @ 7330 IOPS

#### NFS performance:

(Speeds reported are as observed on the client, with the Pi CM4 as the server.)

1x Crucial MX500 500GB (underprovisioned to 256GB) [control]:

> 32GB read, 107MB/sec
> 
> 32GB write, 115MB/sec

1x 8TB QVO: 

> 32GB read, 107MB/sec
> 
> 32GB write, 115MB/sec

4x 8TB QVO, md RAID0:

> 32GB read, 108MB/sec
> 
> 32GB write, 96.7MB/sec

Write to 4x QVO in RAID0 over NFS uses 100% of 1 CPU core, and 25-50% of a further 2

Read from 4x QVO in RAID0 over NFS uses 50% of 1 CPU core

## Observations:

0. Power consumption at 17W idle is low for a NAS, but I had hoped it would be lower.
At current Irish power rates of €0.39/kWh, this setup will cost €58.08 per year to idle, or €88.83 to fully load. 
(My current NAS idles at 44 watts, reaching 63 under load; €150.32 and €215.23 respectively)

1. The Pi4's PCIe 2.0 1x bus hampers extracting the full performance from a single SSD, let alone a group of SSDs, but is still more than sufficient to fill a gigE or even 2.5gigE pipe. 450MB/sec across the bus is about the most you'll see in practice, and that is not assuming the use of multiple devices on a PCIe switch.

2. Samsung QVO 8TB QLC SSD read performance is strong compared to the TLC control SSD used, but its write performance is particularly poor, dropping to 196MB/sec on 32GB writes to a single disk, vs 404MB/sec on 32GB writes for the TLC control.

3. SoftRAID over 4x QVO SSDs introduces overhead that creates a penalty (primarily visible on NFS writes) since there is CPU load associated with both arbitrating the RAID operations as well as shuttling data from the Ethernet card (N.B. on the Pi, the gigE PHY is connected via a proprietary high-speed peripheral bus to the CPU, and is not a PCIe device)

4. This write penalty disappears with single-disk operation-- QVO or other SSDs are within range of each other

5. IOPS performance is not(!) improved by scattering reads across multiple SSDs-- probably due to bus contention and low overall PCIe bus bandwidth

6. NFS read performance is able to fill the gigE pipe with both single-disk and RAID0 configs

7. This setup exhibits good local read/write performance, good NFS read performance, and acceptable NFS write performance that is approx 11% less than the read on long operations. (Short writes in the neighbourhood of 1-2GB complete at line speed, probably thanks to server RAM caching of I/O traffic)

8. The QVO QLC SSDs have a relatively-low write rating for their capacity, and will not tolerate constantly being hammered with write traffic.

Such a setup is thus suited to:
A) read-focused client loads, i.e. "write once and archive forever"
B) a "torrentbox" that primarily downloads from the Internet and serves the content up later to a user via NFS/samba
C) a target for scheduled backups that can trickle through in the background

9. Even with 2.5gigE, while client NFS reads would almost certainly improve, it is unlikely to be suitable as a remote target for write-focused activities such as content editing, given that a 2.5gigE card would need to be connected via the PCIe bus which will further exacerbate the observed bus contention issues.

10. At €1880 for the QVO SSDs, €50 for the Crucial, and about €350 for the rest of the gear (not counting what I pulled from my parts bin), the idle power cost delta of €92.24/annum vs current NAS will require just under 25 years to pay for itself. But it will fail to do so very, very quietly, which makes it OK.
