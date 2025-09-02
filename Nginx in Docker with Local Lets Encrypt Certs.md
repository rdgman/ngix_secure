# Nginx in Docker with Local Let's Encrypt Certs

## 📁 Docker Project Folder Structure

```
docker/
├── nginx/
│   ├── docker-compose.yml
│   ├── cloudflare.ini
│   ├── nginx.conf
│   └── (any additional confs for server blocks)
└── docker-compose.yml
```

---

## 🐳 `docker-compose.yml`

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    network_mode: "host"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./websocket.conf:/etc/nginx/conf.d/websocket.conf
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot
#    ports:  # Expose ports 80 and 443 if not running in host mode
#      - "80:80"
#      - "443:443"
    depends_on:
      - certbot

  certbot:
    image: certbot/dns-cloudflare
    container_name: certbot
    network_mode: "host"
    volumes:
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot
      - ./cloudflare.ini:/etc/letsencrypt/cloudflare.ini
    entrypoint: >
      /bin/sh -c "
      certbot certonly --non-interactive --quiet --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
      --email ${EMAIL} --agree-tos --no-eff-email --expand \
      --domains '*.local.mydomain.com' --domains '*.tail.mydomain.com';
      while :; do  
        sleep 24h;
        certbot renew --non-interactive --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini;
      done"
```

---

## 🔐 `cloudflare.ini`

```ini
dns_cloudflare_api_token=<cloudflare api token with dns edit privilege>
```

---

## ⚙️ `nginx.conf`

```nginx
user nginx;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;
}

http {
    server_tokens off;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
    send_timeout       600s;

    # Redirect HTTP to HTTPS
    server {
        listen 80;
        listen [::]:80;
        server_name *.local.thomasjwilde.com *.tail.thomasjwilde.com;
        return 301 https://$host$request_uri;
    }

    # immich.local
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name immich.local.thomasjwilde.com;

        ssl_certificate        /etc/letsencrypt/live/local.thomasjwilde.com/fullchain.pem;
        ssl_certificate_key    /etc/letsencrypt/live/local.thomasjwilde.com/privkey.pem;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

        include /etc/nginx/conf.d/websocket.conf;

        location / {
            proxy_pass http://127.0.0.1:2283;
        }
    }

    # Vaultwarden tail
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name vault.tail.thomasjwilde.com;

        ssl_certificate        /etc/letsencrypt/live/local.thomasjwilde.com/fullchain.pem;
        ssl_certificate_key    /etc/letsencrypt/live/local.thomasjwilde.com/privkey.pem;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

        location / {
            proxy_pass http://127.0.0.1:1776;
        }
    }
}
```

---

## 🔄 Hot Reload Strategy

Instead of using `entrypoint` or `command` in the container (which interferes with Docker logs), you can create a **host-level timer** to reload Nginx every 48 hours:

```bash
/usr/bin/docker exec -it nginx nginx -s reload
```

---

## 📜 Tail Logs

```bash
tail -10 /home/thomas/docker/nginx/data/nginx/logs/access.log
```

---

Let me know if you'd like this exported to a file or converted into a README format!
