# STEP 1 – PREPARE NFS SERVER

Create an Ec2 instance using RHEL Linux 8 Operating System.
![Ec2 instance](/images/1.png)

Create 3 volumes and attach them to NFS server instance.
![new volumes](/images/2.png)

Verify the attached volumes

```bash
lsblk
```

![new volumes](/images/3.png)

The EBS volumes are attached but not partitioned for use yet. Lets do that using the gdisk utility to perform  a Linux LVM partition.

```bash
sudo gdisk /dev/xvdf
```

![new Linux LVM partion](/images/4.png)

Repeat the above step for the remaining two volumes.

Verify the Partiions

```bash
lsblk
```

![partiions](/images/5.png)

Install Linux volume manager to dynamicially manage the partitions.

```bash
sudo yum update
```

```bash
sudo yum install lvm2
```

![install lvm](/images/6.png)

Create PVs(Phycial Volumes) to added to Volume group.

```bash
sudo pvcreate /dev/xvdf1 /dev/xvdg2 /dev/xvdh3
```

Verify the 3 Pyhical volumes are created

```bash
sudo pvs
```

![create physcial volumes](/images/7.png)

Create Volume Group and add the physical volues to it.

```bash
sudo vgcreate  nfsdata-vg /dev/xvdf1 /dev/xvdg2 /dev/xvdh3
```

Verify the nfsdata-vg is created

```bash
sudo vgs
```

![created volume group](/images/8.png)

Create three Logical volumes from the volume group (nfsdata-vg).

```bash
sudo lvcreate -n lv-opt -L 6G nfsdata-vg
sudo lvcreate -n lv-apps -L 6G nfsdata-vg
sudo lvcreate -n lv-logs -L 5.5G nfsdata-vg
```

Verify the newly created logical volumes

```bash
lvs
```

![created volume group](/images/9.png)

View the full logical and pyhscial volume setup

```bash
sudo vgdisplay -v
```

![volume group setup](/images/10.png)

Format the logical volumes with xfs format.

```bash
sudo mkfs.xfs /dev/nfsdata-vg/lv-opt
sudo mkfs.xfs /dev/nfsdata-vg/lv-apps
sudo mkfs.xfs /dev/nfsdata-vg/lv-logs
```

Create mount points on /mnt directory for the logical volumes

```bash
sudo mkdir /mnt/opt /mnt/apps /mnt/logs
```

Mount the logical volumes
```bash
sudo mount /dev/nfsdata-vg/lv-opt /mnt/opt
sudo mount /dev/nfsdata-vg/lv-apps /mnt/apps
sudo mount /dev/nfsdata-vg/lv-logs /mnt/logs
```

![mount](/images/11.png)