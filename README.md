# RoboShop — 3-Tier Microservices E-Commerce Deployment

This repository documents the manual deployment of **RoboShop**, a microservices-based e-commerce application, on AWS EC2 running RHEL 9. It was built as a hands-on DevOps learning project covering server provisioning, Linux service management, networking, and multi-language application deployment.

## What is RoboShop?

RoboShop is a demo e-commerce platform split into **11 independently deployed services**, each on its own EC2 instance, instead of one monolithic application. This mirrors how real-world companies structure production systems — different teams own different services, often written in different languages, communicating over the network rather than sharing a single codebase.

## Architecture

```
                          AWS Account
 ┌──────────┐      ┌─────────────────────┐      ┌─────────────────┐
 │          │      │   Application Tier  │      │   Data Tier     │
 │  Client  │─────▶│  frontend (nginx)   │      │                 │
 │ (browser)│      │        │            │      │                 │
 └──────────┘      │        ▼            │      │                 │
                    │   catalogue ───────┼─────▶│  MongoDB :27017 │
                    │   user     ───────┼─────▶│  MongoDB :27017 │
                    │   user     ───────┼─────▶│  Redis   :6379  │
                    │   cart     ───────┼─────▶│  Redis   :6379  │
                    │   cart ◀── shipping│      │                 │
                    │   cart ◀── payment │      │                 │
                    │   shipping ───────┼─────▶│  MySQL   :3306  │
                    │   payment  ───────┼─────▶│  RabbitMQ :5672 │
                    │   dispatch ◀──────┼──────│  RabbitMQ :5672 │
                    └─────────────────────┘      └─────────────────┘
```

See [`docs/12-architecture-and-security-groups.md`](docs/12-architecture-and-security-groups.md) for the full request-flow diagram and per-service security group rules.

## Tech Stack

| Service | Language/Tech | Port | Datastore Used |
|---|---|---|---|
| Frontend | Nginx | 80 | — |
| Catalogue | Node.js 20 | 8080 | MongoDB |
| User | Node.js 20 | 8080 | MongoDB + Redis |
| Cart | Node.js 20 | 8080 | Redis |
| Shipping | Java / Maven | 8080 | MySQL |
| Payment | Python 3 | 8080 | RabbitMQ (publisher) |
| Dispatch | Golang | — (consumer only) | RabbitMQ (consumer) |
| MongoDB | Database | 27017 | — |
| Redis | Cache | 6379 | — |
| MySQL | Database | 3306 | — |
| RabbitMQ | Message Queue | 5672 | — |

## Why these specific databases?

Each service's datastore was chosen to match its actual data shape and access pattern, not arbitrarily:

- **MongoDB** (Catalogue, User) — product and account data have variable, nested fields; a flexible document schema avoids constant migrations.
- **Redis** (User, Cart) — session tokens and cart contents are short-lived and read/written on nearly every request; in-memory storage prioritizes speed over durability.
- **MySQL** (Shipping) — city/distance/pricing data is rigidly structured and relational; SQL joins are the right tool for "look up distance, then look up rate."
- **RabbitMQ** (Payment → Dispatch) — not a database at all; it's a message broker that decouples Payment (publisher) from Dispatch (consumer) so a slow/down Dispatch never blocks a payment confirmation.

## Repository Structure

```
roboshop-documentation/
├── README.md                              ← you are here
├── deployment-playbook.md                 ← full ordered, copy-pasteable setup guide
└── docs/
    ├── 01-frontend.md
    ├── 02-mongodb.md
    ├── 03-catalogue.md
    ├── 04-redis.md
    ├── 05-user.md
    ├── 06-cart.md
    ├── 07-mysql.md
    ├── 08-shipping.md
    ├── 09-rabbitmq.md
    ├── 10-payment.md
    ├── 11-dispatch.md
    └── 12-architecture-and-security-groups.md
```

## Deployment Order

Data layer first (no dependencies), then application layer (depends on data layer + sometimes each other), then Frontend last (needs every other server's IP for its reverse proxy):

**MongoDB → Redis → MySQL → RabbitMQ → Catalogue → User → Cart → Shipping → Payment → Dispatch → Frontend**

Full step-by-step commands for every stage are in [`deployment-playbook.md`](deployment-playbook.md).

## Common Pattern Across Every Service

Every module in this repo follows the same five-step shape:

1. **Add a repo/module stream** if the required software/version isn't in RHEL's default sources
2. **Install** via `dnf` (or build via `npm`/`mvn`/`pip`/`go build` for custom app code)
3. **Configure** — typically opening `bind`/`listen` addresses from `127.0.0.1` to `0.0.0.0` so other servers can connect
4. **Enable + start** via `systemd`, so the service survives reboots and crashes
5. **Verify** before moving to the next dependent service

## Security

- Every server accepts SSH (22) only from a specific admin IP.
- Only Frontend accepts public internet traffic (port 80, `0.0.0.0/0`).
- Every other service accepts traffic **only** from the specific security groups of services that legitimately call it (see [`docs/12-architecture-and-security-groups.md`](docs/12-architecture-and-security-groups.md)).
- Application services run under a dedicated non-root `roboshop` system user, not root (with one documented exception in Payment — see that doc for the reasoning).

## License / Purpose

This is a personal learning/training project for practicing DevOps fundamentals: Linux administration, systemd, networking, security groups, and multi-language application deployment. Not intended for production use as-is (several settings — e.g. `gpgcheck=0`, open Redis with no auth — are simplifications appropriate only for a private training environment).
