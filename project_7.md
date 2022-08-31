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
