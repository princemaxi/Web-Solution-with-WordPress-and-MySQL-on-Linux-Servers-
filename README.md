# Building a Scalable Web Solution with WordPress and MySQL on Linux Servers  

Modern web applications rarely live on a single machine. Instead, they are designed around **modularity, scalability, and separation of concerns**. To understand this in practice, I recently implemented a **WordPress-based web solution** backed by a dedicated MySQL database server.  

This project was more than just a WordPress installation — it was an exercise in **infrastructure design, storage configuration, and system administration**. By setting up separate servers for the **web tier** and the **database tier**, and carefully managing disk partitions and volumes, I was able to simulate how real-world enterprise systems are built and maintained.  

---

## Why This Project Matters as a DevOps Engineer  

As DevOps engineers, we are often at the intersection of software and infrastructure. Knowing how the underlying pieces of a web solution fit together — and more importantly, how to troubleshoot them — is critical. This project gave me hands-on experience in:  

- **Storage subsystem configuration**: working directly with raw disks, partitions, and logical volumes on Linux using tools like `gdisk` and `LVM`.  
- **Application deployment**: installing and configuring WordPress (PHP-based CMS) and connecting it to a remote database server.  
- **Architecture design**: implementing a basic **three-tier architecture** — a common pattern in modern applications.  

---

## Understanding the Three-Tier Architecture  

The **three-tier architecture** is a classic client-server model that separates concerns into distinct layers:  

1. **Presentation Layer** – The front end where users interact (in this case, a browser accessing WordPress).  
2. **Application Layer (Business Logic)** – The web server hosting WordPress, responsible for rendering content and handling requests.  
3. **Data Layer** – The database server running MySQL, which stores and manages the application’s content and user data.  

In this project setup:  

- **Client Layer** → My local machine (browser).  
- **Application Layer** → An EC2 Linux server hosting WordPress.  
- **Data Layer** → A separate EC2 Linux server running MySQL.  

This separation mirrors real-world deployments, making the solution more **scalable**, **secure**, and **fault-tolerant**.  

---
# Detailed Architectural Diagram
    

```swift
                                Internet
                                   |
                               [ALB / NAT]
                                   |
                             Public Subnet (optional)
                                   |
                           -------------------------
                          |      Private Subnet      |
                          | (eu-west-2b) - VPC       |
                           -------------------------
                              |                 |
                              |                 |
                SG: Web-SG    |                 |   SG: DB-SG
       (Allow 80/443  from    |                 |  (Allow 3306 only from
        0.0.0.0/0 or ALB)     v                 v   Web-SG / web private IP)
    +---------------------+      +-----------------------------+
    |  Web EC2 (RHEL 10)  |      |  DB EC2 (RHEL 10)           |
    |  Instance: i-0abcd  |      |  Instance: i-0dcba         |
    |---------------------|      |----------------------------|
    | EBS Volume x3 (10GiB)|     | EBS Volume x3 (10GiB)      |
    | -> /dev/xvdf, xvdg,  |     | -> /dev/xvdf, xvdg, xvdh   |
    |    xvdh             |     |                            |
    | gdisk -> create pv  |     | gdisk -> create pv         |
    | pvcreate -> PVs     |     | pvcreate -> PVs            |
    | vgcreate webdata-vg |     | vgcreate dbdata-vg         |
    | lvcreate apps-lv(14G)|    | lvcreate db-lv(14G)        |
    | lvcreate logs-lv(14G)|    | lvcreate logs-lv(14G)      |
    |---------------------|     |----------------------------|
    | filesystems:         |    | filesystems:               |
    | /dev/webdata-vg/apps-lv -> /var/www/html   |  /dev/dbdata-vg/db-lv -> /db
    | /dev/webdata-vg/logs-lv -> /var/log        |  /dev/dbdata-vg/logs-lv -> /var/log
    |---------------------|     |----------------------------|
    | Apache (httpd)      |     | MySQL / MariaDB             |
    | PHP-FPM             |     | mysqld listens on 0.0.0.0:3306 |
    | WordPress on /var/www/html |                        |
    +---------------------+     +----------------------------+

Network rules:
 - Web-SG: inbound 80/443 from Internet (or ALB), outbound 3306 to DB-SG
 - DB-SG: inbound 3306 only from Web-SG (or specific web private IP), no public inbound
```

