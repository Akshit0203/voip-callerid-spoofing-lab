# 03 — Configuration

> Configuring the lab inside MagnusBilling: admin access, SIP users, codecs, routing prefixes,
> rates, and routes. All values shown are **placeholders/demo data**.

---

## 1. First Login & Locale

After installation, open the admin UI at `http://<YOUR_SERVER_IP>/mbilling` and select the
interface language.

![MagnusBilling language selection](../assets/screenshots/magnusbilling-language-selection.png)

> The admin account is `root`. **Change its password immediately** (Clients → Users → root → edit).

---

## 2. MagnusBilling Concepts

MagnusBilling organises telephony around a few core objects:

| Object | Menu | What it is |
|--------|------|------------|
| **User** | Clients → Users | A billing account (a customer / tenant) with a plan and credit balance. |
| **SIP User** | Clients → SIP Users | A SIP endpoint (the credentials a softphone registers with). |
| **Trunk** | Routes → Trunks | An upstream path that carries calls off the PBX. |
| **Prefix** | Rates → Prefixes | Maps a dialled number prefix to a destination (e.g. `91` → India). |
| **Tariff / Rate** | Rates → Tariffs / Rates | The per-minute cost model per destination. |
| **Route** | Routes | Chooses which trunk handles a given prefix. |
| **CDR** | Reports → CDR | One record per call. |

---

## 3. Users (Billing Accounts)

A **User** is the billing identity. In this lab two accounts existed: the `root` administrator and
a `demo` client account with a plan and a small credit balance.

![Client / user management](../assets/screenshots/client-user-management.png)

| Field | Example | Notes |
|-------|---------|-------|
| Username | `demo` | Login / account identifier |
| Plan | `India_tom` | Links the account to a rate plan |
| Credit | `1.500` | Prepaid balance (controls fraud blast-radius) |
| Group | `Client` | Permission profile |
| Status | `Active` | Enable/disable the account |

> **Defensive tip:** low prepaid credit + concurrency caps limit the damage a compromised account
> can do. See [06 — Defenses](06-defenses-and-countermeasures.md).

---

## 4. SIP Users & Codec Negotiation

A **SIP User** holds the credentials a softphone registers with, and the fields that shape the
call — including the **CallerID**.

![SIP user, CallerID and codec configuration](../assets/screenshots/sip-user-codec-config.png)

| Field | Example | Purpose |
|-------|---------|---------|
| SIP user | `demo` | Auth username |
| SIP password | `demo12` | Auth secret *(use a strong, unique value)* |
| **CallerID** | `919999999999` | **The number presented on outbound calls** ⬅ see note |
| Host | `dynamic` | Endpoint registers from any IP (vs. a pinned IP) |
| Codec (allow) | `g729, gsm, alaw, ulaw` | Negotiated media formats |
| Disallow | `all` | Deny everything, then allow the listed codecs |
| Tech prefix | `0` | Optional routing prefix |

### About the CallerID field

> The `CallerID` field is placed into the SIP **`From`** (and often **`P-Asserted-Identity`**)
> header of the outbound `INVITE`. **In legacy SIP this value is simply trusted and relayed** — the
> protocol provides no proof that the account owns that number. This is the mechanism behind CLI
> spoofing and is examined in [04 — Caller ID Spoofing Mechanics](04-callerid-spoofing-mechanics.md).
> For the lab, this field must only ever contain a number **you own and are authorised to present.**

### Codec choices

| Codec | Bandwidth | Notes |
|-------|-----------|-------|
| `g729` | ~8 kbps | Low bandwidth, licensed; common on trunks |
| `gsm` | ~13 kbps | Lightweight, free |
| `alaw` / `ulaw` (G.711) | ~64 kbps | Uncompressed; best quality, high bandwidth |

`Disallow: all` followed by an explicit allow-list is the recommended, predictable pattern.

---

## 5. Routing — Prefixes & Destinations

A **Prefix** maps the leading digits of a dialled number to a destination. This lab used `91 → India`.

![Dial prefix → destination mapping](../assets/screenshots/dial-prefix-rate-routing.png)

```
Prefix   Destination
──────   ───────────
91       India
```

When a call is dialled, MagnusBilling matches the **longest prefix**, looks up the **rate** for that
destination, checks the account's **credit**, and selects a **route/trunk**.

---

## 6. Rates, Tariffs & Routes

- **Tariff** — a named table of per-destination costs.
- **Rate** — the buy/sell price per minute for a prefix within a tariff.
- **Route** — binds a prefix/destination to the **trunk** that will carry it, with priority and
  failover ordering.

A typical chain:

```
Dialled number ─► longest-prefix match ─► destination ─► rate lookup
              ─► credit check ─► route selection ─► trunk ─► outbound INVITE
```

---

## 7. Call Detail Records (CDR)

Every call writes a CDR row (Reports → CDR) with: date/time, SIP user, source CLI, destination,
duration, trunk, call type, and buy/sell cost. CDRs are the primary source for **billing,
troubleshooting, and fraud detection** — sudden spikes in volume, cost, or calls to high-rate
destinations are classic fraud signals.

![Call Detail Records (CDR) report](../assets/screenshots/call-detail-records-cdr.png)

> ⚠️ **Privacy — redact before publishing.** Real CDR contains **third-party personal data** (phone
> numbers and usernames). Black out all real numbers/usernames, or use demo data. Never publish
> unredacted call records.

---

## 8. Monitoring Dashboard

The MagnusBilling home dashboard gives a live operational view: server hardware (CPU, memory,
network), simultaneous calls over time, calls-per-second (CPS), active channels, and per-trunk
distribution.

![MagnusBilling system dashboard overview](../assets/screenshots/system-dashboard-overview.png)

![MagnusBilling admin dashboard](../assets/screenshots/magnusbilling-admin-dashboard.png)

These metrics double as **fraud indicators** — unexpected spikes in concurrent calls, CPS, or spend
warrant investigation (see [06 — Defenses §6](06-defenses-and-countermeasures.md)).

> ⚠️ **Redact before publishing** any real subscriber counts, revenue, or trunk names.

---

## 9. Email Notifications (optional)

MagnusBilling can send alerts/invoices over SMTP. This lab used a Hostinger mailbox; the IMAP/SMTP
settings are the provider's standard values (IMAP `993/SSL`, SMTP `465/SSL`).

![Hostinger IMAP/SMTP settings](../assets/screenshots/hostinger-imap-smtp-setup.png)

![Hostinger POP3 settings](../assets/screenshots/hostinger-pop3-setup.png)

> ⚠️ **Redact before publishing** your real email address/domain, and never put a mailbox password
> in a screenshot or config file.

---

## 10. Configuration Checklist

- [ ] Admin password changed from default
- [ ] SIP passwords are strong and unique (not `demo12`)
- [ ] `Disallow: all` + explicit codec allow-list on every SIP user
- [ ] CallerID values are numbers you own
- [ ] Prepaid credit limits + concurrency caps set per account
- [ ] Prefixes/rates/routes defined for intended destinations only
- [ ] `Host` pinned to a known IP where feasible (vs. `dynamic`)
- [ ] fail2ban + firewall protecting SIP/UI

---

**Next:** [04 — Caller ID Spoofing Mechanics →](04-callerid-spoofing-mechanics.md)
