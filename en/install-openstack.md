Title: Install OpenStack
TODO: Add image of deployment from the Juju GUI
      Add link to the bundle file

# Install OpenStack

Now that we've installed and configured [MAAS][installmaas] and successfully
deployed a [Juju][installjuju] controller, it's time to do some real work;
we're going to use Juju to deploy [OpenStack][openstack], the leading open
cloud platform. 

We have two options when installing OpenStack. 

1. Install and configure each OpenStack component separately. Adding
   Ceph, Compute, Swift, RabbitMQ, Keystone and Neutron in this way allows you
   to see exactly what Juju and MAAS are doing, and consequently, gives you a
   better understanding of the underlying OpenStack deployment. 
1. Use a [`bundle`][bundle] to deploy OpenStack with a single command.  A
   bundle is an encapsulation of a working deployment, including all
   configuration, resources and references. It allows you to effortlessly recreate
   a deployment with a single command.

If this is your first foray into MAAS, Juju and OpenStack territory, we'd
recommend starting with the first option. This will give you a stronger
foundation for maintaining and expanding the default deployment. Our
instructions for this option continue below. 

Alternatively, jump to [Deploying OpenStack as a bundle][osbundle] to learn
about deploying as a bundle.

## Deploy the Juju controller

[Previously][installjuju], we tested our MAAS and Juju configuration by
deploying a new Juju controller called `maas-controller`. You can check this
controller is still operational by typing `juju status`. With the Juju
controller running, the output will look similar to the following:

```no-highlight
Model    Controller           Cloud/Region  Version
default  maas-controller-two  mymaas        2.2-alpha1

App  Version  Status  Scale  Charm  Store  Rev  OS  Notes

Unit  Workload  Agent  Machine  Public address  Ports  Message

Machine  State  DNS  Inst id  Series  AZ
``` 

If you need to remove and redeploy the controller, use the following two
commands:

```bash
juju kill-controller maas-controller
juju bootstrap --constraints tags=juju maas maas-controller
```

During the bootstrap process, Juju will create a model called `default`, as
shown in the output from `juju status` above. [Models][jujumodels] act as
containers for applications, and Juju's default model is great for
experimentation. 

We're going to create a new model called `uos` to hold our OpenStack deployment
exclusively, making the entire deployment easier to manage and maintain.

To create a model called `uos` (and switch to it), simply type the following:

```bash
juju add-model uos
```
## Deploy OpenStack 

We're now going to step through adding each of the various OpenStack components
to the new model. Each application will be installed from the 
[Charm store][charmstore], and we'll be providing the configuration for many of
the charms as a `yaml` file which we include as we deploy them. 

### [Ceph-OSD][charmcephosd]

We're starting with the Ceph object storage daemon. We want to configure Ceph
to use the second drive of a cloud node, `/dev/sdb`, but change or ignore this
to match your own configuration. The configuration is held within the following
file we've called `ceph-osd.yaml`:

```yaml
ceph-osd:
  osd-devices: /dev/sdb
  osd-reformat: "yes"
```

We're going to deploy Ceph-OSD to each of the four cloud nodes we've already
tagged with `compute`. The following command will import the settings above and
deploy Ceph-OSD to each of the four nodes:

```bash
juju deploy --constraints tags=compute --config ceph-osd.yaml -n 4 ceph-osd
```

In the background, Juju will ask MAAS to commission the nodes, powering them on
and installing Ubuntu. Juju then takes over and installs the necessary packages
for the required application.

Remember, you can check on the status of a deployment using the `juju status`
command. To see the status of a single charm of application, append the charm
name:

```bash
juju status ceph-osd
```

In the early stages of deployment, the output will look similar to the
following:

```no-highlight
Model  Controller       Cloud/Region  Version
uoa    maas-controller  mymaas        2.2-beta1

App       Version  Status   Scale  Charm     Store       Rev  OS      Notes
ceph-osd  10.2.6   blocked      4  ceph-osd  jujucharms  241  ubuntu

Unit         Workload  Agent  Machine  Public address   Ports  Message
ceph-osd/0   blocked   idle   0        192.168.100.113         Missing relation: monitor
ceph-osd/1*  blocked   idle   1        192.168.100.114         Missing relation: monitor
ceph-osd/2   blocked   idle   2        192.168.100.115         Missing relation: monitor
ceph-osd/3   blocked   idle   3        192.168.100.112         Missing relation: monitor

Machine  State    DNS              Inst id  Series  AZ       Message
0        started  192.168.100.113  fr36gt   xenial  default  Deployed
1        started  192.168.100.114  nnpab4   xenial  default  Deployed
2        started  192.168.100.115  a83gcy   xenial  default  Deployed
3        started  192.168.100.112  7gan3t   xenial  default  Deployed
```