## Key Takeaways  

- Linux storage management is foundational for server administration.  
- Splitting the web and database tiers provides a clearer view of how modern applications are structured.  
- WordPress, though simple to set up, offers a great playground to explore **infrastructure, networking, and troubleshooting skills**.  

---

👉 In the next section, I’ll walk through the **step-by-step deployment** of this solution — from preparing storage on both servers to configuring WordPress and linking it to the remote MySQL database.  

---

# Deployment Steps

## Step 1 — Prepare the Web Server (RHEL 10 on EC2)

This step provisions storage for the web tier and mounts it in the right places for WordPress:

- `/var/www/html` → website content
- `/var/log` → system & web logs (on a separate LV)

_Why separate LVs? It gives you flexibility (resize snapshots independently), performance isolation, and reduces the risk that logs fill the same filesystem as your web content._

---
### 1. Launch EC2 and Attach EBS Volumes

- Launch an EC2 instance using RHEL 10 as the AMI for your Web Server.
![images](/images/1.png)
- Create three 10 GiB EBS volumes in the same Availability Zone as the instance.
**_Volumes must be in the same AZ to be attachable._**
![images](/images/2.png)
![images](/images/3.png)
![images](/images/4.png)
![images](/images/5.png)
- Attach the volumes to the instance (they’ll appear as /dev/xvdf, /dev/xvdg, /dev/xvdh).
![images](/images/6.png)
![images](/images/7.png)
![images](/images/8.png)

SSH in:
```bash
cp /mnt/c/Users/HP/Downloads/pj6-key.pem ~/
chmod 400 pj6-key.pem
ssh -i pj6-key.pem ec2-user@ec2-18-168-255-72.eu-west-2.compute.amazonaws.com
```

![images](/images/9.png)

### 2. Update the OS
```bash
sudo yum update -y && sudo yum upgrade -y
```
![images](/images/10.png)

### 3. Discover the Disks

- What’s attached?
    ```bash
    lsblk
    ```
    You should see `xvdf`, `xvdg`, `xvdh` with no partitions yet.
![images](/images/11.png)

- All device files:
    ```bash
    ls /dev/
    ```
    ![images](/images/12.png)
- Mounted filesystems & space:
    ```bash
    df -h
    ```
    ![images](/images/13.png)
### 4. Confirm gdisk (GPT tool)
```bash
gdisk --version
```
_On RHEL, `gdisk` may live in /usr/sbin. If gdisk isn’t found, install it._    

**Why GPT? GPT avoids MBR’s 2 TiB limit, supports more partitions, and has better metadata redundancy.**

### 5. Partition Each Disk with GPT
Repeat for each raw disk (`/dev/xvdf`, `/dev/xvdg`, `/dev/xvdh`):
```bash
sudo gdisk /dev/xvdf
# Inside gdisk:
#  - 'n' to create a new partition (accept defaults for start/end to use the full disk)
#  - type code: 8E00 (LVM)
#  - 'w' to write and exit
```
Do the same for `/dev/xvdg` and `/dev/xvdh`.
![images](/images/15.png)
![images](/images/16.png)
![images](/images/17.png)
![images](/images/18.png)

### 6. Install LVM and Scan
```bash
sudo yum install -y lvm2
sudo lvmdiskscan
```
LVM 101:
- PV (Physical Volume) → a disk or partition LVM manages.
- VG (Volume Group) → a pool combining PVs.
- LV (Logical Volume) → “virtual partition” carved from a VG (you format & mount this).
![images](/images/19.png)

### 7. Create Physical Volumes (PVs)
```bash
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1

sudo pvs   # verify
```
![images](/images/20.png)
![images](/images/21.png)

### 8. Create a Volume Group (VG)
```bash
sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
sudo vgs   # verify
```
_With 3 × 10 GiB PVs ≈ 30 GiB total, you’ll allocate most of it to two LVs and keep a little free space for future growth._
![images](/images/22.png)
![images](/images/23.png)

