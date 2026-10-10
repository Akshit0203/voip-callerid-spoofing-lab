# 05 — SIP Traffic Analysis

> Capturing and dissecting SIP signalling and RTP media with `tcpdump` and Wireshark — the
> foundation for understanding call flows and building spoofing **detection** logic.

> ⚠️ Capture only traffic on systems **you own/operate**. Packet captures contain phone numbers and
> can contain decodable audio — treat them as sensitive and **never commit `.pcap` files**
> (they are in [`.gitignore`](../.gitignore)).

---

## 1. Why Packet Analysis

The displayed Caller ID lives in SIP headers on the wire. To understand — or detect — spoofing you
must read those headers directly rather than trusting a UI. Wireshark also reconstructs call flow,
exposes codec negotiation, and reveals media (RTP) paths.

---

## 2. Capturing SIP Traffic

On the PBX host, capture the SIP signalling port (default UDP 5060) plus the RTP media range:

```bash
# SIP signalling only
tcpdump -i any -n -s 0 port 5060 -w /tmp/sip-capture.pcap

# SIP + RTP media range (adjust to your rtp.conf)
tcpdump -i any -n -s 0 'port 5060 or (udp portrange 10000-20000)' -w /tmp/voip-capture.pcap
```

Transfer the `.pcap` to a workstation and open it in **Wireshark**.

> Asterisk CLI alternative — live SIP tracing without a capture file:
> ```
> asterisk -rvvv
> sip set debug on          ; chan_sip
> pjsip set logger on       ; chan_pjsip
> ```

---

## 3. Reading the INVITE in Wireshark

Useful display filters:

| Goal | Filter |
|------|--------|
| All SIP | `sip` |
| Only INVITEs | `sip.Method == "INVITE"` |
| A specific call | `sip.Call-ID == "<call-id>"` |
| Caller ID header | `sip.From` |
| Asserted identity | `sip.P-Asserted-Identity` |
| RTP media | `rtp` |

Use **Telephony → VoIP Calls** to list calls and **Flow Sequence** to draw the ladder diagram
(`INVITE → 100 → 180 → 200 → ACK → … → BYE`).

### What to inspect on the INVITE

```
From:                  "Demo User" <sip:919999999999@pbx.lab.local>   ← displayed CLI
P-Asserted-Identity:   <sip:919999999999@pbx.lab.local>               ← asserted CLI
Contact:               <sip:demo@203.0.113.10:5060>                   ← actual source host
Via:                   SIP/2.0/UDP 203.0.113.10 ...                   ← actual path
```

**Detection insight:** the identity headers (`From`/`PAI`) can be compared against the **real
network origin** (`Via` / `Contact` / source IP) and against the set of numbers the account is
**authorised** to present. A mismatch between *asserted identity* and *authorised identity* is the
core signal of CLI spoofing — and the basis for the detection rules in
[06 — Defenses](06-defenses-and-countermeasures.md).

---

## 4. Inspecting the SDP (Codec Negotiation)

The `INVITE`/`200 OK` bodies carry **SDP**, which negotiates media:

```
m=audio 17654 RTP/AVP 18 3 0 8
a=rtpmap:18 G729/8000
a=rtpmap:3  GSM/8000
a=rtpmap:0  PCMU/8000
a=rtpmap:8  PCMA/8000
```

This confirms the codecs agreed (matching the SIP-user allow-list from
[03 — Configuration](03-configuration.md)) and the RTP port/address for the media stream.

---

## 5. RTP Media

With `rtp` selected, **Telephony → RTP → RTP Streams** shows jitter, packet loss, and stream
direction. For streams you own and are authorised to analyse, Wireshark can also play back audio —
a reminder that **unencrypted RTP is fully interceptable**, which is why **SRTP** matters
(see [06 — Defenses](06-defenses-and-countermeasures.md)).

---

## 6. From Capture → Detection

A practical detection pipeline built on these observations:

1. **Mirror/parse SIP** at the PBX or SBC (e.g. `sip set debug`, `sngrep`, Homer/HEP).
2. **Extract** `From`, `P-Asserted-Identity`, `Contact`, source IP per INVITE.
3. **Compare** asserted CLI against the account's authorised number set.
4. **Score/alert** on mismatches, impossible geographies, or CLI not owned by the account.
5. **Correlate** with CDR for volume/cost anomalies.

> `sngrep` is an excellent terminal tool for live SIP flow inspection on the PBX:
> ```bash
> apt install -y sngrep && sngrep
> ```

---

**Next:** [06 — Defenses & Countermeasures →](06-defenses-and-countermeasures.md)
