# 12 — Architecture & Security Groups

## Services & Ports

| Service | Port | Technology |
|---------|------|------------|
| Frontend | 80 | Nginx |
| Catalogue | 8080 | NodeJS |
| User | 8080 | NodeJS |
| Cart | 8080 | NodeJS |
| Shipping | 8080 | Java/Maven |
| Payment | 8080 | Python |
| Dispatch | - | GoLang (consumer only) |
| MongoDB | 27017 | NoSQL Database |
| Redis | 6379 | In-memory Cache |
| MySQL | 3306 | SQL Database |
| RabbitMQ | 5672 | Message Queue |

## Security Model

Each service runs on its own EC2 instance with its own security group. Two layers of access control exist independently:

1. **Application-level binding** (e.g. `bindIp 0.0.0.0` in `mongod.conf`) — controls whether the *software itself* will accept a connection from another host.
2. **Security group rules** — controls whether the *network* even lets that traffic reach the server at all.

Both layers must agree for a connection to succeed — a very common source of "it should work but doesn't" issues when only one layer is checked.

### Legend
🟡 SSH  🔴 Public Internet  🔵 Internal (Security Group source)

Only **Frontend** accepts 🔴 public traffic (port 80). Every other service accepts only 🔵 traffic from the specific security groups of services that legitimately depend on it — enforcing that, for example, Payment can never reach MongoDB even if it tried, since only Catalogue and User are permitted.

### Rules by service

| Service | Inbound (besides SSH) |
|---|---|
| frontend | 80 TCP from `0.0.0.0/0` |
| catalogue | 8080 from `roboshop-frontend`, `roboshop-cart` |
| user | 8080 from `roboshop-frontend`, `roboshop-payment` |
| cart | 8080 from `roboshop-frontend`, `roboshop-shipping`, `roboshop-payment` |
| shipping | 8080 from `roboshop-frontend` |
| payment | 8080 from `roboshop-frontend` |
| dispatch | none (outbound-only consumer) |
| mongodb | 27017 from `roboshop-catalogue`, `roboshop-user` |
| redis | 6379 from `roboshop-user`, `roboshop-cart` |
| mysql | 3306 from `roboshop-shipping` |
| rabbitmq | 5672 from `roboshop-payment`, `roboshop-dispatch` |

## Request Flow

```
User → Frontend (nginx, :80, public)
Frontend → Catalogue / User / Cart / Shipping / Payment (:8080 each, internal)
Catalogue → MongoDB (:27017)
User → MongoDB (:27017) + Redis (:6379)
Cart → Redis (:6379) + Catalogue (:8080, product lookup)
Shipping → MySQL (:3306) + Cart (:8080, cart lookup)
Payment → Cart (:8080) + User (:8080) + RabbitMQ (:5672, publish)
Dispatch ← RabbitMQ (:5672, consume) — no inbound calls
```

## Dependency Shapes Summary

- **Durable database:** Catalogue/User → MongoDB, Shipping → MySQL
- **Fast/ephemeral cache:** User/Cart → Redis
- **Direct service-to-service HTTP calls:** Cart→Catalogue, Shipping→Cart, Payment→Cart/User
- **Asynchronous decoupled messaging:** Payment (publisher) → RabbitMQ → Dispatch (consumer)
- **Reverse proxy entry point:** Frontend → all app-tier services