### 9. Create Logical Volumes (LVs)
- `apps-lv` (for `/var/www/html`) — ~14 GiB
- `logs-lv` (for `/var/log`) — ~14 GiB

    ```bash
    sudo lvcreate -n apps-lv -L 14G webdata-vg
    sudo lvcreate -n logs-lv -L 14G webdata-vg

    sudo lvs      # verify
    sudo vgdisplay -v
    sudo lsblk
    ```
    _Leaving ~2 GiB free in the VG is intentional; it gives you room for metadata or later expansion._
    ![images](/images/24.png)
    ![images](/images/25a.png)
    ![images](/images/25b.png)

### 10. Create Filesystems
Format both LVs with ext4:
```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
![images](/images/26.png)

### 11. Create Mount Points
```bash
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
```
![images](/images/27.png)

### 12. Mount the Web Content LV
```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
![images](/images/28.png)

### 13. Back Up Existing Logs Before Remounting /var/log
_Mounting over `/var/log` hides the current files under the mount point. Back them up first._
```bash
sudo rsync -av /var/log/ /home/recovery/logs/
```
![images](/images/29.png)

### 14. Mount the Logs LV
```bash
sudo mount /dev/webdata-vg/logs-lv /var/log
```
_Note: Some services keep log files open. After mounting, restarting rsyslog/httpd later ensures they write to the new filesystem. (You can also reboot after finishing fstab to test persistence.)_

### 15. Restore the Logs (Optional but Often Useful)
If you want the old logs visible on the new /var/log:
```bash
sudo rsync -av /home/recovery/logs/ /var/log
```
![images](/images/30.png)

### 16. Persist Mounts with /etc/fstab (Use UUIDs)

Get the UUIDs:
```bash
sudo blkid
# or
lsblk -f
```
![images](/images/31.png)

You’ll see entries like:
```bash
/dev/mapper/webdata--vg-apps--lv: UUID="ce8f13d4-ae3c-4d75-8c90-b87d359dd722"
/dev/mapper/webdata--vg-logs--lv: UUID="28ac4228-389c-4c31-a751-e74123a770d4"
```
![images](/images/32a.png)

Edit fstab:
```bash
sudo nano /etc/fstab
```

Add lines using your UUIDs (no quotes):
```fstab
UUID=ce8f13d4-ae3c-4d75-8c90-b87d359dd722  /var/www/html  ext4  defaults  0 0
UUID=28ac4228-389c-4c31-a751-e74123a770d4  /var/log       ext4  defaults  0 0
```
![images](/images/32b.png)

Test and reload:
```bash
sudo mount -a
sudo systemctl daemon-reload   # (not required for fstab, but harmless)
```
![images](/images/33.png)
If mount -a prints no errors, fstab syntax is good.

### 17. Verify
```bash
df -h
```
![images](/images/34.png)

You should see:
- `/var/www/html` mounted from `/dev/mapper/webdata--vg-apps--lv`
- `/var/log` mounted from `/dev/mapper/webdata--vg-logs--lv`

---

## Step 2 — Prepare the Database Server

In this step, we provision a second EC2 instance running RHEL. This instance will serve as the Database Server, and we will prepare its storage using LVM (just like we did for the Web Server). The difference is that instead of creating a logical volume for applications (apps-lv), we will create one for databases (db-lv) and mount it to the /db directory.

### 1. Launch the Database EC2 Instance

- Launch a Red Hat Enterprise Linux (RHEL) EC2 instance from the AWS console.
- Ensure it is in the same Availability Zone (AZ) as the Web Server so you can later configure networking efficiently.
- Create and attach 3 additional EBS volumes (each 10 GiB).

SSH into the database server:
```bash
ssh -i "pj6-key.pem" ec2-user@<your-dbserver-public-dns>
```

Update and upgrade the instance:
```bash
sudo yum update && sudo yum upgrade -y
```

### 2. Inspect Attached Volumes

Check block devices:
```bash
lsblk
```

All devices in Linux live under /dev:
```bash
ls /dev/
```

Check disk usage:
```bash
df -h
```

### 3. Partition the Volumes

Check if gdisk (for GPT partitioning) is installed:
```bash
gdisk --version
```