Don't worry about the 'Missing relation' messages. We'll connect the relations
together in a later step. You also don't have to wait for a deployment to finish
before adding further applications to Juju. Errors will mostly resolve
themselves as applications are deployed and relations are triggered.

### [Nova Compute][charmcompute]

We're going use three machines to host the OpenStack Nova Compute application.
The first will use the following configuration file, `compute.yaml`, while
we'll use the second and third to scale-out the same application to two other
machines. 

```yaml
nova-compute:
  enable-live-migration: True
  enable-resize: True
  migration-auth-type: ssh
  virt-type: qemu
```

Type the following to deploy `nova-compute` to the first machine:

```bash
juju deploy --to 1 --config compute.yaml nova-compute
```

And use the following commands to scale-out Nova Compute to the second and third
machines:

```bash
juju add-unit --to 2 nova-compute
juju add-unit --to 3 nova-compute
```

!!! Note:
    We don't deploy Compute to machine `0`.

As before, it's worth checking `juju status nova-compute` output to make sure `nova-compute`
has been deployed to three machines. Look for lines similar to this:

```no-highlight
Machine  State    DNS              Inst id  Series  AZ       Message
1        started  192.168.100.117  7gan3t   xenial  default  Deployed
2        started  192.168.100.118  fr36gt   xenial  default  Deployed
3        started  192.168.100.119  nnpab4   xenial  default  Deployed
```

### [Swift storage][charmswiftstorage]

The Swift-storage application is going to be deployed to the first machine
(`machine 0`), and scaled across the other three with the following
configuration file:


```no-highlight
swift-storage:
  block-device: sdc
  overwrite: "true"
```

Here are the four deploy commands for the four machines:

```bash
juju deploy --to 0 --config swift-storage.yaml swift-storage
juju add-unit --to 1 swift-storage
juju add-unit --to 2 swift-storage
juju add-unit --to 3 swift-storage
```

### [Neutron networking][charmneutron]

Next comes Neutron for OpenStack networking. We have just a couple of
configuration options than need to be placed within `neuton.yaml` and we're
going to use this for two applications, `neutron-gateway` and `neutron-api`:

```yaml
neutron-gateway:
  ext-port: 'eth1'
neutron-api:
  neutron-security-groups: True
```

First, deploy the gateway:

```bash
juju deploy --to 0 --config neutron.yaml neutron-gateway
```

We're going to colocate the Neutron API on machine 1 by using an [LXD][lxd]
container. This is a great solution for both local deployment and for managing
cloud instances.  

We'll also deploy Neutron OpenvSwitch at the same time:

```bash
juju deploy --to lxd:1 --config neutron.yaml neutron-api
juju deploy neutron-openvswitch
```

We've got to a stage where we can start to connect applications together.
Juju's ability to add these links, known as a relation in Juju, is one of its
best features. 

```bash
juju add-relation neutron-api neutron-gateway
juju add-relation neutron-api neutron-openvswitch
juju add-relation neutron-openvswitch nova-compute
```
There are still 'Missing relations' messages in the status output, leading to
the status of some applications to be `blocked`. This is because there are many
more relations to be added and they'll resolve themselves as we add them.

See [Managing relationships][relationships] in the Juju documentation for more
information on relations.

### [Percona cluster][charmpercona]

The Percona XtraDB cluster application comes next, and like Neutron API above,
we're going to use LXD.

The following `mysql.yaml` is the only configuration we need:

```yaml
mysql:
  max-connections: 20000
``` 

To deploy Percona alongside MySQL:

```bash
juju deploy --to lxd:0 --config mysql.yaml percona-cluster mysql
```

And there's just a single new relation to add:

```bash
juju add-relation neutron-api mysql
```

