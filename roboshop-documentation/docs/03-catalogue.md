# 03 — Catalogue

Node.js microservice serving the product listing. Reads from MongoDB.

## Install Node.js 20

```shell
dnf module disable nodejs -y
dnf module enable nodejs:20 -y
dnf install nodejs -y
```

RHEL defaults to an older Node stream; the app requires ≥20.

## Non-root app user and code deployment

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
mkdir /app
curl -o /tmp/catalogue.zip https://roboshop-artifacts.s3.amazonaws.com/catalogue-v3.zip
cd /app
unzip /tmp/catalogue.zip
npm install
```

**Why a dedicated non-root user:** limits the blast radius if the app is ever compromised — this pattern repeats for every custom-built service (Catalogue, User, Cart, Shipping, Payment, Dispatch), since packaged software (Mongo, Redis, MySQL, RabbitMQ, Nginx) already creates its own service user automatically on install.

## systemd service

`/etc/systemd/system/catalogue.service`:
```ini
[Unit]
Description = Catalogue Service

[Service]
User=roboshop
Environment=MONGO=true
Environment=MONGO_URL="mongodb://<MONGODB-IP>:27017/catalogue"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=catalogue

[Install]
WantedBy=multi-user.target
```

```shell
systemctl daemon-reload
systemctl enable catalogue
systemctl start catalogue
```

`daemon-reload` is required because this is a brand-new unit file systemd hasn't seen yet.

## Load product data

```shell
dnf install mongodb-mongosh -y
mongosh --host <MONGODB-IP> </app/db/master-data.js
```

Verify:
```shell
mongosh --host <MONGODB-IP>
use catalogue
db.products.find()
```

## Point Frontend at Catalogue

Update `/api/catalogue/` in Frontend's `nginx.conf` with this server's IP.
