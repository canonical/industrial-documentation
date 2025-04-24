# Containers

<!--
Containers intro
-->

Ubuntu Core has been developed specifically to leverage the power of snap packages, with their autonomous upgrades and confined sandbox. However, Ubuntu Core can also be used to run Docker containers.

There many ways to deploy Docker containers on Ubuntu Core. 
For basic use cases, a container can be started from the [command line](https://documentation.ubuntu.com/core/how-to-guides/container-deployment/run-a-docker-container/).

## Deploying Docker via Snaps

<!-- intro -->

- {doc}`Deploy Docker Containers from a snap <how-to/deploy-docker-container-from-a-snap>`
- {doc}`Package Docker images in a snap <how-to/package-docker-images-in-a-snap>`
- {doc}`Build an Ubuntu Core image containing Docker <how-to/build-ubuntu-core-image-containing-docker>`
- {doc}`Docker companion snap <explanation/docker-companion-snap>`


## Deploying Docker via Ansible

<!-- intro -->

- {doc}`Deploy Docker containers on Ubuntu Core using Ansible <how-to/deploy-docker-containers-uc-ansible>`


```{toctree}
:hidden:
:titlesonly:
:maxdepth: 2

how-to/deploy-docker-container-from-a-snap
how-to/package-docker-images-in-a-snap
how-to/build-ubuntu-core-image-containing-docker
explanation/docker-companion-snap
how-to/deploy-docker-containers-uc-ansible
```
