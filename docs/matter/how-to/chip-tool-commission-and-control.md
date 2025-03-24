# How to commission and control Matter devices with Chip Tool

Chip Tool is an open source Matter Controller with a command-line
interface (CLI). It is useful for development and testing of Matter
devices from a Linux machine.

Chip Tool offers a wide range of capabilities, ranging from device
commissioning and control to setup and operational payload generation
and parsing.

This document guides you through setting up and configuring Chip Tool
using a Snap. This makes it extremely easy to securely run and use the
tool on Linux.

## Install

First and foremost, make sure to have SnapD installed. It is
pre-installed on some distributions such as Ubuntu. Refer to
[Installing SnapD] for details.

Install the dependencies:

````{tabs}
```{group-tab} Ubuntu Server / Desktop

```bash
    sudo apt update
    sudo apt install bluez avahi-daemon

```
```{group-tab} Ubuntu Core

```bash
    sudo snap install bluez avahi

```
````
Install the Chip Tool snap:

```shell
    sudo snap install chip-tool
```

```{tip}
Pre-release versions of Chip Tool are available in different [channels]
```







<!-- Links -->
[installing SnapD]: https://snapcraft.io/docs/installing-snapd
[channels]: https://snapcraft.io/docs/channels