### [Keystone][charmkeystone]

As Keystone handles OpenStack identity management and access, we're going to
use the following contents of `keystone.yaml` to set an admin password
for OpenStack:

```yaml
keystone:
  admin-password: openstack
``` 

We'll use an LXD container on machine 3 to help balance the load a little. To
deploy the application, use the following command:

```bash
juju deploy --to lxd:3 --config keystone.yaml keystone
```

Then add these relations:

```bash
juju add-relation keystone mysql
juju add-relation neutron-api keystone
```

### [RabbitMQ][charmrabbit]

We're using RabbitMQ as the messaging server. Deployment requires no further
configuration than running the following command:

```bash
juju deploy --to lxd:0 rabbitmq-server
```

This brings along four new connections that need to be made:

```bash
juju add-relation neutron-api rabbitmq-server
juju add-relation neutron-openvswitch rabbitmq-server
juju add-relation nova-compute:amqp rabbitmq-server
juju add-relation neutron-gateway:amqp rabbitmq-server:amqp
```

### [Nova Cloud Controller][charmnova]

This is the controller service for OpenStack, and includes the nova-scheduler,
nova-api and nova-conductor services.

The following simple `controller.yaml` configuration file will be used:

```yaml
nova-cloud-controller:
  network-manager: "Neutron"
```
To add the controller to your deployment, enter the following:

```bash
juju deploy --to lxd:2 --config controller.yaml nova-cloud-controller
```

Followed by these `add-relation` connections:

```bash
juju add-relation nova-cloud-controller mysql
juju add-relation nova-cloud-controller keystone
juju add-relation nova-cloud-controller rabbitmq-server
juju add-relation nova-cloud-controller neutron-gateway
juju add-relation neutron-api nova-cloud-controller
juju add-relation nova-compute nova-cloud-controller
```

### [OpenStack Dashboard][charmdashboard]

We'll deploy the dashboard to another LXD container with a single command:

```bash
juju deploy --to lxd:3 openstack-dashboard
```

And a single relation:

```bash
juju add-relation openstack-dashboard keystone
```

### [Glance][charmglance]

For the Glance, deploy as follows:

```bash
juju deploy --to lxd:2 glance
```

Relations:

```bash
juju add-relation nova-cloud-controller glance
juju add-relation nova-compute glance
juju add-relation glance mysql
juju add-relation glance keystone
juju add-relation glance rabbitmq-server

```

### [Ceph monitor][charmcephmon]

Ceph, the distributed storage system used by our deployment, needs a couple of extra
parameters.

