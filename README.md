# Odoo 14 setup instructions (Nginx + Let's Encrypt)
This will guide you to install Odoo 14 with Nginx reverse proxy with SSL in Ubuntu and connect it with PostgreSQL instance.

## Update the packages and upgrade them to the latest on your new server

```
$ sudo apt update
$ sudo apt upgrade
```
## Install Wkhtmltopdf (Optional)
Wkhtmltopdf is a package that is used to render HTML to PDF and other image formats. If you are using Odoo to print PDF reports you should install wkhtmltopdf tool.

```
$ wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.bionic_amd64.deb

$ sudo apt install ./wkhtmltox_0.12.5-1.bionic_amd64.deb
```

## Install num2words (Optional)
num2words is a library that converts numbers like 42 to words like forty-two. Amount-to-text features won't be fully available if you didn't install this.

```
$ sudo apt-get install python3-num2words
```

## Install Odoo 14
```
$ wget -O - https://nightly.odoo.com/odoo.key | sudo apt-key add -
$ echo "deb http://nightly.odoo.com/14.0/nightly/deb/ ./" | sudo tee /etc/apt/sources.list.d/odoo.list


$ sudo apt update
$ sudo apt install odoo
```

Check Odoo status
```
$ sudo service odoo status
```
Enable Odoo service to start on system boot.
```
$ sudo systemctl enable --now odoo
```

## Configure Odoo
Open Odoo configuration file from Nano editor.
```
$ sudo nano /etc/odoo/odoo.conf
```
Replace with your values.
```
[options]
; This is the password that allows database operations:
; admin_passwd = admin
db_host = DB_HOST
db_port = False
db_user = DB_USER
db_password = DB_PASSWORD
;addons_path = /usr/lib/python3/dist-packages/odoo/addons
```

Restart Odoo to take effect the configuration changes.
```
$ sudo service odoo restart
```

## Install Nginx
```
$ sudo apt install nginx
```
Remove default Nginx configurations.
```
$ sudo rm /etc/nginx/sites-enabled/default
$ sudo rm /etc/nginx/sites-available/default
```

## Configure Nginx

Install Let’s Encrypt free SSL.
```
$ sudo apt install python3-certbot-nginx -y
$ sudo certbot --nginx certonly
```

Create a new Nginx configuration for Odoo.
```
$  sudo nano /etc/nginx/sites-available/odoo.conf
```

Copy below configurations to odoo.conf and save (Make sure to update **yourdomain.com** with your domain).
```
upstream odoo {
    server 127.0.0.1:8069;
}

upstream odoo-chat {
    server 127.0.0.1:8072;
}

server {
    listen [::]:80;
    listen 80;

    server_name yourdomain.com;

    return 301 https://yourdomain.com$request_uri;
}

server {
    listen [::]:443 ssl http2;
    listen 443 ssl http2;

    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    access_log /var/log/nginx/odoo_access.log;
    error_log /var/log/nginx/odoo_error.log;

    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;

    location / {
        # proxy_redirect off;
        proxy_pass http://odoo;
        proxy_redirect default;
    }

    location /longpolling {
        proxy_pass http://odoo-chat;
        proxy_redirect default;
    }

    location ~* /web/static/ {
        proxy_cache_valid 200 90m;
        proxy_buffering on;
        expires 864000;
        proxy_pass http://odoo;
    }

    gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;
    gzip on;
}
```

Enable newly created configurations.
```
$ sudo ln -s /etc/nginx/sites-available/odoo.conf /etc/nginx/sites-enabled/odoo.conf
```

Check your configuration and restart Nginx for the changes to take effect.
```
$ sudo nginx -t
$ sudo service nginx restart
```

## Renewing SSL Certificate
Certificates provided by Let’s Encrypt are valid for 90 days only. So you can automate this using cronjob.

```
$ sudo crontab -e
```
Add below script to end of the file to check certificate renewing twice daily.

```
0 0,12 * * * certbot renew >/dev/null 2>&1
```