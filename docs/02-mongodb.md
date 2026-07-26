# 02 — MongoDB

MongoDB is a NoSQL document database used by **Catalogue** (products) and **User** (accounts). Chosen because both services store data with variable, nested fields — a rigid SQL schema would force constant migrations or lots of empty columns.

## Add the repo and install

```shell
sudo tee /etc/yum.repos.d/mongo.repo > /dev/null << 'EOF'
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
enabled=1
gpgcheck=0
EOF

dnf install mongodb-org -y
systemctl enable mongod
systemctl start mongod
```

**Why a repo file:** RHEL 9's default repos don't carry MongoDB at all, let alone the specific `7.x` version required. `gpgcheck=0` skips signature verification — acceptable for a private lab, not for production.

## Open to the network

Edit `/etc/mongod.conf`: change `bindIp` from `127.0.0.1` to `0.0.0.0`.

```shell
systemctl restart mongod
```

**Why:** MongoDB defaults to accepting connections only from the same machine. Catalogue and User run on separate servers and need network access — `0.0.0.0` listens on all interfaces. Config changes require a restart to take effect.
