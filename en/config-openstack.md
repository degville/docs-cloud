Title: Configure OpenStack

# Configure OpenStack

Now we've successfully used [Juju][installjuju] and [MAAS][installmaas] to
deploy [OpenStack], it's time to configure OpenStack for use within a typical
production environment. We'll start by setting environment variables and adding
command line utilities, before creating a project with specific network, access
and security requirements, before finally deploying an image.

## nova.rc

Get the IP of Dashboard:

```bash
juju status --format=yaml openstack-dashboard | grep public-address | awk '{print $2}'
```


Get the IP of keystone:

```bash
juju status --format=yaml keystone/0 | grep public-address | awk '{print $2}'
```

Put the details into a file called `nova.rc`:

```yaml
export OS_AUTH_URL=http://192.168.100.95:5000/v2.0/
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_TENANT_NAME=admin
```

```bash
source nova.rc
```

```bash
openstack endpoint list
```

```no-highlight
+----------------------------------+-----------+--------------+--------------+
| ID                               | Region    | Service Name | Service Type |
+----------------------------------+-----------+--------------+--------------+
| 060d704e582b4f9cb432e9ecbf3f679e | RegionOne | cinderv2     | volumev2     |
| 269fe0ad800741c8b229a0b305d3ee23 | RegionOne | neutron      | network      |
| 3ee5114e04bb45d99f512216f15f9454 | RegionOne | swift        | object-store |
| 68bc78eb83a94ac48e5b79893d0d8870 | RegionOne | nova         | compute      |
| 59c83d8484d54b358f3e4f75a21dda01 | RegionOne | s3           | s3           |
| bebd70c3f4e84d439aa05600b539095e | RegionOne | keystone     | identity     |
| 1eb95d4141c6416c8e0d9d7a2eed534f | RegionOne | glance       | image        |
| 8bd7f4472ced40b39a5b0ecce29df3a0 | RegionOne | cinder       | volume       |
+----------------------------------+-----------+--------------+--------------+
```

## Define network

Step through the GUI

(no space when entering allocation pools)

```bash
openstack network list
```

```no-highlight
+--------------------------------------+----------------+--------------------------------------+
| ID                                   | Name           | Subnets
|
+--------------------------------------+----------------+--------------------------------------+
| b11cf9a7-b629-4b7f-b47e-9d8243a3b30d | Public_Network |
7512ceed-bff2-4c0c-b34c-518c4639c2cb |
+--------------------------------------+----------------+--------------------------------------+
```

## Cloud images



```bash
wget http://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
```

check file type with:

```bash
file .img
```

```no-highlight
xenial-server-cloudimg-amd64-disk1.img: QEMU QCOW Image (v2), 2361393152 bytes
```

Get `virtual size` from:

```bash
qemu-img info disk.img
```

```no-highlight
image: xenial-server-cloudimg-amd64-disk1.img
file format: qcow2
virtual size: 2.2G (2361393152 bytes)
disk size: 272M
cluster_size: 65536
Format specific information:
    compat: 0.10
    refcount bits: 16
```

Import with:

```bash
openstack image create --public --min-disk 3 --container-format bare --disk-format qcow2 --property architecture=x86_64 --property hw_disk_bus=virtio --property hw_vif_model=virtio --file ~/cloud_images/xenial-server-cloudimg-amd64-disk1.img "xenial x86_64"
```

check image has been uploaded:

```bash
openstack image list
```

```no-highlight
+--------------------------------------+---------------+--------+
| ID                                   | Name          | Status |
+--------------------------------------+---------------+--------+
| d4244007-5864-4a2d-9cfd-f008ade72df4 | xenial x86_64 | active |
+--------------------------------------+---------------+--------+
```
Or you can check the System>Images page on Horizon. 

## Create a project

Create the project from the GUI

Add a user to the GUI and give them access to the project

Identity > Users
Click on Create user

## Project access

Get the Keystone IP address:

```bash
juju show-status --format=yaml keystone/0 | grep public-address | awk '{print $2}'

```
Create the following project01.rc file:

```yaml
export OS_AUTH_URL=http://192.168.100.95:5000/v2.0/
export OS_USERNAME=proj01user
export OS_PASSWORD=openstack
export OS_TENANT_NAME=Project01
```

make the variables this file contains active:

```bash
source project01.rc
```

you can now access the project from the command line. For example, you can list
all the various endpoints with:

```bash
nova endpoints
```

If this doesn't work, then your variables aren't configured correctly. 

## Keypairs

(from the GUI)

When you create a keypair, your browser will request to download the private
key file (project01-keypair.pem).

!!! Note: The keys are available to download only once. If you don't download
them you will need to delete them and generate another set.

```bash
chmod 600 project01-keypair.pem  
```

This key also needs to be copied to the shell user of the MAAS server (why?):

```bash
scp .ssh/project01-keypair.pem ubuntu@192.168.100.3:~/.ssh
```

The MAAS server will ask for the `ubuntu` user  password. By default this is
`ubuntu`. 



- [ ] Define network
- [ ] Cloud images
- [ ] Create a project
- [ ] Project access
- [ ] Keypairs
- [ ] Security groups/SSH
- [ ] Quotas
- [ ] Virtual networks/Public IPs?
- [ ] Create a cloud instance

<!-- LINKS -->
[installjuju]: ./install-juju.md
[installmaas]: ./install-maas.md
[installos]: ./install-openstack.md
