# 09 — RabbitMQ

Message broker decoupling **Payment** (publisher) from **Dispatch** (consumer). Not a database — its job is asynchronous event delivery, so a slow or temporarily-down Dispatch never blocks a payment confirmation.

## Add repo and install

```shell
sudo tee /etc/yum.repos.d/rabbitmq.repo > /dev/null << 'EOF'
[modern-erlang]
name=modern-erlang-el9
baseurl=https://yum1.novemberain.com/erlang/el/9/$basearch
        https://yum2.novemberain.com/erlang/el/9/$basearch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/rpm/el/9/$basearch
enabled=1
gpgcheck=0
[rabbitmq-el9]
name=rabbitmq-el9
baseurl=https://yum2.novemberain.com/rabbitmq/el/9/$basearch
        https://yum1.novemberain.com/rabbitmq/el/9/$basearch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/rpm/el/9/$basearch
enabled=1
gpgcheck=0
EOF

dnf install rabbitmq-server -y
systemctl enable rabbitmq-server
systemctl start rabbitmq-server
```

**Why two separate package families:** RabbitMQ is built on **Erlang**, a hard dependency RHEL 9 doesn't ship in a modern-enough version — hence a dedicated Erlang repo alongside RabbitMQ's own.

## Create an application user

```shell
rabbitmqctl add_user roboshop roboshop123
rabbitmqctl set_permissions -p / roboshop ".*" ".*" ".*"
```

**Why not use the default `guest` account:** `guest` is restricted to `localhost`-only connections by design. Payment and Dispatch connect from separate servers, so a real user is required.
