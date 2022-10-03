# Web Solutions with WordPress

A WordPress solution using three-tier architecture implemented with EC2, EBS. This project implements a web solution using WordPress CMS using Amazon EC2 instances and MySQL database server. The WordPress CMS is installed on a server while the MySQL database server is installed on another server with three EBS volumes of 10GB capacity attached for logging data and application data respectively.


## Project Requirements
- Amazon EC2 Instances (RedHat OS)
- Amazon EBS attached to the EC2 Instances
- WordPress CMS
- MySQL Database server
- Security Group to allow connection on port 22 for SSH, 80 for HTTP connections and port 3306 on the database server for MySQL.

## Guide
- Launch 2 Amazon EC2 Instances on AWS
- Attach 3 volumes of 10GB capacity to the two instances for application and logging data


### Volume Creation and Attachment to EC2 Instances

EBS Volumes are use as file system for storage, databases or applications that require access to block level storage. This is used for data persistency, when the EC2 instances is terminated the data on it is deleted and EBS gives us the ability to be able to have our data even when the instance is no longer available.

- Login to AWS account
- Type EC2 in the services
- From EC2 Dashboard, from the left hand side, click on volumes
- Follow the screen to create your desired capacity
- Make sure you the volume is created in the same Availability Zone of the instance you want to attached the volume to
- Select a volume from the volume dashboard, click on attach

### Configure the EC2 instances attachment as required

The volumes need to be mounted and formatted to be able to access data from the instance. And they are always in the /dev/ folder. This folder contains all the file storage attached to the system.
This folder can be viewed by running **ls /dev/** or **lsblk** command.

The following steps will also be required on the two instances (WebServer and DBServer)

List all the attached volumes on the instances and noticed their names because the names will be needed when formatting the disk for usage

```bash
    lsblk
    ls /dev/
```

The **lsblk** command lists information about all available or the specified block
devices. The lsblk command reads the sysfs filesystem and udev db to gather information. If the udev db is not available or lsblk is
compiled without udev support, then it tries to read LABELs, UUIDs and
filesystem types from the block device. In this case root permissions
are necessary. Source: **man lsblk**

Use gdisk command to create a single partition on the volumes
gdisk is an interactive GUID partition table (GPT) manipulator and it requires root privilege. After entering the command enter **n** to create a new partition on the disk and **w** to save the changes. Repeat for the remaining two volumes. View the changes using **lsblk** command

```bash
    sudo gdisk /dev/xvdf
    sudo gdisk /dev/xvdg
    sudo gdisk /dev/xvdh
```

Install Logical Volume Manager to be able to create virtual block devices from block storage and run **lvmdiskscan ** to scan for disk that may be used as physical volumes
**YUM** and **DNF** are package managers for RedHat, CentOS and Fedora Linux distribution
```
sudo yum install lvm2
sudo lvmdiskscan
```

Use **pvcreate** utility to initialize a physical volume on a device so that it can be recognized as belonging to the Logical Volume Manager. A PV can be placed on a whole device or partition and use **pvs** command to display information about the physical volumes.


```
    sudo pvcreate /dev/xvdf1
    sudo pvcreate /dev/xvdg1
    sudo pvcreate /dev/xvdh1
    sudo pvs
```

Create Logical Volume group with the following command **vgcreate** and verified that the logical volumes has been created with **vgs**

```
    sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
    sudo vgs
```

Create Logical Volume from the Logical Volume group created above
The **apps-lv** is used for the application data while the **logs-lv** is used for the logging data.
Verify that the logical volume is created with **lvs**
View the complete settings about the PV, LV, VG
```
    sudo lvcreate -n apps-lv -L 14G webdata-vg
    sudo lvcreate -n logs-lv -L 14G webdata-vg
    sudo lvs
    sudo vgdisplay -v #view complete setup - VG, PV, and LV
    sudo lsblk 
```

Format the logical volumes with mkfs to build a linux file system, **ext4** is the file extension.

```
    sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
    sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```

Create **/var/www/html directory** to store website files

```
    sudo mkdir -p /var/www/html
```

