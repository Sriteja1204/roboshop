# 07 — MySQL

Relational database used by **Shipping**. Chosen because shipping cost calculation depends on structured, always-consistent relational data (cities, distances, pricing rules) — a classic SQL-join problem, unlike Catalogue/User's flexible document data.

## Install

```shell
dnf install mysql-server -y
systemctl enable mysqld
systemctl start mysqld
```

**Why no repo file needed:** unlike MongoDB/Redis/RabbitMQ, RHEL 9's default repos already carry MySQL at the required `8.x` version — no extra repo or module-stream switch required.

## Set the root password

```shell
mysql_secure_installation --set-root-pass 'RoboShop@1'
```

**Why this step exists:** unlike MongoDB (installed wide open by default), MySQL ships secure by default — you're locked out until you explicitly set a root password.
