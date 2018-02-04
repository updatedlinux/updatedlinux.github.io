# [Creation of Volumen (LVM) in CentOS 7 with FileSystem Format XFS](https://updatedlinux.github.io)

 * Objetive: Create a file system based on XFS using the LVM scheme to extend and have high availability without damaging the data.

 * Scope: Physical and Virtual Servers of OpenStack or AWS with attacked disks (via SCSI).

 ## Let's start step by step

Create the Physical Volume
```bash
  pvcreate /dev/emcpowera

```

Where **/dev/emcpowera** is the Device associated with the disk.

Continue with:
```bash
vgcreate volgroup_mdb /dev/emcpowera
```

Where Volume Group will have the name: **volgroup_mdb** which will be associated with the device.

Run the following scams:
```bash
vgscan
pvscan
```

Then create the logical volume:

```bash
lvcreate -L400G -n volumen_mdb01 volgroup_mdb
```

Where 400 equals the available disk space, in this case 400 GB, the name **volumen_mdb01** is the logical volume associated with the volume group **volgroup_mdb**.

Run Logical Scam:
```bash
lvscam
```
Now format the partition:
```bash
mkfs.xfs -f -L mariadbdata /dev/volgroup_mdb/volumen_mdb01
```

At this point we have the disk as a logical volume within a group of volumes.

## Extend Volumen Group

If we want to extend the group of volumes, for example to add a new disk on the RAID (if applicable).
```bash
vgextend volgroup_mdb /dev/emcpowera1
```
To extend the logical volume:
```bash
lvextend -L +40G /dev/volgroup_mdb/volumen_mdb01
```
The following command takes the existing storage volume and adds 40 GB in size, taken from the integration of the new physical volume.

In order for the logical volume to take extended space, execute:
```bash
xfs_growfs /dev/volgroup_mdb/volumen_mdb01
```
If **against any recommendation**, you used EXT4, then you should execute:
```bash
resize2fs /dev/volgroup_mdb/volumen_mdb01
```
That is all. -

Easy? comment on my linkedIn


- **By Jonathan Alexander Melendez Duran
- **Caracas, Venezuela
- **soyjonnymelendez AT gmail DOT com
- **[My Linkedin Profile](https://www.linkedin.com/in/updatedlinux/)




