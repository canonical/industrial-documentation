# Deploy Docker containers on Ubuntu Core using the Ansible snap module

Ansible is an open-source automation engine used to automate processes such as provisioning, continuous software deployment, and configuration management. Ansible uses YAML scripts called playbooks to automate tasks and ensures systems remain in the state declared in the playbook. It is compatible with many systems including Ubuntu Core.

This guide shows you how to deploy docker containers on Ubuntu Core using the Ansible Snap module. For demonstration purposes, we will be deploying an instance of RabbitMQ on an Ubuntu Core compatible Raspberry Pi. The same playbook will also deploy an instance of RabbitMQ on another Raspberry Pi running Ubuntu Server to showcase the ease with which you can shift containerized runtimes from Ubuntu Server to Ubuntu Core for added security.

Note: The methods demonstrated here should also work on other distributions that support [Snapd](https://snapcraft.io/docs/installing-snapd).

## Installation

To get started, you will need to install an Ubuntu Core 24 image on the first Raspberry Pi. Instructions on how to do this are available [here](https://ubuntu.com/core/docs/get-started). You can install a pre-built Ubuntu Core image or build your own image and install it.  

You’ll also need to install Ubuntu Server 24.04.1 LTS on the second Raspberry Pi. Instructions for this are available [here](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview). 

Executing some tasks in the playbook requires Sudo privileges. To prevent errors when running it, be sure to enable passwordless access on both installations.

````{Note}

Make sure you can SSH into both devices before you continue. You can test this by running the following command:

```shell
ssh {username}@{ip address}
```
````

## Create the inventory file

The inventory file defines the collection of machines, i.e. hosts, that Docker container(s) will be deployed to. You’ll need the username and IP address of each host.   
Create a new file called ‘inventory.yaml’, and enter each host machine’s details as shown.

```yaml
machines:
  hosts:
 
    # 24.04.1 LTS (Noble Numbat) - Rpi4
    rpi4:
      ansible_host: 192.168.0.57
      ansible_user: ubuntu

    # UC24 - RPi5
    rpi5:
      ansible_host: 192.168.0.58
      ansible_user: locnnil
      ansible_python_interpreter: /usr/bin/python3
```

In the code above, ‘machines’ is the group name for the hosts. You can define different groups for hosts so each group executes different sets of tasks from the same playbook. A host can be placed in multiple groups in the same inventory. 

## Confirm hosts are reachable

It’s good practice to confirm the inventory.yaml file is properly set up and the hosts can be reached before starting on the playbook. Enter the following command to ping each host in the machines group in the inventory. 

```shell
$ ansible machines -i inventory.yaml -m ping
```

If successful, you should get an output similar to this:

```
rpi5 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
rpi4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

If the hosts are not reachable, ensure that your public SSH key is added to the .ssh/authorized\_keys file on each host.

## Prepare the playbook

Ansible uses the playbook to deploy and configure the nodes in the inventory file. The playbook is a list of tasks in YAML format written in the order they will be executed. 

To successfully deploy the RabbitMQ containers to the two nodes using the Ansible Snap Module, the playbook will need to go through the following tasks in the same sequence:

* Pinging the hosts to confirm availability  
* Installing the Docker snap  
* Pulling a suitable RabbitMQ image  
* Configuring RabbitMQ  
* Starting the RabbitMQ container

Therefore, the complete playbook will be as shown:

```yaml
name: Deploy RabbitMQ
  hosts: machines
  tasks:
  
  - name: Ping my hosts
    ansible.builtin.ping:

  - name: Install Docker snap
    become: true
    community.general.snap:
      name:
        - docker
      channel: latest/stable
      state: present

  - name: Pull an Image
    become: true
    community.docker.docker_image_pull:
	    name: rabbitmq:4.0.2-management

  - name: Setup RabbitMQ configurations
    become: true
    ansible.builtin.shell: |
      sudo echo "RABBITMQ_DEFAULT_USER=demo-user" > $HOME/rabbitmq_conf.env
      sudo echo "RABBITMQ_DEFAULT_PASS=SomeReallyStrongPassword" >> $HOME/rabbitmq_conf.env

  - name: Start RabbitMQ container
    become: true
    community.docker.docker_container:
      image: rabbitmq:4.0.2-management
      name: message_broker
      hostname: docker_ansible_demo
      env_file: $HOME/rabbitmq_conf.env
      volumes: /var/snap/docker/current/data:/var/lib/rabbitmq
      ports:
          # Publish container port 5672 as host port 17338
          - 17338:5672
          # Publish container port 15672 as host port 18280
          - 18280:15672
```

## Run the playbook

Type the following command to run the playbook:

```shell
$ ansible-playbook -i inventory.yaml playbook.yaml
```

When successfully deployed, this should yield the following output:

```shell
$ ansible-playbook -i inventory.yaml playbook.yaml

PLAY [Deploy Docker on Ubuntu Core with Ansible] ***************************

TASK [Gathering Facts] *****************************************************
ok: [rpi5]
ok: [rpi4]

TASK [Ping my hosts] *******************************************************
ok: [rpi5]
ok: [rpi4]

TASK [Install Docker snap] *************************************************
ok: [rpi5]
ok: [rpi4]

TASK [Pull an image] *******************************************************
ok: [rpi5]
ok: [rpi4]

TASK [Setup RabbitMQ configurations] ***************************************
changed: [rpi5]
changed: [rpi4]

TASK [Start RabbitMQ container] ********************************************
ok: [rpi5]
ok: [rpi4]

PLAY RECAP *****************************************************************
rpi4                       : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
rpi5                       : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

Note: Ansible does not display an output by default. To find out the status of the containers at any time, you’ll need to run the following command:

```shell
$ ansible machines -i inventory.yaml -m shell -a "sudo docker container ls"
```

The output will be:

```
rpi5 | CHANGED | rc=0 >>
CONTAINER ID   IMAGE                       COMMAND                  CREATED       STATUS       PORTS                                                                                                          NAMES
a427d99efd75   rabbitmq:4.0.2-management   "docker-entrypoint.s…"   3 hours ago   Up 3 hours   4369/tcp, 5671/tcp, 15671/tcp, 15691-15692/tcp, 25672/tcp, 0.0.0.0:17338->5672/tcp, 0.0.0.0:18280->15672/tcp   message_broker
rpi4 | CHANGED | rc=0 >>
CONTAINER ID   IMAGE                       COMMAND                  CREATED       STATUS       PORTS                                                                                                          NAMES
74f18efdd315   rabbitmq:4.0.2-management   "docker-entrypoint.s…"   3 hours ago   Up 3 hours   4369/tcp, 5671/tcp, 15671/tcp, 15691-15692/tcp, 25672/tcp, 0.0.0.0:17338->5672/tcp, 0.0.0.0:18280->15672/tcp   message_broker

```

## Next steps

In this guide, all the devices are assigned the same password. In practice, this does not align with the best practices for maintaining system security. Consider dividing your hosts into groups and creating different login credentials for each. 

You can also take advantage of [Patterns](https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html) to run your playbook or commands against specific groups or hosts. Patterns can be used to refer to a group, set of groups, an IP address, or all the hosts in your inventory.   

Although values such as `env_file` and a specific RabbitMQ image are indicated in the tasks, this does not scale well when you have a large number of hosts. Ideally, these values should be replaced with variables for more efficient management.




















<!-- Links -->

[Snapd]: https://snapcraft.io/docs/installing-snapd