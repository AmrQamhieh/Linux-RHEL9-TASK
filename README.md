### Linux-RHEL9-TASK

# 🧩 Part 1 – LVM Configuration

## 🎯 Objective  
Create an LVM setup on the second disk `/dev/sdb`, with:  
- Physical extents = 16 MB  
- Logical volume = 50 extents  
- Filesystem = `ext4`  
- Mounted automatically at `/mnt/data`

---

## ⚙️ Steps Performed
### 1️⃣ Check the Disk
```bash
fdisk /dev/sdb
```
- Created a new DOS partition table.
- Added one primary partition (/dev/sdb1).

### 2️⃣ Create Physical Volume (PV)
```bash
  pvcreate /dev/sdb1
  ```
  Verification:

```bash
pvdisplay
```
Output Example:
```pgsql
PV Name               /dev/sdb1
VG Name               myvg
PV Size               1023.00 MiB / not usable 15.00 MiB
PE Size               16.00 MiB
Total PE              63
Allocated PE          50
Free PE               13
```

### 3️⃣ Create Volume Group (VG)
```bash
vgcreate myvg /dev/sdb1
```
Verification:
```bash
vgdisplay
```
Output
```pgsql
VG Name               myvg
VG Access             read/write
VG Status             resizable
PE Size               16.00 MiB
Total PE              63
Alloc PE / Size       50 / 800.00 MiB
Free  PE / Size       13 / 208.00 MiB
```

### 4️⃣ Create Logical Volume (LV)
```bash
lvcreate -n mylv -l 50 myvg
```

Verification:
```bash
lvdisplay
```
Output:
```pgsql
LV Path                /dev/myvg/mylv
LV Name                mylv
VG Name                myvg
LV Status              available
LV Size                800.00 MiB
Current LE             50
```
### 5️⃣ Format the LV with ext4
```
mkfs.ext4 /dev/myvg/mylv
```
Example Output:
```yaml
Creating filesystem with 204800 4k blocks and 51296 inodes
Filesystem UUID: b0ab29b5-a60d-4340-993f-42754fbc1e85
Writing inode tables: done
Writing superblocks and filesystem accounting information: done
```
### 6️⃣ Mount Point & Automatic Mounting
Create mount directory:
```bash
mkdir /mnt/data
```
Edit /etc/fstab:
vim /etc/fstab
Added the following line at the bottom:
```bash
/dev/myvg/mylv    /mnt/data    ext4    defaults    0 0
```
Applied the changes:
```bash
mount -a
```
Verfication;
```bash
df -h | grep data
```
✅ Output should confirm /mnt/data is mounted successfully.
_________________________________________________________________________

## 👥 Part 2 – Users, Groups & Permissions

### 🎯 Objective
- Create `user1` (UID=601, password=redhat, non-interactive shell)  
- Add `user1` to `TrainingGroup`  
- Create `user2`, `user3` (both in `admin` group)  
- Give `user3` root privileges (`wheel` group)

---

### ⚙️ Steps
```bash
# Create user1
useradd -u 601 -p redhat -s /sbin/nologin user1
passwd user1

# Create TrainingGroup and add user1
groupadd TrainingGroup
usermod -aG TrainingGroup user1

# Create user2 and user3
useradd user2 && useradd user3
passwd user2 && passwd user3

# Add both to admin group
usermod -aG admin user2
usermod -aG admin user3

# Give user3 root privileges
usermod -aG wheel user3
```

✅ Verification:
```bash
id user1   # shows groups: user1, TrainingGroup
id user2   # shows groups: user2, admin
id user3   # shows groups: user3, admin, wheel
```
Result: All users, groups, and permissions configured successfully.
___________________________________________________________________

## 🔐 Part 3 – SSH Key Authentication (Passwordless Login)

### 🎯 Objective
Generate an SSH key pair and connect to another VM without using a password.

---

### ⚙️ Steps
```bash
# Generate SSH key pair
ssh-keygen

# Copy the public key to the remote host
ssh-copy-id root@10.0.2.15

# Test passwordless connection
ssh root@10.0.2.15
```
✅ Verification

Connection to 10.0.2.15 established without password prompt

Keys stored at /root/.ssh/id_rsa and /root/.ssh/id_rsa.pub

Result: SSH key-based authentication configured successfully.
______________________________________________________________

## 🧾 Part 4 – File Permissions (ACL)

### 🎯 Objective
Copy `/etc/fstab` to `/var/tmp/admin` and set permissions so:
- `user1` → can read, write, and modify  
- `user2` → has no access  

---

### ⚙️ Steps
```bash
cp /etc/fstab /var/tmp/admin
setfacl -m u:user1:rwx /var/tmp/admin
setfacl -m u:user2:--- /var/tmp/admin
```
✅ Verification
```bash
getfacl /var/tmp/admin
```
Shows:
```bash
user:user1:rwx
user:user2:---
```
Result: ACL permissions applied successfully.
___________________________________________________________

## 🛡️ Part 5 – SELinux Configuration

### 🎯 Objective
Ensure SELinux is **Enforcing** permanently (persists after reboot).

---

### ⚙️ Steps
```bash
vim /etc/selinux/config
```
# Change the line:
```
SELINUX=enforcing
```
### Then verify:
```bash
getenforce
# or
sestatus
```
✅ Result

SELinux mode is Enforcing and remains active after reboot.
__________________________________________________________

## ⚙️ Part 6 – Bash Script & Processes

### 🎯 Objective
Create a shell script that runs for 10 minutes in the background, verify it’s running, then kill it.

---

### ⚙️ Steps
```bash
touch file.sh
vim file.sh
```
Script content:
```bash
echo "This file will be running for 10 minutes in the background :')"
sleep 600
```
Run in background:
```
bash file.sh &
```
Check process:
```
ps aux | grep file.sh
```
Kill process:
```
kill <PID>
```

