
## SYSTEMD SERVICES for application RUNTIME process

***/etc/systemd/system/gunicorn.service***         

```
[Unit]
Description=Gunicorn instance to serve Hiringdog
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/Hiringdog-backend

Environment="PATH=/home/ubuntu/shared_venv/bin"
EnvironmentFile=/home/ubuntu/secrets/hiringdog.env
Environment="PYTHONUNBUFFERED=1"

ExecStart=/home/ubuntu/shared_venv/bin/gunicorn \
  --workers 2 \
  --pid /run/gunicorn/gunicorn.pid \
  --bind unix:/run/gunicorn/gunicorn.sock \
  hiringdogbackend.wsgi:application

ExecReload=/bin/kill -s USR2 $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

PIDFile=/run/gunicorn/gunicorn.pid
RuntimeDirectory=gunicorn
RuntimeDirectoryMode=0755
Restart=always

# Redirecting standard output and error to log files
StandardOutput=append:/var/log/hiringdog/gunicorn.log
StandardError=append:/var/log/hiringdog/gunicorn_error.log

# Set limits
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```
***/etc/systemd/system/celery-beat.service***         

```
[Unit]
Description=Celery Beat Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/Hiringdog-backend

Environment="PATH=/home/ubuntu/shared_venv/bin"
EnvironmentFile=/home/ubuntu/secrets/hiringdog.env
Environment="PYTHONUNBUFFERED=1"

ExecStart=/home/ubuntu/shared_venv/bin/celery -A hiringdogbackend beat \
  --scheduler django_celery_beat.schedulers.DatabaseScheduler \
  --loglevel=info

Restart=always
RestartSec=3
TimeoutSec=300
LimitNOFILE=65536

StandardOutput=append:/var/log/hiringdog/celery_beat.log
StandardError=inherit

[Install]
WantedBy=multi-user.target
```
***cat /etc/systemd/system/celery.service***                                                                                  
```
[Unit]
Description=Celery Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/Hiringdog-backend

Environment="PATH=/home/ubuntu/shared_venv/bin"
EnvironmentFile=/home/ubuntu/secrets/hiringdog.env
Environment="PYTHONUNBUFFERED=1"

ExecStart=/home/ubuntu/shared_venv/bin/celery -A hiringdogbackend worker --loglevel=info

# Gracefully stop with SIGTERM
KillSignal=SIGTERM
TimeoutStopSec=300

Restart=always
RestartSec=3
TimeoutSec=300
LimitNOFILE=65536

# Log to journal for easier access to logs
StandardOutput=append:/var/log/hiringdog/celery.log
StandardError=inherit

[Install]
WantedBy=multi-user.target
```
