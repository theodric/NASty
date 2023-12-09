v2 of NASty will use a regular Raspberry Pi 5 with a HatDrive! Bottom providing an NVMe breakout, 
and an NVMe M.2 to PCIe slot converter to connect to the disk controller.

Differential BOM:

| What           | Product           | $$$$ | Link                                                                     |
|----------------|-------------------|------|--------------------------------------------------------------------------|
| SBC            | Pi 5 8GB RAM      | $115 | https://shop.pimoroni.com/products/raspberry-pi-5?variant=41044580171859 |
| USB-C PD 5v5A  | Chinesium Special | $32  | https://www.aliexpress.com/item/1005003967784373.html                    |
| NVMe breakout  | HatDrive! Bottom  | $48  | https://pineberrypi.com/products/hatdrive-bottom-2230-2242-2280-for-rpi5 |
| M.2 to PCIe    | Chinesium Special | $17  | https://www.aliexpress.com/item/1005002663297639.html                    |

Notes:
1. See NASty-setup-qemu-kvm.txt for content specific to later versions of virt-manager/qemu-kvm/Debian
2. The SD controller is *way* better - still plan to boot from an SSD, however
3. The system pisses and moans if not connected to a 5v5A power supply. Purchased, but still to be tested, a device that accepts 12VDC input (as from a picoPSU) and claims to output 5v5A: https://www.aliexpress.com/item/1005003967784373.html

