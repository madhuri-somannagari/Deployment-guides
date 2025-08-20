## LogRotate system

### Install logrotate (if not already installed)

```bash.sh
sudo apt update
sudo apt install logrotate -y
```

### Log Directory and Permissions
Create a dedicated directory to store application logs with restricted access:
```
sudo mkdir -p /var/log/hiringdog
sudo chown www-data:www-data /var/log/hiringdog
sudo chmod 750 /var/log/hiringdog
```
### Logrotate Configuration 
Create a logrotate configuration file specifically for the Hiringdog application.
File: ` /etc/logrotate.d/hiringdog `

```
/var/log/hiringdog/*.log {
    daily                # rotate logs daily (can be weekly/monthly)
    rotate 10             # keep 7 days of logs
    missingok            # don’t throw error if log file missing
    notifempty           # don’t rotate empty files
    compress              # Old logs are gzipped to save space.
    delaycompress         # recent rotated log is not compressed until the next rotation
    notifempty            # Skips rotation if the log is empty.
    copytruncate          #  Truncates the original log after copying, so apps don’t need to reopen log files 
}
```

***What this does:***
***daily:*** Rotates logs every day.

***rotate 10:*** Keeps 10 old log files (plus the current one).

***missingok:*** No error if a log file is missing.

***compress:*** Old logs are gzipped to save space.

***delaycompress:*** The most recent rotated log is not compressed until the next rotation (ensures it’s not in use).

***notifempty:*** Skips rotation if the log is empty.

***copytruncate:*** Truncates the original log after copying, so apps don’t need to reopen log files (important for Gunicorn, Celery, etc.).

***Logrotate is run via cron :*** cat /etc/cron.daily/logrotate
```
if [ -d /run/systemd/system ]; then
    exit 0
fi
```
### Test your logrotate config
```
sudo logrotate -d /etc/logrotate.d/hiringdog

```
If it looks fine, force rotate once:(optional)
```
sudo logrotate -f /etc/logrotate.d/hiringdog
```
### Log rotate execution method and status check
logrotate is run by a systemd timer, not cron. 
logrotate is managed by a systemd timer:
```
systemctl list-timers --all | grep logrotate
systemctl status logrotate.timer
```
### Verify rotation

After rotation, you’ll see files like:
```
/var/log/hiringdog/app.log
/var/log/hiringdog/app.log.1.gz
/var/log/hiringdog/app.log.2.gz
```