Partition each attached volume:
```bash
sudo gdisk /dev/xvdf
sudo gdisk /dev/xvdg
sudo gdisk /dev/xvdh
```

### 4. Configure LVM

Install LVM utilities:
```bash
sudo yum install lvm2 -y
sudo lvmdiskscan
```

Mark partitions as physical volumes:
```bash
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```

Verify:
```bash
sudo pvs
```

Create a Volume Group named dbdata-vg:
```bash
sudo vgcreate dbdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
```

Check:
```bash
sudo vgs
```

Create 2 Logical Volumes:
- db-lv (for database data, ~14GB)
- logs-lv (for database logs, remaining space)
```bash
sudo lvcreate -n db-lv -L 14G dbdata-vg
sudo lvcreate -n logs-lv -L 14G dbdata-vg
```

Verify:
```bash
sudo lvs
sudo vgdisplay -v
lsblk
```

### 5. Format and Mount the Logical Volumes

Format with ext4 filesystem:
```bash
sudo mkfs -t ext4 /dev/dbdata-vg/db-lv
sudo mkfs -t ext4 /dev/dbdata-vg/logs-lv
```

Create required directories:
```bash
sudo mkdir -p /db
sudo mkdir -p /home/recovery/logs
```

Mount db-lv to /db:
```bash
sudo mount /dev/dbdata-vg/db-lv /db
```

Backup existing logs before overwriting:
```bash
sudo rsync -av /var/log/ /home/recovery/logs/
```

Mount logs-lv to /var/log:
```bash
sudo mount /dev/dbdata-vg/logs-lv /var/log
```

Restore logs:
```bash
sudo rsync -av /home/recovery/logs/ /var/log
```

### 6. Persist the Mounts

Get UUIDs of the logical volumes:
```bash
sudo blkid
```

Edit /etc/fstab:
```bash
sudo nano /etc/fstab
```

Example configuration (replace with your UUIDs):
```fstab
UUID=xxxx-xxxx   /db       ext4   defaults  0 0
UUID=yyyy-yyyy   /var/log  ext4   defaults  0 0
```

Reload and test:
```bash
sudo mount -a
sudo systemctl daemon-reload
```

Verify mounts:
```bash
df -h
```
![images](/images/35.png)

### 🔑 Why This Setup?

- Separation of Concerns: Application files are kept on the Web Server (/var/www/html), while database files are stored on the Database Server (/db).
- Resilience: By using LVM, we can expand storage easily in the future without downtime.
- Log Isolation: Mounting /var/log separately ensures logs don’t consume database storage, keeping performance stable.
- Persistence: The fstab configuration guarantees mounts persist across reboots.

_👉 With this, the Database Server is now storage-ready for hosting MySQL, MariaDB, or any relational DBMS._

---

## STEP 3 — Install WordPress on the Web Server (EC2)

After preparing the storage subsystem and setting up the database server, the next step is to deploy WordPress on the web server. WordPress is the application that will serve as the presentation layer of our three-tier architecture.

This step involves installing and configuring Apache, PHP, and WordPress, while also ensuring that the server’s SELinux policies and dependencies are properly configured.

### 1. Update the Repository

Keeping the server up-to-date ensures you’re running with the latest patches and security fixes.
```bash
sudo yum -y update
```

### 2. Install Apache and PHP (LAMP Stack Basics)

Apache acts as the web server responsible for serving web content. PHP is required because WordPress is written in PHP and relies on it for dynamic content rendering.

```bash
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
- `httpd` → Apache Web Server
- `php-mysqlnd` → PHP extension for MySQL communication
- `php-fpm` → Improves PHP performance
- `php-json` → Allows handling of JSON data
![images](/images/36.png)

Start and enable Apache:
```bash
sudo systemctl enable httpd && sudo systemctl start httpd
sudo systemctl status httpd
```
![images](/images/37.png)
![images](/images/38.png)

### 3. Install PHP 8.4 and Dependencies

By default, RHEL might ship with older PHP versions. Since WordPress requires modern PHP (>=7.4), we enable the Remi repository and install PHP 8.4.

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm -y
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm -y
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-8.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
```
![images](/images/39.png)
![images](/images/40.png)
![images](/images/41.png)
![images](/images/42.png)
![images](/images/43.png)
![images](/images/44.png)
![images](/images/45.png)

