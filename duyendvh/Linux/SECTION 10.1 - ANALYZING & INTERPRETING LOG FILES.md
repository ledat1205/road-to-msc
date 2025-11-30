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