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

Mount the logical volumes and verfiy

```bash
sudo mount /dev/nfsdata-vg/lv-opt /mnt/opt
sudo mount /dev/nfsdata-vg/lv-apps /mnt/apps
sudo mount /dev/nfsdata-vg/lv-logs /mnt/logs
```

```bash
findmnt | grep /mnt
```

![mount](/images/11.png)

## Install NFS server, configure it to start on reboot and make sure it is updated and running

```bash
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

![mount](/images/12.png)

Export the mounts for webservers’ subnet cidr to connect as clients. For simplicity, we will install all three Web Servers inside the same subnet, but in production, it should be set up in separate tier inside its own subnet for higher level of security.

```bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

![mount](/images/13.png)

Configure access to the NFS server for clients(wwebservers) within the same subnet (In this case the Subnet CIDR is 172.31.16.0/20)

Open the exports file and add the mounts to be accessable (on web server cidr address)

```bash
sudo vi /etc/exports
```

Update the NFS file system table by adding the below exports.

```bash
/mnt/apps 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
```

![mount](/images/14.png)

Export the directories

```bash
sudo exportfs -arv
```

And check the port
![mount](/images/15.png)

Make sure to also open the following ports for the NFS server in your security group inbound rules.
TCP 111, UDP 111, UDP 2049

![mount](/images/16.png)

## STEP 2 — INSTALL & CONFIGURE THE DATABASE SERVER -Ubuntu 20.04 + MySQL

Upate repo index and install mysql server

```bash
sudo apt update -y
sudo apt install mysql-server -y
```

![mount](/images/17.png)
Enter the mysql, create database named tooling, a user - named webaccess and grant priviledges only accessble via webserver cidr address

```bash
mysql
create database tooling;
create user 'webaccess'@'172.31.80.0/20' identified by 'password';
grant all privileges on tooling.* TO 'webaccess'@'172.31.80.0/20';
FLUSH PRIVILEGES;
show database;
```

![mount](/images/18.png)

## Step 3 — Prepare the Web Servers

Launch a 3 EC2 instance with RHEL 8 Operating System as the web servers.

![mount](/images/19.png)

Update repo index and Install NFS client

```bash
sudo yum update -y
sudo yum install nfs-utils nfs4-acl-tools -y
```

![mount](/images/20.png)

Mount /var/www/ and target the NFS server’s(used nfs private ip) export for apps

```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.28.191:/mnt/apps /var/www
```

![mount](/images/21.png)

Make sure that the changes will persist on Web Server after reboot

```bash
sudo vi /etc/fstab
```

Add

```bash
172.31.28.191:/mnt/apps /var/www nfs defaults  0  0
```

![mount](/images/22.png)

Install Remi’s repository, Apache and PHP on the Web Server

```bash
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm -y

sudo dnf module reset php -y
sudo dnf module enable php:remi-7.4 -y

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd -y

sudo systemctl start php-fpm
sudo systemctl enable php-fpm

sudo setsebool -P httpd_execmem 1
```

![mount](/images/23.png)

>***Repeat the above steps for the Web-Server2 and Web-Server3***

Verify NFS server is properly configure for the 3 web servers. Create a text file ```text.txt``` in ```Web-Server1``` (in ther server /var/www) and check if it is accessble from ```Web-Server2``` and ```Web-server3```

![mount](/images/24.png)
![mount](/images/25.png)
![mount](/images/26.png)

To store the web server logs, Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. 

```bash
sudo mount -t nfs -o rw,nosuid 172.31.28.191:/mnt/logs /var/log/httpd
```

Let's make sure this mount point persist after reboot.

```bash
sudo vi /etc/fstab
```

Add

```bash
172.31.28.191:/mnt/logs                  /var/log/httpd nfs     defaults        0       0
```

![mount](/images/27.png)

Fork the tooling source code from Darey.io Github Account to your Github account.

Next, Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html

```bash
sudo yum install git -y

git init

git clone https://github.com/oayanda/tooling.git
```

![mount](/images/28.png)

Copy the content the html folder from the tooling app into the web server at /var/www/html

```bash
cd tooling
sudo cp -R html/. /var/www/html
```

![mount](/images/29.png)

Open port 80 for client access in the security group
![mount](/images/30.png)

```bash
sudo setenforce 0
```

Make this change permanent 

```bash
sudo vi /etc/sysconfig/selinux
```

Set SELINUX=disabled 

![mount](/images/31.png)

Restrt httpd and check the status to make sure it is up and running.

```bash
sudo systemctl restart httpd
sudo systemctl status httpd
```

![mount](/images/32.png)

Update the website’s configuration to connect to the database.
In the function.php ```(/var/www/html/functions.php)``` update the authentication details you used when creating the DB Server.

![mount](/images/34.png)
Install mysql client

```bash
sudo yum install mysql -y

```
On the DB Server, edit the mysqld.cnf file to allow anywhere Ip

```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

![mount](/images/35.png)

Back on the Web Server apply the tooling-db.sql script from the git clone folder  to DB Server. Make sure to be in the tooling directory.

![mount](/images/36.png)

Check the ```tooling``` database you created ealier in the  DB Server to see the script applied.

![mount](/images/37.png)

Notice the table user was created and now have a user admin(All from the script).

Update the security group for the DB Server to all MYSQL/Aurora and cidr ip for the web servers.

View Web-Server1 from browser using the public ip

![mount](/images/38.png)

Login with the defualt user ```admin```
![mount](/images/39.png)