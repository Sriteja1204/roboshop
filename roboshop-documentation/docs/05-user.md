# 05 — User

Node.js microservice handling login/registration. Uses **both** MongoDB (durable account data) and Redis (fast, disposable session data) — a good example of picking a different datastore per job even within one service.

## Install, user, code (same pattern as Catalogue)

```shell
dnf module disable nodejs -y
dnf module enable nodejs:20 -y
dnf install nodejs -y

useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
mkdir /app
curl -L -o /tmp/user.zip https://roboshop-artifacts.s3.amazonaws.com/user-v3.zip
cd /app
unzip /tmp/user.zip
npm install
```

`-L` follows redirects on the S3 URL — without it, curl can save a redirect response instead of the actual zip.

## systemd service

`/etc/systemd/system/user.service`:
```ini
[Service]
User=roboshop
Environment=MONGO=true
Environment=REDIS_URL='redis://<REDIS-IP>:6379'
Environment=MONGO_URL="mongodb://<MONGODB-IP>:27017/users"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=user
```

```shell
systemctl daemon-reload
systemctl enable user
systemctl start user
```

No data-seeding step here — unlike Catalogue's product data, the `users` collection starts empty and fills up naturally as people register.
