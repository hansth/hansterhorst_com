---
title: Setup a Secure VPS with Ubuntu
date: 2025-10-14 
tags: ["devops", "ubuntu", "vps", "security", "fail2ban", "ufw"]
draft: false
weight: 1
---

After completing the [TWN DevOps Bootcamp](https://www.techworld-with-nana.com/devops-bootcamp), I decided to set up my own DevOps environment using the tools I learned during the program. To start, I rented a VPS running Ubuntu and configured it to host a **K3s Kubernetes cluster**.

Setting up an Ubuntu server is one of the most crucial steps in building reliable infrastructure. Whether you’re managing a small project or running multiple applications, a well-configured server ensures performance, stability, and—most importantly—security.

In this project, I will explain the steps I took to set up an Ubuntu VPS, applying essential security measures, and preparing it to host a **K3s** cluster. The goal is to create a solid, production-ready environment that reflects real-world DevOps practices.

## Provisioning the VPS

I chose a medium-sized VPS to run multiple applications like Jenkins, Nexus, and Prometheus with Grafana in a Kubernetes environment. Here are the specifications:

- **OS**: Ubuntu 24.04 LTS.
- **CPU**: 4 vCPUs.
- **RAM**: 16 GB.
- **Disk**: 200 GB.

Once the VPS is ready, connect to it via **SSH** with the credentials provided by the hosting provider and update the system.

**Login as `root` user**:
```bash
ssh root@<SERVER_IP>
```

**Update the system package list**:
```bash
apt update && apt upgrade -y
```
- `apt update`: Updates the local package index with the latest information from repositories.
- `&&`: A logical operator that runs the second command if the first one succeeds.
- `apt upgrade`: Installs the latest versions of installed packages.
- `-y`: Automatically answers “yes” to prompts.

**Monitoring system logs**:
```bash
cat /var/log/auth.log | less
cat /var/log/syslog | less
```
- `less`: This will show the content of log files in a pager viewer.

## Create a non-root user

When setting up an Ubuntu VPS for the first time, it creates a **root user** account by default. The root user has unrestricted privileges, which can be dangerous—one wrong command can break the entire system. It’s best practice to create a **non-root user** for daily operations.

**Create a new user**:
```bash
adduser <USERNAME>
```
Follow instruction in the terminal.
```
info: Adding user `USERNAME' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `USERNAME' (1001) ...
info: Adding new user `USERNAME' (1001) with group `USERNAME (1001)' ...
info: Creating home directory `/home/USERNAME' ...
info: Copying files from `/etc/skel' ...
New password:
Retype new password:
```

**Add user to the `sudo` group**:
```bash
usermod -aG sudo <USERNAME>
```
- `-aG`: Appends the user to the `sudo` group.

**Verify user added to the group**:
```bash
cat /etc/group | grep <USERNAME>
```
- `grep`: Will filter out the username and display all groups the user belongs to.
```
sudo:x:27:<USERNAME>
users:x:100:<USERNAME>
<USERNAME>:x:1001:
```

**Copy the `root` SSH directory**:
```bash
rsync --archive --chown=<USERNAME>:<USERNAME> /root/.ssh /home/<USERNAME>
```
- `rsync`: A file synchronization tool that efficiently copies and syncs files.
- `--archive`: Preserves file permissions, timestamps, symbolic links, and other attributes while recursively copying directories.
- `--chown=<USERNAME>:<USERNAME>`: Changes ownership of the copied files to the specified username.
- `.ssh`: The source location.
- `/home/<USERNAME>`: The destination location.

The `rsync` command offers an efficient way to transfer your existing SSH keys from the root's `.ssh` directory to a new user's home directory. This process ensures:

- Proper file permissions are preserved, which is critical for maintaining SSH security.
- Correct ownership is set for the new user.
- All SSH configurations and authorized keys are maintained.

By doing this, you can enable immediate SSH access using the existing keys while adhering to security best practices, such as minimizing direct root user access.


**Set only user `rwx` permissions on `.ssh` directory**:
```bash
chmod 700 /home/<USERNAME>/.ssh
```
```
drwx------ 2 USERNAME USERNAME 4096 Jul 26 10:51 .ssh
```
- `700`: `7` = read `4` + write `2` + execute `1`.
- The other two `0` is for `group` and `other` with no permissions.

**Set user `rw-` permissions on `authorized_keys` file**:
```bash
chmod 600 /home/<USERNAME>/.ssh/authorized_keys
```
```
-rw------- 1 USERNAME USERNAME 97 Aug 16 08:38 .ssh/authorized_keys
```

**Optional removes unnecessary keys**:
```bash
vim /home/<USERNAME>/.ssh/authorized_keys
```

**Exit the server as `root` user**:
```bash
exit
```

**Optional create an SSH key on your local computer**:
```bash
ssh-keygen -t ed25519 -C "<COMMENT>"
```
- `-C`: Add a comment for identification.

**Copy SSH public key from the host computer**:
```bash
ssh-copy-id -i ~/.ssh/<SSH_KEY_NAME>.pub <USERNAME>@<SERVER_IP>
```
- `-i`: Specify which public SSH key to copy.

After copying the SSH public key, login as a non-root user.

**Login as non-root user**:
```bash
ssh <USERNAME>@<SERVER_IP>
```

Creating a non-root user in Linux is a small but crucial step to keep the system secure and organized. It helps prevent accidental mistakes and improve the security of your server or computer.

When you need root privileges, simply use `sudo` before the command, or switch to the root user `sudo su`.

## Enable automatic security updates

When managing a Linux server, keeping it secure is important. New security issues are found all the time, and not updating the system quickly can make it vulnerable. You can update manually, but it’s easy to forget.

By using `unattended-upgrades`, you can automatically install security and other updates. It runs in the background, updating the system based on your settings.

**Update the server packages**:
```bash
sudo apt update && sudo apt upgrade -y
```
- `sudo`: Runs the command with administrator (`root`) privileges.

**Install `unattended-upgrades` package**:
```bash
sudo apt install unattended-upgrades -y
```

**Enable automatic updates**:
```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```
- `dpkg-reconfigure`: A utility that re-runs the configuration process for an already installed package.
- `--priority=low`: Shows all available configuration questions.
- `unattended-upgrades`: The package being reconfigured, which handles automatic security updates and package upgrades

**Check the configuration**:
```bash
sudo vim /etc/apt/apt.conf.d/50unattended-upgrades
```
Ensure at least these lines are uncommented:
```
"${distro_id}:${distro_codename}-security";
```

**Check the update scheduler**:
```bash
sudo vim /etc/apt/apt.conf.d/20auto-upgrades
```
Here you can control how often the system checks for updates and applies them.
```bash
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```
This runs daily (`1`). Set to `"7"` for weekly or `"0"` to disable.

This will update the package list and install critical security updates on the system.

**Monitoring the log files**:
```bash
sudo cat /var/log/unattended-upgrades/unattended-upgrades.log
```
Unattended-upgrades keep logs of what it installs. You can review what is updated:
```
2025-08-31 13:39:43,475 INFO Starting unattended upgrades script
2025-08-31 13:39:43,475 INFO Allowed origins are: o=Ubuntu,a=noble, o=Ubuntu,a=noble-security, o=UbuntuESMApps,a=noble-apps-security, o=UbuntuESM,a=noble-infra-security
2025-08-31 13:39:43,475 INFO Initial blacklist:
2025-08-31 13:39:43,475 INFO Initial whitelist (not strict):
2025-08-31 13:39:44,642 INFO No packages found that can be upgraded unattended and no pending auto-removals
2025-08-31 13:39:44,651 INFO Package cloud-init is marked to be held back.
```

Using `unattended-upgrades` is one of the simplest ways to improve server security with minimal effort. It ensures updating the package list and important updates are applied daily, reducing the risk of leaving the system exposed to vulnerabilities.

## Modify `sshd` configuration

**Secure Shell, SSH** is one of the most widely used methods to access and manage Linux systems. By default, it provides encrypted communication between the user and server, but leaving it with default settings can expose the system to unnecessary risks.

By modifying the `sshd_config` config file, allows you to improve security, and optimize system performance.

>**_Important_**: Before changing the configuration, **always** backup the previous file before making changes.

**Backup and recovery**:
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup.$(date +"%Y%m%d_%H%M")
```
- `cp`: Copy command.
- `/etc/ssh/sshd_config`: The source file to copy.
- `/etc/ssh/sshd_config.backup.`: The destination to copy to with the new filename.
- `$(date +"%Y%m%d_%H%M")`: With the current date and time.
```
-rw-r--r-- 1 root root 3447 Aug 30 09:57 sshd_config.backup.20250830_0957
```

**Recover the configuration**:
```bash
sudo cp /etc/ssh/sshd_config.backup.<LAST_BACKUP> /etc/ssh/sshd_config
sudo systemctl restart ssh.service
```

After backup the config file, you can modify the configuration.

**Open `sshd_config` file**:
```bash
sudo vim /etc/ssh/sshd_config
```

**Change the default SSH port**:
```vim
Port 22222
```
Choosing a non-standard port to reduce automated brute-force attacks.

>**_Note_**: When changed the port use `-p` flag to SSH into the server.

**Change log level for more information**:
```vim
LogLevel VERBOSE
```

**Disable `root` access**:
```vim
PermitRootLogin no
```
This will disable direct root login over SSH.

**Use key-based authentication**:
```vim
PubkeyAuthentication yes
PasswordAuthentication no
```
This enforces key-based authentication.

**Disable X11 forward if you don't need it**:
```vim
X11Forwarding no
```
This ensures attackers can’t use the X11 protocol.

After changing the configuration, you need to restart or reload the service to apply changes.

**Restart the SSH service**:
```bash
sudo systemctl restart ssh.service
```

**Check the service status**:
```bash
sudo systemctl status ssh.service
```
```
● ssh.service - OpenBSD Secure Shell server
 Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
 Active: active (running) since Sun 2025-08-31 14:23:26 UTC; 51min ago
TriggeredBy: ● ssh.socket
 Docs: man:sshd(8)
 man:sshd_config(5)
Process: 4627 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
 Main PID: 4629 (sshd)
Tasks: 2 (limit: 19147)
 Memory: 7.9M (peak: 21.7M)
CPU: 46ms
 CGroup: /system.slice/ssh.service
 ├─ 915 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
 └─4629 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
```

Modifying the config file is one of the **most effective ways to secure the server**. By disabling root login, changing the default SSH port, using key-based authentication, you can significantly reduce brute-force attacks.

## Setup Fail2ban

When running a Linux server exposed to the internet, one of the most common threats is brute-force login attempts on services like **SSH**. A powerful way to protect against this is using **Fail2ban**.

Fail2ban is a security tool that monitors log files and automatically blocks IP addresses that show suspicious activity, like repeated failed login attempts.

**Update the server packages**:
```bash
sudo apt update && sudo apt upgrade -y
```

**Install `fail2ban` package**:
```bash
sudo apt install fail2ban -y
```

**Enable and start the service**:
```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
```

Fail2ban uses configuration files stored in `/etc/fail2ban/`. The default settings are in `jail.conf`, but you should **never** edit this file. By copy it to `jail.local`, Fail2ban will use this file to override the default `jail.conf` file.

**Copy the default configuration**:
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

>**_Important_**: Before changing the configuration, **always** backup the previous file before making changes.

**Backup and recovery**
```bash
sudo cp /etc/fail2ban/jail.local /etc/fail2ban/jail.local.backup.$(date +"%Y%m%d_%H%M")
```

After backup the config file, you can modify the configuration.

**Modify the Fail2Ban configuration**:
```bash
sudo vim /etc/fail2ban/jail.local
```

**The default jail settings**:
```vim
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 <YOUR_IP_ADDRESS>
bantime= 24h
findtime = 10m
maxretry = 3
```
- `ignoreip`: IP addresses that Fail2ban will never ban.
- `bantime`: How long to ban an IP.
- `findtime`: Time window to look for repeated failures.
- `maxretry`: Number of failures allowed within the `findtime` window before banning.

**SSH settings**, for monitoring SSH connection:
```
[sshd]
enable= true
port= 22222
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```
- `enabled`: Activates this jail.
- `port`: Monitors the SSH port that I have changed.
- `logpath`: Points to SSH log files.
- `backend`: Specifies how to read the log files.

**recidive settings** that catches repeat IPs that keep getting banned:
```
[recidive] 
enabled = true 
logpath = /var/log/fail2ban.log 
banaction = %(banaction_allports)s 
bantime = 1M # 1 month
findtime = 1w # Look back 1 week 
maxretry = 2
```
- `enabled`: Activates this jail.
- `logpath`: Monitors Fail2ban's own log file to catch repeated failures.
- `banaction`: Uses an action that blocks all ports.
- `bantime`: Much longer ban.
- `findtime`: Looks back 1 week for previous bans.
- `maxretry`: Allows 2 bans before triggering the long-term block

**Restart Fail2ban service**:
```bash
sudo systemctl restart fail2ban.service
```

**Check the status of Fail2ban service**:
```bash
sudo systemctl status fail2ban.service
```
```
● fail2ban.service - Fail2Ban Service
 Loaded: loaded (/usr/lib/systemd/system/fail2ban.service; enabled; preset: enabled)
 Active: active (running) since Sun 2025-08-31 13:58:31 UTC; 25s ago
 Docs: man:fail2ban(1)
 Main PID: 3284 (fail2ban-server)
Tasks: 5 (limit: 19147)
 Memory: 20.0M (peak: 20.8M)
CPU: 184ms
 CGroup: /system.slice/fail2ban.service
 └─3284 /usr/bin/python3 /usr/bin/fail2ban-server -xf start

Aug 31 13:58:31 hansth systemd[1]: Started fail2ban.service - Fail2Ban Service.
Aug 31 13:58:31 hansth fail2ban-server[3284]: 2025-08-31 13:58:31,271 fail2ban.configreader [3284]>
Aug 31 13:58:31 hansth fail2ban-server[3284]: Server ready
```

**Check Fail2ban client status**:
```bash
sudo fail2ban-client status sshd
```
```
Status for the jail: sshd
|- Filter
||- Currently failed: 0
||- Total failed: 0
|`- File list:/var/log/auth.log
`- Actions
 |- Currently banned: 0
 |- Total banned: 0
 `- Banned IP list:
```

### Monitor Fail2ban

Monitoring Fail2ban logs is crucial to make sure the server is being protected effectively without banning the wrong users.

**Fail2ban default logs:
```bash
sudo less /var/log/fail2ban.log
```

**Real time monitoring**:
```bash
sudo tail -f /var/log/fail2ban.log
```
- `tail`: Output the last 10 rows of the file.
- `-f`: Follow the last log rows

**Search for bans and unbans in the logs**:
```bash
sudo grep "Ban" /var/log/fail2ban.log
sudo grep "Unban" /var/log/fail2ban.log
```

**Check Fail2ban client status**:
```bash
sudo fail2ban-client status
```
This will list enabled jails:
```
Status
|- Number of jail:1
`- Jail list: sshd
```

**Check SSH jail details**:
```bash
sudo fail2ban-client status sshd
```
This will show banned IPs for a specific jail:
```
Status for the jail: sshd
|- Filter
||- Currently failed: 0
||- Total failed: 0
|`- File list:/var/log/auth.log
`- Actions
 |- Currently banned: 0
 |- Total banned: 0
 `- Banned IP list:
```

**Monitor logs with `journalctl`**:
```bash
sudo journalctl -u fail2ban -f
```
This follows the `systemd fail2ban.service` logs in real time, similar to `tail -f`.

**Unban an IP manually**:
```bash
sudo fail2ban-client set sshd unbanip <IP_ADDRESS>
```

Fail2ban is a simple and powerful tool that adds a crucial layer of protection to the server. With these adjustments, you can reduce the risk of brute-force attacks dramatically.

## Enable UFW firewall rules

Securing the server is one of the most important steps you can take when managing Linux systems. One of the simplest tools for managing firewall rules is **Uncomplicated FireWall**. UFW provides an easy-to-use interface for configuring powerful `iptables` rules in the background.

**Install UFW, if not already installed**:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw -y
```

**Deny incoming traffic on all ports**:
```bash
sudo ufw default deny incoming
```

**Allow a specific IP address for SSH login on port `22222`**:
```bash
sudo ufw allow from <YOUR_IP_ADDRESS> to any port 22222
```

>**_Note_**: This is one of the most secure ways for users to log in with their IP address via SSH on port `22222`.

**Allow incoming with HTTP**:
```bash
sudo ufw allow http
```
This will open port `80`.

**Allow incoming with HTTPS**:
```bash
sudo ufw allow https
```
This will open port `443`.

**Enable UFW firewall rules**:
```bash
sudo ufw enable
```

**Reload the UFW service**:
```bash
sudo ufw reload
```

**Check UFW status**:
```bash
sudo ufw status verbose
```
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To ActionFrom
-- ----------
22222ALLOW IN<YOUR_IP_ADDRESS>
80/tcp ALLOW INAnywhere
443ALLOW INAnywhere
80/tcp (v6)ALLOW INAnywhere (v6)
443 (v6) ALLOW INAnywhere (v6)
```

**Delete a firewall rule**:
```bash
sudo ufw status numbered
sudo ufw delete <STATUS_NUMBERED> 
```

**Monitoring UFW logs**:
```bash
sudo cat /var/log/ufw.log | less
```

Enabling UFW firewall rules is a simple and powerful way to secure the Linux system. With just a few commands, you can lock down the server, allow only the services you need, and prevent unauthorized access.

## Change hostname

When you set up an Ubuntu VPS, it usually has a random name. You can change this to something more recognizable, which is helpful for management tasks. To customize the hostname in Ubuntu, follow these steps:

**Update the hostname**:
```bash
sudo hostnamectl set-hostname hansth-dev
```

**Reboot the server**:
```bash
sudo reboot
```

After rebooting, the VPS will use the new hostname.


## Bash script

Throughout the setup process, I frequently backed up configuration files before making changes, such as with `sshd_config` and `fail2ban`. To make this step consistent and repeatable, I created a **Bash script** to automate the backup process.

> See [backup-file.sh](https://gitlab.com/devops8614042/ubuntu-vps-setup/-/blob/main/backup-file.sh)

The script copies specified configuration files into a timestamp backup filename. This ensures that every modification is traceable and easily reversible.

**Example recovery configuration**:
```bash
sudo cp /etc/fail2ban/jail.local.backup.<LAST_BACKUP> /etc/fail2ban/jail.local
sudo systemctl restart ssh.service
```

Automating configuration backups ensures you can always revert safely if a configuration breaks.


## Conclusion

Setting up an Ubuntu VPS is more than just installing an operating system, it’s about creating a secure, stable environment for your applications. By creating a non-root user, enable SSH login, using auto-updates, configure Fail2ban, and create firewall rules, I have a secure foundation for running my **K3s cluster**.

This setup marks the first step in my DevOps journey, building a self-managed, production-ready environment that mirrors real-world infrastructure best practices.