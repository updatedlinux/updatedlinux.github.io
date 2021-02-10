# [Installation of OpenStack (PackStack) Victoria version in CentOS 8 - Multinode Environment (Controller + Compute) in VLAN/FLAT mode](https://git.soyjonnymelendez.com)

 * Objetive: Install OpenStack Victoria version using the PackStack test environment on Hosts with CentOS 8, designed for Multinode environments (controller + compute) using VLAN mode networks

 * Scope: At least two servers with CentOS 8 installed "correctly", without using the EPEL repository (due to incompatibility with Hiera and Puppet)

 ## Let's start step by step

First we must identify the controlling nodes of both clouds, the source cloud where the virtual machines are located, and the destination cloud where they will be deployed.

For this laboratory we will use physical servers, we must have at least two servers, one we will designate as Controller and another as Compute.

Important to keep in mind, the controller will manage all the cloud orchestration components, the computer will only perform computing tasks (virtualization and network) with Nova and Neutron.

On the other hand, it is also important that you take this knowledge into account before continuing with the recipe, although it is simple, if you do not have a clear knowledge of these topics, you should reinforce them. OpenStack by itself is a gigantic world.

```bash
- Administration of Linux.
- Virtualization on Linux with kvm / qemu / libvirt.
- LinuxBridge and OpenVSwitch.
- Linux namespaces.
- Networks in general.
- OpenStack.
- NFS, GlusterFS.
- "Correct" Install of CentOS 8
```

In our laboratory we will use 6 servers, one of them as Controller node and the rest as Computes nodes. In your case with at least two servers you can run this lab without any major problem.

```bash
172.16.157.6 -> OpenStack Victoria Controller Node
172.16.157.5 -> OpenStack Victoria Compute Node
172.16.157.7 -> OpenStack Victoria Compute Node
172.16.157.8 -> OpenStack Victoria Compute Node
172.16.157.9 -> OpenStack Victoria Compute Node
172.16.157.10 -> OpenStack Victoria Compute Node
```

It is important that you take into account that you must have at least 2 Network interfaces on the servers (both controller and compute), one for administration and another for traffic management of virtual machines in Trunk Type Networks (VLAN). 

We will use the following scheme.

Interface 1: VLAN on Access Switch
Interface 2: VLAN on Trunk Type Switch with the Following VLANs Tagged: 81, 82, 83, 96

CIDR: 172.16.81.0/24 - 172.16.82.0/24 - 172.16.83.0/24 - 172.16.96.0/24

You can define how many trunk vlan you want and need for your project.

As TIP it is important that the server models are the same to save you headaches, otherwise due to the preceivable name of the Network interfaces, you will have cases where you have names like: eno1, eno2, enps0fx etc ...

This was solved in an elegant way: Using Bonding, with this we gave the same name to the interfaces and additionally we created high network availability for the Instances. For this Lab I will show you, however it is not something limiting or that you should apply, everything will depend on your work environment.


## Step One: Standardization of the work environment.

We must guarantee that the work environment is the same in all the nodes where the solution will be deployed, importantly, disable the EPEL and EPEL-MODULAR repository if you have them active.

```bash
dnf update -y
dnf config-manager --enable PowerTools
dnf -y install network-scripts
readlink $(readlink $(which ifup))
touch /etc/sysconfig/disable-deprecation-warnings
systemctl disable --now NetworkManager
systemctl enable network
systemctl start  network
systemctl disable --now firewalld
```
Important: selinux must be disabled

System Control (vim /etc/sysctl.conf)
```bash
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```

```bash
reboot
```

## Step Two: We set the hostname on the computers

In the controller node:

```bash
hostnamectl set-hostname "controller-victoria.stack.dom"
```

In the computes nodes:

```bash
hostnamectl set-hostname "compute0x-victoria.stack.com"
```
where "x" is the number of the compute, in our case it goes from 05 to 10

Edit each /etc/hosts of all nodes to make clear the FQDN and IP of each one.

In our case:

Output: 
```bash
[root@victoria ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.157.6    controller-victoria.stack.dom
172.16.157.5    compute05-victoria.stack.dom
172.16.157.7    compute07-victoria.stack.dom
172.16.157.8    compute08-victoria.stack.dom
172.16.157.9    compute09-victoria.stack.dom
172.16.157.10   compute10-victoria.stack.dom
[root@victoria ~]#
```

As I mentioned in previous lines, we will make a small workaround with the name of the interfaces, in this lab, our controller has the name of the interfaces: enp7s0fx and our computers have the name: enox

This is not really bad to die, we can change the preceivable name, there is a lot of documentation about this on the net, I recommend:

https://www.itzgeek.com/how-tos/linux/centos-how-tos/how-to-change-network-interface-name-to-eth0-on-centos-8-rhel-8.html

We obtain by using bonding as workaround, to handle the same name (Bond0) in the interfaces that will be used for trunk traffic of the controller and compute nodes.

We create the following file on all nodes: /etc/sysconfig/network-scripts/ifcfg-bond0
```bash
BONDING_OPTS="mode=4 miimon=100"
TYPE=Bond
BONDING_MASTER=yes
BOOTPROTO=none
NAME=bond0
DEVICE=bond0
ONBOOT=yes
```

This is where the interesting thing comes from, depending on the server the interface changes, keep that in mind, for our controller in the trunk interface 1:

/etc/sysconfig/network-scripts/ifcfg-enp7s0f0

```bash
TYPE=Ethernet
BOOTPROTO=none
NAME=enp7s0f1
DEVICE=enp7s0f1
ONBOOT=yes
MASTER=bond0
SLAVE=yes
```

On trunk interface 2: /etc/sysconfig/network-scripts/ifcfg-enp5s0

```bash
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
NAME=enp5s0
DEVICE=enp5s0
ONBOOT=yes
MASTER=bond0
SLAVE=yes
```

Now we execute:

```bash
ifup bond0
systemctl restart NetworkManager
nmcli connection reload
```

In compute nodes the interfaces are:

```bash
/etc/sysconfig/network-scripts/ifcfg-eno2
/etc/sysconfig/network-scripts/ifcfg-eno3
```

The process for creating the bonding is the same as for the controller, you must execute the same commands listed above, counting the obvious difference in the ifcfg where the Device and the name will obviously change.

**Note: It is important that you execute these steps correctly, I think I have explained clearly, if you have doubts it is better that you study the bonding issue more, if you have the facility that all your servers have the same name on the network interface, you can save this step. Bonding is not mandatory and as I told you, I used it as a workaround and high availability (all my servers have 3 or more network interfaces), but you can do this lab with at least 2 on each server.**

Finally we will give the SSH trust relationship between the controller and the nodes, with this we leave the entire work environment ready.

```bash
ssh-keygen
ssh-copy-id -i /root/.ssh/id_rsa root@compute05-victoria.stack.dom
ssh-copy-id -i /root/.ssh/id_rsa root@compute07-victoria.stack.dom
ssh-copy-id -i /root/.ssh/id_rsa root@compute08-victoria.stack.dom
ssh-copy-id -i /root/.ssh/id_rsa root@compute09-victoria.stack.dom
ssh-copy-id -i /root/.ssh/id_rsa root@compute10-victoria.stack.dom
```
## Step Three: Install OpenStack Victoria Repository.

These steps will be executed **ONLY** from the controller node

```bash
dnf search centos-release-openstack
CentOS-8 - Advanced Virtualization                                                                                                257 kB/s | 133 kB     00:00
CentOS-8 - Ceph Nautilus                                                                                                          530 kB/s | 388 kB     00:00
CentOS-8 - RabbitMQ 38                                                                                                            239 kB/s | 137 kB     00:00
CentOS-8 - NFV OpenvSwitch                                                                                                         35 kB/s |  16 kB     00:00
CentOS-8 - OpenStack victoria                                                                                                     6.6 MB/s | 2.7 MB     00:00
============================================================= Name Matched: centos-release-openstack =============================================================
centos-release-openstack-train.noarch : OpenStack from the CentOS Cloud SIG repo configs
centos-release-openstack-ussuri.noarch : OpenStack from the CentOS Cloud SIG repo configs
centos-release-openstack-victoria.noarch : OpenStack from the CentOS Cloud SIG repo configs
```

