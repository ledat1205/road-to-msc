# ✅ **1. Basic journalctl output (default)**

`Dec 01 10:15:23 ubuntu-server systemd[1]: Started Daily apt upgrade and clean activities. Dec 01 10:15:24 ubuntu-server CRON[12345]: pam_unix(cron:session): session opened for user root(uid=0) by (uid=0) Dec 01 10:15:24 ubuntu-server CRON[12345]: pam_unix(cron:session): session closed for user root Dec 01 10:15:27 ubuntu-server sshd[2298]: Accepted publickey for ubuntu from 192.168.1.50 port 53522 ssh2: RSA SHA256:abcd1234 Dec 01 10:15:27 ubuntu-server sshd[2298]: pam_unix(sshd:session): session opened for user ubuntu(uid=1000) by (uid=0) Dec 01 10:15:31 ubuntu-server kernel: [  521.123456] eth0: link becomes ready Dec 01 10:15:33 ubuntu-server systemd[1]: Stopping User Manager for UID 1000...`

---

# ✅ **2. `journalctl -u nginx` (service logs)**

`Dec 01 11:00:12 ubuntu-server systemd[1]: Starting A high performance web server and a reverse proxy server... Dec 01 11:00:12 ubuntu-server nginx[2145]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok Dec 01 11:00:12 ubuntu-server nginx[2145]: nginx: configuration file /etc/nginx/nginx.conf test is successful Dec 01 11:00:12 ubuntu-server systemd[1]: Started A high performance web server and a reverse proxy server. Dec 01 11:05:41 ubuntu-server nginx[2149]: 192.168.1.101 - - [01/Dec/2025:11:05:41 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0" Dec 01 11:06:02 ubuntu-server nginx[2149]: 192.168.1.102 - - [01/Dec/2025:11:06:02 +0000] "GET /favicon.ico HTTP/1.1" 404 162 "-" "Mozilla/5.0"`

---

# ✅ **3. `journalctl -k` (kernel logs)**

`Dec 01 10:44:12 ubuntu-server kernel: [    0.000000] Linux version 5.15.0-105-generic (buildd@ubuntu) ... Dec 01 10:44:12 ubuntu-server kernel: [    2.345678] ata1: SATA link up 6.0 Gbps (SStatus 133 SControl 300) Dec 01 10:44:12 ubuntu-server kernel: [    2.567890] EXT4-fs (nvme0n1p2): mounted filesystem with ordered data mode Dec 01 10:44:14 ubuntu-server kernel: [   15.234567] IPv6: ADDRCONF(NETDEV_UP): eth0: link is not ready Dec 01 10:44:16 ubuntu-server kernel: [   17.123456] eth0: link becomes ready Dec 01 10:45:57 ubuntu-server kernel: [  118.123456] Out of memory: Killed process 1324 (java) total-vm:12G, anon-rss:8G`

---

# ✅ **4. `journalctl -f` (tailing logs live)**

`Dec 01 11:14:32 ubuntu-server sshd[3542]: Failed password for root from 203.0.113.42 port 44512 ssh2 Dec 01 11:14:34 ubuntu-server sshd[3542]: Failed password for root from 203.0.113.42 port 44512 ssh2 Dec 01 11:14:36 ubuntu-server sshd[3542]: Failed password for root from 203.0.113.42 port 44512 ssh2 Dec 01 11:14:39 ubuntu-server sshd[3542]: Received disconnect from 203.0.113.42: 11: Bye Bye Dec 01 11:14:45 ubuntu-server sshd[3551]: Accepted password for ubuntu from 192.168.1.50 port 54788 ssh2`

---

# ✅ **5. `journalctl --priority=3` (error logs only)**

`Dec 01 08:54:23 ubuntu-server systemd[1]: Failed to start MySQL Server. Dec 01 08:54:23 ubuntu-server mysqld[1874]: ERROR: Can't find messagefile '/usr/share/mysql/errmsg.sys' Dec 01 09:12:56 ubuntu-server sshd[2012]: error: kex_exchange_identification: Connection closed by remote host Dec 01 09:13:20 ubuntu-server kernel: [  345.123456] ext4-disk corruption detected on /dev/sda1`

# ✅ **6. `journalctl -b` (logs from current boot only)**

`-- Logs begin at Mon 2025-12-01 10:10:01 +0000, end at Mon 2025-12-01 12:04:03 +0000. -- Dec 01 10:10:02 ubuntu kernel: [    0.000000] Command line: BOOT_IMAGE=/vmlinuz-5.15.0 root=/dev/nvme0n1p2 ro quiet splash Dec 01 10:10:03 ubuntu systemd[1]: Mounted /boot/efi. Dec 01 10:10:05 ubuntu systemd[1]: Started Network Manager. Dec 01 10:10:06 ubuntu systemd[1]: Reached target Multi-User System. Dec 01 10:10:09 ubuntu systemd[1]: Started OpenSSH server daemon.`