Enable and start PHP-FPM (FastCGI Process Manager) for efficient PHP processing:
```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -p httpd_execmem 1
```
![images](/images/46.png)

Restart Apache to apply PHP configurations:
```bash
sudo systemctl restart httpd
sudo systemctl status php
```
![images](/images/47.png)

### 4. Download and Configure WordPress

Next, download the latest WordPress release and extract it into Apache’s web root (/var/www/html).
```bash
mkdir wordpress
cd wordpress
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html/
```
![images](/images/48.png)
![images](/images/49.png)
At this point, WordPress files are placed in /var/www/html/wordpress, and we now need to ensure that Apache can serve them correctly.

### 5. Configure SELinux Policies

Since RHEL has SELinux enabled by default, additional configurations are needed to allow Apache to interact with WordPress files and the database.
```bash
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
```
![images](/images/50.png)
- chown → gives Apache ownership of WordPress files
-  chcon → changes the SELinux security context to allow web content read/write
- setsebool → allows Apache to connect to external resources (like the database server)

✅ At this stage, WordPress is installed on your web server and ready to be configured to connect with the remote MySQL database.

---

## Step 4 — Install and Configure MySQL on the Database Server (DB EC2)

Now that the web server is prepared and WordPress is installed, the next step is to set up the database layer of our three-tier architecture. WordPress requires a relational database (MySQL or MariaDB) to store critical data such as posts, pages, user accounts, and site configurations.

We’ll use MySQL Community Server 8.0, which is open-source and widely supported.

### 🔹 Update Your Packages

Before installing MySQL, ensure your server packages are up to date:
```bash
sudo yum update -y
```

### 🔹 Add the MySQL Yum Repository

MySQL provides an official repository for RHEL-based distributions. Install it with:
```bash
sudo yum install https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm -y
```
This ensures you’re pulling the latest stable MySQL 8.0 packages.

### 🔹 Install MySQL Server
```bash
sudo dnf install mysql-server -y
```
![images](/images/51.png)
This installs the MySQL server and supporting utilities.

### 🔹 Verify MySQL Service

After installation, check whether the service is running:
```bash
sudo systemctl status mysqld
```
![images](/images/52.png)

If the service isn’t active, start and enable it:
```bash
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```

Enabling ensures MySQL automatically starts on system boot.

### 🔹 Secure the MySQL Installation

When MySQL is first installed, the root user has a temporary password. You’ll need to set your own strong password and secure the installation.

Log in to the MySQL shell:
```bash
sudo mysql -u root -p
```

