# How to Test J-EDI PaaS Platform

These instructions are provided to make it easy for a developer to stand up a test environment that deploys the J-EDI PaaS platform for test purposes.  Supply your configuration, and the deployment should just work.

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Configuring Test environment](#configuring-test-environment)
	- [System Requirements](#system-requirements)
	- [OpenStack configuration](#openstack-configuration)
	- [SSH configuration](#ssh-configuration)
- [Executing Tests](#executing-tests)

<!-- /TOC -->

## Configuring Test environment

### System Requirements

You must have access to a jump host for your internal IP address range.  That IP address range should be able to access the internet via NAT.  These instructions leverage the Jump Host Subrata configured: 10.13.84.72.

Any system with a recent version of Vagrant with access to the internet and our intranet should suffice.

On my Mac, I set it up via [brew](https://docs.brew.sh/Installation.html):
1. ```brew cask install https://raw.githubusercontent.com/caskroom/homebrew-cask/e8816187ae43f52b598f15f45b3453e22727ac99/Casks/virtualbox.rb```
1. ```brew cask install vagrant```

### OpenStack configuration

#### Cluster Requirements

Please refer to the *OpenStack Cluster Requirements* section in the [Deployment Guide](openstack/jedipass-vmrg-heat/README.md#openstack-cluster-requirements) for information on how to configure OpenStack.

#### Key Pair

When you have generated your Key Pair file for your user ID, copy the .pem file to ```~/.ssh```.

#### RC (Environment) file

Create a file named ```~/.config/openstack/26635-ATTM-EDI.rc```.  __You *MUST* set the values to reflect your OpenStack user and cluster information!__

```
# OpenStack API URL
export OS_AUTH_URL=http://10.13.82.11:5000/v2.0

#
# USER-SPECIFIC Configuration
#
# OpenStack Project Name
export OS_TENANT_NAME="CHANGE_ME"
# OpenStack User Credentials
export OS_USERNAME=CHANGE_ME
export OS_PASSWORD=CHANGE_ME

#
# APPLICATION-SPECIFIC Configuration
#
VNF_NAME=zrdm3frwl98oam

# If your configuration has multiple regions, we set that information here.
# OS_REGION_NAME is optional and only valid in certain environments.
export OS_REGION_NAME="RegionOne"
```

### Docker Hub configuration

Speciify your docker hub configuration file in the path ```~/.config/docker/hub.rc```.  Follow the following template:

```
declare hub_username="ejacques"
declare hub_email="ejacques@juniper.net"
declare hub_password="S3Fua2luczc3Nwo="
```

Note that the ```hub_password``` should be encoded in ```base64```.

### SSH configuration

__YOU MUST REPLACE ```CHANGE_ME_KEY``` WITH THE NAME OF YOUR KEY IN OPENSTACK!__

For example, Edwin would replace it with ```ejacques-key```.

#### Generate Your Public Key

```ssh-keygen -y -f ~/.ssh/CHANGE_ME_KEY.pem > ~/.ssh/CHANGE_ME_KEY.pem.pub```

#### Set SSH Permissions

```chmod 600 ~/.ssh/CHANGE_ME_KEY.pem*```

#### Update SSH Client Configuration

Place the following in your ```~/.ssh/config``` file:
```
# Create alias for jump host, and ID file used to access it.
Host jumphost-vm 10.13.84.72
  HostName 10.13.84.72
  IdentityFile=~/.ssh/CHANGE_ME_KEY.pem

# Internal IP addresses are proxied through the jump host.
Host 172.30.1.*
  StrictHostKeyChecking no
  ProxyCommand ssh -o "StrictHostKeyChecking no" -i ~/.ssh/CHANGE_ME_KEY.pem -q %r@jumphost-vm nc %h %p
```

#### Copy ID to Jump Host

```ssh-copy-id -i ~/.ssh/CHANGE_ME_KEY.pem ubuntu@jumphost-vm```

## Executing Tests

If all configuration is done, then testing the deployment is as simple as running this script.  It will:
1. Bring up the test environment in Vagrant.
2. Remove any existing stack instance.
3. Deploy a new stack instance.
4. Execute smoke tests (if any).
5. Report non-zero error status if there is any failure.

More advanced test suites may be invoked in the future via Jenkins.

Just one command:

```
./test.sh
```

Here is some example output:

```
ejacques-mbp:26635-ATTM-EDI ejacques$ ./test.sh
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Checking if box 'ubuntu/xenial64' is up to date...
==> default: Machine already provisioned. Run `vagrant provision` or use the `--provision`
==> default: flag to force provisioning. Provisioners marked to run always will still run.
INFO:  Deleting any existing stack...
Connection to 127.0.0.1 closed.
4d6b3a70-502f-4ef2-b95e-a6fbd22832b9
Connection to 127.0.0.1 closed.
INFO:  Deploying VM's...
+ ansible-playbook -vvv -e stack_name=STACK_201711041634_JEDIPASS_VMRG --extra-vars=@/home/ubuntu/.config/openstack/default.yml -e heat_template_file_path=../template/jedipass_vmrg.yaml -e heat_env_j2_file_path=../env/jedipass_vmrg.env.j2 -e tenant_name=EJ_Demo -e vnf_name= jedipass_vmrg_heat_create_play.yml
ansible-playbook 2.4.1.0
  config file = None
  configured module search path = [u'/home/ubuntu/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python2.7/dist-packages/ansible
  executable location = /usr/local/bin/ansible-playbook
  python version = 2.7.12 (default, Nov 19 2016, 06:48:10) [GCC 5.4.0 20160609]
No config file found; using defaults
...
...
...
PLAY RECAP *********************************************************************************************************************************************************************************
node1                      : ok=23   changed=16   unreachable=0    failed=0   
node2                      : ok=4    changed=1    unreachable=0    failed=0   
node3                      : ok=4    changed=1    unreachable=0    failed=0   
node4                      : ok=4    changed=1    unreachable=0    failed=0   

INFO:  DONE
```
