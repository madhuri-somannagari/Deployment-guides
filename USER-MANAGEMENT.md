## User Management

#### Objective:
Implement a secure, auditable, and maintainable access control system for servers with the principle of least privilege.

#### Principle: Least Privilege & Role-Based Access
- Never share the root account.
- No one logs in as root directly.
- Each person has their own user account.
- Access is granted via groups and sudoers, not by giving out the root password.
- Everything is least privilege, auditable, revocable.

### User and Group SetUp
Create a new user and group:
```bash 
sudo add user dev1
sudo groupadd dev
sudo usermod -aG dev dev1
```

### Project Directory Ownership and Permissions
Assign the project directory and its contents to the correct user and group (e.g., ubuntu:dev):
``` 
sudo chown -R ubuntu:dev /home/ubuntu/git-source
chmod 755 /home/ubuntu/git-source
```
Set read/write for owner and group, none for others (all regular files except .env):
``` 
sudo find /home/ubuntu/git-source -type f ! -name ".env" -exec chmod 664 {} \;
```
Use chmod 2770 for directories (for group inheritance and security).
``` 
sudo find /home/ubuntu/git-source -type d -exec chmod 2770 {} \;
``` 
Find all .env files and apply ownership and permissions
``` 
sudo find /home/ubuntu/ -type f -name ".env" -exec chown ubuntu:dev {} \;
sudo find /home/ubuntu/ -type f -name ".env" -exec chmod 640 {} \;
chmod 775 /home/ubuntu/git-source/deploy.sh
chmod 775 /home/ubuntu/git-source/manage.py
```
Specific .env File Handling
```
sudo chown ubuntu:dev /home/ubuntu/git-source/.env
sudo chmod 640 /home/ubuntu/git-source/.env
```
Parent Directory Execute Bit
Allows users to traverse into the directory.
``` 
sudo chmod o+x /home/ubuntu
sudo chmod o+x /home/ubuntu/git-source
# ...repeat for any other parent directories as needed
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
### Controlled sudo Access
Edit (with visudo) /etc/sudoers.d/dev-group:
``` 
sudo visudo -f /etc/sudoers.d/dev-group
Allow limited systemctl and logs:
# Allow dev group to restart and check status of gunicorn, celery, and celery-beat
%dev ALL=NOPASSWD: /bin/systemctl restart gunicorn, /bin/systemctl status gunicorn, /bin/systemctl reload gunicorn
%dev ALL=NOPASSWD: /bin/systemctl restart celery, /bin/systemctl status celery
%dev ALL=NOPASSWD: /bin/systemctl restart celery-beat, /bin/systemctl status celery-beat
%dev ALL=NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl status nginx
```

**Results:** When user try to access to list or uses sudo it will denied the permissions

### Best Practice Checklist for Real-Time Production
- No root login via SSH
- Each user has their own account and SSH key
- Least privilege: only needed sudo permissions
- Sensitive files (.env) are protected
- App runs as non-root user, managed by systemd or supervisor
- App directory permissions restrict access to only necessary users/groups
- Regularly audit users, groups, and sudoers
- Document all changes and access grants

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

