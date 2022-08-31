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

