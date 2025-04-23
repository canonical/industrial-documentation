# Containers

Ubuntu Core has been developed specifically to leverage the power of snap packages, with their autonomous upgrades and confined sandbox.

However, Ubuntu Core can also be used to run and deploy Docker container images, either from the command line or embedded within an Ubuntu Core image.

## Deploying Docker

Install and run Docker from within the Ubuntu Core environment. Build a snap which embeds the container images you wish to deploy from a device.

- {ref}`Deploy Docker containers from a snap <how-to-guides-container-deployment-deploy-docker-from-a-snap>`
- {ref}`Understand the Docker companion snap <explanation-docker-companion-snap>`


## Packaging Docker

How Docker images can be bundled inside snaps to be deployed together with other resources within the package.

Ship containerized applications with Ubuntu Core by building a custom image that can then be used to create bootable media and can automate deployment of a production system.


- {ref}`Package Docker images in a snap <how-to-guides-container-package-docker-images-in-a-snap>`
- {ref}`Build an Ubuntu Core image for Docker deployment <containers-how-to-build-uc-image-containing docker>`


```{toctree}
:hidden:
:titlesonly:
:maxdepth: 2
:glob:

Deploy Docker Container from a snap <how-to/deploy-docker-container-from-a-snap>
Package Docker images in a snap <how-to/package-docker-images-in-a-snap>
Build an Ubuntu Core image containing Docker <how-to/build-ubuntu-core-image-containing-docker>
Docker companion snap <explanation/docker-companion-snap>