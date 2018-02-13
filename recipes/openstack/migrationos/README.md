# [Migration of OpenStack Instances Havana/IceHouse/Juno/Kilo/Liberty/Mitaka to OCATA or Pike Release](https://updatedlinux.github.io)

 * Objetive: Migrate the Virtual Instances of an OpenStack platform at the end of life (Icehouse, Havana and Juno) to Ocata or Pike.

 * Scope: From OpenStack versions Havana, Icehouse, Juno, Kilo, Liberty, Mitaka to Ocata or Pike

 ## Let's start step by step

First we must identify the controlling nodes of both clouds, the source cloud where the virtual machines are located, and the destination cloud where they will be deployed.

For this case study the nodes are:
```bash
172.16.1.184 -> OpenStack Havana Controller Node
172.16.1.189 -> OpenStack OCATA Controller Node
```


## Step One: Creation of the snapshots of virtual instances

From Horizon of the Havana cloud, look for the instances that you want to migrate and press the instantaneous creation button.

If you want to do it from the CLI, enter the controller and with the embedded environment variables for authentication with Keystone, execute:
```bash
nova list
```
You will see the list of instances, take the UUID and execute
```bash
nova image-create <instanceid> "snapshot 1"
```
Where **snapshot 1** is the name of the snapshort you are assigning to the virtual instance.


## Step Two: Creation of the QCOW2 image based on the Snapshot

We entered the Havana controller server and in the Bash console, you must be authenticated with Keystone.
Execute:
```bash
nova image-list
```

You will see the list of images (and among them the snapshot 1)

Output: 
```bash
[root@srvp2p-openstack-01 ~(keystone_admin)]$ nova image-list
+--------------------------------------+-----------------+--------+--------------------------------------+
| ID                                   | Name            | Status | Server                               |
+--------------------------------------+-----------------+--------+--------------------------------------+
| a46efd3a-aecf-4e24-8feb-880bb2081e76 | snapshot 1      | ACTIVE |                                      |
| d6c9eafe-2b05-4fe0-95ca-ed0bae695081 | DB-MASTER-SNAP  | ACTIVE | 8950ace3-ec32-427e-b6f7-eafbff1dcf57 |
| 7aafb80e-73ba-4847-860b-d9b195a4ce32 | Debian-7-64bits | ACTIVE |                                      |
+--------------------------------------+-----------------+--------+--------------------------------------+
```

Identify the ID of the snapshot that interests us, which in this case is: **a46efd3a-aecf-4e24-8feb-880bb2081e76**

Make the call to glance identifying the id of the snapshot 1
```bash
glance image-download --file snapshot.qcow2 a46efd3a-aecf-4e24-8feb-880bb2081e76
```
**Note: It is important that where you run the following command you have enough disk space to generate the QCOW2 file**

## Step Three: Copy of file qcow2 to Ocata controller.

There are multiple ways to copy a file between servers. Unix gives us two very good utilities, scp or rsync, whatever you use is fine, for this example case, we'll use scp:
```bash
scp snapshot.qcow2 root@172.16.1.189:/workdir
```
Note: /workdir is a directory that I use to work in clean, **in your case use the one that suits you best.**

## Step Four: Optimization and Deployment of QCOW2 file in the new Ocata cloud.

Enter via ssh the controller of the Ocata cloud, authenticated with the Keystone environment variables execute the following commands:

```bash
qemu-img convert -c -O qcow2 -p snapshot.qcow2 /workdir/snapshot-compressed.qcow2
```

In this step, I compressed the snapshot image to optimize the space and not upload garbage data to Glance.

This command is executed from the path / workdir where we are working the entire deployment, in your case adapt it your way.

```bash
glance image-create --name "Image-Snapshot-Migrated" --disk-format qcow2 --visibility public --container-format bare --progress --file /workdir/snapshot-compressed.qcow2
```

Or

If you like the native openstack client:
```bash
openstack image create --disk-format qcow2 --container-format bare \
  --public --file /workdir/snapshot-compressed.qcow2 Image-Snapshot-Migrated
```

Select the one that is to your liking. (if you have the openstack client installed of course)

If everything went well, you should see something like this:

```bash
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | b9f5f0d5bc23e3c2b95c92f473729be6     |
| container_format | bare                                 |
| created_at       | 2015-11-17T22:39:32.000000           |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 2760f2ae-7b66-4131-bee4-fb19022227bc |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | Image-Snapshot-Migrated              |
| owner            | None                                 |
| protected        | False                                |
| size             | 1830804480                           |
| status           | active                               |
| updated_at       | 2015-11-17T22:39:45.000000           |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
```


At this point the migration was completed,

You should create a Flavor similar to the one you had in the Havana cloud for the instance that you migrated, and simply from Horizon create the new instance based on the image that you just imported.

You can also use the openstack command in the CLI. I leave it to your choice, in any case I always recommend reading:

https://docs.openstack.org/python-openstackclient/latest/cli/command-list.html

That is all. -

Easy? comment on my linkedIn


- **By Jonathan Alexander Melendez Duran**
- **Caracas, Venezuela**
- **soyjonnymelendez AT gmail DOT com**
- **[My Linkedin Profile](https://www.linkedin.com/in/updatedlinux/)**




