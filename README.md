# DEVOPS-TOOLING-WEBSITE-SOLUTION.

This project demonstrates the implementation of a scalable three-tier DevOps Tooling Website on AWS. The application is distributed across multiple Apache/PHP web servers that share application files and logs through a centralized Network File System (NFS) server, while a dedicated MySQL server manages persistent data.

By separating the web, storage, and database layers, the architecture keeps the web servers stateless, ensuring consistent data, improved scalability, resilience, and easier server replacement without affecting the application.


### Architecture Overview

- **Client Tier** – End users accessing the application through a web browser.
- **Web Tier** – Multiple Apache/PHP web servers running on RHEL 8. Each server communicates with the NFS server for shared files and logs, and with the MySQL server for database operations.
- **Database Tier** – A dedicated MySQL server responsible for storing all application data.
- **NFS Server** – Provides centralized shared storage for website files and Apache logs, ensuring consistency across all web servers.

![img.(i)](./images/img.(i).png)

# The following steps were executed to implement the project.

# Step 1:  - Prepare NFS Server.

1. Create and configure a Linux-based virtual server (EC2 instance) on AWS.

![img1](./images/img1.png)

2. Create and attach EBS-volumes to the ec2 instance and then connect to the instance.

![img2](./images/img2.png)
![img3](./images/img3.png)

3. Update the package manager 
```bash 
sudo yum update
```
4. Confirm the disk or storage in the server
   List the existing or attached volumes

```bash
lsblk
```
![img5](./images/img5.png)

5. Use the gdisk utility to partition each of the disks. 
```bash
sudo gdisk /dev/nvme1n1
sudo gdisk /dev/nvme2n1
sudo gdisk /dev/nvme3n1
```
![img6](./images/img6.png)

6. Check the partitioned disk. 
![img7](./images/img7.png)

7. Install lvm2 to set-up the logical volumes

```bash
sudo yum install lvm2
```
```bash
sudo pvcreate
```
![img8](./images/img8.png)
![img9](./images/img9.png)

8. Check the physical volumes. 
```bash
sudo pvs
```
![img10](./images/img10.png)

9. Add the physical volumes into a single volume group. 

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```
![img11](./images/img11.png)

10. Create three logical volumes - "apps" "opt" and "logs"

```bash 
sudo lvcreate -n apps-lv -L 10G webdata-vg
sudo lvcreate -n opt-lv -L 10G webdata-vg
sudo lvcreate -n logs-lv -L 10G webdata-vg
```
![img12](./images/img12.png)

11. Format the logical volumes using xfs file system
```bash
sudo mkfs -t xfs /dev/webdata-vg/apps-lv
sudo mkfs -t xfs /dev/webdata-vg/opt-lv
sudo mkfs -t xfs /dev/webdata-vg/logs-lv
```
![img13](./images/img13.png)

12. Create mount directories 

```bash
sudo mkdir /mnt/apps
sudo mkdir /mnt/opt
sudo mkdir /mnt/logs
```

13. Mount each logical volume into its respective mount directory:

```bash
sudo mount /dev/webdata-vg/apps-lv /mnt/apps 
sudo mount /dev/webdata-vg/opt-lv /mnt/opt
sudo mount /dev/webdata-vg/logs-lv /mnt/logs
```
![img14](./images/img14.png)

14. Install NFS server 
```bash
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```
![img15](./images/img15.png)
![img16](./images/img16.png)

15. Set up file permission that will allow web servers to read, write, and execute files on the NFS server.

```bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/opt
sudo chown -R nobody: /mnt/logs
sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/opt
sudo chmod -R 777 /mnt/logs
```
![img17](./images/img17.png)

16. Configure access to the NFS server within the same subnet.

Export the mounts for webservers' subnet cidr to connect as clients. For simplicity, install your all three Web Servers inside the same subnet. To check your subnet cidr - open your EC2 details in AWS web console and locate 'Networking' tab and open a Subnet link:

open the etc/export file 

```bash
sudo vi /etc/exports  - 172.31.32.0/20
```
![img18](./images/img18.png)

```bash
sudo exportfs -arv
```
![img19](./images/img19.png)

17. Check the port used by the NFS server 
```bash
rpcinfo -p | grep nfs
```
![img20](./images/img20.png)

18. In order for NFS server to be accessible from the client, open the inbound rules and add the ports: 

Go to aws management console and open the inbound rules and add the port ```2049``` TCP ```111``` UDP ```111```

![img21](./images/img1.png)
![img22](./images/img22.png)

# Step 2: Configure the Database

1. Launch an EC2 instance for the MySQL database server and connect to it

![img23](./images/img23.png)
![img24](./images/img24.png)

2. Update the package manager
```bash
sudo apt update
```
![img25](./images/img25.png)

3. Install mysql server 
```bash
sudo apt install -y mysql-server
sudo systemctl start mysql 
sudo systemctl enable mysql 
sudo systemctl status mysql 
```
![img26](./images/img26.png)
![img27](./images/img27.png)

4. Log into mysql console and create a new Database. 

```bash
sudo mysql
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'172.31.32.0/20' IDENTIFIED BY 'Password123';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.32.0/20';
FLUSH PRIVILEGES;
```
![img28](./images/img28.png)

5. Edit MySQL configuration file and bind it to all IP address. 

![img29](./images/img29.png)
![img30](./images/img30.png)

6. Open port 3306 in the database server's security group inbound rules.

![img31](./images/img31.png)

# Step 3: Set up the webservers 
1. Launch and configure two ec2 instances for the web-servers

![img32](./images/img32.png)
![img33](./images/img33.png)

2. Configure NFS client on the web-servers

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```
3. Create the /var/www directory and mount it to the NFS server's apps export:


