# Build a custom Ubuntu Core image for Lora Basics™ station

Although you can install Lora Basics station on existing Ubuntu Core devices, you can also create a custom Ubuntu Core image that includes LoRa Basics station software and add fleet management using Canonical's Landscape tool. In this guide, we will show you how to build this image and flash it onto a device. 

Before you continue, you should know how to [building an Ubuntu Core image](https://ubuntu.com/core/docs/build-an-image).

```{note}

These instructions are written for, and tested with, a Raspberry Pi 5. With some modification they should also work on other platforms.

```

## Build the gadget snap

The gadget snap defines and configures system properties that are specific to one or more devices. A custom version of this snap must be included as part of the Ubuntu Core image to setup the bootloader and other components. <!-- Further clarification needed -->

Clone the [LoRa Basics stations Ubuntu Core image repository](https://github.com/canonical/lora-basicstation-snap/tree/main/ubuntu-core-image) and replace the original pi-gadget directory with this [version of the Raspberry Pi gadget snap](https://github.com/canonical/pi-gadget/tree/24).

Inside the `pi-gadget` directory, edit the file `gadget.yaml` by adding the content of `additions-to-gadget.yaml` (shown below) at the end.

```yaml

defaults:
  # Preconfigure Landscape Client
  ffnH0sJpX3NFAclH777M8BdXIWpo93af:
    landscape-url: https://landscape.canonical.com
    account-name: "<REDACTED>"
    registration-key: ""
    auto-register:
      enabled: true
      computer-title-pattern: ${mac}
      wait-for-serial-as: false

  system:
    # Disable console-conf so that we do not have an interactive installer at first bootup
    service:
      console-conf:
        disable: true

    # Add this if you want to preconfigure wifi credentials
    system:
      network:
        netplan:
          network:
            version: 2
            wifis:
              wlan0:
                dhcp4: true
                access-points:
                  "my-wifi-ssid":
                    password: "my-wifi-password"
```

You will also need to update the additions with your Landscape account name, and perhaps the Landscape server URL.
If you also want to preconfigure WiFi, update the WiFi SSID and password.

Then, to build the Gadget snap, run:

```shell
cd pi-gadget
snapcraft -v
cd ..
```

## Set up the model

The example `model.json` file in the `ubuntu-core-image` directory includes `landscape-client`, `lora-basicstation` and `docker`.

```json
{
    "type": "model",
    "series": "16",
    "model": "uc24-pi-arm64-lora",
    "architecture": "arm64",
    "authority-id": "<REDACTED>",
    "brand-id": "<REDACTED>",
    "timestamp": "2024-08-20T12:20:00+00:00",
    "base": "core24",
    "grade": "dangerous",
    "snaps": [
        {
            "name": "pi",
            "type": "gadget"
        },
        {
            "name": "pi-kernel",
            "type": "kernel",
            "default-channel": "24/stable",
            "id": "jeIuP6tfFrvAdic8DMWqHmoaoukAPNbJ"
        },
        {
            "name": "core24",
            "type": "base",
            "default-channel": "latest/stable",
            "id": "dwTAh7MZZ01zyriOZErqd1JynQLiOGvM"
        },
        {
            "name": "core22",
            "type": "base",
            "default-channel": "latest/stable",
            "id": "amcUKQILKXHHTlmSa7NMdnXSx02dNeeT"
        },
        {
            "name": "snapd",
            "type": "snapd",
            "default-channel": "latest/stable",
            "id": "PMrrV4ml8uWuEUDBT8dSGnKUYbevVhc4"
        },
        {
            "name": "docker",
            "type": "app",
            "default-channel": "latest/stable",
            "id": "sLCsFAO8PKM5Z0fAKNszUOX0YASjQfeZ"
        },
        {
            "name": "lora-basicstation",
            "type": "app",
            "default-channel": "latest/edge",
            "id": "Ym43FmNEiZgqaAciYg6EMqRC4dUjcYYb"
        },
        {
            "name": "landscape-client",
            "type": "app",
            "default-channel": "latest/beta",
            "id": "ffnH0sJpX3NFAclH777M8BdXIWpo93af"
        }
    ]
}

```

All you need to do is to update the `authority-id` and `brand-id` fields, and then sign the model:

```
snap sign -k my-model-key model.json > model.model
```

## Build the image

After the model is signed, build an image from it by executing:

```shell
$ ubuntu-image snap --snap pi-gadget/pi_24-1_arm64.snap --validation=enforce model.model
```
The path to the custom gadget snap you built earlier is set because it must be side-loaded into the image during the build process.

This should return:

```
[0] prepare_image
WARNING: the kernel for the specified UC20+ model does not carry assertion max formats information, assuming possibly incorrectly the kernel revision can use the same formats as snapd
WARNING: "pi" installed from local snaps disconnected from a store cannot be refreshed subsequently!
[1] load_gadget_yaml
[2] set_artifact_names
[3] populate_rootfs_contents
[4] generate_disk_info
[5] calculate_rootfs_size
[6] populate_bootfs_contents
[7] populate_prepare_partitions
[8] make_disk
[9] generate_snap_manifest
Build successful
```

During the build process you will see two warnings.

- The warning about the kernel is discussed [here](https://forum.snapcraft.io/t/ubuntu-image-warning-kernel-snap/37774/3).
- The warning about the `pi` snap is because we side-loaded a custom locally built snap. This snap is not provided via the Snap Store and will therefore not get updates. To update custom Gadget snaps like these you will need a [dedicated Snap Store](https://ubuntu.com/core/docs/dedicated-snap-stores).

If the build was successful you will find a file called `pi.img` in the working directory.

## Using the image

You can flash this image file to an SD card using Raspberry Pi Imager.
After flashing, insert the SD card into a Raspberry Pi and power it on.
If you have a monitor connected, you will see a login prompt after a couple of minutes.
It is not possible to log in here as there is no user configured.

At this point the Pi should have registered with Landscape.

Follow the the guide [here](https://github.com/canonical/lora-basicstation-snap/blob/main/landscape/README.md) to control your gateway via Landscape.