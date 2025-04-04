# Deploy Docker containers on Ubuntu Core using the Ansible snap module

Ansible is an open-source automation engine used to automate processes such as provisioning, continuous software deployment, and configuration management. Ansible uses YAML scripts called playbooks to automate tasks and ensures systems remain in the state declared in the playbook. It is compatible with many systems including Ubuntu Core.

This guide shows you how to deploy docker containers on Ubuntu Core using the Ansible Snap module. For demonstration purposes, we will be deploying an instance of RabbitMQ on a Raspberry Pi. The same playbook will also deploy an instance of RabbitMQ on another Raspberry Pi running Ubuntu Server to showcase the ease with which you can shift containerized runtimes from Ubuntu Server to Ubuntu Core for added security.

Note: The methods demonstrated here should also work on other distributions that support [Snapd](https://snapcraft.io/docs/installing-snapd).

## Installation

To get started, you’ll need to install Ansible on the control node. This is the machine you’ll be using to run Ansible CLI tools. You can do this by executing:

```shell
$ sudo apt install ansible
```

For additional information and alternative installation options, check out the full [Ansible installation guide](https://docs.ansible.com/ansible/latest/installation_guide/index.html).

## Create the inventory file

The inventory file defines the Ansible managed nodes (or hosts) that Docker containers will be deployed to. To create this file, you’ll need each host’s username and IP address. 

You must be able to SSH into any nodes in your inventory file. To confirm this, run:

```shell
$ ssh {username}@{ip address}
```

To create the inventory file, create a new file called ‘inventory.yaml’ in your preferred directory, and enter each host machine’s details as shown:

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
      ansible_user: ubuntu
      ansible_python_interpreter: /usr/bin/python3
```

`machines` is the group name for the hosts. You can define different groups for hosts so each group executes different sets of tasks from the same playbook. 

For nodes running Ubuntu Core, you must specify the `ansible_python_interpreter` to avoid getting a warning message about future python interpreters. 

Executing some tasks in the playbook will require Sudo privileges. You can enable passwordless sudo access or  provide the user password for each host by setting the `ansible_password` field.

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

Ansible must perform a series of actions, i.e. tasks, to deploy a docker container to a node. These tasks are written in YAML format in the order they should be executed and saved in a file called a playbook. 

To deploy the RabbitMQ containers to the two nodes:

### Create the playbook and ping the hosts

Create the playbook.yaml file in the same directory as the inventory. Inside the playbook, write the name of the playbook and specify the targeted host group. As the first task, ping the hosts using ansible’s builtin ping module. This will confirm which hosts are available:

```yaml
name: Deploy RabbitMQ
  hosts: machines
  tasks:
  
  - name: Ping my hosts
    ansible.builtin.ping:
```

### Install the Docker snap

Installing the Docker snap in the nodes is the second task in the playbook. The field `become` must be set to `true` because sudo privileges are required for this and subsequent tasks. 

```yaml
  - name: Install Docker snap
    become: true
    community.general.snap:
      name:
        - docker
      channel: latest/stable
      state: present
```

### Pull a RabbitMQ image

This task downloads the RabbitMQ image that will be deployed. You’ll need the name of the specific version of RabbitMQ you want to deploy as shown:

```yaml
- name: Pull an image
  become: true
  community.docker.docker_image_pull:
    name: rabbitmq:4.0.2-management
```

### Configure RabbitMQ

This task sets the default username and password for the RabbitMQ instances using Ansible’s builtin shell:

```yaml
  - name: Setup RabbitMQ configurations
    become: true
    ansible.builtin.shell: |
      sudo echo "RABBITMQ_DEFAULT_USER=demo-user" > $HOME/rabbitmq_conf.env
      sudo echo "RABBITMQ_DEFAULT_PASS=SomeReallyStrongPassword" >> $HOME/rabbitmq_conf.env

```

### Start the RabbitMQ container

This is the last task in the playbook. It launches the instances of RabbitMQ in the managed nodes:

```yaml
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

Save the playbook and execute it by running:

```shell
$ ansible-playbook -i inventory.yaml playbook.yaml
```

If successfully deployed, it should return:

```
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

Ansible does not display an output by default. To find out the status of the containers at any time, you’ll need to run the following command:

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