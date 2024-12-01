# CS-Cart High Availability Setup with NFS, Apache, and Load Balancer
This repository contains the instructions and setup for deploying a CS-Cart application with two web servers, a MySQL server, and an NFS server. Additionally, we use a load balancer to distribute traffic between the two web servers.

## Overview
This guide outlines how to set up a high-availability CS-Cart environment using:
- 2 Apache-based Web Servers running CS-Cart.
- A Shared File Storage Server using NFS.
- A MySQL Database Server.
- A Load Balancer using Apache or HAProxy.

The aim is to provide a scalable, fault-tolerant CS-Cart environment where both web servers share data and the load balancer distributes traffic.

## Prerequisites:
- 4 servers:
  - Web Server 1: Apache Web Server (IP: <web_server1_ip>)
  - Web Server 2: Apache Web Server (IP: <web_server2_ip>)
  - File Storage Server: NFS server (IP: <shared_storage_ip>)
  - MySQL Server: MySQL database server (IP: <mysql_server_ip
- Load Balancer (Could be Apache or HAProxy)

## Step 1: Setting Up the File Storage Server (NFS)
**Install NFS Server**

  1) Install the NFS server:

```bash
sudo apt update
sudo apt install nfs-kernel-server
```

  2) Create the directory to be shared:

```bash
sudo mkdir -p /var/www/cscart
sudo chown nobody:nogroup /var/www/cscart
sudo chmod 777 /var/www/cscart
```

  3) Configure /etc/exports to Share Directory
  
  Edit /etc/exports file:

```bash
sudo nano /etc/exports
```
Add the following line:
```bash
/var/www/cscart *(rw,sync,no_subtree_check)
```
  4) Restart NFS Server
```bash
sudo systemctl restart nfs-kernel-server
```
  5) Verify NFS Server
```bash
sudo systemctl status nfs-kernel-server
```

## Step 2: Mount Shared Storage on Web Servers
**Install Apache2, PHP, and NFS Client**
On both **webserver 1** and **webserver 2**:

```bash
sudo apt update
sudo apt install apache2 -y
sudo apt install nfs-common
sudo add-apt-repository ppa:ondrej/php -y
sudo apt install php8.2 libapache2-mod-php8.2 php8.2-mysql php8.2-curl php8.2-zip php8.2-gd php8.2-mbstring php8.2-xml php8.2-cli php8.2-intl unzip -y
sudo systemctl restart apache2
```

  2) Configure Apache2 to Serve CS-Cart
  
  Edit Apache configuration:

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

  Set DocumentRoot to /var/www/html/cscart:

```apache
DocumentRoot /var/www/html/cscart
```
  3) Mount NFS Share on Web Servers

Mount the NFS share from NFS:

```bash
sudo mount <shared_storage_ip>:/var/www/cscart /var/www/html/cscart
```

  4) Configure Automatic NFS Mount at Boot

  Edit /etc/fstab:

```bash
sudo nano /etc/fstab
```

  Add the following line:

```bash
<shared_storage_ip>:/var/www/cscart /var/www/html/cscart nfs defaults 0 0
```
  5) Install CS-Cart on Web Servers

  Copy and extract CS-Cart:

```bash
cd /var/www/html/cscart
sudo apt install unzip
scp -i "C:\Users\AS\Downloads\secret_key.pem" "C:\Users\AS\Downloads\multivendor_v4.18.2.SP1.zip" ubuntu@13.229.81.83:/home/ubuntu/
sudo unzip multivender_v4.18.2.SP1.zip
```

  6) Set Permissions for CS-Cart
```bash
sudo chown -R www-data:www-data /var/www/html/cscart
sudo chmod -R 755 /var/www/html/cscart
```
  7) Restart Apache
```bash
sudo systemctl restart apache2
```

  8) Verify the mount:

```bash
df -h
```

  Repeat these steps for Web Server 2.

## Step 3: Setting Up the MySQL Database Server
**Install MySQL Server**

  1) Install MySQL:

```bash
sudo apt update
sudo apt install mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
```


  2) Allow Remote Connections

  Edit the MySQL config file:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Change the bind-address to:

```bash
bind-address = 0.0.0.0
```

  3) Restart MySQL:

```bash
sudo systemctl restart mysql
```

  4) Create MySQL User and Database for CS-Cart
  
  Log into MySQL:

```bash
sudo mysql -u root -p
```

  5) Create a CS-Cart database and user:

```sql
CREATE DATABASE cscart_db;
CREATE USER 'cscart_user'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON cscart_db.* TO 'cscart_user'@'%';
FLUSH PRIVILEGES;
EXIT;
```

  6) Test the MySQL connection from Web Server 1 and Web Server 2:

```bash
mysql -u cscart_user -p -h <mysql_server_ip> cscart_db
```


## Step 4: AWS Load Balancer Setup

To distribute traffic between  **webserver 1** and **webserver 2**, an AWS Load Balancer is used.

a. Create an AWS Load Balancer
- Go to the AWS Management Console.
- Navigate to EC2 > Load Balancers.
- Click Create Load Balancer.
- Choose an Application Load Balancer (for HTTP/HTTPS traffic).
- Configure the Load Balancer settings:
  - Name: cscart-lb
  - Scheme: Internet-facing (for public access)
  - IP address type: ipv4
  - Listeners: Add a listener on port 80 (HTTP)
- Choose a VPC and appropriate Subnets.

b. Add Target Group for Web Servers
- Under Target groups, click Create target group.
- Select Instances as the target type.
- Configure the target group:
  - Name: cscart-targets
  - Protocol: HTTP
  - Port: 80
  - Health check: Use default settings (HTTP, /).
- Register **webserver 1** and **webserver 2** as targets.

c. Attach the Load Balancer

- In the Load Balancer configuration, select the target group cscart-targets and attach it.
- Save and create the load balancer.

d. Test Load Balancer

After creating the load balancer, access the DNS name of the Load Balancer provided by AWS (e.g., cscart-lb-123456.elb.amazonaws.com). This will automatically distribute the traffic between **webserver 1** and **webserver 2**.



## Step 5: CS-Cart Installation

On either  **webserver 1** and **webserver 2**, open a browser and navigate to the public IP of the Load Balancer (e.g., http://cscart-lb-123456.elb.amazonaws.com).

During the installation process, provide the following database details:
  - Database Host: 172.31.16.42 (MySQL server's private IP)
  - Database Name: cscart_db
  - Database Username: cscart_user
  - Database Password: chirag

Complete the installation by following the on-screen instructions.


## Step 6: Test the Setup

- CS-Cart Website: Verify that the CS-Cart website is accessible through the Load Balancer and that traffic is being distributed between **webserver 1** and **webserver 2**.
- Database Connection: Ensure that the web servers can connect to the MySQL database on the mysqlcs instance.
-  NFS Mount: Ensure that the NFS share is properly mounted on both **webserver 1** and **webserver 2**.
- Load Balancer: Verify that the Load Balancer is distributing traffic correctly between the two web servers.


## Additional Notes
- Backup: Regularly back up the MySQL database and CS-Cart files to avoid data loss.
- Security: Ensure that your MySQL, NFS, and Load Balancer configurations are properly secured, and use firewall rules to restrict access.
- Scalability: You can add more web servers to the Load Balancer by registering them as additional targets.

## Conclusion

This setup ensures high availability, scalability, and fault tolerance for CS-Cart with shared storage and MySQL replication. Apache-based load balancing ensures that traffic is distributed between web servers efficiently.
