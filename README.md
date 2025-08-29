# hacked_limo
I was unable to reflash the nano with agilex image and firmware. This is my attempt to create a workaround.


# AgileX LIMO with Jetson Nano — Custom Setup & ROS2 Eloquent Integration

This guide outlines how to reflash the Jetson Nano on the AgileX LIMO, get custom SD-card support via a device-tree overlay, and configure ROS2 Eloquent in containers saved to the SD card.

---

##  Prerequisites

- Ubuntu 18.04 (host machine)
- NVIDIA SDK Manager version 2.3 supporting jetpack 4.6.6 for Jetson Nano flash
- AgileX LIMO robot using the nano
- USB-Micro (must allow data) 
- Micro-SD card 


---

## 1. Flash Jetson Nano Image

1. On the **Ubuntu 18.04** host machine, install an **older version of NVIDIA SDK Manager** to support Jetson Nano JetPack 4.x. and start the program.
2. Put the nano in force recover mode by jumping/connecting the recovery pin to ground. 
3. Connect the micro usb 
4. power on
5. The sdk manager should recognize the nano.
6. follow instructions to start flashing


---

## 2. Bring Up & Discover SD-Card Module via Device-Tree Overlay

Inside the nano's case, the nano module is mounted on a carrier board. This carrier board has the sd card reader module on it.

Since were not using the original agilex image, we wont have access to the peripherals like the sd card reader.

Since the custom carrier board doesn’t auto-detect the SD card by default, we will need to create and apply a device-tree overlay.

###  Steps:

1. Connect the limo to a network either wifi or ethernet.
2. SSH in from the host machine
3. make sure device tree compiler exists on the nano
    ```bash
        sudo apt-get update && sudo apt-get install -y device-tree-compiler
4. check the dtb folder for what DTB you are using. 
mine says `kernel_tegra210-p3448-0002-p3449-0000-b00.dtb`
    ```bash
        ls /boot/dtb/
5. Decompile the DTB so you can search through it
    ```bash
    sudo dtc -I dtb -O dts -o /tmp/base.dts /boot/dtb/kernel_tegra210-p3448-0002-p3449-0000-b00.dtb
6. your looking for stuff related to this `sdhci@700b0400`

7. Create a new .dts file ` limo-sdmmc3-overlay.dts`
8. Compile the `.dts` into `.dtbo`:
   ```bash
   sudo dtc -@ -I dts -O dtb -o /boot/limo-sdmmc3.dtbo limo-sdmmc3-overlay.dts

9. Merge Overlay with Base DTB

    Merge the overlay into a standalone DTB so U-Boot loads the combined version directly:
    ```bash
    sudo fdtoverlay   -i /boot/dtb/kernel_tegra210-p3448-0002-p3449-0000-b00.dtb -o /boot/dtb/kernel_tegra210-p3448-0002-p3449-0000-b00-merged.dtb  /boot/limo-sdmmc3.dtbo


10. Point extlinux.conf to the Merged DTB
    Update U-Boot’s configuration to load your merged DTB instead of the default:
    ```bash
    sudo vim /boot/extlinux/extlinux.conf


11. Within the primary label, set the FDT directive:
    ```conf
    TIMEOUT 30
    DEFAULT primary

    MENU TITLE L4T boot options

    LABEL primary
        MENU LABEL primary kernel
        LINUX /boot/Image
        INITRD /boot/initrd
        FDT /boot/dtb/kernel_tegra210-p3448-0002-p3449-0000-b00-merged.dtb
        APPEND ${cbootargs} quiet root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0 sdhci_tegra.en_boot_part_access=1  root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0 sdhci_tegra.en_boot_part_access=1 nv-auto-config

12. Reboot & Verify
    After rebooting, confirm everything is active:
    ```bash
    cat /proc/device-tree/sdhci@700b0400/status
    # Expected: okay

## 3. Configure boot priority (TODO)
- Boot priority hasn’t been customized yet; Jetson will default to the standard SD‑card rootfs.
- The overlay entry will add a new label to `/boot/extlinux/extlinux.conf`, pointing to a new combined DTB
- You can leave the default label for now; later you may adjust priority via `extlinux.conf`.

## 4. Set Up Container Storage on the SD Card



## 5. Install ROS2 Eloquent
