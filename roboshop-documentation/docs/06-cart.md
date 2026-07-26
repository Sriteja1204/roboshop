# 06 — Cart

Node.js microservice managing shopping cart contents. Uses Redis only — no MongoDB, since cart data doesn't need to survive restarts.

## Install, user, code (same pattern)

```shell
dnf module disable nodejs -y
dnf module enable nodejs:20 -y
dnf install nodejs -y

useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
mkdir /app
curl -L -o /tmp/cart.zip https://roboshop-artifacts.s3.amazonaws.com/cart-v3.zip
cd /app
unzip /tmp/cart.zip
npm install
```

## systemd service — first example of service-to-service communication

`/etc/systemd/system/cart.service`:
```ini
[Service]
User=roboshop
Environment=REDIS_HOST=<REDIS-SERVER-IP>
Environment=CATALOGUE_HOST=<CATALOGUE-SERVER-IP>
Environment=CATALOGUE_PORT=8080
ExecStart=/bin/node /app/server.js
SyslogIdentifier=cart
```

**Why `CATALOGUE_HOST`/`CATALOGUE_PORT`:** Cart doesn't store product details itself — when displaying cart contents, it calls Catalogue's API directly to fetch product name/price/image. This is a peer-service dependency, not a datastore dependency, and introduces an implicit ordering requirement: Catalogue must be reachable for Cart to fully function.

```shell
systemctl daemon-reload
systemctl enable cart
systemctl start cart
```
