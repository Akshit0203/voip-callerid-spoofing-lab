# 06 — Defenses & Countermeasures

> **The most important document in this repository.** Understanding how Caller ID is forged is only
> useful if it leads to defense. This covers the controls that detect, prevent, and deter CLI
> spoofing — from protocol-level authentication to operational anti-fraud.

---

## 1. STIR/SHAKEN — Cryptographic Caller ID Authentication

**The direct answer to the trust gap** described in [04 — Mechanics](04-callerid-spoofing-mechanics.md).

| Term | Meaning |
|------|---------|
| **STIR** | *Secure Telephony Identity Revisited* — IETF framework (RFC 8224) that signs the calling number in SIP. |
| **SHAKEN** | *Signature-based Handling of Asserted information using toKENs* — the carrier deployment profile (RFC 8588). |
| **PASSporT** | A signed JWT (RFC 8225) carrying the identity claim, attached via the SIP **`Identity`** header. |

### How it works

1. The **originating carrier** cryptographically signs the calling number using a certificate from
   a trusted Certificate Authority, and asserts an **attestation level**:
   - **A (Full):** carrier authenticated the customer **and** confirmed they own the number.
   - **B (Partial):** customer authenticated, but number ownership not confirmed.
   - **C (Gateway):** call accepted from an unverifiable source (e.g. an international gateway).
2. The signature travels in the SIP **`Identity`** header as a PASSporT token.
3. The **terminating carrier** verifies the signature and attestation before the call is presented.

> Spoofed calls either carry **no valid signature** or only a **low attestation (C)**, letting
> networks label, deprioritise, or block them. STIR/SHAKEN is mandated/encouraged by regulators
> (e.g. FCC in the US) precisely to kill the forgery this lab studies.

**References:** [RFC 8224](https://www.rfc-editor.org/rfc/rfc8224) ·
[RFC 8588](https://www.rfc-editor.org/rfc/rfc8588) ·
[FCC Call Authentication](https://www.fcc.gov/call-authentication)

---

## 2. Secure the Signalling & Media (SIP/TLS + SRTP)

Plain UDP/5060 (used in this lab for visibility) is the worst case: readable, modifiable, spoofable.

| Control | Protects against | Asterisk mechanism |
|---------|------------------|--------------------|
| **SIP over TLS (5061)** | Signalling tampering, header injection, eavesdropping on SIP | `transport=tls` (pjsip), certificates |
| **SRTP** | Media interception / playback | `media_encryption=sdes` or DTLS-SRTP |
| **Mutual TLS to trunks** | Rogue peers injecting calls | Certificate-pinned trunk transports |

```ini
; pjsip.conf (illustrative)
[transport-tls]
type = transport
protocol = tls
bind = 0.0.0.0:5061
cert_file = /etc/asterisk/keys/asterisk.crt
priv_key_file = /etc/asterisk/keys/asterisk.key

[demo]
type = endpoint
transport = transport-tls
media_encryption = sdes      ; SRTP
```

---

## 3. Trunk-Side CLI Validation (The Operator's Duty)

The single most effective operator control: **do not blindly relay customer-supplied CLI.**

- **Allow-list numbers per account.** Only permit a SIP account to present numbers it has **proven
  it owns** (verified DIDs). Reject or overwrite anything else.
- **Overwrite, don't pass through.** On outbound trunks, set the CLI to an allocated, owned number
  rather than trusting the INVITE's `From`/`PAI`.
- **Strip/normalise identity headers** from untrusted sources (`P-Asserted-Identity`,
  `Remote-Party-ID`) before forwarding.

```
; Asterisk dialplan concept — enforce an authorised CLI
exten => _X.,1,Set(ALLOWED=${DB(authcli/${CHANNEL(accountcode)})})
 same => n,ExecIf($["${CALLERID(num)}" != "${ALLOWED}"]?Set(CALLERID(num)=${ALLOWED}))
 same => n,Dial(PJSIP/${EXTEN}@trusted-trunk)
```

---

## 4. Account & Access Hardening

| Control | Why |
|---------|-----|
| **Strong, unique SIP secrets** | Stops registration brute-force / account takeover (never `demo12`). |
| **`Host` pinned to known IP** | Rejects registrations/INVITEs from unexpected sources (vs. `dynamic`). |
| **Concurrency caps per account** | Limits blast radius of a compromised account. |
| **Low prepaid credit** | Caps financial damage from toll fraud. |
| **Destination allow-lists** | Block unexpected high-cost international destinations by default. |
| **Codec allow-lists** | `Disallow: all` then explicit allow — predictable, smaller attack surface. |
| **Admin UI IP allow-list + HTTPS** | Keep MagnusBilling off the open internet. |

---

## 5. Brute-Force & Scanning Protection

VoIP servers are constantly scanned for weak SIP accounts.

```bash
# fail2ban for Asterisk SIP auth failures
apt install -y fail2ban
```

```ini
# /etc/fail2ban/jail.d/asterisk.conf
[asterisk]
enabled  = true
filter   = asterisk
logpath  = /var/log/asterisk/messages
maxretry = 5
bantime  = 86400
action   = iptables-allports[name=ASTERISK, protocol=all]
```

Complement with: a restrictive firewall (only known peers on 5060/5061), `alwaysauthreject=yes`
(don't leak which extensions exist), and disabling guest/anonymous calls.

---

## 6. Fraud Detection via CDR & Behaviour

CDR is a rich detection source. Alert on:

- **Volume spikes** — sudden surges in CPS or concurrent channels.
- **Cost spikes** — calls to expensive/premium or unusual international destinations.
- **CLI anomalies** — asserted CLI not owned by the account (cross-check with §3 allow-list).
- **Off-pattern timing** — heavy traffic at unusual hours.
- **Geo-impossibility** — same account registering/calling from distant IPs rapidly.

> Pipe SIP metadata (e.g. via **Homer/HEP** or `sngrep`) and CDR into a SIEM, and alert when
> *asserted identity* ≠ *authorised identity*.

---

## 7. Regulatory & Carrier-Level Controls

- **STIR/SHAKEN mandates** (US FCC, and growing adoption elsewhere).
- **Robocall mitigation / traceback** programs (e.g. Industry Traceback Group).
- **Know-Your-Customer (KYC)** for number allocation and trunk provisioning.
- **National Do-Not-Call / CLI integrity rules** (e.g. Ofcom, TRAI).

These push authentication and accountability **upstream**, where individual CLI forgery is hardest.

---

## 8. Defender's Checklist

- [ ] SIP/TLS (5061) + SRTP enabled; plain 5060 restricted or disabled externally
- [ ] Per-account **authorised CLI allow-list**; untrusted identity headers stripped/overwritten
- [ ] Strong unique SIP secrets; `Host` pinned where possible
- [ ] Concurrency + credit + destination limits per account
- [ ] fail2ban + firewall + `alwaysauthreject=yes`; guest calls disabled
- [ ] Admin UI behind HTTPS + IP allow-list
- [ ] CDR/SIP anomaly alerting (volume, cost, CLI mismatch, geo)
- [ ] STIR/SHAKEN in place (or required of your upstream carrier)
- [ ] Incident/traceback process documented

---

## Key Takeaway

> Caller ID spoofing exists because **legacy SIP authenticates nothing about the caller's identity.**
> Every effective defense works by **adding that missing verification** — cryptographically
> (STIR/SHAKEN), at the trunk (CLI allow-listing), and operationally (anomaly detection). Studying
> the attack in a controlled lab is what makes building those defenses possible.

---

**Next:** [07 — Softphone Clients →](07-softphone-clients.md)
