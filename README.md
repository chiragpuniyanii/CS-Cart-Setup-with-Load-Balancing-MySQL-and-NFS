# CS-Cart Setup with Load Balancing, NFS, and Master-Master MySQL Replication
This guide provides step-by-step instructions for setting up CS-Cart with a load balancer, NFS (Network File System) for shared storage, and Master-Master MySQL replication to ensure high availability and scalability.

## Overview
This guide outlines how to set up a high-availability CS-Cart environment using:
- 2 Apache-based Web Servers running CS-Cart.
- A Shared File Storage Server using NFS.
- Two MySQL servers (SQL Server 1 and SQL Server 2) with public IPs.
- A Load Balancer using Apache or HAProxy.

The aim is to provide a scalable, fault-tolerant CS-Cart environment where both web servers share data and the load balancer distributes traffic.

## Prerequisites:
- 5 servers:
  - Web Server 1: Apache Web Server (IP: <web_server1_ip>)
  - Web Server 2: Apache Web Server (IP: <web_server2_ip>)
  - File Storage Server: NFS server (IP: <shared_storage_ip>)
  - MySQL Server: Two MySQL servers (SQL Server 1 and SQL Server 2) with public IPs.
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
sudo apt install php8.2 libapache2-mod-php8.2 php8.2-mysql php8.2-curl php8.2-zip php8.2-gd php8.2-mbstring php8.2-xml php8.2-cli php8.2-intl php8.2-soap unzip -y
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
```
**In cmd or powershell**
scp -i "C:\Users\AS\Downloads\secret_key.pem" "C:\Users\AS\Downloads\multivendor_v4.18.2.SP1.zip" ubuntu@13.229.81.83:/home/ubuntu/
```

```
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

## Step 3: Setting Up the Master-Master MySQL Replication Setup
**Install MySQL Server**

  1) Install MySQL:

```bash
sudo apt update
sudo apt install mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
```


  2) Allow Remote Connections

  On both servers, update the MySQL configuration file:

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
  - On both servers, run:

  Log into MySQL:

```bash
sudo mysql -u root -p
```

  5) Create a CS-Cart database and user:

```sql
CREATE DATABASE cscart_db;
CREATE USER 'cscart_user'@'%' IDENTIFIED BY 'chirag';
GRANT ALL PRIVILEGES ON cscart_db.* TO 'cscart_user'@'%';
FLUSH PRIVILEGES;
EXIT;
```

  6) Enable Binary Logging
**Update /etc/mysql/mysql.conf.d/mysqld.cnf:**
  - On Server 1:
```ini
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = row
binlog_do_db = cscart_db
```

  - On Server 2:
```ini
server-id = 2
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = row
binlog_do_db = cscart_db
```

  - Restart MySQL:
```bash
sudo systemctl restart mysql
```

  7) Create Replication User
  - On both servers:
```sql
CREATE USER 'replication_user'@'%' IDENTIFIED BY 'replication_password';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
FLUSH PRIVILEGES;
```

  8) Set Up Master-Master Replication
  - On Server 1:
```sql
CHANGE MASTER TO
MASTER_HOST = '3.91.191.237',  # Server 2 IP
MASTER_USER = 'replication_user',
MASTER_PASSWORD = 'replication_password',
MASTER_LOG_FILE = 'mysql-bin.000001',
MASTER_LOG_POS = 4;
START SLAVE;
```
  - On Server 2:
```sql
CHANGE MASTER TO
MASTER_HOST = '54.90.194.35',  # Server 1 IP
MASTER_USER = 'replication_user',
MASTER_PASSWORD = 'replication_password',
MASTER_LOG_FILE = 'mysql-bin.000001',
MASTER_LOG_POS = 4;
START SLAVE;
```

  9) Verify Replication
  - On both servers, check the status:
```sql
SHOW SLAVE STATUS\G
```

Ensure Slave_IO_Running and Slave_SQL_Running are Yes.



  10) Test the MySQL connection from Web Server 1 and Web Server 2:(OPTIONAL)

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
  - Database Host: 172.31.16.42 (any MySQL server's private IP)
  - Database Name: cscart_db
  - Database Username: cscart_user
  - Database Password: chirag

Complete the installation by following the on-screen instructions.


## Step 6: Test the Setup

- CS-Cart Website: Verify that the CS-Cart website is accessible through the Load Balancer and that traffic is being distributed between **webserver 1** and **webserver 2**.
- Database Connection: Insert test data into cscart_db on one MySQL server and verify replication on the other.
-  NFS Mount: Ensure that the NFS share is properly mounted on both **webserver 1** and **webserver 2**.
- Load Balancer: Verify that the Load Balancer is distributing traffic correctly between the two web servers.


## Additional Notes
- Backup: Regularly back up the MySQL database and CS-Cart files to avoid data loss.
- Monitoring: Regularly monitor replication and NFS performance.
- Security: Ensure that your MySQL, NFS, and Load Balancer configurations are properly secured, and use firewall rules to restrict access.
- Scalability: You can add more web servers to the Load Balancer by registering them as additional targets.

## Conclusion

By following this guide, you have set up a scalable and high-availability infrastructure for CS-Cart using load balancing to distribute traffic, NFS for shared storage, and Master-Master MySQL replication for fault-tolerant, synchronized databases. This configuration ensures scalability to handle growing traffic, redundancy to minimize downtime, and centralized storage for seamless file access. To enhance the setup further, consider enabling SSL for secure connections, implementing monitoring tools like Datadog or AWS CloudWatch for real-time insights, and optimizing database performance with caching solutions like Redis. This setup ensures your CS-Cart platform is resilient, efficient, and ready for high traffic demands.