Then, update the root user’s password:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Strong.Passw0rd!';
```
![images](/images/53.png)

🔑 Replace Strong.Passw0rd! with a strong password following MySQL’s policy requirements.

_Once complete, you now have a functioning database server ready to host WordPress data._

---

## Step 5 — Configure MySQL Database for WordPress

With MySQL installed and secured, the next step is to configure it so that WordPress can store and retrieve data properly. WordPress needs a dedicated database and a database user with the right permissions. This ensures that WordPress only has access to its own data and doesn’t interfere with other databases.

### 🔹 Log in to MySQL

Use the root account to access the MySQL shell:
```bash
sudo mysql -u root -p
```
Enter the root password you created earlier in Step 4.

### 🔹 Create the WordPress Database
```mysql
CREATE DATABASE wordpress;
```
This creates a dedicated database named wordpress where all site data will be stored (posts, pages, themes, settings, etc.).

### 🔹 Create a Database User
```mysql
CREATE USER 'myuser'@'<web-server-private-ip-address>' IDENTIFIED BY 'mypass';
```
- myuser → a new user specifically for WordPress
- `<web-server-private-ip-address>` → replace this with the private IP of your web server EC2 instance. This ensures that only the web server can connect to the database, not the public internet.
- mypass → replace with a strong password.

### 🔹 Grant Permissions
```mysql
GRANT ALL PRIVILEGES ON wordpress.* TO 'myuser'@'<web-server-private-ip-address>';
```
This gives the new user (myuser) full control over the wordpress database only (not other databases). This follows the principle of least privilege.

### 🔹 Apply Changes
```mysql
FLUSH PRIVILEGES;
```
This reloads the MySQL privileges so the changes take effect immediately.

### 🔹 Verify the Database
```mysql
SHOW DATABASES;
```
You should see wordpress listed among the databases.
![images](/images/54.png)

Finally, exit the MySQL shell:
```mysql
EXIT;
```

**✅ At this stage, you now have:**
-  A dedicated WordPress database
-  A secure database user tied to your web server’s private IP
-  Proper privileges for WordPress to store and manage its data

Next, we’ll configure WordPress to connect to this database and complete the setup.

---

## Step 6 — Configure WordPress to Connect to the Remote Database

By default, WordPress connects to a local database. Since our database is on a separate EC2 instance, we must configure connectivity between the web server and the database server.

### 🔹 Open MySQL Port (3306) on the DB Server

On your DB server EC2, ensure that port 3306 (the default MySQL port) is open.
For security, allow inbound traffic on this port only from your web server’s private IP.
![images](/images/55.png)

This prevents outside hosts from connecting directly to your database.

### 🔹 Install MySQL Client on the Web Server

To verify connectivity, install the MySQL client on the web server EC2:
```bash
sudo rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo yum install mysql -y
```
![images](/images/56.png)
This installs the MySQL client, which allows the web server to connect to the remote DB server.

### 🔹 Test Database Connection

From the web server, connect to the DB server:
```bash
mysql -u <admin> -p -h <db-server-private-ip-address>
```
- Replace <admin> with the MySQL user you created in Step 5 (myuser).
- Replace <db-server-private-ip-address> with the private IP of your DB server EC2 instance.
![images](/images/57.png)

### 🔹 Verify the Connection

Once logged in, confirm that the WordPress database exists:
```mysql
SHOW DATABASES;
```
![images](/images/58.png)
You should see wordpress listed. If successful, our web server is now able to connect to the database server over the internal AWS network.

---

## Step 7 — Configure Apache Permissions and WordPress Setup

Now that the database connectivity is ready, we need to ensure that Apache (httpd) can properly serve WordPress. By default, Apache may not have the right ownership or SELinux context to access the WordPress files, so let’s fix that.

### 🔹 Change Ownership of WordPress Files

Set Apache (apache user in RHEL) as the owner of the WordPress directory:
```bash
sudo chown -R apache:apache /var/www/html/wordpress
```
This ensures that Apache has full ownership of all WordPress files and directories.

### 🔹 Set Correct Permissions

Next, configure directory and file permissions so Apache can read and write where necessary:
```bash
sudo find /var/www/html/wordpress/ -type d -exec chmod 755 {} \;
sudo find /var/www/html/wordpress/ -type f -exec chmod 644 {} \;
```
![images](/images/59.png)
- Directories (755): Read, write, execute for owner; read + execute for group and others.
- Files (644): Read/write for owner; read-only for group and others.

### 🔹 Configure SELinux Context (If Enabled)

If SELinux is enforcing, allow Apache to access WordPress content:
```bash
sudo chcon -t httpd_sys_content_t /var/www/html/wordpress -R
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress/wp-content -R
```
- httpd_sys_content_t → grants Apache read-only access to WordPress files.
- httpd_sys_rw_content_t → allows write access (needed for uploads and plugins).

### 🔹 Configure Apache Virtual Host

Edit Apache configuration to point to your WordPress directory:
```bash
sudo vi /etc/httpd/conf.d/wordpress.conf
```
![images](/images/60a.png)

Add the following configuration:
```bash
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/wordpress
    ServerName <your-domain-or-public-ip>

    <Directory /var/www/html/wordpress>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/wordpress_error.log
    CustomLog /var/log/httpd/wordpress_access.log combined
</VirtualHost>
```
![images](/images/60.png)

Save and exit.

### 🔹 Enable and Restart Apache

Finally, reload Apache to apply changes:

```bash
sudo systemctl restart httpd
sudo systemctl enable httpd
```

---

## Step 8 — Configure wp-config.php and Complete WordPress Installation

At this point, Apache is ready to serve WordPress. The final step is to configure WordPress so it can connect to the remote MySQL database created in Step 5.

### 🔹 Step 8.1: Create the wp-config.php File

Inside the WordPress directory, copy the sample configuration file:
```bash
cd /var/www/html/wordpress
sudo cp wp-config-sample.php wp-config.php
```

### 🔹 Step 8.2: Edit the wp-config.php File

Open the file and update it with your database details:
```bash
sudo vi wp-config.php
```

Update these lines:

```php
/** The name of the database for WordPress */
define( 'DB_NAME', 'database_name' );

