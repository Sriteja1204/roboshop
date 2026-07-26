# 11 — Dispatch

Go microservice that consumes "order paid" messages from RabbitMQ and simulates dispatching the order. This is the consumer side of the queue Payment publishes to.

## Install Go

```shell
dnf install golang -y
```

## User, code, and build

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
mkdir /app
curl -L -o /tmp/dispatch.zip https://roboshop-artifacts.s3.amazonaws.com/dispatch-v3.zip
cd /app
unzip /tmp/dispatch.zip
go mod init dispatch
go get
go build
```

`go build` compiles to a single static binary with no external runtime dependency — unlike Node (needs `node` at runtime) or Java (needs the JVM), the compiled `dispatch` executable can run directly.

## systemd service

`/etc/systemd/system/dispatch.service`:
```ini
[Service]
User=roboshop
Environment=AMQP_HOST=<RABBITMQ-IP>
Environment=AMQP_USER=roboshop
Environment=AMQP_PASS=roboshop123
ExecStart=/app/dispatch
SyslogIdentifier=dispatch
```

**Why this config is so minimal compared to every other service:** Dispatch has no API and isn't called by anything — it's a pure background worker that only listens to RabbitMQ. No `CART_HOST`, no `USER_HOST`, no port to expose.

```shell
systemctl daemon-reload
systemctl enable dispatch
systemctl start dispatch
```
