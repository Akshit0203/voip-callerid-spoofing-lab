# 02 — Installation

> Building the telephony stack on a fresh **Debian 11 (Bullseye)** VPS: system prep, dependencies,
> Asterisk (compiled from source), and MagnusBilling.

> ⚠️ Perform this only on a server **you own**. Replace every placeholder (IP, domain, password)
> with your own values and **never commit credentials** — see [`.gitignore`](../.gitignore).

---

## 1. Prerequisites

| Requirement | Value used in this lab |
|-------------|------------------------|
| OS | Debian 11 (Bullseye), 64-bit |
| RAM | 8 GB |
| Disk | ≥ 40 GB |
| Access | `root` (or sudo) over SSH |
| Network | Static public IP; firewall control |

> MagnusBilling ships an **automated installer** that pulls in Asterisk, MariaDB, Apache, PHP, and
> all Perl/ODBC dependencies. The steps below mirror what that installer does and what was observed
> during this build.

---

## 2. System Preparation

```bash
# Update the base system
apt update && apt upgrade -y

# Core build + utility tooling
apt install -y wget curl sudo gnupg2 lsb-release ca-certificates git
```

Set a correct hostname and timezone (accurate timestamps matter for CDR):

```bash
timedatectl set-timezone Asia/Kolkata   # adjust to your region
```

---

## 3. Dependencies (Perl / ODBC / build toolchain)

The installer resolves a large set of Perl modules and the unixODBC stack that Asterisk and
MagnusBilling rely on (MySQL connectivity via ODBC, CDR handling, and build utilities such as
`devscripts`, `equivs`, and `lintian`).

![Debian Perl & ODBC dependencies being configured](../assets/screenshots/debian-perl-dependencies.png)

Representative packages pulled in during this phase:

```
libwww-perl        liblwp-protocol-https-perl   libxml-parser-perl
unixodbc           unixodbc-dev                 odbcinst / odbcinst1debian2
equivs             devscripts                   lintian         licensecheck
```

---

## 4. Regional Telephony Code (libvpb1)

During dependency setup, `libvpb1` prompts for the **ITU-T country code** that sets regional
telephony defaults. This lab used **`91` (India)**.

![libvpb1 ITU-T country code prompt](../assets/screenshots/libvpb1-country-code-config.png)

> Use the code matching the region your lab is legally operating in.

---

## 5. Compiling Asterisk from Source

Asterisk is built from source so the exact modules (SIP channel driver, codecs, RTP engine) are
compiled against the host. The build compiles hundreds of C objects — core PBX subsystems, the SIP
API, SDP/SRTP handling, the Stasis message bus, and more.

![Asterisk compiling from source (CC object build)](../assets/screenshots/asterisk-source-compilation.png)

Conceptually (handled by the installer):

```bash
# fetch → ./configure → menuselect → make → make install → make samples → make config
cd /usr/src
# (installer downloads and builds the pinned Asterisk release here)
```

Key modules relevant to this lab:

| Module | Role |
|--------|------|
| `chan_sip` / `chan_pjsip` | SIP channel driver (registrations + INVITEs) |
| `res_rtp_asterisk` | RTP media engine |
| `codec_*` | Codec transcoding (g729, gsm, ulaw, alaw, …) |
| `cdr_adaptive_odbc` | Writes CDR to MySQL via ODBC |
| `res_odbc` | ODBC connection pool |

---

## 6. Downloading & Installing MagnusBilling

MagnusBilling is fetched from the official distribution (`MagnusBilling-current.tar.gz`, ~65 MB)
and its installer wires MagnusBilling to Asterisk and the database.

![MagnusBilling installer downloading the current release](../assets/screenshots/magnusbilling-update-download.png)

Typical bootstrap (always review a script before running it):

```bash
cd /usr/src
wget https://raw.githubusercontent.com/magnussolution/magnusbilling7/master/install.sh
# REVIEW the script first:
less install.sh
# Then run it on a server you own:
bash install.sh
```

The installer will:

1. Install MariaDB and create the `mbilling` database.
2. Deploy the PHP app under `/var/www/html/mbilling`.
3. Configure Apache + PHP.
4. Link MagnusBilling to Asterisk via the **Asterisk Manager Interface (AMI)**.
5. Generate the admin and database credentials.

---

## 7. Installation Complete

On success the installer prints the access URL and the generated credentials
(admin login + MySQL root password).

![MagnusBilling installation result screen](../assets/screenshots/installation-success-result.png)

> 🔒 **Security — important:** The completion screen displays **live credentials** (admin login and
> the generated MySQL root password). **Change the default admin password and the MySQL root
> password immediately**, restrict the admin UI to trusted IPs, and never expose these values.
> **Redact any real password before publishing this screenshot** (see the note at the end of this
> page).

After install, the MagnusBilling application lives under `/var/www/html/mbilling`:

![MagnusBilling application folder contents](../assets/screenshots/mbilling-folder-contents.png)

> 🔒 **Redact your real server IP/hostname** from any terminal screenshot before publishing.

Default landing point:

```
http://<YOUR_SERVER_IP>/mbilling
Username: root
Password: <changed on first login>
```

---

## 8. Post-Install Hardening (do this before anything else)

```bash
# 1. Change the MagnusBilling admin password via the UI (Clients → Users → root)
# 2. Change the MySQL root password
mysqladmin -u root -p'OLD_PASS' password 'NEW_STRONG_PASS'

# 3. Restrict the admin panel + SIP to known sources with a firewall (example: ufw)
apt install -y ufw
ufw allow from <YOUR_ADMIN_IP> to any port 80,443 proto tcp
ufw allow 5060/udp         # SIP  (tighten to known peers where possible)
ufw allow 10000:20000/udp  # RTP media
ufw enable

# 4. Install fail2ban for SIP brute-force protection
apt install -y fail2ban
```

More hardening — SIP/TLS, SRTP, and anti-fraud controls — is covered in
[06 — Defenses & Countermeasures](06-defenses-and-countermeasures.md).

---

## 9. ⚠️ Screenshot Redaction Note

Some screenshots in this repo were taken from a live build and may show sensitive values. **Before
making this repository public, black out:**

- **Passwords / secrets** — e.g. the MySQL root password on the installation-result screen.
- **Server IPs & hostnames** — e.g. in terminal screenshots.
- **Phone numbers & usernames** — any real Caller IDs, DIDs, or client names in CDR/dashboards.
- **Financial / subscriber figures** — real revenue, credit, or active-user counts.

Publishing live credentials or third-party phone numbers can be a security incident and a privacy
violation. Redact first, or replace with `demo`/placeholder values.

---

**Next:** [03 — Configuration →](03-configuration.md)
