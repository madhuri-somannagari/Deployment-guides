
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
  --workers 2 --pid /run/gunicorn/gunicorn.pid 
  --bind 127.0.0.1:8000 
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

### After (bind on gunicorn.sock, accessible to NGINX):
```
ExecStart=/home/ubuntu/shared_venv/bin/gunicorn \
  --workers 2 \
  --pid /run/gunicorn/gunicorn.pid \
  --bind unix:/run/gunicorn/gunicorn.sock \
  hiringdogbackend.wsgi:application

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
***UNIX Socket***
Instead of binding Gunicorn to a TCP port, it binds to a Unix socket (/run/gunicorn/gunicorn.sock).
Nginx proxies requests to this socket.
Benefits:
- Faster than TCP localhost. lower latency 
- No open port exposed. 
- Clear boundary: Nginx ⇄ Gunicorn via socket.
***cat /etc/nginx/sites-available/hiringdogbackend***                                    
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;  # Close connection for requests to the IP
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;
    ssl_certificate /etc/letsencrypt/live/api.hdiplatform.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.hdiplatform.in/privkey.pem;
    return 444;  # Close connection for HTTPS requests to the IP
}
server {
    listen 80;
    server_name api.hdiplatform.in;
    return 301 https://$host$request_uri;  # Redirect HTTP to HTTPS
}

server {
    listen 443 ssl;
    server_name api.hdiplatform.in;

    ssl_certificate /etc/letsencrypt/live/api.hdiplatform.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.hdiplatform.in/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;  # Change port if needed
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    client_max_body_size 25M;
    error_log /var/log/nginx/hiringdogbackend_error.log;
    access_log /var/log/nginx/hiringdogbackend_access.log;
}
```

### Test and Reload: 
` sudo nginx -t && sudo systemctl reload nginx`

- Check if the Socket and PID Are Active: ` ls -l /run/gunicorn/gunicorn.sock `
- View the PID file (if needed): ` cat /run/gunicorn/gunicorn.pid `

### Fix Permission Issues (If Nginx can't access the socket) 
Give socket to group www-data (Nginx user): 
``` 
sudo chown ubuntu:www-data /run/gunicorn/gunicorn.sock
```
Ensure group (Nginx) can read/write:
```
sudo chmod 660 /run/gunicorn/gunicorn.sock
```
- Test Your Deployment ` https://api.hdiplatform.in `
- Access Log:
```
sudo tail -f /var/log/nginx/hiringdogbackend_access.log
```
- Error Log: 
```
sudo tail -f /var/log/nginx/hiringdogbackend_error.log
```

### Change to (local gunicorn.sock for single VM):
```
location / {
    #proxy_pass http://127.0.0.1:8000;
    proxy_pass http://unix:/run/gunicorn/gunicorn.sock:;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```