I’ll install Victoria release repository package

```bash
dnf -y install centos-release-openstack-victoria
dnf update -y
reboot
```


## Step Four: Install PackStack and generate answers file

Install packstack which is provided by openstack-packstack package.

```bash
dnf install -y openstack-packstack
$ packstack --version
packstack 17.0.0
```

Generate answers file which defines variables that modifies installation of OpenStack services.

```bash
packstack --os-neutron-ml2-tenant-network-types=vlan \
--os-neutron-l2-agent=openvswitch \
--os-neutron-ml2-type-drivers=vlan,flat \
--os-neutron-ml2-mechanism-drivers=openvswitch \
--keystone-admin-passwd=password \
--nova-libvirt-virt-type=kvm \
--provision-demo=n \
--cinder-volumes-create=n \
--os-heat-install=y \
--gen-answer-file /root/answers.txt
```

Set the Keystone / admin user password --keystone-admin-passwd. If you don’t have extra storage for Cinder you can use loop device for volume group by cinder-volumes-create=y but performance will not be good. Above are the standard settings but you can pass as many options as it suites your desired deployment.

For this LAB we will use Cinder with Backend on a Centralized NFS Server.

Take a good look at the parameters that we are going to edit in the answers.txt, you should customize it according to your environment, we add some more, to handle cinder with NFS, however it is not mandatory that you handle it that way, you can leave the configuration By default in the txt and a 20GB loop device will be created as Cinder's Backend, take into account that the performance IS NOT THE BEST.

vi /root/answers.txt

```bash
CONFIG_HEAT_INSTALL=y
CONFIG_CONTROLLER_HOST=172.16.157.6
CONFIG_COMPUTE_HOSTS=172.16.157.5,172.16.157.7,172.16.157.8,172.16.157.9,172.16.157.10
CONFIG_NETWORK_HOSTS=172.16.157.6
CONFIG_KEYSTONE_ADMIN_USERNAME=admin
CONFIG_KEYSTONE_ADMIN_PW=password
CONFIG_NEUTRON_L3_EXT_BRIDGE=br-ex
CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vlan,flat
CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES=vlan
CONFIG_NEUTRON_ML2_VLAN_RANGES=physnet1:10:200,physnet0
CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=physnet1:br-bond0,physnet0:br-ex
CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-bond0:bond0,br-ex:enp7s0f0
CONFIG_NEUTRON_OVS_BRIDGES_COMPUTE=br-bond0
CONFIG_PROVISION_DEMO=n
CONFIG_PROVISION_TEMPEST=n
CONFIG_CINDER_NFS_MOUNTS=172.16.17.2:/NFS/cinder-victoria
CONFIG_CINDER_BACKEND=nfs
```

## Step Five: Install OpenStack Victoria on CentOS 8 With Packstack

If satisfied with the contents in the answers file initiate deployment of OpenStack Victoria on CentOS 8 With Packstack:

```bash
packstack --answer-file /root/answers.txt --timeout=3000
```

Installation process should be started and may take some time to complete but taking into account the number of nodes compute::

