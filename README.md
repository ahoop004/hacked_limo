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

2. Compile the `.dts` into `.dtbo`:
   ```bash
   dtc -O dtb -o my-sdcard.dtbo -@ my-sdcard.dts
3. Copy `my-sdcard.dtbo` to `/boot/` on the Jetson.

4. Apply the overlay:
    ```bash
    sudo /opt/nvidia/jetson-io/config-by-hardware.py -n "My SD-Card Overlay"


5. Reboot, then verify:
    ```bash
    ls /proc/device-tree/…-sdcard-detect

## 3. Configure boot priority (TODO)
- Boot priority hasn’t been customized yet; Jetson will default to the standard SD‑card rootfs.
- The overlay entry will add a new label to `/boot/extlinux/extlinux.conf`, pointing to a new combined DTB
- You can leave the default label for now; later you may adjust priority via `extlinux.conf`.

## 4. Set Up Container Storage on the SD Card

- Create a partition on the SD card (if not already configured) to store Docker/containers.

- Mount it (e.g., `/mnt/containers`) in your flash image.

- Configure Docker storage to use that mount:
    ```json
    {
  "data-root": "/mnt/containers/docker"
    }
- Restart Docker so container persists on sd card

## 5. Install ROS2 Eloquent

1. Add ROS2 Eloquent repositories and keys on Jetson Nano (Ubuntu 18.04).

2. Install ros-eloquent-desktop or desired ROS2 components.

3. Initialize dependencies:
    ```bash
    sudo apt update && sudo apt install curl gnupg2 lsb-release
4. Follow ROS2 installation guide for Eloquent on Bionic.

5. Optionally, use a Docker container (e.g., `osrf/ros:eloquent-desktop`) and mount your ROS2 workspace in, with container storage pointed to the SD‑card partition.