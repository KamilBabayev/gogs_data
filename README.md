# MAP EDI solution for AT&T Mobility

<!-- TOC -->

- [MAP EDI solution for AT&T Mobility](#map-edi-solution-for-att-mobility)
- [Design and Architecture](#design-and-architecture)
    - [High-level design](#high-level-design)
    - [Networking requirements](#networking-requirements)
        - [Inter-node Communications](#inter-node-communications)
        - [Networking Security](#networking-security)
        - [Swarm-mode Routing Mesh](#swarm-mode-routing-mesh)
        - [Reverse Proxy for web services](#reverse-proxy-for-web-services)
- [Architecture](#architecture)
    - [Core JEDI service stack](#core-jedi-service-stack)
    - [VCS stack](#vcs-stack)
    - [AT&T stack](#att-stack)
    - [Reverse proxy](#reverse-proxy)
- [Deployment](#deployment)
    - [VM preparation](#vm-preparation)
        - [Prerequsites](#prerequsites)
        - [Roles](#roles)
            - [glusternode](#glusternode)
            - [dockernode](#dockernode)
        - [Playbooks](#playbooks)
        - [Inventory](#inventory)
        - [Run the playbook](#run-the-playbook)
    - [All-in-one deployment playbook](#all-in-one-deployment-playbook)
    - [Deploy the core JEDI service stack](#deploy-the-core-jedi-service-stack)
        - [Prerequsites](#prerequsites-1)
        - [Playbook](#playbook)
        - [Inventory](#inventory-1)
        - [Run the playbook](#run-the-playbook-1)
    - [Deploy the VCS stack](#deploy-the-vcs-stack)
        - [Prerequisites](#prerequisites)
        - [Playbook](#playbook-1)
        - [Inventory](#inventory-2)
        - [Run the playbook](#run-the-playbook-2)
    - [Deploy the Proxy stack](#deploy-the-proxy-stack)
        - [Prerequisites](#prerequisites-1)
        - [Playbook](#playbook-2)
        - [Inventory](#inventory-3)
        - [Run the playbook](#run-the-playbook-3)

<!-- /TOC -->

# Design and Architecture

## High-level design

At a very high level, this solution is designed to be scalable and platform agnostic. While it starts as 4 nodes, it could just as easily be 2 nodes or 20. Further, the solution is not tied to any specific environment or container orchestration. The solution, as developed here, is designed to start with 4 vanilla Ubuntu 16.04 virtual machines (VMs) running in Openstack, but with minor changes to the deploy playbooks, can be made to run on any Docker-enabled platform.

Based on customer requirements and infrastructure, Docker swarm-mode on 4 nodes was the chosen infrastructure. The 4 nodes will run Ubuntu 16.04 and we start with the assumption that these nodes are vanilla and any required packages and dependencies will need to be installed and configured.

## Networking requirements

### Inter-node Communications

Between the Docker hosts (nodes), orchestration communications will occur across 3 ports: TCP 2377, UDP 4789, and TCP/UDP 7946. This is shown in Figure 1.

![swarm-networking](docs/images/swarm-networking.png "Swarm Networking")

_Figure 1_


### Networking Security

Communications between the nodes are encrypted using AES. This includes daemon to daemon communications as well as container to container communications. Container networking occurs across a user-defined network using the overlay driver. This user-defined network enables all containers to communicate with one another on any exposed port without publishing those ports to the host. This provides an additional level of security by not exposing service ports to the host (and therefore the external network). Finally, all web interface communications will occur via a reverse proxy, limiting the exposure to 2 ports (80/443) from 5. In all, only 4 ports are exposed to the external network, as seen in Figure 2.

![container-networking](docs/images/container-networking.png "Container Networking")

_Figure 2_

### Swarm-mode Routing Mesh

One of the great features of using Docker's swarm-mode with user-defined networks is the concept of a routing mesh. What this means is that any service within the swarm which publishes a port will be reachable from any node in the swarm. This is beneficial as we can use external DNS round-robin to reach any published services. This is shown in Figure 3.

![routing-mesh](docs/images/routing-mesh.png "Routing Mesh")

_Figure 3_

### Reverse Proxy for web services

Any web service should not be exposed to the host/external network and instead only expose their respective ports to the user-defined network. The reverse proxy will connect to the same use-rdefined network and proxy all services using ACL mapping.

# Architecture

The solution is comprised of 3 service stacks; Core JEDI, Version Control System (VCS), and an AT&T-specific stack. This is demonstrated in Figure 4.

![solution](docs/images/solution.png "Architecture")

_Figure 4_

## Core JEDI service stack

The core service stack is comprised of the following services (and shown in Figure 5):

- RabbitMQ
- API Gateway
- Portainer
- Redis
- MongoDB
- Syslog

![jedi-core](docs/images/core-jedi.png "J-EDI Core")

_Figure 5_

## VCS stack

The VCS stack provides source-code management, a CI/CD engine, and a Docker private registry. It consists of the following services (and shown in Figure 6):

- __Gogs:__ Gogs provides git repositories, and a graphical UI for repository management, issue tracking, branch/pull requests, and commit history.

- __Jenkins:__ Jenkins is a CI/CD and workflow engine with a graphical UI and the ability to create workflow "pipelines" via the GUI or as code stored with the source code.

- __Docker private registry:__ The Docker private registry is used to store and retreive Docker container images.

![vcs](docs/images/vcs.png "VCS")

_Figure 6_

## AT&T stack

While the Core JEDI and VCS stacks can be re-used for other solutions, the AT&T stack is comprised of the services that are specific only to the AT&T solution.

## Reverse proxy

The reverse proxy is a standalone service which connects to the user-defined network and will reverse proxy all web-services in the solution.  This is illustrated in Figure 7.

![proxy](docs/images/reverse-proxy.png "Reverse Proxy")

_Figure 7_

# Deployment

## VM preparation

The VMs start out as vanilla Ubuntu 16.04. Ansible playbooks are used to configure the VMs to run the Docker stack. This is accomplished with the following playbooks and roles:

### Prerequsites

- Clone this repo - `git clone git@github.com:Juniper-PS-Automation/26635-ATTM-EDI.git`

### Roles

#### glusternode

This role will configure GlusterFS on all of the nodes. This role is already included in `configure_jedivm.yml` and stored [here](ansible/roles/glusternode).

#### dockernode

This role will configure swarm-mode on all nodes. Currently, it's a static role to configure 1 manager and the rest as workers, but will be updated to allow easier configuration. This role is already included in `configure_jedivm.yml` and stored [here](ansible/roles/dockernode).

### Playbooks

The primary playbook to prepare the VMs is [configure_jedivm.yml](ansible/configure_jedivm.yml). This playbook calls 2 other playbooks; [confgure_gluster.yml](ansible/configure_gluster.yml) and [configure_swarm.yml](ansible/configure_swarm.yml), creates the jedi overlay network, and ensures the gluster volume is ready before proceeding.

### Inventory

The Ansible inventory is stored in the `ansible/inventory` directory using the format `environment.inv`. So, for example, the PS lab inventory is called `ansible/inventory/lab.inv` and looks like this:

```
[jedivms]
node1 ansible_host=10.8.51.220 ansible_user=jedi
node2 ansible_host=10.8.51.221 ansible_user=jedi
node3 ansible_host=10.8.51.222 ansible_user=jedi

[glustervms]
node[1:3]

[jedivms:vars]
ansible_python_interpreter=/usr/bin/python
gluster_iface=ens32

[manager]
node1 swarm_iface=ens32

[worker]
node[2:3] swarm_iface=ens32
```

__DO NOT MODIFY THIS FILE!__ Instead, create your own file using `lab.inv` as a guide and use that file when executing the playbook.

### Run the playbook

```bash
$ cd 26635-ATTM-EDI/ansible
$ ansible-playbook -i inventory/lab.inv configure_jedivm.yml -Kk
```

_NOTE:_ The `-Kk` option prompts for the sudo password and user password, respectively. If you have your SSH key loaded in the target machine(s), the `-k` option won't be necessary. Likewise, if you can invoke `sudo` without a password, the `-K` option is not necessary.

## All-in-one deployment playbook

The `deploy_solution.yml` playbook will deploy the entire solution by running the 2 playbooks plus below all with a single playbook.

## Deploy the core JEDI service stack

### Prerequsites

- If this playbook is run independently of `deploy_solution.yml`, you will need to log in to the Docker Hub manually, or run `hub_login.yml` to accomplish that function.

### Playbook

The core service stack is deployed using the playbook [deploy_jedi.yml](ansible/deploy_jedi.yml). This playbook will perform the following tasks:

- Login to Docker Hub
- Install some Python packages via `pip`
- Copy the docker directory from the local repository to the swarm in a tmp directory (to be executed later)
- Copy the RabbitMQ configuration to a volume (this is needed since RabbitMQ needs to be able to write to the configuration and this is not possible using Docker configs)
- Deploy the core JEDI stack

### Inventory

The Ansible inventory is stored in the `ansible/inventory` directory using the format `environment.inv`. So, for example, the PS lab inventory is called `ansible/inventory/lab.inv` and looks like this:

```
[jedivms]
node1 ansible_host=10.8.51.220 ansible_user=jedi
node2 ansible_host=10.8.51.221 ansible_user=jedi
node3 ansible_host=10.8.51.222 ansible_user=jedi

[glustervms]
node[1:3]

[jedivms:vars]
ansible_python_interpreter=/usr/bin/python
gluster_iface=ens32

[manager]
node1 swarm_iface=ens32

[worker]
node[2:3] swarm_iface=ens32
```

__DO NOT MODIFY THIS FILE!__ Instead, create your own file using `lab.inv` as a guide and use that file when executing the playbook.

### Run the playbook

```bash
$ cd 26635-ATTM-EDI/ansible
$ ansible-playbook -i inventory/lab.inv deploy_jedi.yml -Kk
```

_NOTE:_ The `-Kk` option prompts for the sudo password and user password, respectively. If you have your SSH key loaded in the target machine(s), the `-k` option won't be necessary. Likewise, if you can invoke `sudo` without a password, the `-K` option is not necessary.

## Deploy the VCS stack

### Prerequisites

- If this playbook is run independently of `deploy_solution.yml`, you will need to log in to the Docker Hub manually, or run `hub_login.yml` to accomplish that function.
- Set the Gogs configuration by modifying `docker/gogs/app.ini` and changing __only__ the following fields:

```
[server]
DOMAIN           = localhost
ROOT_URL         = http://localhost:3000/  #Only change the  `localhost` portion to match `DOMAIN`

[mailer]
ENABLED = true
HOST    = mail.local:25
FROM    = gogs@mail.local
USER    = gogs@mail.local
PASSWD  = password
```

### Playbook

The VCS stack is deployed using the playbook [deploy_vcs.yml](ansible/deploy_vcs.yml). This playbook will perform the following tasks:

- Login to Docker Hub
- Install some Python packages via `pip`
- Copy the docker directory from the local repository to the swarm in a tmp directory (to be executed later)
- Deploy the VCS stack

### Inventory

The Ansible inventory is stored in the `ansible/inventory` directory using the format `environment.inv`. So, for example, the PS lab inventory is called `ansible/inventory/lab.inv` and looks like this:

```
[jedivms]
node1 ansible_host=10.8.51.220 ansible_user=jedi
node2 ansible_host=10.8.51.221 ansible_user=jedi
node3 ansible_host=10.8.51.222 ansible_user=jedi

[glustervms]
node[1:3]

[jedivms:vars]
ansible_python_interpreter=/usr/bin/python
gluster_iface=ens32

[manager]
node1 swarm_iface=ens32

[worker]
node[2:3] swarm_iface=ens32
```

__DO NOT MODIFY THIS FILE!__ Instead, create your own file using `lab.inv` as a guide and use that file when executing the playbook.

### Run the playbook

```bash
$ cd 26635-ATTM-EDI/ansible
$ ansible-playbook -i inventory/lab.inv deploy_vcs.yml -Kk
```

_NOTE:_ The `-Kk` option prompts for the sudo password and user password, respectively. If you have your SSH key loaded in the target machine(s), the `-k` option won't be necessary. Likewise, if you can invoke `sudo` without a password, the `-K` option is not necessary.

## Deploy the Proxy stack

### Prerequisites
- If this playbook is run independently of `deploy_solution.yml`, you will need to log in to the Docker Hub manually, or run `hub_login.yml` to accomplish that function.

### Playbook

The Proxy stack is deployed using the playbook [deploy_proxy.yml](ansible/deploy_proxy.yml). This playbook will perform the following tasks:

- Create the data volume directory
- Copy the configuration
- Deploy the Proxy stack

### Inventory

The Ansible inventory is stored in the `ansible/inventory` directory using the format `environment.inv`. So, for example, the PS lab inventory is called `ansible/inventory/lab.inv` and looks like this:

```
[jedivms]
node1 ansible_host=10.8.51.220 ansible_user=jedi
node2 ansible_host=10.8.51.221 ansible_user=jedi
node3 ansible_host=10.8.51.222 ansible_user=jedi

[glustervms]
node[1:3]

[jedivms:vars]
ansible_python_interpreter=/usr/bin/python
gluster_iface=ens32

[manager]
node1 swarm_iface=ens32

[worker]
node[2:3] swarm_iface=ens32
```

__DO NOT MODIFY THIS FILE!__ Instead, create your own file using `lab.inv` as a guide and use that file when executing the playbook.

### Run the playbook

```bash
$ cd 26635-ATTM-EDI/ansible
$ ansible-playbook -i inventory/lab.inv deploy_solution.yml -Kk
```

_NOTE:_ The `-Kk` option prompts for the sudo password and user password, respectively. If you have your SSH key loaded in the target machine(s), the `-k` option won't be necessary. Likewise, if you can invoke `sudo` without a password, the `-K` option is not necessary.

# Testing

See the [Test Automation Guide](TEST.md).
