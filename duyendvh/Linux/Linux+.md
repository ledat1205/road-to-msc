.

# 🖥️ SYSTEM MANAGEMENT (23%)

## 1. Linux basics

**Identify boot process steps, kernel, filesystems, architectures**

### What it is

Understanding how Linux **starts**, what the **kernel** does, how filesystems are organized, and how Linux differs across CPU architectures.

### Key concepts

- Boot sequence: **Firmware → Bootloader → Kernel → init/systemd → services**
    
- Kernel = core of OS (hardware abstraction, process/memory mgmt)
    
- Filesystems = ext4, xfs, btrfs
    
- Architectures = x86_64, ARM64, etc.
    

### Examples

`systemctl list-units --type=service uname -r lsblk -f df -Th arch`

### Practice advice

- Draw the boot flow **from memory**
    
- Break & fix:
    
    - Edit GRUB kernel parameters (temporarily)
        
    - Boot into rescue mode
        
- Compare `/`, `/boot`, `/var`, `/etc` and explain their purpose **out loud**
    

---

## 2. Device management

**Manage kernel modules, hardware components, and device utilities**

### What it is

Linux treats hardware as **devices**, often represented as files under `/dev`.  
Kernel modules are drivers loaded dynamically.

### Examples

`lsmod modprobe usb_storage ls /dev udevadm info --query=all --name=/dev/sda lspci lsusb`

### Practice advice

- Plug/unplug USB → observe `/dev`
    
- Load/unload a harmless module
    
- Learn **when to use**:
    
    - `lspci` (PCI)
        
    - `lsusb` (USB)
        
    - `lsblk` (storage)
        

---

## 3. Storage management

**Configure LVM, RAID, partitions, and mounted storage**

### What it is

How Linux **organizes disks**, from raw devices → partitions → logical volumes → mounted filesystems.

### Examples

`fdisk /dev/sdb pvcreate /dev/sdb1 vgcreate data_vg /dev/sdb1 lvcreate -L 5G -n data_lv data_vg mkfs.ext4 /dev/data_vg/data_lv mount /dev/data_vg/data_lv /mnt/data`

RAID:

`mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdc /dev/sdd`

### Practice advice

- Create **throwaway virtual disks**
    
- Practice:
    
    - Extend LVM volume **without downtime**
        
    - Fix a missing mount in `/etc/fstab`
        
- Understand **why LVM exists** (flexibility)
    

---

## 4. Network configuration

**Set up hosts, DNS, interfaces, and network tools**

### What it is

Linux networking: IPs, routing, DNS, hostname resolution.

### Examples

`ip addr ip route nmcli device status cat /etc/resolv.conf cat /etc/hosts ping ss -tulnp`

### Practice advice

- Manually assign static IP
    
- Break DNS → fix it
    
- Understand difference:
    
    - `/etc/hosts` vs DNS
        
    - `ping` vs `curl`
        

---

## 5. Shell operations

**Navigation, editing, redirection, environment variables**

### What it is

Command-line fluency — this is **everywhere** in the exam.

### Examples

`cd ../ grep "error" file.log > errors.txt cat file | awk '{print $1}' export PATH=$PATH:/opt/bin vim file.txt`

### Practice advice

- Live in terminal
    
- Practice **pipes and redirection daily**
    
- Rewrite commands without copy-paste
    
- Learn `bash` shortcuts (`Ctrl+A`, `Ctrl+R`)
    

---

## 6. Backups and restores

**Archiving, compression, data recovery**

### What it is

Protecting data using archives and compression.

### Examples

`tar -czvf backup.tar.gz /etc tar -xzvf backup.tar.gz rsync -av /data /backup gzip file gunzip file.gz`

### Practice advice

- Backup → delete original → restore
    
- Compare:
    
    - `tar` vs `rsync`
        
- Know **when compression helps**
    

---

## 7. Virtualization

**Deploy hypervisors, create VMs, manage disk images**

### What it is

Running **virtual machines** using KVM/QEMU or tools like VirtualBox.

