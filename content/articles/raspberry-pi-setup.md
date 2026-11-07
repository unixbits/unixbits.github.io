---
title: "Setting Up a Raspberry Pi for Linux Real-Time Experiments"
description: "Preparing a headless Raspberry Pi 4 as a Linux laboratory for kernel, GPIO and real-time experiments."
topics:
  - real-time-linux
  - linux-internals
reading_room: true
weight: 10
---

A Raspberry Pi is an extremely useful platform for exploring Linux internals: inexpesive,
widely tested and documented, it runs a conventional Linux kernel, provides direct access
to hardware interfaces, and can be controlled via remote access.

This article gives step-by-step instruction how prepare a Raspberry Pi 4 running the 64-bit
Raspberry Pi OS Lite distribution with remote access.

The objective is to create a reproducible starting point for later experiments involving
GPIO, kernel scheduling, timing and real-time behaviour.

> **Tested configuration:** Raspberry Pi 4 Model B Rev 1.5 (1 GB RAM), Raspberry Pi OS Lite 64-bit, Linux kernel 6.18.50+rpt-rpi-v8. Price: AUD $90

## 1. Hardware and software

- Raspberry Pi 4 Model B, revision 1.5, with 1 GB RAM
- 64 GB microSD card
- Raspberry Pi power supply
- Debian Linux workstation
- Local network connection

We will use Raspberry Pi OS Lite (64-bit), without a graphical desktop.

The Raspberry Pi will operate *headlessly*: no monitor, keyboard or mouse.

## 2. Prepare the microSD card

Insert the microSD card into the Linux workstation using a suitable USB reader.

Identify the device:

```sh
lsblk -o NAME,SIZE,MODEL
```

In my case, the relevant device appeared as:

```text
sde    58.2G    MassStorageClass
└─sde1 58.2G
```

The reported capacity of 58.2 GiB is consistent with a nominal 64 GB card.

**Note:** Device names are not fixed. The card may appear as `/dev/sdb`, `/dev/sdc` or something else. Identify the correct device carefully before writing an image.

There is no need to format the card beforehand; raspberry Pi Imager writes the
operating-system image, including its partition table and filesystems.

## 3. Install Raspberry Pi OS

From Linux workstation, download the Raspberry Pi Imager AppImage:

```sh
wget https://downloads.raspberrypi.com/imager/imager_latest_amd64.AppImage
chmod +x imager_latest_amd64.AppImage
./imager_latest_amd64.AppImage
```

Run Raspberry Pi Imager and select:

- Device: Raspberry Pi 4
- Operating system: Raspberry Pi OS Lite (64-bit)
- Storage: 64 GB microSD card
- Hostname: `pi4`
- Username: `pi`
- SSH: Enabled
- Raspberry Pi Connect: Disabled

Network credentials were configured during imaging, allowing the Pi to join the
network automatically on its first boot.

After the image was written and verified, insert the card into the Raspberry Pi
and connected the power supply.

## 4. Establish SSH access

Once the Pi has booted and joined the network, from the Linux workstation:

```sh
ssh pi@pi4.local
```

The `.local` hostname uses multicast DNS (mDNS), avoiding the need to discover or
configure a fixed IP address.

During the first boot, the intended hostname did not resolve immediately. This
required inspecting the network advertisements using Avahi:

```sh
avahi-browse -art | grep -i pi
```

The Pi initially appeared as `pi4-2.local`. After a subsequent reboot, the expected
hostname `pi4.local` became available.

This illustrates a useful practical point: network discovery and hostname resolution
may require investigation even when the underlying SSH service is working correctly.

## 5. Configure and update the system

The initial SSH session reported that the workstation's Australian locale was unavailable
on the Pi.  Inspect the installed locales:

```sh
locale
locale -a
```

Configure the required locale:

```sh
sudo dpkg-reconfigure locales
```

In this case, I enabled `en_AU.UTF-8` and selected it as the default.

Next, update Raspberry Pi OS:

```sh
sudo apt update
sudo apt full-upgrade
sudo reboot
```

After rebooting, reconnected using SSH.

## 6. Inspect the Linux system

Identify the running kernel:

```sh
uname -a
```

Raspberry Pi reported:

```text
Linux pi4 6.18.50+rpt-rpi-v8 #1 SMP PREEMPT
```

The system runs a 64-bit ARM Linux kernel.

Next, inspect the processor:

```sh
lscpu
```

The relevant characteristics:

| Property | Observed value |
|---|---|
| Architecture | aarch64 |
| Processor | ARM Cortex-A72 |
| CPU cores | 4 |
| Maximum CPU frequency | 1800 MHz |
| Minimum CPU frequency | 600 MHz |

Inspect memory:

```sh
free -h
```

The Pi reported approximately 905 MiB of usable RAM, consistent with its
nominal 1 GB configuration.

Identify the board:

```sh
cat /sys/firmware/devicetree/base/model
```

Result:

```text
Raspberry Pi 4 Model B Rev 1.5
```

Finally, inspect the SoC temperature:

```sh
cat /sys/class/thermal/thermal_zone0/temp
```

The measurement was:

```text
41868
```

The value is expressed in millidegrees Celsius, corresponding to approximately **41.9 °C**.

## 7. Summary

This tutorial built:

- A 64-bit ARM Linux system
- Remote SSH access
- A conventional Linux kernel
- Command-line access to system and hardware information
- GPIO interfaces available for subsequent experiments

The system has no GUI and is operated entirely from a remote Linux workstation.

A particularly interesting detail is the kernel's `PREEMPT` designation. Does that mean
the Raspberry Pi is running a real-time Linux kernel?

**Not necessarily.**

In the next article, we inspect the kernel configuration and distinguish ordinary kernel
preemption from the `PREEMPT_RT` real-time kernel.
