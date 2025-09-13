+++
title = "How to install Home Assistant OS on an Incus VM"
template = "post.html"
date = 2025-09-13
path = "/how-to-install-home-assistant-os-on-an-incus-vm"
+++

Install Home Assistant OS on an Incus VM via the image import command.
Includes how to passthrough the Conbee II ZigBee USB stick to the VM.

<!-- toc -->

## Download HAOS image and create metadata
Download the `ova qcow2` asset from (the Home Assistant GitHub page)[https://github.com/home-assistant/operating-system/releases] 
from the release you wish. For example `haos_ova-16.2.qcow2.xz`.

Uncompress it with `unxs` (on NixOS this is part of the `xs` package):

```bash
unxs haos_ova-16.2.qcow2.xz
```

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

## Import/convert the image into Incus
```bash
incus image import 
```