```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.36.99:/mnt/apps /var/www
```
![img34](./images/img34.png)

4. Verify the mount succeeded with df -h, then make it persistent across reboots by adding it to /etc/fstab:

```bash
sudo vi /etc/fstab
172.31.36.99:/mnt/apps /var/www nfs defaults 0 0
```

![img35](./images/img35.png)
![img36](./images/img36.png)

5. Install Apache, PHP, and Remi's repository on each web server:

```bash
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

setsebool -P httpd_execmem 1
```
![img37](./images/img37.png)
![img38](./images/img38.png)
![img39](./images/img39.png)
![img40](./images/img40.png)
![img41](./images/img41.png)
![img42](./images/img42.png)
![img43](./images/img43.png)


6. Create the Apache log directory and mount it to the NFS server's logs export:

```bash
sudo mkdir /var/log/httpd
sudo mount -t nfs -o rw,nosuid 172.31.36.99:/mnt/logs /var/log/httpd
sudo vi /etc/fstab
```

![img44](./images/img44.png)
![img45](./images/img45.png)

7. To confirm the shared storage is configured correctly, create a test file on one web server and verify that it appears on the other web server too

![img46](./images/img46.png)

8. Install the MySQL client on the web server to test connectivity to the database server:

```bash
sudo yum install mysql-server
```
9. Test the remote connection to the MySQL server from the web server:

![img48](./images/IMG48.png)

10. Deploy the website to the web server. 

![img49](./images/img49.png)

11. Deploy the ```html``` directory from the tooling directory to ```/var/www/```

![img50](./images/img50.png)

12. Update the database connection details in /var/www/html/functions.php. Then import the tooling-db.sql into the database:

```bash
sudo mysql -h 172.31.36.136 -u webaccess -p tooling <tooling-db.sql
```

![img51](./images/img51.png)

![img52](./images/img52.png)


13. Create an admin user by inserting a row into the users table:

![img53](./images/img53.png)
 
14. Open port 80 (HTTP) in each web server's security group inbound rules, so the site is reachable from the browser:

![img54](./images/img54.png)
![img54b](./images/img54b.png)
![img55](./images/img55.png)
![img55b](./images/img55b.png)



Conclusion: 

This project successfully demonstrated the implementation of a scalable three-tier DevOps Tooling Website on AWS. By separating the web, storage, and database layers, the solution provides improved scalability, centralized data management, and easier maintenance. It also highlights the practical application of AWS infrastructure, Linux administration, NFS, MySQL, and Apache in deploying a production-style web application.







