---
layout: post
title:  "IaC: test environment to explore Ansible"
date:   2023-11-02
categories: jekyll update
---

Audience: 
* You want to start using Ansible
* You are familiar with linux, bash, python, ssh, linux, Docker, Ansible, IaC
* You want to start coding *without any costs* for now.


In this guide, we will 
* introduce Ansible, a remote management automation utility 
* set up an environment to run Ansible 
* use docker to simulate a fleet of hosts to be managed
* run basic ansible commands on managed hosts


## Introduction
The main goal is to install Ansible tools on a local environment and have a fleet of ansible remotely managed hosts to 
explore basic Ansible concepts, mainly for educational purposes. 

* This environment can be used as an Ansible playground or sandbox, or it can be reproduced and used to
develop and test Ansible modules.  

* The examples focus on Ansible foundational procedures: ssh, uploading public ssh  keys to managed hosts, 
authentication, connectivity and requirements for state monitoring and management of services and configurations.  

The general instructions can be run on any local development environment where ansible and docker can be installed.
In this guide, we will use [Google Cloud Shell](https://console.cloud.google.com/home/) for convenience:   
* It can be accessed from a browser with just a Google Account, no credit card, no upfront costs, available online, 
persistent 5GB home dir,  quickly provisioned.  
* It runs a debian distribution, includes docker, python3 and all the tools and packages needed already installed.

 
## Introducing Ansible
[Ansible](https://ansible.com/) is an open source automation platform with several components, the main being:  
* A language that can describe infrastructure, state and actions required to transition between states.  
* An automation engine and tool able to run and orchestrate those actions.

Ansible is a strong option to apply  [IaC](https://www.redhat.com/en/topics/automation/what-is-infrastructure-as-code-iac), 
DevOps and CI/CD procedures:
  * Open source
  * Modular and customizable, including built-in modules for admin tasks.
  * Extensive [community](https://www.ansible.com/community) contribution.
  * Ample [documentation](https://docs.ansible.com/)
  * Remote configuration management standard tool in enterprise Linux, see [Red Hat Ansible](https://www.redhat.com/en/technologies/management/ansible)

### Ansible architecture
**Control nodes and managed hosts**  
* Ansible is installed and run from a control node
* The control node have access to Ansible project files locally or from centralized storage
* Ansible project files are ideally managed with source version control 

**Agentless**  
* No need to install dedicated software on managed hosts
* Managed hosts have basic standard and connectivity requirements

**Control node-managed hosts communications: ssh**  
* Control nodes connect to managed hosts with ssh and secure communication using ssh key pairs.
* Ansible for Windows hosts uses [WinRM](https://learn.microsoft.com/en-us/windows/win32/winrm/portal), a Windows remote management protocol.


### Concepts
**Inventory**  
* List of managed hosts, defined as a text file or as scripts to dynamically infer the hosts list.

**Playbooks**  
* YAML files that declare tasks that Ansible performs on managed nodes.
* **Playbooks** contain a number of **plays**, each play being a group of one or more **tasks**.

**Tasks**
 * Actions on system files, software installations, API calls, services management operations, custom tasks.
 * New tasks can be authored, named, grouped and published for general use by 
 [developing Ansible modules](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_general.html#environment-setup).

**Configuration files**
* Configuration files with settings for ansible tools, like inventory location, default user when running tasks on 
 remote hosts, etc. 


## Running Ansible
### Use Google Cloud Shell
1. Create a [Google Cloud](https://console.cloud.google.com/home/dashboard)  platform account if you do not already have it.  
*You can use any existing Google account (Gmail, Drive, Workspace or any other Google product) to access Google Cloud.*

2. To start coding right away, launch [Google Cloud Shell](https://console.cloud.google.com/home/)
3. You can use the [Code Editor](https://ide.cloud.google.com) to create and edit files in your workspace.

### Alternatively use a local dev environment
You can also use a *local dev environment*, with docker and python. 
Please check specific [Ansible control node requirements](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#managed-node-requirements)  

### Install Ansible on control node
Install Ansible on Cloud Shell or your local dev environment to use them as an ansible control node for the managed hosts fleet.

Ansible can be installed in different ways, check the [Ansible installation guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for a complete list of options.
In this case, for convenience we will install ansible as a python package with pip. 

```shell
# Install with user scope
python3 -m pip install --user ansible
ansible --version
# Output: ansible [core 2.15.5] ...python version = 3.9.2 ...jinja version = 3.1.2 ...libyaml = True

# Or within a venv environment 
# If you do not want to install ansible permanently 
export VENV_NAME=ans-venv
python -m venv ${VENV_NAME}
source ${VENV_NAME}/bin/activate
python3 -m pip install  ansible
```

### Prepare a Docker image for managed hosts
In order to **emulate ansible managed hosts** we will run a number of containers with a Docker image that:  
* exposes port 22 for incoming ssh connections
* runs minimal requirements for ansible managed hosts: sshd, python  
 
The local host acting as the Ansible control node will communicate with those containers on port 22 via ssh. 
The image is based on a stable version of CentOS, so you can also run software installation packages and control services remotely
as you would on a fleet of physical or virtual machines.  

Following separation of concerns principle:  
* a base image will install and configure the ansible requirements: *Dockerfile.base*
* a utility image will define a custom entry-point: *Dockerfile + init.sh*
#### Ansible requirements base image
The ansible requirements life-cycle can be managed and versioned independently with *Dockerfile.base*
```console
## Dockerfile.base..................... 
FROM debian:latest
RUN apt-get -y update 
RUN apt-get -y install  openssh-server python3 systemctl
RUN mkdir /var/run/sshd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
EXPOSE 22
```

```shell
# Generate Docker image for hosts
cd ~
mkdir ans-host
cd ans-host
# Create and edit required files
# Ansible requirements base image
nano Dockerfile.base
docker build -f Dockerfile.base -t ans-host:base .

docker image ls
# Output
# REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
# ans-host     base      73bde1a5d197   59 seconds ago   173MB

```

#### Custom managed host container image
Used to expose other services not related to ansible, or run bootstrap tasks unrelated to 
ansible requirements:  *Dockerfile*  and *init.sh*

```shell
# Generate Docker image for hosts
cd ~/ans-host
# Create and edit required files
# Custom commands image
nano Dockerfile
nano init.h
# See file contents below
docker build -f Dockerfile -t ans-host:alpha .
```
**Dockerfile and init.sh**
```console
## Dockerfile.........................
FROM ans-host:base
COPY  --chmod=777 init.sh /init.sh
EXPOSE 80 443
ENTRYPOINT ["sh",  "-c" , "/init.sh"]
```
During the initialization of the container,  init.sh:  
* dynamically reads *SSH_PUBKEY_CONTENT*  a required environment 
variable, containing the **SSH public key file content** of the Ansible control node SSH key pair  
* and adds it to the list of ssh authorized keys on the container acting as Ansible managed host.

**Note**: 
* In this case, for convenience, the managed host identity corresponding to the ssh_key is *root*
* The Dockerfile and init.sh scripts can be modified to use a different identity by creating a user with admin access 
and copying the key to */home/<user_id>/.ssh/authorized_keys*  


```console
## init.sh ..........................
#!/bin/bash
export RUN_OK=0
if [ -z ${SSH_PUBKEY_CONTENT} ]
then
        echo "Please set env var SSH_PUBKEY_CONTENT to allow connections"
        export RUN_OK=1
else
        # Add public key to authorized keys
        if ! [ -d /root/.ssh ]
        then 
            mkdir /root/.ssh
        fi
        echo ${SSH_PUBKEY_CONTENT}>>/root/.ssh/authorized_keys 
        echo "Waiting for ssh connections"
        /usr/sbin/sshd -D
fi
exit $RUN_OK

```



### Install and configure a fleet of managed hosts 
Whether a test environment or a production environment, the critical requirement for managing a host with ansible is 
making sure the hosts can communicate with ansible:  
* ssh connectivity between ansible and the managed host
* managed hosts have an ssh public key corresponding to the ansible agent managing them

This ssh key  would be used by ansible to connect and operate on managed hosts.  
* You can choose to generate a key per managed domain, a key per hosts, etc.
* Have in mind that in production environments keys need to be stored securely, managed and rotated

```shell
# Generate a random run ID to identify test environment assets
# You can use a dynamic input parameter if integrating in a CI/CD pipeline  
export RID=${RANDOM}
echo $RID
# Output 8241

# Generate a ssh key pair from the local environment that will run ansible
# The public key will be later installed on every managed host
# The ansible agent will retain also the private key at ~/.ssh 

ssh-keygen -N "" -t rsa -f ~/.ssh/ansible_id_rsa_${RID}

ls ~/.ssh/ansible_id_rsa*
# Output
# ~/.ssh/ansible_id_rsa_8241  ~/.ssh/ansible_id_rsa_8241.pub
 
ssh-agent bash
ssh-add ~/.ssh/ansible_id_rsa_${RID}


# Create an isolated docker network for the managed hosts fleet
# Docker network and managed domain name based on run ID
export ANSIBLE_TEST_DOMAIN="ansible.test${RID}.local"
export ANSIBLE_TEST_NETWORK="ansible-test-net-${RID}"

docker network create ${ANSIBLE_TEST_NETWORK}
docker network list --filter "name= ${ANSIBLE_TEST_NETWORK}"
# Output
# NETWORK ID     NAME                     DRIVER    SCOPE
# 9fca299ba532   ansible-test-net-8241   bridge    local

# Prepare host lists and ips
# Backup any existing env with the same run ID
if [ -f new_hosts_$RID ]
then 
	mv new_hosts_$RID new_hosts_$RID.org
fi
touch new_hosts_$RID

# Change VM_SET to customize hosts names
# Using the format vm-<runID>-##  
export BASE_NAME="vm-${RID}-"
export VM_SET="01 02 03 04 05"
export SSH_PUBKEY_CONTENT=$(cat ~/.ssh/ansible_id_rsa_${RID}.pub)
export DOCKER_IMAGE='ans-host:alpha'

for vm_id in $(echo ${VM_SET})
do
    export VM_NAME=${BASE_NAME}${vm_id}
	docker run -d --name ${VM_NAME} --network ${ANSIBLE_TEST_NETWORK}  -e SSH_PUBKEY_CONTENT ${DOCKER_IMAGE} 
	export VM_IP=$(docker inspect ${VM_NAME} |jq -r .[0].NetworkSettings.Networks[].IPAddress)
	echo ${VM_IP} ${VM_NAME}.${ANSIBLE_TEST_DOMAIN} >>new_hosts_${RID}
done

docker ps -n 5
# Note container exposed ports. Only port 22/tcp (ssh) is required for ansible 
# Since we will later be installing web services, 80 and 443 are also exposed to allow web connections 

# Output
# CONTAINER ID   IMAGE            COMMAND ...                 PORTS                     NAMES
# 4fec3ce914c6   ans-host:alpha   "sh -c /init.sh -"   ...    22/tcp, 80/tcp, 443/tcp   vm-8241-05
# 9144003d5792   ans-host:alpha   "sh -c /init.sh -"   ...    22/tcp, 80/tcp, 443/tcp   vm-8241-04
# df0e45684f0c   ans-host:alpha   "sh -c /init.sh -"   ...    22/tcp, 80/tcp, 443/tcp   vm-8241-03
# 6c37d5251559   ans-host:alpha   "sh -c /init.sh -"   ...    22/tcp, 80/tcp, 443/tcp   vm-8241-02
# 3eaa900ebbb3   ans-host:alpha   "sh -c /init.sh -"   ...    22/tcp, 80/tcp, 443/tcp   vm-8241-01


# Add docker hosts to local host DNS resolution 
sudo cp /etc/hosts hosts.org
sudo sh -c "cat new_hosts_${RID} >> /etc/hosts"

cat /etc/hosts
# Ouput
# ...
172.18.0.2 vm-8241-01.ansible.test8241.local
172.18.0.3 vm-8241-02.ansible.test8241.local
172.18.0.4 vm-8241-03.ansible.test8241.local
172.18.0.5 vm-8241-04.ansible.test8241.local
172.18.0.6 vm-8241-05.ansible.test8241.local

if [ -f ~/.ssh/known_hosts ]
  then 
    cp ~/.ssh/known_hosts ~/.ssh/known_hosts.org.${RID}
fi


# Use ssh-keyscan to add managed hosts to ssh known hosts 
for vm_id in $(echo "01 02 03 04 05")
do
    export VM_NAME="${BASE_NAME}${vm_id}"    
    ssh-keyscan  -H ${VM_NAME}.${ANSIBLE_TEST_DOMAIN} >> ~/.ssh/known_hosts
done

# Connect to hosts to check ssh key authentication
for vm_id in $(echo "01 02 03 04 05")
do
  export VM_NAME="${BASE_NAME}${vm_id}";
  ssh root@${VM_NAME}.${ANSIBLE_TEST_DOMAIN} -i ~/.ssh/ansible_id_rsa_${RID}  hostname
done
```
### Use source control for your Ansible inventories and playbooks
```shell
# Create local repo for your ansible code
mkdir ans-intro
cd ans-intro
git init

echo "# Ansible Intro" > README.md
git add -A
git commit -a -m "Initial commit"

# Suggestion: use GitHub as a remote repository
gh auth login
export GH_REPO=ans-intro
gh repo create ${GH_REPO} --private

# Add newly created repository to remotes list
export GIT_REMOTE_URL="https://github.com/${GITHUB_USER}/${GH_REPO}.git"
export GIT_REMOTE_ID=origin-GH-${GH_REPO}
git remote add ${GIT_REMOTE_ID} ${GIT_REMOTE_URL}
git remote -v

```
### Create an Ansible inventory
*Static inventory file*  
The basic form of an inventory is a text file with a list of hostnames or IP addresses of managed hosts, each on a single line.
```console
# i0.txt
vm-8241-01.ansible.test8241.local
vm-8241-02.ansible.test8241.local
vm-8241-03.ansible.test8241.local
vm-8241-04.ansible.test8241.local
vm-8241-05.ansible.test8241.local
```

You can also organize hosts in groups and nested groups using INI files style labels or YAML files with key values.
Reference: [Ansible inventory guide](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)

Ansible allows running playbooks on hosts in specific groups, referenced by their group label or alias in the inventory.
```console
# ig0.txt
[prod]
vm-8241-01.ansible.test8241.local
vm-8241-02.ansible.test8241.local
[dev]
vm-8241-03.ansible.test8241.local
vm-8241-04.ansible.test8241.local
[test]
vm-8241-05.ansible.test8241.local
```
```console
# ig0.yaml
prod:
  hosts:
    vm-01:
      ansible_host: vm-8241-01.ansible.test8241.local
    vm-02:
      ansible_host: vm-8241-02.ansible.test8241.local
dev:
  hosts:
    vm-03:
      ansible_host: vm-8241-03.ansible.test8241.local
    vm-04:
      ansible_host: vm-8241-04.ansible.test8241.local
test:
  hosts:
    vm-05:
      ansible_host: vm-8241-05.ansible.test8241.local
```

#### Basic ansible inventory related commands
* **Listing hosts**
```shell
# Create and edit file, see contents  above
nano i0.txt

# Flat list of hosts in an inventory, within all groups
ansible all --list-hosts -i i0.txt
# Output
  hosts (5):
    vm-8241-01.ansible.test8241.local
    vm-8241-02.ansible.test8241.local
    vm-8241-03.ansible.test8241.local
    vm-8241-04.ansible.test8241.local
    vm-8241-05.ansible.test8241.local
```
* **Inspecting host groups**  
Inventory hosts can be organized in host groups.
Two ansible default host groups always exist, even for simple lists of names or addresses:  
  * The **all** host  contains every host explicitly listed in the inventory.
  * The **ungrouped** host group contains every host explicitly listed in the inventory that is not a member of any other group.
  * Plus other groups defined by labels.  

```shell
# Shows inventory group ansible hierarchy
ansible-inventory -i i0.txt --list
# Output
{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "ungrouped"
        ]
    },
    "ungrouped": {
        "hosts": [
            "vm-8241-01.ansible.test8241.local",
            "vm-8241-02.ansible.test8241.local",
            "vm-8241-03.ansible.test8241.local",
            "vm-8241-04.ansible.test8241.local",
            "vm-8241-05.ansible.test8241.local"
        ]
    }
}

ansible-inventory -i ig0.txt --list
# Output
{
    "_meta": { "hostvars": {} },
    "all": { 
        "children": [ "ungrouped", "prod", "dev",  "test" ]
    },
    "dev":  {  "hosts": [  "vm-8241-03.ansible.test8241.local"] },
    "prod": {  "hosts": [  "vm-8241-01.ansible.test8241.local", "vm-8241-02.ansible.test8241.local" ] },
    "test": {  "hosts": [  "vm-8241-04.ansible.test8241.local", "vm-8241-05.ansible.test8241.local" ] }
}

# Check group hierarchy (top group is 'all')
# jq is a json parser
ansible-inventory -i i0.txt --list |jq .all.children
# Output
# [ "ungrouped"]

ansible-inventory -i ig0.txt --list |jq .all.children
# Output
# [ "ungrouped", "prod",   "dev",  "test" ]

```

### Test connection from Ansible control node to managed hosts
Once the fleet of managed hosts is ready, you can use the inventory and built-in  ansible modules to test basic 
communication with the managed hosts.  

Ansible commands accept as general parameter an inventory path.  Alternatively you can set a default inventory using configuration files.

#### Inspecting inventory
These tasks do not involve connecting to the remote hosts, just processing the inventory file.
* **ansible-inventory** and **ansible all --list-hosts** 
```shell
# Inspecting inventory
ansible-inventory -i ig0.yaml --list
# Output
{
    "_meta": {
        "hostvars": {
            "vm-01": {                "ansible_host": "vm-8241-01.ansible.test8241.local"},
            ...
            "vm-05": {                "ansible_host": "vm-8241-05.ansible.test8241.local"},
        }
    },
    "all": {
        "children": [
            "ungrouped",
            "prod",
            "dev",
            "test"
        ]
    },
    "dev": {  "hosts": ["vm-03","vm-04"]  },
    "prod": { "hosts": ["vm-01","vm-02"]  },
    "test": { "hosts": ["vm-05"] }
}
ansible all --list-hosts -i ig0.yaml
# Output 
  hosts (5):
    vm-01   vm-02   vm-03   vm-04    vm-05
```
#### Ansible options and configuration files   
You can run ansible tasks from built-in modules or from modules that you have added to the default ansible system.  
You can run them on the inventory or just one of the host groups defined in it.  
You can specify which authentication methods to use to connect to the managed hosts:  
* user and ssh key
* user and password
* whether to impersonate another identity on the hosts, etc.

Same as with the default inventory, you can set default values for authentication and other general options 
on ansible configuration files.

#### Run basic built-in modules tasks
In this case, we will use basic built-in modules tasks to test communication and ansible requirements on managed hosts.

* **ansible -m debug** and **ansible -m ping** 
```shell    
# Run an Ansible builtin module on just a host group
export HOST_GROUP=dev
export ANS_INV='ig0.yaml'
export ANS_ADMIN=root
export SSH_PRIV_KEY="~/.ssh/ansible_id_rsa_${RID}"

# Debug checks connectivity with selected hosts
ansible ${HOST_GROUP} -m debug -i ${ANS_INV}  --private-key  ${SSH_PRIV_KEY} -u $ANS_ADMIN 
# Output
#
# vm-03 | SUCCESS => { "msg": "Hello world!" }
# vm-04 | SUCCESS => { "msg": "Hello world!" }

# Ping checks connectivity and some requirements
ansible ${HOST_GROUP} -m ping -i ${ANS_INV}  --private-key  ${SSH_PRIV_KEY} -u $ANS_ADMIN
# Output
#
# vm-03 | SUCCESS => {
#     "ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3" },
#     "changed": false,
#     "ping": "pong"
# }
# vm-04 | SUCCESS => {
#     "ansible_facts": { "discovered_interpreter_python": "/usr/bin/python3"},
#     "changed": false,
#     "ping": "pong"
# }
```
* Reference:  
  * [Ansible built-in modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html)
  * [Ansible built-in debug module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)
  * [Ansible built-in ping module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html)
  * [Ansible facts](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html#id3)


### Use Ansible to install a software package on remote nodes
#### Create a playbook
A playbook is a set of tasks to be run on a set of managed hosts. The tasks are part of modules referenced by a 
unique hierarchical name.  
* The example playbook  executes a task with a custom name, using the module *ansible.builtin.apt*
* the task is executed on the host group called *dev*
* the task have as parameters the package and installation state
* the task is executed by the remote host user root

```console
# pb0.yaml 
---
- name: Install Apache package and update to latest version
  hosts: dev
  remote_user: root

  tasks:
  - name: Ensure apache is at the latest version
    ansible.builtin.apt:
      name: apache2
      state: latest
```

```shell
# Create playbook using content from above
nano pb0.yaml
```

#### Use a playbook to check compliance  
* **ansible-playbook --check**

```shell
# Initially used playbook to check state compliance on the selected host group dev
export ANS_PB='pb0.yaml'
export ANS_INV='ig0.yaml' 
ansible-playbook --check ${ANS_PB} -i ${ANS_INV} --private-key ~/.ssh/ansible_id_rsa_${RID} -u $ANS_ADMIN

# Output
# PLAY [Install Apache and update to latest version] 
# 
# TASK [Gathering Facts]  
# ok: [vm-03]
# ok: [vm-04]
# TASK [Ensure apache is at the latest version]  
# fatal: [vm-03]: FAILED! => {"changed": false, "msg": "python3-apt must be installed to use check mode. If run normally this module can auto-install it."}
# fatal: [vm-04]: FAILED! => {"changed": false, "msg": "python3-apt must be installed to use check mode. If run normally this module can auto-install it."}
#
# PLAY RECAP  
# vm-03                      : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
# vm-04                      : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
```
* **Ansible Playbook report**  

  Note the different parts of the playbook output:  
  * every play in the playbook and the play tasks are identified
  * notice the default *Gathering Facts* task, not explicitly included in source code
  * notice the individual tasks result report
  * notice the error message for the task called *Ensure apache is at the latest version*
  * notice the play recap, a result summary for the play presenting ansible checks, tasks results and state changes
        
  Summary: the check fails because there is a missing requirement for a task

 
* **Ansible smart automation**  
Ansible is able to automatically fix missing requirements on control nodes and managed hosts to achieve the desired state.

```shell
# Run the playbook.
# Note --check is not a command option this time 
# Ansible fixes any missing dependencies and install software to achieve desired state
ansible-playbook - ${ANS_PB} -i ${ANS_INV} --private-key ~/.ssh/ansible_id_rsa_${RID} -u $ANS_ADMIN
# PLAY [Install Apache and update to latest version] 
#
# TASK [Gathering Facts]  
# ok: [vm-03]
# ok: [vm-04]

# TASK [Ensure apache is at the latest version]  
# changed: [vm-04]
# changed: [vm-03]
#
# PLAY RECAP 
# vm-03                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
# vm-04

# Check compliance after installation 
ansible-playbook --check ${ANS_PB} -i ${ANS_INV} --private-key ~/.ssh/ansible_id_rsa_${RID} -u $ANS_ADMIN
# Output
# PLAY [Install Apache and update to latest version]
#
# TASK [Gathering Facts]  
# ok: [vm-03]
# ok: [vm-04]
# 
# TASK [Ensure apache is at the latest version]  
# ok: [vm-03]
# ok: [vm-04]
#
# PLAY RECAP  
# vm-03                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
# vm-04                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
     
# Activate verbose mode
ansible-playbook --check -vvv ${ANS_PB} -i ${ANS_INV} --private-key ~/.ssh/ansible_id_rsa_${RID} -u $ANS_ADMIN
```
### Use Ansible to control service state on remote nodes
To control service state you can use the [ansible.builtin.service](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html#ansible-collections-ansible-builtin-service-module)
```console
# pb1.yaml 
---
- name: Check http service is enabled and started
  hosts: dev
  remote_user: root

  tasks:
  - name: Enable service apache2
    ansible.builtin.service:
      name: apache2
      enabled: yes
  - name: Start service apache2
    ansible.builtin.service:
      name: apache2
      state: started
```

```shell
export ANS_PB='pb1.yaml'
ansible-playbook --check ${ANS_PB} -i ${ANS_INV} --private-key ~/.ssh/ansible_id_rsa_${RID} -u $ANS_ADMIN
# Output
# PLAY [Check http service is enabled and started]
# TASK [Gathering Facts]  
# ok: [vm-03]  ok: [vm-04]
# TASK [Enable service apache2]  
# ok: [vm-03]  ok: [vm-04]
# TASK [Start service httpd]  
# changed: [vm-03]  changed: [vm-04]
# PLAY RECAP  
# vm-03                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
# vm-04                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

ansible-playbook  ${ANS_PB} -i ${ANS_INV} --private-key ~/.ssh/ansible_id_rsa_${RID} -u $ANS_ADMIN
# Ouutput
# PLAY [Check http service is enabled and started]
# TASK [Gathering Facts]  
# ok: [vm-03] ok: [vm-04]
# TASK [Enable service apache2]  
# ok: [vm-03] ok: [vm-04]
# TASK [Start service httpd] 
# changed: [vm-03] changed: [vm-04]
# PLAY RECAP  
# vm-03                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
# vm-04                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


ansible-playbook --check ${ANS_PB} -i ${ANS_INV} --private-key ~/.ssh/ansible_id_rsa_${RID} -u $ANS_ADMIN
# Ouput
# PLAY [Check http service is enabled and started] 
# TASK [Gathering Facts] 
# ok: [vm-04]  ok: [vm-03]
# TASK [Enable service apache2]  
# ok: [vm-03] ok: [vm-04]
# TASK [Start service httpd] 
# ok: [vm-04]  ok: [vm-03]
# vm-03                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
# vm-04                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

# Test apache2 services are up and running on selected hosts
curl -I -s vm-8241-03.ansible.test8241.local
curl -I -s vm-8241-04.ansible.test8241.local

# But not for the rest
curl -I -s vm-8241-01.ansible.test8241.local
curl -I -s vm-8241-02.ansible.test8241.local
curl -I -s vm-8241-05.ansible.test8241.local


```

### That's all for now 
At this point you have a playground to explore Ansible commands on a free, fast to set up environment, accessible over 
the internet, with no costs involved.  
You can also use the [CloudShell IDE](https://ide.cloud.google.com), with git UI included, to comfortably edit and 
manage your Ansible project files.

This is just a very basic Ansible introduction, based on core foundations. There's way much more: [Ansible docs](https://docs.ansible.com/).  

Ansible is very demanded in the modern DevOps industry and it's a standard management tool for 
[Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux).

There are many tools based or built on top of Ansible for Infrastructure management and automation, both for on-premises and public cloud: 
* [Ansible Tower](https://docs.ansible.com/ansible-tower/)
* [Ansible AWX](https://github.com/ansible/awx/blob/devel/README.md)
* [Semaphore, Ansible GUI](https://www.ansible-semaphore.com/)
* [Ansible on Azure](https://learn.microsoft.com/en-us/azure/developer/ansible/overview)
* [Ansible on AWS](https://www.ansible.com/integrations/cloud/amazon-web-services)
* [Ansible on GCP](https://www.ansible.com/integrations/cloud/google-cloud-platform)

Next time we'll dive a bit deeper, but for now I hope this is a useful intro to Ansible for beginners.

 

### Environment decommission  
* If on Cloud Shell, you can reset your Cloud VM using the Cloud Shell menu to go back to a pristine environment.  
* If on a local environment, check the following instructions.

```shell
# Check environment resources******
printenv |grep $RID
# Output
# ANSIBLE_TEST_DOMAIN=ansible.test 8241.local
# BASE_NAME=vm- 8241-
# ANSIBLE_TEST_NETWORK=ansible-test-net- 8241
# RID= 8241
# VM_NAME=vm- 8241-05
# SSH_PRIV_KEY=~/.ssh/ansible_id_rsa_ 8241

# Remove temp files******
rm new_hosts_$RID
rm new_hosts_$RID.org

# Undo changes to system files******
sudo  /etc/hosts hosts.org /etc/hosts
rm /etc/hosts hosts.org 

# Remove docker resources******
# Every container based on emulated managed host image 
export DOCKER_IMAGE='ans-host:alpha'
docker ps --filter ancestor=${DOCKER_IMAGE} -q|xargs docker stop|xargs docker rm

# Optionally remove docker images
docker image remove ans-host:alpha
docker image remove ans-host:base

# Remove docker network
docker network rm ${ANSIBLE_TEST_NETWORK}

# SSH config cleanup******
# Original known hosts
sudo cp ~/.ssh/known_hosts.org.${RID} ~/.ssh/known_hosts
rm  ~/.ssh/known_hosts.org.${RID}

# Remove ssh keys
# Make sure they have only been used for the lab before
rm ~/.ssh/ansible_id_rsa_${RID}
rm ~/.ssh/ansible_id_rsa_${RID}.pub
```

