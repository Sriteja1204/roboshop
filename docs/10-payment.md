# 10 — Payment

Python microservice handling payment processing. Publishes an "order paid" event to RabbitMQ once payment succeeds — this is the publisher side of the queue.

## Install Python

```shell
dnf install python3 gcc python3-devel -y
```

`gcc` and `python3-devel` are needed because many pip packages have C extensions that must be compiled on install — a Python-specific gotcha Node/Java don't have.

## User, code, dependencies

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
mkdir /app
curl -L -o /tmp/payment.zip https://roboshop-artifacts.s3.amazonaws.com/payment-v3.zip
cd /app
unzip /tmp/payment.zip
pip3 install -r requirements.txt
```

## systemd service

`/etc/systemd/system/payment.service`:
```ini
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
```

Notes:
- **`User=root`** — breaks the non-root pattern used everywhere else; likely a lab-only shortcut around a `uwsgi` privilege issue, worth revisiting in a real deployment.
- **`WorkingDirectory=/app`** — required because `ExecStart` references `payment.ini` as a relative path.
- **`AMQP_*` variables** — the credentials created in the RabbitMQ module; this is what lets Payment publish messages onto the queue.
- **`ExecStop=/bin/kill -9`** — force-kills the process on stop rather than a graceful shutdown; a workaround, not best practice.

```shell
systemctl daemon-reload
systemctl enable payment
systemctl start payment
```