✅ Result
Script runs for 10 minutes in the background and can be manually terminated using kill.
__________________________________________________________________________


## 📦 Part 7 – Yum Repository (Zabbix Local HTTP Repo)

### 🎯 Objective
Create a **local Yum repository** (available via HTTP) hosting Zabbix RPMs, disable all other repos, and install Zabbix packages from it.

---

### 🖥️ On the Repository Server (has Internet)
```bash
# 1. Install Apache and tools
dnf install -y httpd wget createrepo dnf-plugins-core
systemctl enable --now httpd

# 2. Add official Zabbix repo for RHEL 9 (7.0 branch)
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-7.0-4.el9.noarch.rpm
dnf clean all && dnf repolist

# 3. Download Zabbix packages + dependencies
mkdir ~/zabbix_rpms && cd ~/zabbix_rpms
dnf download --resolve zabbix-server-mysql zabbix-web-mysql zabbix-agent php

# 4. Move RPMs to Apache web directory
mkdir -p /var/www/html/zabbix
cp ~/zabbix_rpms/*.rpm /var/www/html/zabbix/

# 5. Create repo metadata
createrepo /var/www/html/zabbix

# 6. Set permissions and restart Apache
chown -R apache:apache /var/www/html/zabbix
restorecon -Rv /var/www/html/zabbix
systemctl restart httpd

# 7. Verify
curl http://<repo-server-ip>/zabbix/repodata/
```
### 💻 On Each Client VM (no Internet)
```bash
# 1. Disable all other repos
dnf config-manager --set-disabled '*'

# 2. Create new repo file
vim /etc/yum.repos.d/zabbix-local.repo

[zabbix-local]
name=Zabbix Local HTTP Repo
baseurl=http://10.0.2.10/zabbix
enabled=1
gpgcheck=0


# 3. Clean cache and verify repo
dnf clean all
dnf makecache

# 4. Install from local repo
dnf install -y zabbix-server-mysql zabbix-web-mysql zabbix-agent
```
✅ Result

Zabbix 7.0 repo for RHEL 9 hosted via Apache

Clients install packages without Internet access

All external repos disabled — only local HTTP repo in use
-------------------------------------------------------------------------

## 🌐 Part 8 – Network Management

### 🎯 Objective
- Open ports **80** (HTTP) and **443** (HTTPS)  
- Make the changes **permanent**  
- Block SSH access from a specific colleague IP

---

### ⚙️ Steps
```bash
# 1. Open HTTP and HTTPS ports
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp

# 2. Reload firewall to apply changes
firewall-cmd --reload

# 3. Verify open ports
firewall-cmd --list-ports

# 4. Block SSH from a specific IP (example: 192.168.1.50)
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.50" service name="ssh" reject'

# 5. Reload and verify rules
firewall-cmd --reload
firewall-cmd --list-rich-rules
```
✅ Result

Ports 80 and 443 opened and persistent across reboots

SSH blocked for the specified IP successfully
----------------------------------------------------------------------

## ⏰ Part 9 – Cronjob (User Log Collection)

### 🎯 Objective
Create a cronjob that runs daily at **1:30 AM**, collects logged-in users, and saves the output in a file in the format:  
`timestamp – users`

---

### ⚙️ Steps
```bash
# 1. Create a script to log users
vim /usr/local/bin/log_users.sh
```
#Script content:
```bash
#!/bin/bash
echo "$(date '+%Y-%m-%d %H:%M:%S') - $(who | awk '{print $1}' | sort | uniq)" >> /var/log/users.log
```
Make it executable:
```bash
chmod +x /usr/local/bin/log_users.sh
```
🕐 Add Cronjob
```bash
crontab -e
```
Add:
```bash
30 1 * * * /usr/local/bin/log_users.sh
```
## ✅ Result

Script runs daily at 1:30 AM

Logged-in users appended to /var/log/users.log with timestamps
----------------------------------------------------------------
## 🗄️ Part 10 – MariaDB Setup & Database Configuration

### 🎯 Objective
Install **MariaDB** from the local repo, configure access, create a database and user, and verify connection using the new credentials.

---

### ⚙️ Steps
```bash
# 1. Install MariaDB from the local repo
dnf install -y mariadb-server --disablerepo='*' --enablerepo=zabbix-local

# 2. Enable and start the service
systemctl enable --now mariadb

# 3. Secure the installation
mysql_secure_installation
```
I Followed prompts to set the root password and remove test users.

# 🧱 Create Database and User
```bash
mysql -u root -p
```
Inside MariaDB shell:
```bash
CREATE DATABASE studentdb;
CREATE USER 'studentuser'@'localhost' IDENTIFIED BY 'pass123';
GRANT ALL PRIVILEGES ON studentdb.* TO 'studentuser'@'localhost';
FLUSH PRIVILEGES;
```
# 🔍 Verify Access
```bash
mysql -u studentuser -p studentdb
```
# 🧩 Example Data
Inside the studentdb:
```sql
CREATE TABLE students (
  student_number VARCHAR(10),
  firstname VARCHAR(20),
  lastname VARCHAR(20),
  program VARCHAR(30),
  grad_year INT
);

INSERT INTO students VALUES
('110-001', 'Barry', 'Allen', 'Mechanical', 2017),
('110-002', 'David', 'Brown', 'Mechanical', 2017),
('110-003', 'Mary', 'Green', 'Mechanical', 2018);
```

# Verification:

```sql
SELECT * FROM students
```

# ✅ Result

MariaDB installed and running from the local repo

Database studentdb and user studentuser created

Data inserted successfully and verified with SQL queries























