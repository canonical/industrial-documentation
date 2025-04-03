# How to set up LoRa Basics™ Station on Ubuntu Core
The LoRa Basics Station snap is a companion snap for LoRa Basics™ Station for Docker. It allows you to run and configure a LoRa gateway on Ubuntu Core devices. This guide will take you through the steps of setting up, running, and debugging the snap.

## Install the snap
To install the snap from the Snap Store, run:

```shell
sudo snap install lora-basicstation
```

This snap needs to interact with the Docker Engine on the host. To allow that, we connect the required docker interfaces:

```shell
sudo snap connect lora-basicstation:docker docker:docker-daemon
sudo snap connect lora-basicstation:docker-executables docker:docker-executables
```

## Configuration
Configuration of the gateway is done similar to the Docker container, using the same config options as listed in the [upstream documentation]. The only difference is that we use snap config options rather than environment variables. It would be possible to create an environment variable file and pass that to Docker, but we prefer to use the standard snap configuration options. This allows us to leverage the snap ecosystem to maintain and update our Ubuntu Core devices.

To translate from environment variables to snap options, we only need to change the name or key of the configuration option. The values remain the same. For the variable name, change it to lower case and replace underscores with hyphens. Then prefix it with `env..`

Example: `GATEWAY_EUI=DEADFFFEBEEF` becomes `env.gateway-eui=DEADFFFEBEEF`

After you have the configs in this format, you can set them one by one using the `snap set` command:

```shell
sudo snap set lora-basicstation env.tc-uri="wss://eu1.cloud.thethings.network:8887"
sudo snap set lora-basicstation env.tc-key=NNSXS...
```

Or with a single command:

```shell
sudo snap set lora-basicstation \
    env.gateway-eui=DEADFFFEBEEF \
    env.model=SX1301 \
    env.tc-key=NNSXS... \
    env.tc-uri=wss://eu1.cloud.thethings.network:8887 \
    env.device=/dev/spidev0.0 \
    env.interface=SPI \
    env.spi-speed=200000 \
    env.use-libgpiod=1 \
    env.gpio-chip=gpiochip4 \
    env.reset-gpio=22
```

An option can be cleared with the `snap unset` command, e.g.:

```shell
sudo snap unset lora-basicstation env.gateway-eui
```

You can also verify the options with `snap get`. For example directly after installation you will see a number of defaults that are already set:

```shell
$ sudo snap get lora-basicstation env
Key            Value
env.device     /dev/spidev0.0
env.gpio-chip  gpiochip0
```

These configs are required to forward the correct devices to the Docker container. If you are using a Raspberry Pi 5 you will need to change the `env.gpio-chip` value to `gpiochip4`, otherwise the Docker container won't be able to reset the LoRa concentrator.

## Running the snap
Start the snap and enable the daemon to autostart in the future:

```shell
sudo snap start --enable lora-basicstation
```
The gateway software can be stopped using:

```shell
sudo snap stop lora-basicstation
```
To view the log output, run:

```shell
sudo snap logs -f lora-basicstation
```
Removing the snap is done with this command, which will also remove the Docker container and image:

```shell
sudo snap remove lora-basicstation
```
## Debugging
### Verify environment
The snap configs are translated to an environment file when this snap is started. This environment file is passed to the Docker container when it is created.

You can verify the contents of the environment file with this command:

```shell
cat /var/snap/lora-basicstation/common/conf.env
```
### Verify that the Docker container is running
To make sure the Docker container is running, execute the following command.
If it does not list any containers with the name `lora-basicstation`, the container is not running.

```
sudo docker ps
```

### Find the Gateway EUI

This snap provides a tool to generate an EUI from the device MAC address.
Refer to the [upstream documentation]) for details on its usage.

```shell
$ lora-basicstation.gateway-eui
Gateway EUI: 2CCF67FFFE1CBBA1 (based on interface eth0)
```

If the `env.gateway-eui` snap option is set, this tool will output the following:

```shell
$ lora-basicstation.gateway-eui
Gateway EUI: 0242ACFFFE110002 (based on environment variable)
```

### Find attached concentrators

A script to scan for connected LoRa concentrators is included.
Please see the [upstream documentation](https://github.com/xoseperez/basicstation-docker?tab=readme-ov-file#find-the-concentrator) for its limitation.

```
$ lora-basicstation.find-concentrator

Looking for devices, this might take some time...

DEVICE             DESIGN      ID
---------------------------------------------------

0 device(s) found!
```







<!-- Links -->
[upstream documentation]: https://github.com/xoseperez/basicstation-docker?tab=readme-ov-file#configure-the-gateway