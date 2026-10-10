# 04 — Caller ID Spoofing Mechanics

> **Why** Caller ID can be forged in legacy SIP, explained at the protocol level. This is a
> conceptual, defensive explanation — **not** an operational guide. It exists so defenders can
> understand the weakness they are mitigating.

> ⚠️ Presenting a Caller ID you are not authorised to use, to anyone other than your own test
> endpoints, is illegal in most countries. See [`DISCLAIMER.md`](../DISCLAIMER.md).

---

## 1. The Core Problem: SIP Trusts the Caller

Caller ID (CLI / CLID) in VoIP is **not a verified property of the network** — it is **metadata
asserted by the originating endpoint** and relayed hop by hop. In the classic PSTN and in legacy
SIP, each network largely **trusts** the identity presented by the network before it. There is no
built-in cryptographic binding between "the number displayed" and "the entity that actually placed
the call."

That trust model is the vulnerability. STIR/SHAKEN (covered in
[06 — Defenses](06-defenses-and-countermeasures.md)) was created specifically to add the missing
proof.

---

## 2. Where the Identity Lives in a SIP INVITE

Caller identity is carried in headers of the SIP `INVITE`:

```
INVITE sip:919876543210@trunk.example.net SIP/2.0
Via: SIP/2.0/UDP 203.0.113.10:5060;branch=z9hG4bK-xxxx
From: "Demo User" <sip:919999999999@pbx.lab.local>;tag=as1b2c3d4   ◄── displayed CLI
To:   <sip:919876543210@trunk.example.net>
Contact: <sip:demo@203.0.113.10:5060>
P-Asserted-Identity: <sip:919999999999@pbx.lab.local>              ◄── "network-asserted" CLI
Call-ID: 8f2a...@pbx.lab.local
CSeq: 102 INVITE
```

| Header | Role | Trust in legacy SIP |
|--------|------|---------------------|
| **`From`** | The display identity chosen by the UAC (user agent). | Set by the caller; freely editable. |
| **`P-Asserted-Identity` (PAI)** | The identity a *trusted* network element asserts (RFC 3325). | Only meaningful **inside** a trusted domain; often passed through unverified. |
| **`Remote-Party-ID`** | Older, pre-PAI identity header. | Same weakness. |
| **`Contact`** | Where to reach the UAC directly. | Routing, not identity. |

**The gap:** nothing in plain SIP cryptographically proves the sender is entitled to the number in
`From` / `PAI`. If an intermediate carrier accepts and forwards whatever it receives, the displayed
number is whatever the originator put there.

---

## 3. Why the Trunk Relays It

A wholesale/ITSP trunk decides whether to:

- **Pass through** the customer-supplied CLI (common for "bring-your-own-CLI" wholesale routes), or
- **Overwrite** it with an allocated number the customer is verified to own (the correct, defensive behaviour).

Legacy and loosely-configured trunks often do the former. Combined with SIP's lack of origin
authentication, that is the entire mechanism: **the PBX sets a `From`/`CallerID` value, and an
unverifying trunk delivers it to the called party's phone.**

In MagnusBilling terms, this maps directly to the **`CallerID` field** on the SIP account
(see [03 — Configuration §4](03-configuration.md)), which is injected into the outbound INVITE.

---

## 4. Why It Matters (Threat Context)

Understanding the mechanism explains why these attacks are so effective and so common:

| Abuse pattern | How the CLI weakness enables it |
|---------------|---------------------------------|
| **Vishing / impersonation** | Display a bank, government, or support number to gain trust. |
| **Neighbor spoofing** | Forge a CLI sharing the victim's area/prefix to boost answer rates. |
| **Robocall campaigns** | Rotate forged CLIs to evade simple block-lists. |
| **SIM-swap / account takeover** | Spoof a victim's number to pass caller-ID-based verification. |
| **TDoS** | Flood a target with calls from forged, untraceable numbers. |

Each of these relies on the **absence of origin authentication**. Remove that gap — with
attestation and signed identity — and the attack collapses. That is the thesis of this entire lab.

---

## 5. What This Lab Shows vs. What It Does Not

| ✅ This lab demonstrates (educationally) | ❌ This lab does **not** provide |
|------------------------------------------|----------------------------------|
| That `From`/`PAI` are caller-set fields | Scripts to mass-dial or target victims |
| That unverifying trunks relay CLI as-is | Instructions to impersonate real entities |
| How to read this in a packet capture | Any real numbers, DIDs, or recipient lists |
| Why STIR/SHAKEN closes the gap | Any way to bypass carrier authentication |

The forgery is only ever exercised **between endpoints the operator owns**, to study the signalling
and to validate detection. Any other use is fraud.

---

## 6. From Mechanism → Defense

Now that the weakness is clear, the remaining documents turn it into defensive knowledge:

- **[05 — SIP Traffic Analysis](05-sip-traffic-analysis.md)** — see the `From`/`PAI` headers on the
  wire, which is the basis of any detection logic.
- **[06 — Defenses & Countermeasures](06-defenses-and-countermeasures.md)** — STIR/SHAKEN
  attestation, SIP/TLS + SRTP, trunk-side CLI validation, and CDR anomaly detection.

---

**Next:** [05 — SIP Traffic Analysis →](05-sip-traffic-analysis.md)
