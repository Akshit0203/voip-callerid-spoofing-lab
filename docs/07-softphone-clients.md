# 07 — Softphone Clients

> Registering SIP endpoints (softphones) against the lab PBX and the codecs they negotiate.

> ⚠️ Register only to a PBX **you own**, using credentials **you created**, and place calls only to
> endpoints/numbers **you control**.

---

## 1. What a Softphone Is

A **softphone** is a software SIP **User Agent (UAC)** that registers to Asterisk with a SIP
username/password and can originate/receive calls. It is the endpoint that sends the `REGISTER` and
`INVITE` messages discussed throughout this repo.

---

## 2. Clients Used / Tested

| Platform | Softphones |
|----------|------------|
| **Windows** | PortSIP, MicroSIP, Zoiper |
| **Android** | MizuDroid, PortSIP, Zoiper |
| **macOS** | Zoiper, MicroSIP, Telephone (Elite) |
| **iOS** | MicroSIP, Zoiper |

> **MicroSIP** (Windows) and **Zoiper** (cross-platform) are the simplest for a quick lab endpoint.

---

## 3. Registering an Endpoint

Each softphone needs the SIP account from [03 — Configuration §4](03-configuration.md):

| Setting | Value (example) |
|---------|-----------------|
| SIP server / domain | `<YOUR_PBX_IP>` |
| Username / SIP user | `demo` |
| Password | `demo12` *(use a strong value in practice)* |
| Transport | UDP 5060 *(prefer TLS 5061 — see [06 — Defenses](06-defenses-and-countermeasures.md))* |
| Codecs | g729, gsm, alaw, ulaw |

### MicroSIP (example)

```
Menu → Add Account
  Account name : Lab Demo
  SIP server   : <YOUR_PBX_IP>
  Username     : demo
  Domain       : <YOUR_PBX_IP>
  Password     : ********
  Transport    : UDP  (use TLS where configured)
→ Save → status should show "Online / Registered"
```

A successful registration appears in **MagnusBilling → Clients → Calls Online** and in the
dashboard's channel count.

---

## 4. Codec Negotiation

The codecs enabled on the softphone must intersect with the SIP user's allow-list, or the call
fails to negotiate media.

| Codec | Bandwidth | Trade-off |
|-------|-----------|-----------|
| **G.729** | ~8 kbps | Low bandwidth; licensed |
| **GSM** | ~13 kbps | Lightweight, free |
| **G.711 (alaw/ulaw)** | ~64 kbps | Best quality; high bandwidth |

Negotiation is visible in the SDP body of the INVITE/200 OK — see
[05 — SIP Traffic Analysis §4](05-sip-traffic-analysis.md).

---

## 5. Verifying a Call End to End

1. Register two endpoints you own to the PBX.
2. Place a call between them (internal) or to a number/DID **you own**.
3. Confirm:
   - `Calls Online` shows the active channel.
   - A **CDR** row is written (Reports → CDR).
   - A packet capture shows the full `REGISTER`/`INVITE`/`BYE` flow.

---

## 6. Endpoint Hardening

- Use **unique, strong SIP secrets** per endpoint.
- Prefer **TLS transport** and **SRTP** where the PBX supports it.
- Keep softphone apps updated (they are SIP-facing software).
- Don't save credentials in shared/synced notes or commit them to version control.

---

**Back to:** [README →](../README.md) · [Defenses →](06-defenses-and-countermeasures.md)
