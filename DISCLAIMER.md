# Disclaimer & Responsible-Use Policy

## 1. Purpose

This repository is an **educational and defensive security** resource. It documents how a VoIP
telephony stack (Asterisk + MagnusBilling) works and how the Session Initiation Protocol (SIP)
handles — and fails to authenticate — Caller Line Identification (CLI). Its goal is to help
students, network engineers, and defenders **understand and mitigate** Caller ID spoofing, which
is one of the most common enablers of telephone fraud.

## 2. No Operational Attack Material

This repository intentionally contains:

- **No credentials, API keys, server IPs, or hostnames.**
- **No real subscriber, customer, or call-detail-record (CDR) data.**
- **No scripts, dialers, or tooling for mass calling or victim targeting.**
- **No instructions for impersonating real people, businesses, or government entities.**

All screenshots have been reviewed and sanitised; example values (e.g. `demo`, `919999999999`)
are placeholders.

## 3. Legality

Transmitting misleading or inaccurate Caller ID information **with intent to defraud, cause harm,
or wrongfully obtain anything of value is a crime** in most countries, including but not limited to:

- 🇺🇸 United States — **Truth in Caller ID Act of 2009**, 47 U.S.C. § 227(e); enforced by the FCC.
- 🇬🇧 United Kingdom — **Communications Act 2003**, s.127; enforced by Ofcom.
- 🇮🇳 India — **Indian Telegraph Act** and **TRAI** regulations; unauthorised CLI manipulation is prohibited.
- 🇪🇺 European Union — national implementations of the **EECC (Directive 2018/1972)**.

Operating a telephony service, SIP trunk, or DID without the proper licensing and carrier
agreements may **independently** violate telecom regulations, regardless of how calls are placed.

## 4. Authorised Use Only

You may use the information in this repository **only** when **all** of the following are true:

1. You own or have explicit written authorisation for every system involved.
2. All endpoints, numbers, and recipients are **yours** or are part of a closed test lab.
3. Your activity complies with every applicable local, national, and international law.
4. You are acting for **learning, research, detection-engineering, or defensive** purposes.

## 5. No Warranty / No Liability

This material is provided **"as is"**, without warranty of any kind. The author(s) accept **no
liability** for any damage, loss, legal consequence, or misuse arising from the use of this
information. **You are solely responsible for your own actions.**

## 6. If You Are a Defender

If you arrived here to protect users and networks, start with
[`docs/06-defenses-and-countermeasures.md`](docs/06-defenses-and-countermeasures.md). That document
covers STIR/SHAKEN attestation, SIP over TLS + SRTP, call-pattern anomaly detection, and carrier-side
CLI validation.

---

**By cloning, forking, or using this repository you acknowledge that you have read and accepted
this disclaimer.**
