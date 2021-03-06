---
layout: post
title: "Getting started with OpenStack"
date: 2021-03-17
tags: openstack vm
---

### Installing OpenStack

The fastest way I found (after reading on components, which gazillions of parts there are and which you probably won't need) was
to just run [DevStack](https://docs.openstack.org/devstack/latest/#).
**Note:** DevStack is not for production and makes A LOT of changes to your system, so install it on a dedicated machine!

The install for Ubuntu 18.04 I followed can be found [here](https://docs.openstack.org/devstack/latest/#).
First add a dedicated user (or just use *ubuntu*). The following are run as *root*, but can also be done with *sudo*.
```
# useradd -s /bin/bash -d /opt/stack -m stack
# echo "stack ALL=(ALL) NOPASSWD: ALL" | tee /etc/sudoers.d/stack
# su - stack
```
Download DevStack:
```
$ git clone https://opendev.org/openstack/devstack
$ cd devstack
```
Create a config file. I added Heat and Ceilometer, but you might do just fine with the first five lines.
```
$ cat <<EOF > local.conf
[[local|localrc]]
ADMIN_PASSWORD=YourSuperSecretPassword
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

#Enable heat services
enable_service h-eng h-api h-api-cfn h-api-cw

CEILOMETER_BACKEND=gnocchi
enable_plugin ceilometer https://opendev.org/openstack/ceilometer
enable_plugin aodh https://opendev.org/openstack/aodh
EOF
```
After that just run
```
$ ./stack.sh
```
and then the install will kick off. This will take a few minutes.
If you run into any errors like `ERROR: Cannot uninstall 'wrapt'. during upgrade` make sure the version you need is installed.
For me it was fixed with `pip install --ignore-installed  wrapt==1.12.1 simplejson==3.17.2`.

After the install you should have access to keystone, glance, nova, placement, cinder, neutron, horizon and also ceilometer and heat if you used my local.conf.
Try pointing a browser to the IP of the Ubuntu machine, it should give you the OpenStack interface where you can login as *admin* with the password defined in
your config file.

# Commandline access
To use the command line install the OpenStack client locally
```
$ pip install python-openstackclient
```
and then create an openrc file:
```
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=YourSuperSecretPassword
export OS_AUTH_URL=http://<ServerIP>/identity
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
and don't forget to `source openrc`.
Try it with e.g.
```
$ openstack user list
```

### Creating your first VM

Start of by either importing a public key or creating a keypair in OpenStack.
```
$ openstack keypair create --public-key ~/.ssh/yourkey.pub NEW_KEY_NAME
```
Create a new security group to allow ping and SSH:
```
$ openstack security group create ssh_icmp_access --description Allows ICMP and SSH access from anywhere
$ openstack security group rule list ssh_icmp_access #Should return nothing
$ openstack security group rule create ssh_icmp_access --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
$ openstack security group rule create --protocol icmp ssh_icmp_access
```
Create a router with a public address:
```
$ openstack router create router
```
Get the public network ID (DevStack creates a public, private and shared network by default), the public-subnet ID and the router ID.
```
$ openstack router list
$ openstack network list
$ openstack subnet list
```
And then set up the router:
```
$ openstack router set ROUTER_ID --external-gateway PUBLIC_NETWORK_ID
$ openstack router add subnet ROUTER_ID PUBLIC_SUBNET_ID
```

Add an image after downloading it (for example [Debian](http://cdimage.debian.org/cdimage/openstack/current-9/)):
```
$ openstack image create --container-format bare --disk-format qcow2 \
        --file debian-9-openstack-amd64.qcow2 debian-9-openstack-amd64
```
Launch an instance. The flavor is the machine size (e.g. **1** is 512MB RAM and 1vCPU, **3** is 4096MB RAM and 4vCPUs),
the image ID can be checked with `openstack image list`.
```
$ openstack server create --flavor 3 --image IMAGE_ID --key-name NEW_KEY_NAME \
  --security-group ssh_icmp_access DEBIAN_INSTANCE_NAME
```
Connect the instance to the private network
```
```
Assign floating IP to instance interface (make sure it’s the private network)
```
$ openstack floating ip create public
$ openstack server add floating ip DEBIAN_INSTANCE_NAME FLOATING_IP
```

Now you should be able to SSH into that server with the private key and floating IP address, using the debian user
```
$ ssh -i ~/.ssh/yourkey debian@FLOATING_IP
```
For more info check out the [OpenStack docs](https://docs.openstack.org/ocata/user-guide/).
