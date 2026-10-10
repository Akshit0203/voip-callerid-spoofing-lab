# 01 — Architecture

> How the lab is designed, which components talk to each other, and how a single call travels
> from a softphone to the outside world.

---

## 1. Design Goals

The lab was built on a **single Debian 11 VPS** with three goals:

1. **Transparency** — compile and configure each layer by hand so the SIP/RTP internals are visible.
2. **Isolation** — a self-contained environment where every endpoint and number is controlled.
3. **Observability** — full Call Detail Records (CDR), a monitoring dashboard, and packet capture.

---

## 2. Component Stack

```
┌──────────────────────────────────────────────────────────────────────┐
│                          DEBIAN 11 VPS (host)                          │
│                                                                        │
│   ┌────────────────────┐        ┌──────────────────────────────────┐  │
│   │   MagnusBilling     │  AMI   │           Asterisk PBX           │  │
│   │  (PHP / Apache UI)  │◄──────►│  (compiled from source)          │  │
│   │                     │        │                                  │  │
│   │  • SIP users        │        │  • chan_sip / PJSIP              │  │
│   │  • Trunks           │        │  • Dialplan (extensions)         │  │
│   │  • Rates / Prefixes │        │  • Codec negotiation             │  │
│   │  • Routes           │        │  • RTP media engine              │  │
│   │  • CDR / Reports    │        │                                  │  │
│   └─────────┬───────────┘        └───────────────┬──────────────────┘  │
│             │ MySQL (mbilling DB)                 │                     │
│   ┌─────────▼───────────┐                         │                     │
│   │      MariaDB        │                         │                     │
│   └─────────────────────┘                         │                     │
└───────────────────────────────────────────────────┼─────────────────────┘
          ▲  SIP REGISTER / INVITE (UDP 5060)        │  SIP INVITE
          │  RTP media (UDP 10000–20000)             ▼  (to upstream trunk)
   ┌──────┴───────┐                           ┌─────────────────────────┐
   │  Softphones  │                           │   Upstream SIP Trunk /   │
   │ MicroSIP /   │                           │   ITSP (wholesale VoIP)  │
   │ Zoiper /     │                           └─────────────────────────┘
   │ MizuDroid    │
   └──────────────┘
```

### Roles at a glance

| Component | Function |
|-----------|----------|
| **Asterisk** | The PBX engine. Terminates SIP registrations, runs the dialplan, negotiates codecs, bridges RTP media, and hands outbound legs to upstream trunks. |
| **MagnusBilling** | A web front-end + billing layer over Asterisk. Manages SIP accounts, trunks, rate tables, routing prefixes, and records every call in the CDR. Talks to Asterisk via the **Asterisk Manager Interface (AMI)**. |
| **MariaDB / MySQL** | Stores the MagnusBilling configuration and CDR. |
| **Apache + PHP** | Serves the MagnusBilling admin UI. |
| **Softphones** | SIP user agents (UACs) that register to Asterisk and originate calls. |
| **Upstream trunk** | A wholesale VoIP provider that carries the call onto the PSTN. *(Out of scope for the lab — numbers/recipients must be ones you control.)* |

---

## 3. Protocols

| Protocol | Transport | Purpose | Port(s) |
|----------|-----------|---------|---------|
| **SIP** | UDP/TCP/TLS | Signalling: `REGISTER`, `INVITE`, `ACK`, `BYE` | 5060 (UDP/TCP), 5061 (TLS) |
| **RTP** | UDP | Media (audio) stream | 10000–20000 (configurable) |
| **AMI** | TCP | Asterisk ⇄ MagnusBilling control | 5038 (localhost) |
| **HTTP/HTTPS** | TCP | MagnusBilling admin UI | 80 / 443 |
| **MySQL** | TCP | Database | 3306 (localhost) |

> **Security note:** In this lab, `chan_sip` over plain UDP/5060 is used for clarity, which is
> precisely why Caller ID is trivially forgeable. Production systems should use **SIP over TLS
> (5061)** and **SRTP** — covered in [06 — Defenses](06-defenses-and-countermeasures.md).

---

## 4. Call Flow — Step by Step

A simplified outbound call from a registered softphone:

```
Softphone (UAC)                Asterisk / MagnusBilling                 Upstream Trunk
     │                                   │                                    │
     │ 1. REGISTER (auth) ──────────────►│   SIP account authenticated        │
     │ ◄─────────── 200 OK               │   against mbilling DB              │
     │                                   │                                    │
     │ 2. INVITE  (From: <CallerID>) ───►│                                    │
     │    SDP: offered codecs            │ 3. Match dialled prefix (e.g. 91)  │
     │                                   │    → select rate + route           │
     │                                   │ 4. Apply account CallerID field    │
     │                                   │ 5. INVITE ───────────────────────► │
     │                                   │    (From/PAI carried to trunk)     │
     │ ◄─────── 180 Ringing ─────────────│ ◄──────────── 180 Ringing          │
     │ ◄─────── 200 OK (SDP answer) ─────│ ◄──────────── 200 OK               │
     │ 6. ACK ──────────────────────────►│ ───────────────────────────────► │
     │ ═══════════ RTP media (bidirectional audio) ═══════════════════════════│
     │ 7. BYE ──────────────────────────►│ ───────────────────────────────► │
     │                                   │ 8. Write CDR row                   │
```

**The critical step is #4.** The number shown to the far end is taken from the **`CallerID`
field of the SIP account** (or the `From` / `P-Asserted-Identity` header of the INVITE). Legacy SIP
has **no mechanism to verify that the originator is entitled to that number** — the trunk simply
relays whatever it receives. That single design gap is the entire basis of CLI spoofing, and the
reason STIR/SHAKEN was created. See
[04 — Caller ID Spoofing Mechanics](04-callerid-spoofing-mechanics.md).

---

## 5. Observability

- **Dashboard** — live view of simultaneous calls, CPS (calls-per-second), channel count, and
  server hardware (CPU / memory / network).
- **CDR** — one row per call: timestamp, SIP user, source CLI, destination, duration, trunk, and cost.
- **Packet capture** — `tcpdump` / Wireshark on UDP 5060 to inspect raw SIP headers
  ([05 — SIP Traffic Analysis](05-sip-traffic-analysis.md)).

---

**Next:** [02 — Installation →](02-installation.md)
