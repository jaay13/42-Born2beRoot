*This project has been created as part of the 42 curriculum by jakoch.*

# Born2beRoot

## Table of Contents

- [Description](#description)
- [Instructions](#instructions)
- [System Overview](#system-overview)
- [Mandatory Part](#mandatory-part)
  - [Sudo](#sudo)
  - [SSH](#ssh)
  - [UFW Firewall](#ufw-firewall)
  - [Hostname](#hostname)
  - [Users and Groups](#users-and-groups)
  - [Password Policy](#password-policy)
  - [AppArmor](#apparmor)
  - [LVM and Partitions](#lvm-and-partitions)
  - [Monitoring Script and Cron](#monitoring-script-and-cron)
- [Bonus Part](#bonus-part)
  - [WordPress Stack (lighttpd + MariaDB + PHP)](#wordpress-stack)
  - [Fail2ban](#fail2ban)
- [Technical Choices and Comparisons](#technical-choices-and-comparisons)
- [Cheat-Sheet](#cheat-sheet)
- [Signature & Evaluation Procedure](#signature--evaluation-procedure)
- [What I Learned](#what-i-learned)
- [Resources](#resources)

---

## Description

Born2beRoot is a system administration project whose goal is to set up a
minimal, secured server inside a virtual machine while following a strict set
of rules around partitioning, security, user management, and monitoring.

The server runs **Debian 13 (Trixie)** in **VirtualBox** with no graphical
interface. The disk is fully encrypted with LUKS and organised using LVM. A
strong password policy, a strict `sudo` configuration, an SSH service on a
non-default port, and an active firewall enforce a security-first setup. A
`bash` monitoring script broadcasts key system information to every terminal
every 10 minutes via `cron`.

On top of the mandatory part, this project includes the full bonus: a working
WordPress website (lighttpd + MariaDB + PHP) and Fail2ban as an extra security
service.

---

## Instructions

**Requirements:** VirtualBox and a Debian 13 (Trixie) netinst ISO.

1. Create the VM in VirtualBox and install Debian using **encrypted LVM**
   partitioning. Do **not** select any desktop environment during install.
2. Configure the base system in this order: `sudo`, users/groups, password
   policy, SSH, firewall, hostname (see the Mandatory Part below).
3. Add `monitoring.sh` and schedule it with `cron`.
4. (Bonus) Install the WordPress stack and Fail2ban.
5. Change **all** account passwords (including root) after the config files
   are in place.

> All passwords, database names and credentials in the commands below are
> shown as **placeholders** (`<...>`). Never commit real secrets to git.

---

## System Overview

| Item | Value |
|---|---|
| OS | Debian 13 (Trixie), minimal, no GUI |
| Hostname | `jakoch42` |
| Regular user | `jakoch` (groups: `sudo`, `user42`) |
| SSH port | `4242` (root login disabled) |
| Firewall | UFW active — ports `4242` and `80` (bonus) only |
| Security module | AppArmor (loaded at boot) |
| Disk | LUKS-encrypted, LVM volumes |
| Monitoring | `monitoring.sh` via `cron`, broadcast with `wall` |

Quick "who am I / where am I" commands:

```bash
hostnamectl            # hostname + OS + kernel
whoami                 # current user
uname -a               # architecture and kernel
lsblk                  # disk / partition layout
```

---

## Mandatory Part

### Sudo

`sudo` lets a permitted user run commands as another user (usually root). The
strict config lives in the sudoers file, edited only via `visudo` (it
validates syntax before saving).

```bash
sudo apt install sudo -y            # install sudo
sudo visudo                         # safely edit /etc/sudoers
sudo -V                             # check sudo version
```

Required `Defaults` lines added in `visudo`:

```
Defaults        passwd_tries=3                              # 3 tries then fail
Defaults        badpass_message="<custom message>"         # shown on wrong pw
Defaults        logfile="/var/log/sudo/sudo.log"           # log location
Defaults        log_input, log_output                      # log I/O
Defaults        iolog_dir="/var/log/sudo"                  # I/O log dir
Defaults        requiretty                                 # TTY mode required
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"   # restrict paths
```

```bash
sudo mkdir -p /var/log/sudo         # create the sudo log folder
sudo touch /var/log/sudo/sudo.log   # create the log file
sudo cat /var/log/sudo/sudo.log     # read logged sudo actions
```

### SSH

SSH (Secure Shell) gives an encrypted remote login to the server. Here it runs
on port **4242** and root may not log in. The server config is `sshd_config`
(the `d` = daemon = the server side).

```bash
sudo apt install openssh-server -y                 # install the SSH server
sudo nano /etc/ssh/sshd_config                     # set: Port 4242 / PermitRootLogin no
sudo systemctl restart ssh                         # apply the new port
sudo systemctl status ssh                          # check it's running
sudo ss -tlnp | grep 4242                          # confirm sshd listens on 4242
```

Connecting:

```bash
# From inside the VM (proves sshd is really on 4242):
ssh <user>@127.0.0.1 -p 4242

# From the host machine via VirtualBox NAT port forwarding.
# Forward set as: host port 4241 -> guest port 4242.
ssh <user>@localhost -p 4241
```

> Host and guest ports don't have to match. The guest must be 4242 (where
> sshd listens); the host port is just where you knock from your own machine.

### UFW Firewall

UFW ("Uncomplicated Firewall") is a simple front-end for the kernel firewall.
Only the ports we need are opened.

```bash
sudo apt install ufw -y             # install UFW
sudo ufw enable                     # turn the firewall on (and at boot)
sudo ufw allow 4242                 # allow SSH port
sudo ufw allow 80                   # allow HTTP (bonus WordPress only)
sudo ufw status                     # list active rules
sudo ufw status numbered            # same, with rule numbers (to delete)
sudo ufw delete <number>            # remove a rule by number
```

### Hostname

The hostname must be the login followed by `42`.

```bash
hostnamectl                                 # show current hostname
sudo hostnamectl set-hostname <newname>     # change it (used live at eval)
sudo nano /etc/hosts                        # update the 127.0.1.1 line to match
```

### Users and Groups

In addition to root, a user named after the login must exist and belong to the
`sudo` and `user42` groups.

```bash
sudo adduser <username>                 # create a user (+home, +password prompt)
sudo addgroup <groupname>               # create a group
sudo adduser <username> <groupname>     # add a user to a group
getent passwd <username>                # confirm the user exists
getent group <groupname>                # confirm membership (user listed at end)
groups <username>                       # list a user's groups
sudo deluser <username> <groupname>     # remove a user from a group
```

Setup used here:

```bash
sudo adduser jakoch sudo                # add jakoch to sudo
sudo addgroup user42                    # create user42 group
sudo adduser jakoch user42              # add jakoch to user42
getent group user42 sudo                # verify both
```

### Password Policy

Two layers. `/etc/login.defs` sets aging rules for **new** accounts;
`libpam-pwquality` enforces strength rules; `chage` applies aging to
**existing** accounts.

```bash
sudo nano /etc/login.defs               # PASS_MAX_DAYS 30 / PASS_MIN_DAYS 2 / PASS_WARN_AGE 7
sudo apt install libpam-pwquality -y    # strength-checking library
sudo nano /etc/pam.d/common-password    # add the pwquality options (below)
```

Options appended to the `pam_pwquality.so` line:

```
retry=3 minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```

| Option | Meaning |
|---|---|
| `minlen=10` | at least 10 characters |
| `ucredit=-1` | at least 1 uppercase |
| `lcredit=-1` | at least 1 lowercase |
| `dcredit=-1` | at least 1 digit |
| `maxrepeat=3` | no more than 3 identical chars in a row |
| `reject_username` | password can't contain the username |
| `difok=7` | at least 7 chars different from the old password |
| `enforce_for_root` | root must obey the rules too |

Apply aging to existing accounts and check:

```bash
sudo chage -M 30 -m 2 -W 7 <username>   # max 30 / min 2 / warn 7
sudo chage -l <username>                # show a user's aging info
passwd                                  # change my own password
sudo passwd root                        # change root's password
```

### AppArmor

AppArmor is Debian's Mandatory Access Control system; it confines programs to
what their profiles allow. It must run at startup.

```bash
sudo aa-status                          # show loaded profiles
sudo systemctl is-enabled apparmor      # confirm it starts at boot
```

### LVM and Partitions

**What LVM is.** LVM (Logical Volume Manager) is a layer that sits between the
physical disk and the filesystems. Instead of carving fixed partitions
directly on the disk, LVM groups physical space into a **volume group (VG)**
and then splits that VG into **logical volumes (LVs)** — one per mount point
(`/`, `/home`, `/var`, ...). The advantage is flexibility: logical volumes can
be **resized** without repartitioning the disk, and a volume group can even
span multiple physical disks. That is why this project uses it.

**Encryption.** The whole thing sits inside a single **LUKS-encrypted**
container (`sda5_crypt`). So the LVM volume group — and every logical volume in
it — is encrypted at rest. This satisfies the mandatory rule of "at least 2
encrypted partitions using LVM."

**Mandatory vs bonus partitioning (decided at install time).** The subject
shows two different layouts, and the difference matters:

- **Mandatory** needs only a minimal encrypted-LVM layout — e.g. `root`,
  `swap`, `home` (the two+ encrypted volumes rule).
- **Bonus** asks for an expanded structure with separate volumes for `/var`,
  `/srv`, `/tmp`, and `/var/log` on top of the mandatory ones.

This build uses the **bonus** layout (7 logical volumes). Note that the subject
never explicitly defines LVM or labels the expanded layout as "bonus" — you are
expected to understand it. Importantly, the partition layout is created **during
the Debian installation**, so the decision of mandatory-only vs bonus
partitioning has to be made *up front*; changing it afterwards means
repartitioning.

This build's layout:

| Volume | Mount | Purpose |
|---|---|---|
| root | `/` | system root |
| swap | `[SWAP]` | swap space |
| home | `/home` | user files |
| var | `/var` | variable data (logs, web, DB) |
| srv | `/srv` | served data |
| tmp | `/tmp` | temporary files |
| var-log | `/var/log` | logs isolated from the rest of `/var` |

Inspecting and managing:

```bash
lsblk                                   # tree view of disk / crypt / lvm
sudo cryptsetup status <name>_crypt     # confirm LUKS encryption on the container
sudo vgs                                # volume groups
sudo lvs                                # logical volumes
sudo pvs                                # physical volumes underlying the VG
```

Resizing logical volumes (used to tune sizes):

```bash
sudo lvresize -r -L 3G /dev/<VG>/root   # resize + filesystem in one step
sudo swapoff -a                         # disable swap before resizing it
sudo lvresize -L 1G /dev/<VG>/swap      # resize swap volume
sudo mkswap /dev/<VG>/swap              # re-make swap
sudo swapon -a                          # re-enable swap
```

### Monitoring Script and Cron

`monitoring.sh` (bash) prints system info to all terminals at boot and every 10
minutes using `wall`. `cron` is the scheduler that runs it.

```bash
sudo nano /usr/local/bin/monitoring.sh   # write the script
sudo chmod 755 /usr/local/bin/monitoring.sh   # make it executable
bash /usr/local/bin/monitoring.sh        # run it manually to test (no errors!)
```

Schedule it in root's crontab:

```bash
sudo systemctl enable cron.service       # cron starts at boot
sudo systemctl status cron               # confirm cron is running
sudo crontab -u root -e                  # edit root's crontab, then add:
#   */10 * * * * /usr/local/bin/monitoring.sh   (run every 10 minutes)
sudo crontab -u root -l                  # list root's crontab
```

The script reports: architecture + kernel, physical CPUs, vCPUs, RAM usage %,
disk usage %, CPU load %, last boot, LVM active, TCP connections, logged-in
users, IP + MAC, and number of sudo commands.

**Script content:**

```bash
#!/bin/bash
arch=$(uname -a)
pcpu=$(grep "physical id" /proc/cpuinfo | sort | uniq | wc -l)
vcpu=$(grep "^processor" /proc/cpuinfo | wc -l)
ram_total=$(free -m | awk '$1 == "Mem:" {print $2}')
ram_use=$(free -m | awk '$1 == "Mem:" {print $3}')
ram_percent=$(free -m | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')
disk_total=$(df -BG | grep '^/dev/' | grep -v '/boot$' | awk '{disk_t += $2} END {print disk_t}')
disk_use=$(df -BG | grep '^/dev/' | grep -v '/boot$' | awk '{disk_u += $3} END {print disk_u}')
disk_percent=$(df -BG | grep '^/dev/' | grep -v '/boot$' | awk '{disk_u += $3} {disk_t += $2} END {printf("%d", disk_u/disk_t*100)}')
cpuload=$(grep '^cpu ' /proc/stat | awk '{usage=($2+$4)*100/($2+$4+$5)} END {printf("%.1f", usage)}')
lb=$(who -b | awk '$1 == "system" {print $3 " " $4}')
lvmuse=$( if lsblk -o TYPE | grep -iq "^lvm$"; then echo yes; else echo no; fi )
tcpcon=$(ss -Ht state established | wc -l)
ulog=$(users | wc -w)
ip=$(hostname -I | awk '{print $1}')
mac=$(ip link show | grep "ether" | head -n1 | awk '{print $2}')
sudocmd=$(journalctl _COMM=sudo | grep COMMAND | wc -l)
wall "#Architecture: $arch
#Physical CPU: $pcpu
#vCPU: $vcpu
#Memory usage: $ram_use/${ram_total}MB ($ram_percent%)
#Disk Usage: $disk_use/${disk_total}Gb ($disk_percent%)
#CPU Load: $cpuload%
#Last Boot: $lb
#LVM Use: $lvmuse
#TCP Connections: $tcpcon ESTABLISHED
#User log: $ulog
#Network: IP $ip ($mac)
#Sudo: $sudocmd cmd"
```

**How each field works:**

| Field | Command | Explanation |
|---|---|---|
| Architecture | `uname -a` | kernel name, host, kernel version, hardware arch |
| Physical CPU | `grep "physical id" ... \| sort \| uniq \| wc -l` | unique socket ids = physical chips |
| vCPU | `grep "^processor" ... \| wc -l` | one line per logical core (cores × threads) |
| RAM | `free -m` cols `$2/$3` | total vs used MB, percent = used/total×100 |
| Disk | `df -BG`, exclude `/boot`, sum `$2/$3` | total vs used GB across real filesystems |
| CPU load | `/proc/stat` cpu line: `(user+sys)*100/(user+sys+idle)` | busy time over total, from kernel counters |
| Last boot | `who -b` fields `$3 $4` | date + time of last system boot |
| LVM use | `lsblk -o TYPE \| grep -iq "^lvm$"` | yes if any volume is type lvm |
| TCP conns | `ss -Ht state established \| wc -l` | active established connections (no header) |
| User log | `users \| wc -w` | logged-in sessions counted as words |
| Network | `hostname -I` + `ip link \| grep ether` | first IPv4 + first MAC |
| Sudo | `journalctl _COMM=sudo \| grep COMMAND \| wc -l` | count of logged sudo commands |

Common eval follow-ups: physical vs vCPU differ (sockets vs logical cores);
"User log" can be >1 because each session (console + SSH) counts; and to
**interrupt it without editing the script**, comment the `*/10 * * * *` line in
root's crontab or `sudo systemctl stop cron` — the schedule lives in cron, not
in the script.

---

## Bonus Part

> The bonus is only graded if the mandatory part is perfect. Extra ports
> (like 80) are allowed for bonus services, with UFW adjusted accordingly.

### WordPress Stack

A functional WordPress site served by **lighttpd** (web server) + **PHP** (runs
the code via FastCGI) + **MariaDB** (stores the data). The three pieces:
browser → lighttpd → PHP → MariaDB.

**lighttpd** — lightweight web server:

```bash
sudo apt install lighttpd -y             # install web server
sudo ufw allow 80                        # open HTTP
sudo systemctl enable --now lighttpd     # start + enable
sudo systemctl status lighttpd           # check
```

(VirtualBox port forward for the browser: host `8080` → guest `80`.)

**MariaDB** — database (a MySQL fork):

```bash
sudo apt install mariadb-server -y       # install database
sudo mariadb-secure-installation         # harden defaults (no,no,yes,yes,yes,yes)
sudo mariadb                             # open the DB shell as root
```

Inside the MariaDB shell:

```sql
CREATE DATABASE <db_name>;
GRANT ALL ON <db_name>.* TO '<db_user>'@'localhost' IDENTIFIED BY '<db_password>' WITH GRANT OPTION;
FLUSH PRIVILEGES;                        -- reload privilege tables
EXIT;
```

```bash
mariadb -u <db_user> -p                  # log in as the DB user to test
# then:  SHOW DATABASES;   EXIT;
```

**PHP** — processes WordPress pages:

```bash
sudo apt install php-fpm php-mysql php-curl php-gd php-zip -y   # PHP + extensions
php -v                                                          # confirm version
ls -l /run/php/                                                 # find the fpm socket name
sudo nano /etc/lighttpd/conf-available/15-fastcgi-php.conf      # point socket to php<ver>-fpm.sock
sudo lighttpd-enable-mod fastcgi                                # enable FastCGI
sudo lighttpd-enable-mod fastcgi-php                            # enable FastCGI-PHP
sudo systemctl reload lighttpd                                  # apply
```

**WordPress** — download, place in web root, configure:

```bash
sudo apt install wget -y
sudo wget https://wordpress.org/latest.tar.gz -P /var/www/html      # download
sudo tar -xzvf /var/www/html/latest.tar.gz -C /var/www/html         # extract
sudo rm /var/www/html/latest.tar.gz                                 # remove archive
sudo cp -r /var/www/html/wordpress/* /var/www/html                  # move to web root
sudo rm -rf /var/www/html/wordpress                                 # clean up
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php   # config from sample
sudo chown -R www-data:www-data /var/www/html                       # web server owns files
sudo chmod 755 /var/www/html                                        # permissions
sudo nano /var/www/html/wp-config.php                               # set DB_NAME/USER/PASSWORD/HOST
sudo systemctl reload lighttpd
```

Then open `http://localhost:8080/` and finish the install wizard. A rendered
wizard (not raw PHP) proves lighttpd + PHP + MariaDB all work together.

### Fail2ban

My bonus extra choice of a service.

**Fail2ban** watches the auth logs and temporarily bans IPs that repeatedly
fail SSH logins, by adding a firewall rule.

Why I chose this service. The bonus asks for a service that is genuinely
useful (NGINX/Apache2 excluded), and I have to justify the choice at the
defense. Fail2ban is the strongest fit for this specific project because:

- It directly extends the project's theme. Born2beRoot is about hardening
a server that exposes SSH on port 4242. Fail2ban is the natural next layer:
the mandatory setup makes the door strong (non-default port, no root login,
strong passwords), and Fail2ban throws out anyone who keeps rattling the
handle. The whole project is SSH hardening, so this service is on-topic
rather than bolted on.

- It complements UFW instead of duplicating it. UFW is a static rule set
(which ports are open). Fail2ban is dynamic and behaviour-based (it reacts
to what shows up in the logs and bans specific IPs for a while). They solve
different halves of the problem, so adding Fail2ban is not redundant unlike
adding a second web server.

- It's a real defence against a real threat. Any internet-facing SSH port
gets automated brute-force attempts. Rate-limiting and banning repeat
offenders is a standard, production-grade mitigation, not a toy.

- It's easy to demonstrate and explain. I can show a live ban and account
for exactly how it works (journal → filter → nftables rule), which is what
the defense is really testing.

Debian 13 specifics: it reads the **systemd journal** (no `/var/log/auth.log`),
bans through **nftables**, and the SSH unit is `ssh.service`.

```bash
sudo apt install fail2ban -y                                # install
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local    # never edit jail.conf directly
sudo nano /etc/fail2ban/jail.local                          # configure (below)
sudo fail2ban-client -t                                     # validate config
sudo systemctl enable --now fail2ban                        # start + enable
```

`[sshd]` jail on Trixie (the `port` and `backend` lines are critical):

```
[sshd]
enabled  = true
port     = 4242
backend  = systemd
maxretry = 3
```

Check and manage:

```bash
sudo fail2ban-client status                 # list active jails
sudo fail2ban-client status sshd            # failed/banned counts + port
sudo fail2ban-client set sshd unbanip <ip>  # manually unban an IP
```

---

## Technical Choices and Comparisons

**Design choices made:** Debian 13 minimal (no GUI); LUKS-encrypted disk with
LVM split into `/`, `/home`, `/var`, `/srv`, `/tmp`, `/var/log`, and swap;
strong password policy via login.defs + pwquality + chage; strict sudo with
logging, TTY mode, attempt limit and restricted path; SSH on 4242 with no root
login; UFW allowing only 4242 (and 80 for the bonus); AppArmor at boot;
monitoring via cron. Bonus: WordPress (lighttpd/MariaDB/PHP) and Fail2ban.

**Debian vs Rocky Linux** — Debian is beginner-friendly, huge community, uses
`apt`/`dpkg` and AppArmor. Rocky is RHEL-compatible (enterprise), uses
`dnf`/`rpm` and SELinux, more complex to set up for this project.

**AppArmor vs SELinux** — both are Mandatory Access Control (LSM). AppArmor
(Debian) is **path-based** and simpler to read/manage. SELinux (RHEL/Rocky) is
**label-based**, more granular and powerful, but steeper to learn.

**UFW vs firewalld** — both front-ends for netfilter. UFW (Debian) has a simple
syntax (`ufw allow 4242`), ideal for a single static server. firewalld
(RHEL/Rocky) uses **zones** and runtime-vs-permanent rules, more flexible but
heavier.

**VirtualBox vs UTM** — VirtualBox is a free cross-platform (x86) hypervisor,
used here. UTM is a macOS QEMU front-end mainly for Apple Silicon (ARM) Macs
where VirtualBox isn't available.

---

## Cheat-Sheet

**Change the hostname (then change it back):**

```bash
hostnamectl                                    # show current
sudo hostnamectl set-hostname newname42        # change
sudo nano /etc/hosts                           # update 127.0.1.1 line to match
sudo hostnamectl set-hostname jakoch42         # change back
```

**Create a new user and add to a group** (use a NEW group name — check
`getent group` first so it isn't one that already exists):

```bash
sudo adduser evaluser42                         # new user
sudo addgroup demogroup                         # new group (unused name)
sudo adduser evaluser42 demogroup               # add to group
getent group demogroup                          # proof of membership
```

**Interrupt monitoring.sh WITHOUT editing the script** (stop the schedule, not
the file):

```bash
sudo crontab -u root -e        # comment out the line: # */10 * * * * ...
# or, blunt version:
sudo systemctl stop cron       # stops all cron jobs
```

**Show SSH is correct:**

```bash
sudo grep -Ei '^port|permitrootlogin' /etc/ssh/sshd_config   # Port 4242 / PermitRootLogin no
ssh <user>@127.0.0.1 -p 4242                                  # login works
# root login should be refused
```

**Show the firewall:**

```bash
sudo ufw status                # active, 4242 (+80 bonus) only
```

**Demonstrate Fail2ban actually banning an IP:**

```bash
# 1. inside VM, watch live:
sudo journalctl -u fail2ban -f
# 2. from HOST, fail SSH >3 times (arrives as 10.0.2.2 in the guest):
ssh fakeuser@localhost -p 4241
# 3. inside VM, confirm the ban + the enforced rule:
sudo fail2ban-client status sshd
sudo nft list ruleset | grep -A5 f2b       # rule must cover the IP / port 4242
# 4. prove it: from HOST the real user gets NO password prompt (refused/hang):
ssh -o BatchMode=yes -o ConnectTimeout=5 <user>@localhost -p 4241
# 5. unban (from the VM CONSOLE, host path is blocked):
sudo fail2ban-client set sshd unbanip 10.0.2.2
```

**Verify password aging:**

```bash
sudo chage -l <user>           # Max 30 / Min 2 / Warn 7
sudo chage -l root             # same for root
```

**Check the sudo log works:**

```bash
sudo cat /var/log/sudo/sudo.log
```

---

## Signature & Evaluation Procedure

The `signature.txt` file contains the **SHA1 hash of the VM's virtual disk
(`.vdi`)**. At the eval, the hash in this file is compared against the hash of
the actual disk. **If they don't match → grade 0.** The tricky part: the hash
**changes every time the VM boots**, yet the VM must run during the eval. The
fix is snapshots.

### Why the hash breaks (and how snapshots save it)

- Booting the VM writes to the disk, so the `.vdi` hash changes immediately.
- A **snapshot** freezes the current disk state and sends new writes to a
  separate file. When you **restore** the snapshot afterwards, the base
  `.vdi` returns to exactly its pre-boot state → the original hash matches
  again. **Important** restore the snapshot at first **BEFORE** deleting the snapshot. 
  Otherwise the changes of it get written to the disk hence changing the hash.
- So: compute the hash from a fully shut-down VM, then only ever run it from
  snapshots, deleting them after each session to keep the base disk intact.

### Step 1 — Create the signature (do this once, before submitting)

```bash
# 1. FULLY shut down the VM (not paused, not saved state):
#    inside the VM:
sudo poweroff
#    or in VirtualBox: Machine > ACPI Shutdown, and wait until it's Powered Off.

# 2. On the HOST (e.g. macOS), hash the .vdi (shasum defaults to SHA1):
shasum ~/VirtualBox\ VMs/Born2beRoot/*.vdi
#    Linux:   sha1sum <disk>.vdi
#    Windows: certUtil -hashfile <disk>.vdi sha1

# 3. Put ONLY the 40-character hash into signature.txt (no filename):
echo "<the_hash>" > signature.txt
```

> After step 2, do **not** boot the VM again before submitting, or the hash
> won't match what you wrote.

### Step 2 — Submit

```bash
git add README.md signature.txt      # NEVER add the .vdi itself
git commit -m "born2beroot"
git push
```

The repo must contain **only** `README.md` and `signature.txt`. Including the
VM/disk in git = grade 0.

### Step 3 — Evaluation (keep the signature valid)

The subject requires each defense to **start with no snapshots**. So:

1. Arrive with the VM **powered off** and **zero snapshots** on it.
2. Show the evaluator that `signature.txt` matches the current disk hash
   (re-run `shasum` on the `.vdi` while it's still off — it must equal the file).
3. **Create a snapshot now** (VirtualBox: Machine > Take Snapshot), *then*
   boot the VM from it. Because you booted from a snapshot, all the eval's
   changes land in the snapshot file, not the base `.vdi`.
4. Do the whole evaluation (logins, live user creation, monitoring, fail2ban,
   etc.) from the running snapshot.
5. **Shut down the VM normally.**
6. **RESTORE the Snapshot at first, BEFORE deleting it.**
    Otherwise the changes of it get written to the disk hence changing the hash.
    The base `.vdi` reverts now its re-boot state, so its hash still matches `signature.txt` — ready for the
   next evaluation.
7. **At the end, delete/discard the snapshot.** 

### Golden rules

- Hash the disk **only** from a fully powered-off VM.
- **Never** boot the base VM directly after hashing — always through a snapshot.
- Start each defense with **no** pre-existing snapshots.
- Take the snapshot **before** booting, delete it **after** finishing.
- If you ever boot the base disk by accident, you must re-hash and re-commit
  `signature.txt` before that becomes a problem.
- Safety net: keep a **copy of the `.vdi`** (duplicate the file) that matches
  the submitted hash, in case a snapshot goes wrong.

---

## What I Learned

- **Virtualization & partitioning:** creating a VM, and how a LUKS-encrypted
  LVM stack layers (physical partition → crypt container → volume group →
  logical volumes) so each mount point is both encrypted and resizable. Also
  that the partition layout is chosen at **install time**, so mandatory-only
  (minimal encrypted LVM) vs bonus (expanded `/var`, `/srv`, `/tmp`,
  `/var/log` volumes) has to be decided before installing — the subject
  doesn't spell out what LVM is or label the expanded layout as bonus, so you
  have to understand it yourself.
- **`apt` vs `aptitude`:** both manage packages; apt is the standard CLI,
  aptitude is an alternative front-end with interactive dependency solving.
- **AppArmor vs SELinux:** what Mandatory Access Control is and why Debian
  ships AppArmor (path-based) while RHEL uses SELinux (label-based).
- **SSH:** server vs client config, why moving off port 22 and disabling root
  login hardens the box, and how VirtualBox NAT port forwarding maps a host
  port to guest 4242 (they need not match).
- **sudo hardening:** attempt limits, I/O logging, TTY mode, and `secure_path`,
  and why sudoers is edited only through `visudo`.
- **Password policy layering:** `login.defs` (new accounts) vs `chage`
  (existing accounts) vs `pam_pwquality` (strength) — all three must agree.
- **cron:** scheduling jobs, and that a script is passive — cron is what runs
  it, so you interrupt the schedule, not the file.
- **Fail2ban (deepest lesson):** deciding to ban (adding an IP to a set) is
  **separate** from enforcing it (the firewall rule). My first ban rule
  targeted port 22 while SSH was on 4242, so the ban registered but wasn't
  enforced; fixing the jail port rebuilt the nftables rule and the packet
  counter proved packets were finally being dropped. Also learned that on
  Debian 13 fail2ban reads the systemd journal (no auth.log) and bans via
  nftables, that `ignoreself` (not `ignoreip`) is what protects loopback, and
  that a wrong password gives a prompt while a ban gives no prompt at all.
- **The web stack:** how lighttpd, PHP-FPM and MariaDB fit together to serve a
  dynamic WordPress site.

---

## Resources

- Debian Administrator's Handbook — https://debian-handbook.info/
- Debian Wiki — https://wiki.debian.org/ (LVM, AppArmor, cron)
- OpenSSH `sshd_config` manual — https://man.openbsd.org/sshd_config
- UFW community docs — https://help.ubuntu.com/community/UFW
- Fail2ban / jail.conf manuals — https://manpages.debian.org/
- lighttpd documentation — https://redmine.lighttpd.net/projects/lighttpd/wiki
- WordPress installation guide — https://wordpress.org/documentation/
- Born2beRoot guides:
    - https://noreply.gitbook.io/born2beroot
    - https://www.youtube.com/watch?v=VE7X5d19svc

### Use of AI

AI was used as a **learning and explanation tool**, not to generate the setup
for me. Specifically it was used to:

- clarify concepts I then implemented and tested myself (how `ignoreself` and
  NAT-gateway addressing affect Fail2ban, why the systemd journal vs `auth.log`
  matters on Debian 13, how cron relates to the monitoring script, how the LVM
  encryption layers work);
- help debug specific errors during testing (correcting a `fail2ban-regex`
  command, and diagnosing why a ban was registered but not enforced because the
  jail port didn't match the SSH port);
- help draft and organise documentation (this README and my personal notes).

Every configuration decision, command, and verification was carried out and
understood by me.
