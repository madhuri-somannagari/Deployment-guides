## User Management

#### Objective:
Implement a secure, auditable, and maintainable access control system for servers with the principle of least privilege.

#### Admin Users
****CAN****: Full system access (via IAM/sudo) for emergency management

### User and Group SetUp
Create a new user and group:
```bash
# create user
sudo adduser dev1
sudo adduser devops1

# create groups
sudo groupadd developers
sudo groupadd devops

# check groups
getent group developers devops

# add user to group
sudo usermod -aG developers dev1
sudo usermod -aG devops devops1

```

### Project Directory Ownership and Permissions
Assign the project directory and its contents to the correct user and group (e.g., ubuntu:dev):
``` 
sudo chown -R hdiplatform:developers /home/hdiplatform/Hiringdog-backend

sudo chgrp developers /home/hdiplatform

sudo find /home/hdiplatform/Hiringdog-backend -type d -exec chmod 755 {} \;

# -- (OR) --  Use chmod 2770 for directories (for group inheritance and security)
sudo find /home/hdiplatform/Hiringdog-backend -type d -exec chmod 2755 {} \;

sudo find /home/hdiplatform/Hiringdog-backend -type f ! -name ".env" -exec chmod 644 {} \;
```

Find all .env files and apply ownership and permissions
``` 
sudo find /home/hdiplatform/Hiringdog-backend -type f -name "*.py" -exec chmod 744 {} \;
sudo find /home/hdiplatform/Hiringdog-backend -type f -name "*.md" -exec chmod 644 {} \;
sudo find /home/hdiplatform/Hiringdog-backend -type f -name "*.txt" -exec chmod 644 {} \;
sudo find /home/hdiplatform/Hiringdog-backend -type f -name "*.log" -exec chmod 644 {} \;
sudo find /home/hdiplatform/Hiringdog-backend -type f -name ".git*" -exec chmod 644 {} \;

sudo chmod 755 /home/hdiplatform/Hiringdog-backend/manage.py
sudo chmod 644 /home/hdiplatform/Hiringdog-backend/hdip_architechture.png
```

Set read/write for owner and group, none for others (all regular files except .env):
``` 
sudo chown hdiplatform:hdiplatform /home/hdiplatform/Hiringdog-backend/.env
sudo find /home/hdiplatform -name ".env*" -exec chmod 600 {} \; 
```
Parent Directory Execute Bit
Allows users to traverse into the directory.
``` 
sudo chmod o+x /home/hdiplatform/Hiringdog-Backend
sudo chmod o+x /home/hdiplatform/Hiringdog-Backend/hiringdog
# ...repeat for any other parent directories as needed
```

### Controlled sudo Access
Edit (with visudo) /etc/sudoers.d/:
``` 
sudo visudo -f /etc/sudoers.d/developer-group
```
Developers - APPLICATION services management only
```
# Developer group limited systemctl privileges
%developers ALL=(ALL) NOPASSWD: \
  /bin/systemctl restart gunicorn.service, \
  /bin/systemctl reload gunicorn.service, \
  /bin/systemctl status gunicorn.service, \
  /bin/systemctl restart celery.service, \
  /bin/systemctl status celery.service, \
  /bin/systemctl restart celery-beat.service, \
  /bin/systemctl status celery-beat.service, \
  /bin/systemctl restart nginx.service, \
  /bin/systemctl status nginx.service, \
  /bin/systemctl restart redis.service, \
  /bin/systemctl status redis.service, \
  /bin/systemctl restart rabbitmq-server.service, \
  /bin/systemctl status rabbitmq-server.service

```
``` 
sudo visudo -f /etc/sudoers.d/devops-group
```
Devops - Application services management and read log access
```
# Developer group limited systemctl privileges
%devops ALL=(ALL) NOPASSWD: \
  /bin/journalctl -u gunicorn, \
  /bin/journalctl -u celery, \
  /bin/journalctl -u celery-beat, \
  /bin/journalctl -u nginx, \
  /bin/journalctl -u redis-server, \
  /bin/journalctl -u rabbitmq-server

%devops ALL=(ALL) NOPASSWD: \
  /bin/systemctl restart gunicorn.service, \
  /bin/systemctl reload gunicorn.service, \
  /bin/systemctl status gunicorn.service, \
  /bin/systemctl restart celery.service, \
  /bin/systemctl status celery.service, \
  /bin/systemctl restart celery-beat.service, \
  /bin/systemctl status celery-beat.service, \
  /bin/systemctl restart nginx, \
  /bin/systemctl status nginx,\
  /bin/systemctl restart redis-server, \
  /bin/systemctl status redis-server, \
  /bin/systemctl restart rabbitmq-server, \
  /bin/systemctl status rabbitmq-server
```



### SSH Configuration
Edit /etc/ssh/sshd_config: 
```bash.sh
PasswordAuthentication no 
PermitRootLogin no
PubkeyAuthentication yes
```
```
sudo systemctl reload sshd
```
#### Steps to Enable SSH Login for users 
On the worker VM:
1. Generate an SSH Key Pair on the Worker VM (if you don't have one)   
On the worker VM, run:
```bash.sh
ssh-keygen -t ed25519
```
Private key: ~/.ssh/id_ed25519
Public key: ~/.ssh/id_ed25519.pub

2. On the server:
``` 
sudo mkdir -p /home/dev1/.ssh
sudo touch /home/dev1/.ssh/authorized_keys
sudo chown -R dev1:dev1 /home/dev1/.ssh
sudo chmod 700 /home/dev1/.ssh
sudo chmod 600 /home/dev1/.ssh/authorized_keys
# Paste public key into authorized_keys using editor or automation
sudo nano /home/dev1/.ssh/authorized_keys
```

### Best Practice Checklist for Real-Time Production
- No root login via SSH
- Each user has their own account and SSH key
- Least privilege: only needed sudo permissions
- Sensitive files (.env) are protected
- App runs as non-root user, managed by systemd or supervisor
- App directory permissions restrict access to only necessary users/groups
- Regularly audit users, groups, and sudoers


### Core Concepts
| Concept          | Why It Matters                                                                |
|------------------|-------------------------------------------------------------------------------|
| Users            | Identify who can log in, run commands, or own processes/files                 |
| Groups           | Control access to resources (e.g., files, sudo, services)                     |
| Sudo access      | Restrict who can run commands as root — very critical for secure operations   |
| Home directories | Manage developer environments and personal data                               |
| Root directory   | Critical system files — must be restricted                                    |
| SSH configuration| Manage who can SSH into a server and with what keys  

### View Current Users and Groups
| Command              | Purpose                                        |
|----------------------|------------------------------------------------|
| `cat /etc/passwd`    | List all users                                 |
| `cat /etc/group`     | List all groups                                |
| `id devuser`         | See groups a specific user belongs to          |
| `who` / `w`          | See who is logged in now                       |
| `last`               | Show login history  