Create **/home/recovery/logs** to store backup of log data

```
    sudo mkdir -p /home/recovery/logs
```

Mount **/var/www/html** on **apps-lv** logical volume

```
    sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```

Use **rsync** utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)

```
    sudo rsync -av /var/log/. /home/recovery/logs/
```

Mount **/var/log** on **logs-lv** logical volume. (This will delete the existing data on /var/log will be deleted. That is why the previous step above is very
important)
```
sudo mount /dev/webdata-vg/logs-lv /var/log
```

Restore the log files back into /var/log directory using the rsync utility

```
sudo rsync -av /home/recovery/logs/. /var/log
```

Update **/etc/fstab** file so that the mount configuration will persist after restart of the server.
The fstab file typically lists all available disks and disk partitions, and indicates how they are to be initialized or otherwise integrated into the overall system's file system.
Run **blkid** to print block block devices information, the most important information is the UUID. Each entry line in the fstab file contains six fields, each one of them describes a specific information about a filesystem.
- First field – The block device. ...
- Second field – The mountpoint. ...
- Third field – The filesystem type. ...
- Fourth field – Mount options. ...
- Fifth field – Should the filesystem be dumped ? ...
- Sixth field – Fsck order.

```
    sudo blkid
    sudo vim /etc/fstab
```

Sample /etc/fstab file

Test the configuration and reload the daemon

```
   sudo mount -a
   sudo systemctl daemon-reload
```

**Repeat the same steps on the DBServer, but instead of logical volume **apps-lv** create a **db-lv** and mount it to **/db** directory instead of **/var/www/html/**.**

### Install WordPress on your Web Server EC2

WordPress is a free and open-source content management system written in PHP and paired with a MySQL or MariaDB database with supported HTTPS. Features include a plugin architecture and a template system, referred to within WordPress as Themes.

    sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
    sudo systemctl enable httpd
    sudo systemctl start httpd

**Install PHP and it's package dependencies**

    sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
    sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
    sudo yum module list php
    sudo yum module reset php
    sudo yum module enable php:remi-7.4
    sudo yum install php php-opcache php-gd php-curl php-mysqlnd
    sudo systemctl start php-fpm
    sudo systemctl enable php-fpm
    setsebool -P httpd_execmem 1

Download WordPress and copy it to **/var/www/html**

```
    mkdir wordpress
    cd   wordpress
    sudo wget http://wordpress.org/latest.tar.gz
    sudo tar xzvf latest.tar.gz
    sudo rm -rf latest.tar.gz
    cp wordpress/wp-config-sample.php wordpress/wp-config.php
    cp -R wordpress /var/www/html/
```
**Configure SELinux policy**

The SELinux Policy is the set of rules that guide the SELinux security engine. It defines types for file objects and domains for processes. It uses roles to limit the domains that can be entered, and has user identities to specify the roles that can be attained.

```
    sudo chown -R apache:apache /var/www/html/wordpress
    sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
    sudo setsebool -P httpd_can_network_connect=1
```

### Configure MySQL server on the DBServer instance

Install and enable MySQL server to start on boot

```bash
    sudo yum update -y
    sudo yum install -y mysql-server
    sudo systemctl restart mysqld
    sudo systemctl enable mysqld
```
Configure the MySQL server to accept connection from the WebServer instance

```bash
    sudo mysql
    CREATE DATABASE wordpress;
    CREATE USER mysql_user@<Web-Server-Private-IP-Address> IDENTIFIED BY 'password';
    GRANT ALL ON wordpress.* TO mysql_user@<Web-Server-Private-IP-Address>;
    FLUSH PRIVILEGES;
    SHOW DATABASES;
    exit
```

### Start WordPress Configuration on the WebServer IP 
Launch your browser
Type the IP-Address of the WebServer and append <IP-Address>/wordpress
Follow the onscreen-instructions and enter the detailed for the database configured in the steps above.


## Challenges

I had database connection error, I removed the wp-config file from the wordpress folder in /var/www/html/wordpress and reload the page and everything works fine.

[Awesome README](https://github.com/matiassingers/awesome-readme)