### Examples

`virt-install virsh list qemu-img create -f qcow2 disk.qcow2 10G`

### Practice advice

- Use **VirtualBox or KVM**
    
- Create VM → snapshot → revert
    
- Understand VM vs container difference (important!)
    

---

# 👥 SERVICES & USER MANAGEMENT (20%)

## 1. Files & directories

**Permissions, links, special files**

### What it is

Linux access control via permissions and ownership.

### Examples

`chmod 755 script.sh chown user:group file ln file hardlink ln -s file symlink`

Special bits:

- SUID, SGID, sticky bit
    

### Practice advice

- Predict permissions before running `ls -l`
    
- Test access using different users
    

---

## 2. Account management

**Users and groups**

### Examples

`useradd alice passwd alice usermod -aG sudo alice groupadd devs`

### Practice advice

- Create role-based users
    
- Lock/unlock accounts
    
- Understand `/etc/passwd`, `/etc/shadow`
    

---

## 3. Process control

**States, priorities, scheduling**

### Examples

`ps aux top nice -n 10 command renice 5 -p PID crontab -e at now + 5 minutes`

### Practice advice

- Kill runaway processes
    
- Schedule backups with `cron`
    
- Explain zombie vs sleeping process
    

---

## 4. Software management

**Packages and repositories**

### Examples

`apt update && apt install nginx dnf install httpd apt remove nginx`

### Practice advice

- Add/remove repos
    
- Downgrade a package
    
- Know distro differences
    

---

## 5. Systems management

**Services, logs, timers**

### Examples

`systemctl start nginx systemctl enable nginx journalctl -u nginx systemctl list-timers`

### Practice advice

- Replace cron with systemd timers
    
- Debug failed services via logs
    

---

## 6. Containers

**Runtimes, images, networks**

### Examples

`docker run -d -p 80:80 nginx docker images docker network create app_net`

### Practice advice

- Run same app in VM vs container
    
- Bind mount vs volume
    
- Understand **why containers are not VMs**
    

---

# 🔐 SECURITY (18%)

## 1. Auth & accounting

**PAM, LDAP, Kerberos, auditing**

### What it is

Authentication stack & tracking actions.

### Examples

`cat /etc/pam.d/login auditctl -l`

### Practice advice

- Trace login flow
    
- Enable auditing on file access
    

---

## 2. Firewalls

**iptables, nftables, UFW**

### Examples

`ufw allow ssh ufw enable iptables -L`

### Practice advice

- Block everything, allow SSH only
    
- Lock yourself out → recover via console
    

---

## 3. OS hardening

**Permissions, sudo, SSH**

### Examples

`visudo PermitRootLogin no`

### Practice advice

- Harden SSH
    
- Principle of least privilege
    

---

## 4. Account security

**Password policies, shells, MFA**

### Examples

`chage -l user usermod -s /sbin/nologin user`

### Practice advice

- Enforce password aging
    
- Disable unused accounts
    

---

## 5. Cryptography

**Encryption, hashing, certificates**

### Examples

`openssl sha256 file gpg -c secret.txt`

### Practice advice

- Compare hash vs encryption
    
- Encrypt → decrypt files
    

---

## 6. Compliance

**Integrity, scanning, standards**

### Examples

`aide --check`

### Practice advice

- Baseline system
    
- Detect unauthorized changes
    

---

# ⚙️ AUTOMATION & SCRIPTING (17%)

## Automation (Ansible, CI/CD)

- Write playbooks
    
- Automate installs
    

## Shell scripting

`for f in *.log; do   grep error $f done`

## Python basics

- Files, loops, packages
    

## Git

`git clone git commit git tag`

## AI best practices

- Verify generated scripts
    
- Never run blindly as root
    

---

# 🛠️ TROUBLESHOOTING (22%)

This is **integration of everything above**.

### Practice method (very important):

1. Break system intentionally
    
2. Use logs
    
3. Verify assumptions
    
4. Fix root cause
    

### Tools to master

`journalctl dmesg top htop free -m iostat ss`