+++
title = "How to install Home Assistant OS on an Incus VM"
template = "post.html"
date = 2025-09-14
path = "/how-to-install-home-assistant-os-on-an-incus-vm"
[taxonomies]
tags = ["Home Assistant", "Incus", "Virtualization"]
+++

Install Home Assistant OS on an Incus VM via the image import command.
Includes how to passthrough the ConBee II ZigBee USB stick to the VM.

<!-- toc -->

## Download HAOS image
Download the `ova qcow2` asset from [the Home Assistant GitHub page](https://github.com/home-assistant/operating-system/releases) 
from the release you wish. For example `haos_ova-16.2.qcow2.xz`.

Uncompress it with `unxs` (on NixOS this is part of the `xs` package):

```bash
unxs haos_ova-16.2.qcow2.xz
```

## Create image metadata
Create a metadata file to go with the image. I've used Debian 13.1 here.

```yml
architecture: x86_64
creation_date: 1757437487
properties:
  description: Home Assistant image
  os: Debian
  release: trixie 13.1
```

Now compress it.

```bash
tar -xzfv metadata.yml
```

## Import the image into Incus
```bash
incus image import metadata.tar.gz haos_ova-16.2.qcow2 --alias haos
```

## Create a VM instance
The following command uses the imported image to create a new VM instance with secure boot off and a disk size of 50gb.
The `bridged` profile here refers to a custom brigded network config that exposes the server on the network. Read more [here](@/posts/how-to-install-incus-lxd-on-nixos.md).

```bash
incus launch haos homeassistant --vm -c security.secureboot=false -d root,size=50GiB --profile bridged
incus stop homeassistant -f
incus config set homeassistant limits.cpu=2 limits.memory=6GiB
incus start homeassistant
incus console --show-log homeassistant
```

When the logs show "Welcome to Home Assistant" you can browse the VM's IP on the 8123 port.

Note that you can't `shell` into the VM. You can however install the Home Assistant SSH addon and get in that way.

## Enable passthrough of USB or PCI devices
In my case I have a ConBee II ZigBee USB stick that I want Home Assistant to know about. 

In most cases a USB device can be forwarded to an instance like this: 

```bash
# usb device ids can be read with lsusb (usbutils on NixOS)
incus config device add homeassistant conbee usb vendorid=xxxx productid=yyyy
```

However the ConBee II requires us to forward the entire USB controller that it's attached to:

```bash
# pci device address can be read with lspci (pciutils on NixOS)
incus config device homeassistant conbee pci address=xx:yy.z
```

You need to stop and start the instance to passthrough a PCI device.
