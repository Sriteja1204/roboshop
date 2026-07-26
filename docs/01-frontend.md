# 01 — Frontend (Nginx)

The Frontend serves RoboShop's static web content and acts as a reverse proxy for every backend microservice.

## Install Nginx (specific version)

```shell
dnf module disable nginx -y
dnf module enable nginx:1.24 -y
dnf install nginx -y
systemctl enable nginx
systemctl start nginx
```

**Why a module stream switch:** RHEL 9 offers multiple Nginx version streams. Pinning `1.24` avoids config-syntax or default-behavior mismatches with the config file this project ships.

## Deploy the static site

```shell
rm -rf /usr/share/nginx/html/*
curl -o /tmp/frontend.zip https://roboshop-artifacts.s3.amazonaws.com/frontend-v3.zip
cd /usr/share/nginx/html
unzip /tmp/frontend.zip
```

**Why clear the default content first:** `/usr/share/nginx/html` is Nginx's default document root — leaving the placeholder page in place risks stale/unrelated files being served alongside the real site.

## Configure the reverse proxy

Edit `/etc/nginx/nginx.conf`, and inside the `server {}` block, replace every `localhost` with the real private IP of each backend server:

```nginx
location /api/catalogue/ { proxy_pass http://<CATALOGUE-IP>:8080/; }
location /api/user/      { proxy_pass http://<USER-IP>:8080/; }
location /api/cart/      { proxy_pass http://<CART-IP>:8080/; }
location /api/shipping/  { proxy_pass http://<SHIPPING-IP>:8080/; }
location /api/payment/   { proxy_pass http://<PAYMENT-IP>:8080/; }
```

**Why a reverse proxy at all:** the browser only ever talks to one address (Frontend's public IP). Nginx routes each `/api/...` path to the correct backend server, hiding the internal architecture and avoiding CORS issues that direct browser-to-backend calls would trigger.

```shell
systemctl restart nginx
```

Config changes require a restart — Nginx only reads its config file at startup/reload.