The first parameter is a UUID to ensure each cluster has a unique identifier.
This is simply generated by running the `uuid` command (`apt install uuid`, if
it's not already installed).  We'll use this value as the `fsid` in the
`ceph-mon.yaml` configuration file.

The second parameter is a `monitor-secret` for the configuration file. This is
generated on the MAAS machine by first installing the `ceph-common` package and
then by typing the following:

```bash
ceph-authtool /dev/stdout --name=mon. --gen-key
```
The output will be similar to the following:

```no-highlight
[mon.]
        key = AQAARuRYD1p/AhAAKvtuJtim255+E1sBJNUkcg==
```

This is what the configuration file looks like with the values added:

```yaml
ceph-mon:
  fsid: "a1ee9afe-194c-11e7-bf0f-53d6"
  monitor-secret: AQAARuRYD1p/AhAAKvtuJtim255+E1sBJNUkcg==
```

Finally, deploy and scale the application as follows:

```bash
juju deploy --to lxd:1 --config ceph-mon.yaml ceph-mon
juju add-unit --to lxd:2 ceph-mon
juju add-unit --to lxd:3 ceph-mon
```

With these additional relations:

```bash
juju add-relation ceph-osd ceph-mon
juju add-relation nova-compute ceph-mon
juju add-relation glance ceph-mon
```

### [Cinder][charmcinder]

For Cinder block storage, use the following `cinder.yaml` file:

```yaml
cinder:
  glance-api-version: 2
  block-device: None
```

And deploy with this:

```bash
juju deploy --to lxd:1 --config cinder.yaml cinder
```

Relations:

```bash
juju add-relation nova-cloud-controller cinder
juju add-relation cinder mysql
juju add-relation cinder keystone
juju add-relation cinder rabbitmq-server
juju add-relation cinder:image-service glance:image-service
juju add-relation cinder ceph-mon
```

### [Swift proxy][charmswiftproxy]

Swift also needs a unique identifier, best generated with the `uuid` command we
used previously. The output UUID is used for the `swift-hash` value in the
`wift-proxy.yaml` configuration file:

```yaml
swift-proxy:
  zone-assignment: auto
  swift-hash: "a1ee9afe-194c-11e7-bf0f-53d662bc4339"
```

Use the following command to deploy:

```bash
juju deploy --to lxd:0 --config swift-proxy.yaml swift-proxy
```

These are its two relations:

```bash
juju add-relation swift-proxy swift-storage
juju add-relation swift-proxy keystone
```

### [NTP][charmntp]

The final component we need to deploy is a Network Time Protocol client, to
keep everything in time. This is added with the following simple command:

```bash
juju deploy ntp
```

With these last few `add-relation` commands to finish all the connections:

```bash
juju add-relation neutron-gateway ntp
juju add-relation nova-compute ntp
juju add-relation ceph-osd ntp
```

All that's now left to do is wait for the output of `juju status` to show when
everything is ready.

## Deploying OpenStack as a bundle

Stepping through the deployment of each application is the best way to
understanding how OpenStack operates with Juju and how each application relates
to one another. But it's a labour intensive process.

The same deployment can be accomplished with a single command:

```bash
juju deploy openstack.bundle
```

A [bundle][bundle], as used above, encapsulates the entire deployment process,
including all applications, their configuration and relations. You can use
a local file, as above, or deploy a curated bundle from the [charm
store][osbundle]. 

[Download the OpenStack][downloadbundle] bundle for this project and deploy
using the above command. The speed of the deployment depends on our hardware,
but may take some time. Watch the output of `juju status` to see when
everything is ready.

## Testing OpenStack

After everything has deployed and the output of `juju status` settles, you can
check to make sure OpenStack is working by logging into the Horizon dashboard.

The quickest way to get the IP address for the machine running the dashboard is
to type the following:

```bash
juju status --format=yaml openstack-dashboard | grep public-address | awk '{print $2}'
```

The URL will be `http://<IP ADDRESS>/horizon`, which you should enter
into your browser. Login with `admin` and `openstack`, unless you changed the
password in the configuration file. 

If everything works, you will see something similar to the following:

![Horizon dashboard][install-openstack_horizon] 

## Next steps

Congratulations, you've successfully deployed a working OpenStack environment
using both Juju and MAAS. The next step is to start playing with OpenStack to
see what it's capable of.

<!-- LINKS -->
[osbundle]: #deploying-openstack-as-a-bundle
[installmaas]: ./install-maas.md
[installjuju]: ./install-juju.md
[openstack]: https://www.openstack.org/
[bundle]: https://jujucharms.com/docs/stable/charms-bundles
[jujumodels]: https://jujucharms.com/docs/stable/models
[lxd]: https://www.ubuntu.com/containers/lxd
[charmstore]: https://jujucharms.com
[osbundle]: https://jujucharms.com/openstack-base/
[downloadbundle]: 
[relationships]: https://jujucharms.com/docs/stable/charms-relations
[charmcephosd]: https://jujucharms.com/ceph-osd
[charmcompute]: https://jujucharms.com/nova-compute/
[charmswiftstorage]: https://jujucharms.com/swift-storage/
[charmneutron]: https://jujucharms.com/neutron-api/
[charmpercona]: https://jujucharms.com/percona-cluster/
[charmkeystone]: https://jujucharms.com/keystone/
[charmrabbit]: https://jujucharms.com/rabbitmq-server/
[charmnova]: https://jujucharms.com/nova-cloud-controller/
[charmdashboard]: https://jujucharms.com/openstack-dashboard/
[charmglance]: https://jujucharms.com/glance/
[charmcephmon]: https://jujucharms.com/ceph/
[charmcinder]: https://jujucharms.com/cinder/
[charmswiftproxy]: https://jujucharms.com/swift-proxy/
[charmntp]: https://jujucharms.com/ntp/

<!-- IMAGES -->
[install-openstack_horizon]: ../media/install-openstack_horizon.png