---

# ✅ **7. `journalctl -b -1` (previous boot)**

`-- Reboot -- Nov 30 22:18:44 ubuntu systemd[1]: Shutting down. Nov 30 22:18:45 ubuntu systemd-shutdown[1]: Syncing filesystems and block devices... Nov 30 22:18:47 ubuntu kernel: [  176.123456] ACPI: Preparing to enter system sleep state S5 Nov 30 22:18:47 ubuntu systemd-shutdown[1]: Powering off.`

---

# ✅ **8. `journalctl -xe` (errors with explanations)**

This shows errors + context + suggestions.

`Dec 01 11:20:34 ubuntu sshd[3882]: Failed password for invalid user admin from 141.98.10.2 port 51555 ssh2 Dec 01 11:20:34 ubuntu sshd[3882]: Received disconnect from 141.98.10.2 port 51555:11: Bye Bye Dec 01 11:20:34 ubuntu sshd[3882]: Disconnected from 141.98.10.2 port 51555 Dec 01 11:20:34 ubuntu sshd[3882]: PAM service(sshd) ignoring max retries; 4 > 3 Dec 01 11:20:34 ubuntu systemd[1]: ssh.service: Failed with result 'authentication-failure'. Dec 01 11:20:34 ubuntu systemd[1]: Failed to start OpenBSD Secure Shell server. Hint: Some lines were ellipsized, use -l to show in full.`

---

# ✅ **9. `journalctl -u docker` (Docker logs)**

`Dec 01 11:30:13 ubuntu dockerd[1024]: time="2025-12-01T11:30:13.123456789Z" level=info msg="Starting Docker" Dec 01 11:30:14 ubuntu dockerd[1024]: time="2025-12-01T11:30:14.987123456Z" level=warning msg="overlay: the backing xfs filesystem is formatted without d_type support" Dec 01 11:30:17 ubuntu dockerd[1024]: time="2025-12-01T11:30:17.123456789Z" level=info msg="Daemon has completed initialization"`

---

# ✅ **10. `journalctl -u kubelet` (Kubernetes node logs)**

`Dec 01 11:32:41 worker-node kubelet[2233]: Starting kubelet v1.30.2 Dec 01 11:32:42 worker-node kubelet[2233]: error: Image pull failed for "my-app:latest": ErrImagePull Dec 01 11:32:47 worker-node kubelet[2233]: Back-off pulling image "my-app:latest" Dec 01 11:33:15 worker-node kubelet[2233]: Created pod sandbox: 1234abcd-5678 Dec 01 11:33:20 worker-node kubelet[2233]: Started container app`

---

# ✅ **11. `journalctl -u cron` (cron logs)**

`Dec 01 12:00:01 ubuntu CRON[4423]: pam_unix(cron:session): session opened for user root by (uid=0) Dec 01 12:00:01 ubuntu CRON[4423]: (/usr/bin/php /var/www/backup.php) completed with exit status 0 Dec 01 12:00:01 ubuntu CRON[4423]: pam_unix(cron:session): session closed for user root`

---

# ✅ **12. `journalctl -u ssh` (SSH service logs)**

`Dec 01 12:10:32 ubuntu sshd[5042]: Accepted publickey for ubuntu from 192.168.0.50 port 40022 ssh2  Dec 01 12:10:32 ubuntu sshd[5042]: pam_unix(sshd:session): session opened for user ubuntu Dec 01 12:12:43 ubuntu sshd[5042]: pam_unix(sshd:session): session closed for user ubuntu`

---

# ✅ **13. `journalctl -t <process>` (logs for a specific syslog tag)**

Example: `journalctl -t sudo`

`Dec 01 12:15:44 ubuntu sudo[6234]:    ubuntu : TTY=pts/0 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/bin/systemctl restart nginx Dec 01 12:15:44 ubuntu sudo[6234]: pam_unix(sudo:session): session opened for user root Dec 01 12:15:44 ubuntu sudo[6234]: pam_unix(sudo:session): session closed for user root`

---

# ✅ **14. Logs filtered by user (`--user`)**

`Dec 01 12:20:05 ubuntu systemd[873]: Started /usr/bin/code. Dec 01 12:21:02 ubuntu systemd[873]: app-gnome-terminal-5678.scope: Succeeded.`

---

# ✅ **15. Time-filtering example**

### _Last 1 hour_

`Dec 01 12:04:02 ubuntu systemd[1]: Starting Cleanup of Temporary Directories... Dec 01 12:04:03 ubuntu systemd[1]: Finished Cleanup of Temporary Directories.`