# RoboShop — Full Deployment Playbook

Execution order below is **by dependency**, not by the order the docs were written in. Data stores first, then app services that need them, Frontend absolutely last (it needs every other server's IP to wire up its reverse proxy).

You need **11 servers total**, one per row:
`mongodb, redis, mysql, rabbitmq, catalogue, user, cart, shipping, payment, dispatch, frontend`

Before starting: note down every server's **private IP** as you create it — you'll be pasting these into config files constantly. Keep a scratch note like:

```
mongodb   : 
redis     : 
mysql     : 
rabbitmq  : 
catalogue : 
user      : 
cart      : 
shipping  : 
payment   : 
dispatch  : 
frontend  : 
```

---

## BLOCK 0 — Prerequisites (per server)

- Security groups set up per the SG table (SSH 22 from your IP always; internal 🔵 rules as documented per service).
- All commands below assume you're SSH'd into the correct server for that block, with sudo/root access.

---

## BLOCK 1 — MongoDB

```
docs/02-mongodb.md
```

**Verify:** `systemctl status mongod` shows `active (running)`.

---

## BLOCK 2 — Redis

```shell
sudo dnf module disable redis -y
sudo dnf module enable redis:7 -y
sudo dnf install redis -y
```

Edit `/etc/redis/redis.conf`:
- `bind 127.0.0.1` → `bind 0.0.0.0`
- `protected-mode yes` → `protected-mode no`

```shell
sudo systemctl enable redis
sudo systemctl start redis
```

**Verify:** `systemctl status redis` shows `active (running)`.

---

## BLOCK 3 — MySQL

```shell
sudo dnf install mysql-server -y
sudo systemctl enable mysqld
sudo systemctl start mysqld
sudo mysql_secure_installation --set-root-pass 'RoboShop@1'
```

**Verify:** `mysql -uroot -pRoboShop@1 -e "SELECT 1;"` returns a result without error.

*(Note: if Shipping later fails to connect remotely, check MySQL's bind-address / firewall — this doc doesn't list that step explicitly, flag it if you hit it.)*

---

## BLOCK 4 — RabbitMQ

```shell
sudo tee /etc/yum.repos.d/rabbitmq.repo > /dev/null << 'EOF'
[modern-erlang]
name=modern-erlang-el9
baseurl=https://yum1.novemberain.com/erlang/el/9/$basearch
        https://yum2.novemberain.com/erlang/el/9/$basearch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/rpm/el/9/$basearch
enabled=1
gpgcheck=0
[modern-erlang-noarch]
name=modern-erlang-el9-noarch
baseurl=https://yum1.novemberain.com/erlang/el/9/noarch
        https://yum2.novemberain.com/erlang/el/9/noarch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/rpm/el/9/noarch
enabled=1
gpgcheck=0
[modern-erlang-source]
name=modern-erlang-el9-source
baseurl=https://yum1.novemberain.com/erlang/el/9/SRPMS
        https://yum2.novemberain.com/erlang/el/9/SRPMS
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/rpm/el/9/SRPMS
enabled=1
gpgcheck=0
[rabbitmq-el9]
name=rabbitmq-el9
baseurl=https://yum2.novemberain.com/rabbitmq/el/9/$basearch
        https://yum1.novemberain.com/rabbitmq/el/9/$basearch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/rpm/el/9/$basearch
enabled=1
gpgcheck=0
[rabbitmq-el9-noarch]
name=rabbitmq-el9-noarch
baseurl=https://yum2.novemberain.com/rabbitmq/el/9/noarch
        https://yum1.novemberain.com/rabbitmq/el/9/noarch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/rpm/el/9/noarch
enabled=1
gpgcheck=0
EOF

sudo dnf install rabbitmq-server -y
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server

sudo rabbitmqctl add_user roboshop roboshop123
sudo rabbitmqctl set_permissions -p / roboshop ".*" ".*" ".*"
```

**Verify:** `sudo rabbitmqctl list_users` shows `roboshop`.

*(Data layer done. Every remaining app service now has something to connect to.)*

---

## BLOCK 5 — Catalogue (needs MongoDB IP)

```shell
sudo dnf module disable nodejs -y
sudo dnf module enable nodejs:20 -y
sudo dnf install nodejs -y

sudo useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
sudo mkdir /app

curl -o /tmp/catalogue.zip https://roboshop-artifacts.s3.amazonaws.com/catalogue-v3.zip
cd /app
sudo unzip /tmp/catalogue.zip
sudo npm install
```

Create `/etc/systemd/system/catalogue.service`:
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
*(Replace `<MONGODB-IP>` with Block 1's server IP.)*

```shell
sudo systemctl daemon-reload
sudo systemctl enable catalogue
sudo systemctl start catalogue
```

Load master data (from Catalogue server, install Mongo client first):
```shell
sudo tee /etc/yum.repos.d/mongo.repo > /dev/null << 'EOF'
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
enabled=1
gpgcheck=0
EOF
sudo dnf install mongodb-mongosh -y
mongosh --host <MONGODB-IP> </app/db/master-data.js
```

**Verify:**
```shell
mongosh --host <MONGODB-IP>
use catalogue
db.products.find()
```
Should return product documents, not empty.

---

## BLOCK 6 — User (needs MongoDB IP, Redis IP)

```shell
sudo dnf module disable nodejs -y
sudo dnf module enable nodejs:20 -y
sudo dnf install nodejs -y

sudo useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
sudo mkdir /app

curl -L -o /tmp/user.zip https://roboshop-artifacts.s3.amazonaws.com/user-v3.zip
cd /app
sudo unzip /tmp/user.zip
sudo npm install
```

Create `/etc/systemd/system/user.service`:
```ini
[Unit]
Description = User Service

[Service]
User=roboshop
Environment=MONGO=true
Environment=REDIS_URL='redis://<REDIS-IP>:6379'
Environment=MONGO_URL="mongodb://<MONGODB-IP>:27017/users"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=user

[Install]
WantedBy=multi-user.target
```

```shell
sudo systemctl daemon-reload
sudo systemctl enable user
sudo systemctl start user
```

**Verify:** `systemctl status user` shows `active (running)`, no crash loop.

---

## BLOCK 7 — Cart (needs Redis IP, Catalogue IP)

```shell
sudo dnf module disable nodejs -y
sudo dnf module enable nodejs:20 -y
sudo dnf install nodejs -y

sudo useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
sudo mkdir /app

curl -L -o /tmp/cart.zip https://roboshop-artifacts.s3.amazonaws.com/cart-v3.zip
cd /app
sudo unzip /tmp/cart.zip
sudo npm install
```

Create `/etc/systemd/system/cart.service`:
```ini
[Unit]
Description = Cart Service

[Service]
User=roboshop
Environment=REDIS_HOST=<REDIS-IP>
Environment=CATALOGUE_HOST=<CATALOGUE-IP>
Environment=CATALOGUE_PORT=8080
ExecStart=/bin/node /app/server.js
SyslogIdentifier=cart

[Install]
WantedBy=multi-user.target
```

```shell
sudo systemctl daemon-reload
sudo systemctl enable cart
sudo systemctl start cart
```

**Verify:** `systemctl status cart` active, no repeated restarts (check `journalctl -u cart -n 50` if it is).

---

## BLOCK 8 — Shipping (needs MySQL IP, Cart IP)

```shell
sudo dnf install maven -y

sudo useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
sudo mkdir /app

curl -L -o /tmp/shipping.zip https://roboshop-artifacts.s3.amazonaws.com/shipping-v3.zip
cd /app
sudo unzip /tmp/shipping.zip
sudo mvn clean package
sudo mv target/shipping-1.0.jar shipping.jar
```

Create `/etc/systemd/system/shipping.service`:
```ini
[Unit]
Description=Shipping Service

[Service]
User=roboshop
Environment=CART_ENDPOINT=<CART-IP>:8080
Environment=DB_HOST=<MYSQL-IP>
ExecStart=/bin/java -jar /app/shipping.jar
SyslogIdentifier=shipping

[Install]
WantedBy=multi-user.target
```

```shell
sudo systemctl daemon-reload
sudo systemctl enable shipping
sudo systemctl start shipping

sudo dnf install mysql -y
mysql -h <MYSQL-IP> -uroot -pRoboShop@1 < /app/db/schema.sql
mysql -h <MYSQL-IP> -uroot -pRoboShop@1 < /app/db/app-user.sql
mysql -h <MYSQL-IP> -uroot -pRoboShop@1 < /app/db/master-data.sql

sudo systemctl restart shipping
```

**Verify:**
```shell
mysql -h <MYSQL-IP> -uroot -pRoboShop@1 -e "USE shipping; SHOW TABLES;"
```
Should list tables, not be empty.

---

## BLOCK 9 — Payment (needs Cart IP, User IP, RabbitMQ IP)

```shell
sudo dnf install python3 gcc python3-devel -y

sudo useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
sudo mkdir /app

curl -L -o /tmp/payment.zip https://roboshop-artifacts.s3.amazonaws.com/payment-v3.zip
cd /app
sudo unzip /tmp/payment.zip
sudo pip3 install -r requirements.txt
```

Create `/etc/systemd/system/payment.service`:
```ini
[Unit]
Description=Payment Service

[Service]
User=root
WorkingDirectory=/app
Environment=CART_HOST=<CART-IP>
Environment=CART_PORT=8080
Environment=USER_HOST=<USER-IP>
Environment=USER_PORT=8080
Environment=AMQP_HOST=<RABBITMQ-IP>
Environment=AMQP_USER=roboshop
Environment=AMQP_PASS=roboshop123
ExecStart=/usr/local/bin/uwsgi --ini payment.ini
ExecStop=/bin/kill -9 $MAINPID
SyslogIdentifier=payment

[Install]
WantedBy=multi-user.target
```

```shell
sudo systemctl daemon-reload
sudo systemctl enable payment
sudo systemctl start payment
```

**Verify:** `systemctl status payment` active.

---

## BLOCK 10 — Dispatch (needs RabbitMQ IP)

```shell
sudo dnf install golang -y

sudo useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
sudo mkdir /app

curl -L -o /tmp/dispatch.zip https://roboshop-artifacts.s3.amazonaws.com/dispatch-v3.zip
cd /app
sudo unzip /tmp/dispatch.zip
sudo go mod init dispatch
sudo go get
sudo go build
```

Create `/etc/systemd/system/dispatch.service`:
```ini
[Unit]
Description = Dispatch Service

[Service]
User=roboshop
Environment=AMQP_HOST=<RABBITMQ-IP>
Environment=AMQP_USER=roboshop
Environment=AMQP_PASS=roboshop123
ExecStart=/app/dispatch
SyslogIdentifier=dispatch

[Install]
WantedBy=multi-user.target
```

```shell
sudo systemctl daemon-reload
sudo systemctl enable dispatch
sudo systemctl start dispatch
```

**Verify:** `systemctl status dispatch` active — no inbound test possible since it has no listening port; check `journalctl -u dispatch -f` for a "connected to RabbitMQ" style log line.

---

## BLOCK 11 — Frontend (needs every app-tier IP — do this LAST)

```shell
sudo dnf module disable nginx -y
sudo dnf module enable nginx:1.24 -y
sudo dnf install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

**Check in browser now** — should see default Nginx page.

```shell
sudo rm -rf /usr/share/nginx/html/*
curl -o /tmp/frontend.zip https://roboshop-artifacts.s3.amazonaws.com/frontend-v3.zip
cd /usr/share/nginx/html
sudo unzip /tmp/frontend.zip
```

**Check in browser again** — should see RoboShop content now (product calls will still fail until the next step).

Edit `/etc/nginx/nginx.conf`, inside the `server {}` block, replacing every `localhost` with the real IPs:

```nginx
location /api/catalogue/ { proxy_pass http://<CATALOGUE-IP>:8080/; }
location /api/user/      { proxy_pass http://<USER-IP>:8080/; }
location /api/cart/      { proxy_pass http://<CART-IP>:8080/; }
location /api/shipping/  { proxy_pass http://<SHIPPING-IP>:8080/; }
location /api/payment/   { proxy_pass http://<PAYMENT-IP>:8080/; }
```

```shell
sudo systemctl restart nginx
```

**Verify:** Open the site in browser → product list should load (Catalogue), you should be able to register/login (User), add to cart (Cart), get a shipping quote (Shipping), and complete checkout (Payment → RabbitMQ → Dispatch).

---

## End-to-end sanity checklist

Run through the site in this exact order and confirm each works before moving to the next — it matches the dependency chain:

1. Homepage loads with products → **Frontend + Catalogue + MongoDB** all correct
2. Register/login a user → **User + MongoDB + Redis** all correct
3. Add item to cart → **Cart + Redis + Catalogue-lookup** all correct
4. Get shipping estimate at checkout → **Shipping + MySQL + Cart-lookup** all correct
5. Complete payment → **Payment + Cart/User-lookup + RabbitMQ** all correct
6. Check `journalctl -u dispatch -f` shows it picked up the order → **Dispatch consuming from RabbitMQ** confirmed

If any step fails, the service that owns that step is where the problem is — check with `systemctl status <service>` and `journalctl -u <service> -n 50` first, then come back here with the exact error.