```bash
....
Gathering ssh host keys for Nova migration           [ DONE ]
Preparing Nova Compute entries                       [ DONE ]
Preparing Nova Scheduler entries                     [ DONE ]
Preparing Nova VNC Proxy entries                     [ DONE ]
Preparing OpenStack Network-related Nova entries     [ DONE ]
Preparing Nova Common entries                        [ DONE ]
Preparing Neutron API entries                        [ DONE ]
Preparing Neutron L3 entries                         [ DONE ]
Preparing Neutron L2 Agent entries                   [ DONE ]
Preparing Neutron DHCP Agent entries                 [ DONE ]
Preparing Neutron Metering Agent entries             [ DONE ]
Checking if NetworkManager is enabled and running    [ DONE ]
Preparing OpenStack Client entries                   [ DONE ]
Preparing Horizon entries                            [ DONE ]
Preparing Swift builder entries                      [ DONE ]
Preparing Swift proxy entries                        [ DONE ]
Preparing Swift storage entries                      [ DONE ]
Preparing Heat entries                               [ DONE ]
Preparing Heat CloudFormation API entries            [ DONE ]
Preparing Gnocchi entries                            [ DONE ]
Preparing Redis entries                              [ DONE ]
Preparing Ceilometer entries                         [ DONE ]
Preparing Aodh entries                               [ DONE ]
Preparing Puppet manifests                           [ DONE ]
Copying Puppet modules and manifests                 [ DONE ]
Applying 192.168.10.11_controller.pp
192.168.10.11_controller.pp:                         [ DONE ]
Applying 192.168.10.11_network.pp
192.168.10.11_network.pp:                            [ DONE ]
Applying 192.168.10.11_compute.pp
192.168.10.11_compute.pp:                            [ DONE ]
Applying Puppet manifests                            [ DONE ]
Finalizing                                           [ DONE ]

 **** Installation completed successfully ******

Additional information:
 * Time synchronization installation was skipped. Please note that unsynchronized time on server instances might be problem for some OpenStack components.
 * File /root/keystonerc_admin has been created on OpenStack client host 172.16.157.6. To use the command line tools you need to source the file.
 * To access the OpenStack Dashboard browse to http://172.16.157.6/dashboard .
Please, find your login credentials stored in the keystonerc_admin in your home directory.
 * The installation log file is available at: /var/tmp/packstack/20210116-023529-0df1tgus/openstack-setup.log
 * The generated manifests are available at: /var/tmp/packstack/20210116-023529-0df1tgus/manifests
```

You can now source the keystone admin profile in your terminal session.

```bash
source ~/keystonerc_admin
```

Check if you can call the openstack CLI to interact with OpenStack services.


```bash
$ openstack service list
+----------------------------------+------------+----------------+
| ID                               | Name       | Type           |
+----------------------------------+------------+----------------+
| 016e1a0f299e4188a4ff2f0951041890 | swift      | object-store   |
| 02b03ebfe32a48a8ba1b4eb886fea509 | cinderv2   | volumev2       |
| 0ee374b1619e44dd8c3f1f8c8792b08b | nova       | compute        |
| 4eddc25d9c6c42c29ed4aaf3a690e073 | aodh       | alarming       |
| 51ec76355583449aac07c7570750bfda | heat       | orchestration  |
| 75797c5e394f419f9de85e8f424914fa | neutron    | network        |
| 75e2d698d2114d028769621995232a35 | glance     | image          |
| 84da19176cb84382a7a87d9461ab926e | placement  | placement      |
| 8d228baf96b24d97934d1f722337f0ee | heat-cfn   | cloudformation |
| 9e944a5b9a3d474ebc60fd85f0c080bd | cinderv3   | volumev3       |
| 9e9507529ec4454daebeb30183a06d16 | gnocchi    | metric         |
| bf915960baff410db3583cc66ee55daa | keystone   | identity       |
| fbb3e1eb3d6b489386648476e1c55877 | ceilometer | metering       |
+----------------------------------+------------+----------------+
```


To login to Horizon Dashboard I’ll use the URL: http://172.16.157.6/dashboard

At this point the installation was completed successfully.

From now on you must create vlan type networks, configure projects, and service quotas, that is, the entire administrative issue within OpenStack. We will do documentations for this in the near future, however this is where this recipe goes.

This Lab is a compendium of multiple sources on the internet, below I quote them to give respective credit:

- http://www.tuxfixer.com/openstack-pike-vlan-and-flat-network-based-installation-using-packstack/
- https://ahelpme.com/linux/centos-8/adding-bonding-interface-to-centos-8-editing-configuration-files-only/
- https://www.linuxtechi.com/install-openstack-centos-8-with-packstack/
- https://github.com/tigerlinux/openstack-queens-installer-centos7/blob/master/DOCS/NOTES.txt

That is all. -

Easy? comment on my linkedIn


- **By Jonathan Alexander Melendez Duran**
- **Caracas, Venezuela**
- **soyjonnymelendez AT gmail DOT com**
- **https://git.soyjonnymelendez.com**
- **[My Linkedin Profile](https://www.linkedin.com/in/updatedlinux/)**




