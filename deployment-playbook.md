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

Checkout this folder for complete guide: [`docs/02-mongodb.md`](docs/02-mongodb.md)

---

## BLOCK 2 — Redis

Checkout this folder for complete guide: [`docs/04-redis.md`](docs/04-redis.md)

## BLOCK 3 — MySQL

Checkout this folder for complete guide: [`docs/07-mysql.md`](docs/07-mysql.md)

*(Note: if Shipping later fails to connect remotely, check MySQL's bind-address / firewall — this doc doesn't list that step explicitly, flag it if you hit it.)*

## BLOCK 4 — RabbitMQ

Checkout this folder for complete guide: [`docs/09-rabbitmq.md`](docs/09-rabbitmq.md)

## BLOCK 5 — Catalogue (needs MongoDB IP)

Checkout this folder for complete guide: [`docs/03-catalogue.md`](docs/03-catalogue.md)

## BLOCK 6 — User (needs MongoDB IP, Redis IP)

Checkout this folder for complete guide: [`docs/05-user.md`](docs/05-user.md)

## BLOCK 7 — Cart (needs Redis IP, Catalogue IP)

Checkout this folder for complete guide: [`docs/06-cart.md`](docs/06-cart.md)

## BLOCK 8 — Shipping (needs MySQL IP, Cart IP)

Checkout this folder for complete guide: [`docs/08-shipping.md`](docs/08-shipping.md)

## BLOCK 9 — Payment (needs Cart IP, User IP, RabbitMQ IP)

Checkout this folder for complete guide: [`docs/10-payment.md`](docs/10-payment.md)

## BLOCK 10 — Dispatch (needs RabbitMQ IP)

Checkout this folder for complete guide: [`docs/11-dispatch.md`](docs/11-dispatch.md)

## BLOCK 11 — Frontend (needs every app-tier IP — do this LAST)

Checkout this folder for complete guide: [`docs/01-frontend.md`](docs/01-frontend.md)

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