/** MySQL database username */
define( 'DB_USER', 'myuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'mypass' );

/** MySQL hostname */
define( 'DB_HOST', '<db-server-private-ip-address>' );
```
![images](/images/62.png)

Save and exit.

### 🔹 Step 8.3: Set Proper Permissions

Ensure `wp-config.php` has restricted permissions for security:
```bash
sudo chmod 640 wp-config.php
```

### 🔹 Step 8.4: Access WordPress from Browser

Now, go to your web server’s public IP or domain in your browser:
```bash
http://<your-web-server-public-ip>
```
![images](/images/63.png)

You should see the WordPress Setup Page. Complete the installation by providing:

- Site Title
- Admin Username
- Admin Password
- Email Address
- Click Install WordPress.
![images](/images/64.png)
![images](/images/65.png)

✅ Congratulations! 🎉 You’ve now fully deployed WordPress on a Red Hat-based LAMP stack with the web and database servers separated.

---

# 🔧 Final Notes & Troubleshooting

During this project, I ran into several challenges while configuring the two-tier WordPress deployment. Below are the issues, what caused them, and how I solved them:

#### 1️⃣ MySQL Not Accepting Remote Connections
- Symptom: WordPress couldn’t connect to the database, even though the MySQL service was running.
- Cause: By default, MySQL only listens locally (127.0.0.1). Remote access from the web server was blocked.
- Fix:

    - Verified listening ports with:
        ```bash
        sudo ss -tulnp | grep mysqld
        ```
        Confirmed it was listening on *:3306.

> Opened port 3306 in the DB server’s Security Group, but restricted it only to the web server’s private IP for security.

#### 2️⃣ Apache Not Serving WordPress Properly
- Symptom: Visiting the web server’s public IP showed a blank page or Apache test page instead of WordPress.
- Cause: SELinux and directory permissions were restricting Apache from accessing /var/www/html/wordpress.
- Fix:

  - Adjusted permissions so Apache could read the WordPress files:
    ```bash
    sudo chown -R apache:apache /var/www/html/wordpress
    sudo chmod -R 755 /var/www/html/wordpress
    ```

  - Allowed SELinux to serve content from the directory:
    ```bash
    sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
    ```

#### 3️⃣ wp-config.php Permissions
- Symptom: Security concerns with WordPress complaining about insecure file permissions.
- Fix: Restricted the config file to only be readable by root and Apache:
    ```bash
    sudo chmod 640 /var/www/html/wordpress/wp-config.php
    ```

#### 4️⃣ Database User Privileges
- Symptom: Even after creating the myuser, WordPress threw "Error establishing a database connection".
- Cause: The database user was created but without the correct host entry or privileges.
- Fix: Explicitly granted permissions to the web server private IP:

    ```sql
    CREATE USER 'myuser'@'<web-server-private-ip>' IDENTIFIED BY 'mypass';
    GRANT ALL PRIVILEGES ON wordpress.* TO 'myuser'@'<web-server-private-ip>';
    FLUSH PRIVILEGES;
    ```

#### 5️⃣ MySQL Client Testing

Lesson Learned: Always test DB connectivity from the web server before configuring WordPress.
```bash
mysql -u myuser -p -h <db-server-private-ip>
SHOW DATABASES;
```

---

## ✅ Key Takeaways

- Always separate web and database servers for scalability and security.

- Use least-privilege principles for MySQL users.

- Double-check SELinux, permissions, and firewall rules — they’re often the silent blockers.

- Troubleshoot step by step:

  - Can the DB accept remote connections?

  - Can the web server reach the DB server?

  - Are WordPress configs correct?

👉 With these lessons, you’ll avoid hours of debugging and deploy WordPress more efficiently on Red Hat EC2 instances.