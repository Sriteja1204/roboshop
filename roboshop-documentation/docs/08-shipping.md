# 08 — Shipping

Java/Maven microservice calculating shipping cost based on distance. First compiled-language service in the stack.

## Install Java + Maven

```shell
dnf install maven -y
```

Installing Maven pulls in a JDK automatically since it depends on Java.

## User, code, and build

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
mkdir /app
curl -L -o /tmp/shipping.zip https://roboshop-artifacts.s3.amazonaws.com/shipping-v3.zip
cd /app
unzip /tmp/shipping.zip
mvn clean package
mv target/shipping-1.0.jar shipping.jar
```

**Why a build step (unlike Node services):** Java is compiled, not interpreted — source must be turned into bytecode and packaged into a runnable `.jar` before it can execute. `mvn clean package` also resolves dependencies declared in `pom.xml`, Java's equivalent of `package.json`.

## systemd service

`/etc/systemd/system/shipping.service`:
```ini
[Service]
User=roboshop
Environment=CART_ENDPOINT=<CART-IP>:8080
Environment=DB_HOST=<MYSQL-IP>
ExecStart=/bin/java -jar /app/shipping.jar
SyslogIdentifier=shipping
```

`CART_ENDPOINT` is another service-to-service call — Shipping needs cart weight/quantity to calculate a price.

```shell
systemctl daemon-reload
systemctl enable shipping
systemctl start shipping
```

## Load schema and data

```shell
dnf install mysql -y
mysql -h <MYSQL-IP> -uroot -pRoboShop@1 < /app/db/schema.sql
mysql -h <MYSQL-IP> -uroot -pRoboShop@1 < /app/db/app-user.sql
mysql -h <MYSQL-IP> -uroot -pRoboShop@1 < /app/db/master-data.sql
systemctl restart shipping
```

**Why schema loading is required (and wasn't for Catalogue/User):** MySQL is rigid — tables must be explicitly created before any row can be inserted, unlike MongoDB's schema-less documents. The final restart lets Shipping start fresh now that tables and data actually exist.
