# Manage a fleet of LoRa  gateways on Ubuntu Core with Landscape
[Landscape](https://ubuntu.com/landscape) is a Canonical service for fleet management of devices running ubuntu. 

Although you can install LoRa Basics station on existing Ubuntu Core devices, you can also create a custom Ubuntu Core image that includes LoRa Basics station software, deploy the image to multiple devices and use Landscape to manage your new fleet. 

In this guide, we will show you how to build the Ubuntu Core image containing a LoRa gateway, flash it onto a device(s), and configure it using Landscape. These instructions are written for, and tested with, a Raspberry Pi 5. However, with some modification, they should also work on other platforms.

```{note}
Before you continue, you should know how to [build an Ubuntu Core image](https://ubuntu.com/core/docs/build-an-image).
You also require a Landscape account on a self-hosted or SAAS Landscape server.
```

Create a working directory for the configuration of the image.

```bash
mkdir lora-ubuntu-core-image
cd lora-ubuntu-core-image
```

## Build the gadget snap

The gadget snap defines and configures system properties that are specific to one or more devices. A custom version of this snap must be included as part of the Ubuntu Core image to set up the bootloader and other components. <!-- Further clarification needed -->

To build the gadget snap, start by cloning the [Raspberry Pi gadget snap](https://github.com/canonical/pi-gadget/tree/24) repository into the working directory.

```bash
git clone https://github.com/canonical/pi-gadget --branch 24
```

The repository will be cloned into a new `pi-gadget` subdirectory.
Inside this directory, edit the file `gadget.yaml`, and append this code:

```yaml

defaults:
  # Preconfigure Landscape Client
  ffnH0sJpX3NFAclH777M8BdXIWpo93af:
    landscape-url: https://landscape.canonical.com
    account-name: "REPLACE-ME"
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

You will also need to update the Landscape account name and the Landscape server URL to match your server and account.
If you also want to preconfigure WiFi, update the WiFi SSID and password.

Finally, build the gadget snap by running:

```shell
cd pi-gadget
snapcraft -v
cd ..
```

## Set up the model

Create a new model assertion by creating an empty file in the working directory called `model.json`, and then adding the code below.

```json
{
    "type": "model",
    "series": "16",
    "model": "uc24-pi-arm64-lora",
    "architecture": "arm64",
    "authority-id": "REPLACE-ME",
    "brand-id": "REPLACE-ME",
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

This model assertion includes the `landscape-client`, `lora-basicstation`, and `docker` snaps.

Update the `authority-id` and `brand-id` fields with your [developer id](https://ubuntu.com/core/docs/create-ubuntu-one).
Next update the `timestamp` field to the current time, making sure it falls withing in the validity period of your snap signing key.
Finally execute this command to sign the model:

```
snap sign -k my-model-key model.json > model.model
```

## Build the image

To build an image you require the `ubuntu-image` application. Install it:

```bash
sudo snap install ubuntu-image --classic
```

You can now build a disk image from the signed model:

```shell
ubuntu-image snap --snap pi-gadget/pi_24-1_arm64.snap --validation=enforce model.model
```
The path to the custom gadget snap built earlier is set because it must be side-loaded into the image during the build process.

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

If the build was successful you will find a file called `pi.img` in the working directory.

```{note}
During the build process you will see two warnings.

- The warning about the kernel is discussed [here](https://forum.snapcraft.io/t/ubuntu-image-warning-kernel-snap/37774/3).
- The warning about the `pi` snap is because we side-loaded a custom locally built snap. This snap is not provided via the Snap Store and will therefore not get updates. To update custom gadget snaps like these you will need a [dedicated Snap Store](https://ubuntu.com/core/docs/dedicated-snap-stores).
```
## Flash the image to a device

You can flash this image file to an SD card using Raspberry Pi Imager.
After flashing, insert the SD card into a Raspberry Pi and power it on.
If you have a monitor connected, you will see a login prompt after a couple of minutes.
It is not possible to log in here as there is no user configured.

## Locate the device on Landscape dashboard
After your Raspberry Pi has started up, it should automatically register with Landscape. 
It will then appear on your Landscape dashboard as a pending device.
In the gadget snap, `landscape-client` is configured to use the device mac address as name, so that is what you should see listed here.

![pending device](manage-fleet-uc-gateways-landscape/dashboard-pending-device.png)

Click on this name to configure the device.
You can also customize the device title here.
Click accept to add the device to your Landscape instance.

![accept device](manage-fleet-uc-gateways-landscape/accept-device.png)

After accepting you will see the info page for the device.

You'll need to wait for several minutes for the device to check in. Once this is done, the info page will show some basic parameters of the device, like total memory, model and MAC addresses.

![computer info](manage-fleet-uc-gateways-landscape/computer-info.png)

Devices are called computers on Landscape.
To get back to this page in the future, go to the computers menu and select the device you are interested in from the list.

## Verify installed snaps

To see which snaps are installed on a device, click on the Snaps tab and then click on **Installed snaps**.
You will see the list of installed snaps. 
You can update or remove a Snap from here.
Verify that `lora-basicstation` is in the list.

To install more software on your device, you can click on **Install snaps** under the Snaps tab.

## Configure the gateway

Configuration of an Ubuntu Core device with Landscape is done using scripts.
To configure your gateway, you'll need to set snap options for `lora-basicstation` via a script.
An example script is provided [here](manage-fleet-uc-gateways-landscape/configure-lora-basicstation.sh).

On Landscape, after selecting your device from the list of computers, go to the scripts tab.
If presented with a choice, select **Run a new script** and then click **Next**.
You will now see a form to create a new script.
Give it a name, e.g., *Configure Basicstation*.

In the code block paste the contents of the example script above.
You can modify the configuration in the script to suite your needs.
The configuration follows the guide [here]`(../companion-snap-compose/README.md#configuration).`

Enter `root` as the user the script should run as.
Save the script and click **Run**.

![create scripts](manage-fleet-uc-gateways-landscape/create-script.png)

You will be redirected to the Activities tab where you will see the progress of the script.
After a couple of minutes you should see a success.

![activity](manage-fleet-uc-gateways-landscape/activity-succeeded.png)

The text in the status column is clickable and will show the output of the script. `stdout` on success and `stderr` on failure. 

![stdout](manage-fleet-uc-gateways-landscape/activity-stdout.png)

Log into your LoRa network server and check if your gateway is connected.

![LNS](manage-fleet-uc-gateways-landscape/lns-gateway-status.png)

## Debug

Debugging can be done by running scripts on the device.
Checkout this page for [example scripts for Ubuntu Core devices on Landscape](https://github.com/canonical/landscape-scripts/tree/main/core).

Another debugging option is to create a user account on the device, then log into the device over SSH or locally with a keyboard and mouse.
The user management interface of Landscape does not currently work for Ubuntu Core.
Follow this link to see [example scripts to add users](https://github.com/canonical/landscape-scripts/tree/main/core/users).

For instance, the `add-sso-user.py` script will add a user with an SSH key obtained from a Launchpad account.