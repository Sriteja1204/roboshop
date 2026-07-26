# 04 — Redis

In-memory cache used by **User** (sessions) and **Cart** (cart contents). Chosen over MongoDB for these specifically because the data is short-lived and needs to be as fast as possible — losing it on restart is acceptable, unlike account or order data.

## Install (specific version)

```shell
dnf module disable redis -y
dnf module enable redis:7 -y
dnf install redis -y
```

## Open to the network

Edit `/etc/redis/redis.conf`:
- `bind 127.0.0.1` → `0.0.0.0`
- `protected-mode yes` → `no`

**Why both changes are needed:** `bind` controls which network interfaces Redis listens on. `protected-mode` is a second safety lock — even with `bind` opened, Redis refuses outside connections unless a password is set or protected mode is explicitly disabled.

> Note: this combination (open bind, no protected mode, no password) is fine for a private training VPC, but in production you'd set `requirepass` and restrict access via security groups instead.

```shell
systemctl enable redis
systemctl start redis
```
