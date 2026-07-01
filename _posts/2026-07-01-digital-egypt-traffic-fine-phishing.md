---
layout: post
title: "Inside a Live 'Digital Egypt' Traffic-Fine Phishing Campaign (Chinese Live-Interception PhaaS)"
date: 2026-07-01 18:00:00 +0200
categories: threat-intel
tags: [phishing, threat-intel, phaas, egypt, otp-interception, digital-egypt]
---

> **Disclosure note.** This write-up documents the *observable behaviour* of a live phishing kit impersonating the Egyptian government. All findings come from the attacker-controlled lure pages and their publicly reachable client-side code. **No real victim data was accessed or retrieved**, and no working exploitation steps are published here. The relevant national CERT, the impersonated brand, and the hosting provider / registrars were notified through abuse channels; indicators are shared to help defenders and to warn the public. (TLP:CLEAR)

---

## TL;DR

A phishing operation is impersonating **"مصر الرقمية / Digital Egypt"** — the Egyptian government's official digital-services brand — using a fake **traffic-violation ("إشعار بمخالفة مرورية") payment notice**. Mobile users are lured to a convincing Arabic government-styled page (citing Egyptian Traffic Law 66/1973 and a "50% discount within 24 hours" hook), then funneled through a flow that harvests **payment-card data, personal information, and live bank OTPs**.

The site is a deployment of a **Chinese-language commercial "live-interception" Phishing-as-a-Service (PhaaS) kit** that self-brands as **"Iron Man System" (钢铁侠)** — the same category as the well-documented Darcula / YYlaiyu / Haozi families. It captures OTPs in real time so the operator can defeat OTP/3-D-Secure and authorise fraud on the spot, and it ships a **bank-branded OTP-interception page** aimed at customers of a major Egyptian bank.

If you receive an SMS about a "traffic fine" with a payment link: **it's a scam. Don't enter card details or OTPs.** Egyptian government services are not collected via card on throwaway `.fun` / `.sbs` domains.

---

## 1. How It Starts — the SMS / RCS Blast

The campaign opens with an unsolicited text message. Victims receive an Arabic SMS/RCS message claiming the "Automated Traffic Monitoring System" recorded a violation by a vehicle registered in their name, citing **Traffic Law 66/1973**, threatening account/asset freezes, and pushing the same **"50% early-payment discount within 24 hours"** urgency hook. It ends with a payment link and — in RCS form — an interactive *"reply 1 to view the violation photos and geolocation"* prompt to boost engagement.

<p align="center">
  <img src="/assets/img/2026-07-01-digital-egypt/lure-sms-rcs.jpg" alt="Phishing SMS/RCS impersonating Egyptian traffic authority, linking to digittal.sbs" width="300">
</p>

*Real lure message (victim-reported). Note the sender is a **+212 (Morocco)** mobile number — not an Egyptian government shortcode — and the link points to `digittal[.]sbs/AvVGcmoPXY`, one of the cluster domains. iOS already flags it "may be spam."*

Message body (transcribed, Arabic):

> سجّل نظام المراقبة الآلي للمرور مخالفة لأحكام قانون المرور ارتكبتها المركبة المسجلة باسمكم. وفقاً لقانون المرور رقم 66 لسنة 1973 وتعديلاته … ميزة السداد المبكر – خصم 50% خلال 24 ساعة من استلام هذا الإشعار … رابط الاستئناف والدفع: `hxxps://digittal[.]sbs/AvVGcmoPXY`

**Red flags in the message itself:** foreign (`+212`) sender, a government "service" on a random `.sbs` link, threats + a countdown discount, and an RCS "reply 1" hook. Egyptian authorities do not collect fines this way.

## 2. The Lure

Victims land on a mobile-only page that renders as an Egyptian government portal:

- **Branding:** "مصر الرقمية" (Digital Egypt) + the Ministry of Communications & Information Technology logos.
- **Pretext:** *"Your vehicle was detected by the Smart Traffic Monitoring system,"* citing **قانون المرور رقم 66 لسنة 1973** (Traffic Law No. 66 of 1973).
- **Urgency hook:** *"Early-payment feature — 50% discount within 24 hours."*
- **Authenticity tricks:** the page **clones real `digital.gov.eg` content** (footer links, and it even hotlinks the genuine government `logoDark.svg`) plus real government social-media links.

Interestingly, the underlying HTML template was **cloned from an unrelated government portal (`gov.si`, Slovenia)** — a residual `lang="sl"` and template path give it away — then rebranded into Arabic. Template reuse across countries is a hallmark of this PhaaS family.

## 3. The Harvest Flow

The victim is walked through a multi-step funnel:

```
Lure (fake traffic-fine)  →  enter vehicle plate ("inquire about the violation")
   →  owner-data form (full name, national ID, address, city, phone, email)
   →  card entry (number, name, expiry, CVV)   → BIN lookup for card routing
   →  OTP / authenticator / QR "verification"   (live interception step)
   →  "payment"                                 → "success"
```

The screens, step by step (reconstruction below — **not** the live site; rebuilt from the recovered client bundle so no attacker infrastructure was contacted and no victim ever sees these here):

<p align="center">
  <img src="/assets/img/2026-07-01-digital-egypt/harvest-flow-reconstruction.png" alt="Four-step reconstruction: fake fine notice, plate inquiry, full-PII form, card entry" width="100%">
</p>

1. **Fine notice** — the fake violation + 50% discount countdown; a single "الاستعلام عن المخالفة" (inquire about the violation) button.
2. **Plate inquiry** — victim enters their vehicle **plate number** ("رقم لوحة المركبة") to "look up the violation." This makes it feel real and confirms a live target.
3. **Owner-data form ("بيانات مالك المركبة")** — framed as *identity verification only, "no charges until the next step"* to lower guard. Harvests **full name, national ID (الرقم القومي), address, city, postal code, phone, email**.
4. **Payment** — card number, cardholder name, expiry, CVV — then the OTP/authenticator/QR "verification" step.

Data captured across the flow: card number, cardholder name, expiry, CVV, BIN-derived metadata, OTP, PIN, full name, national ID, address, city, postal code, phone, and email — plus a running **history of every card attempt** (`cardHistory[]`).

### 3.1 What happens when the card is entered (technical)

Card submission is not passive collection — the kit **enriches and validates** on the spot:

- **BIN lookup.** The first 6–8 digits are sent to a **BIN-lookup service** (observed: `bincheck.io`) to resolve the card's **scheme / type / issuing bank / country**. This lets the operator route the card, apply the campaign's **BIN allow/deny** rules, and — critically — pick the **matching bank-branded OTP page** for the next step.
- **Validation-type branching.** The kit's config carries multiple verification templates and drops the victim onto one of them: `otpValid1` (6-digit SMS OTP), `appValid1` (banking-app "approve this login" push), or `scanValid1` (QR-scan + screenshot upload). Same funnel, three ways to defeat 2FA.
- **Live relay.** Each field (`cardNumber`, `cardHolder`, `expires`, `cvv`, `cardBIN`, later `otp`/`pin`) is pushed in real time over `wss://…/ws?token=<uuid>` to the operator console (JSON events `submit_card` / `input_text`), with a Telegram notification. Socket payloads are AES-encrypted client-side using the kit's **hardcoded default passphrase** `your-secret-key`.

Crucially, this is a **live-interception** kit: submissions surface on the operator's console in **real time**, so an operator can trigger a genuine transaction on the victim's card and then present a fake OTP page to capture the one-time code seconds before it expires — defeating OTP-based 2FA and even app/QR verification. This is the same technique recent reporting attributes to Chinese PhaaS families (Darcula/Magic Cat, YYlaiyu, Lighthouse, Haozi/"Magic Mouse", ByteDance Live Panel).

## 4. Bank-Targeted OTP Interception

The campaign configuration contains a **dedicated, bank-branded OTP-interception page** for **a major Egyptian bank** (identity withheld). It:

- displays the bank's genuine logo,
- shows an Arabic prompt asking for the *"6-digit dynamic PIN sent to your registered mobile,"*
- echoes the victim's own (masked) card number for authenticity,
- mimics the real "Confirm / Resend" OTP UI.

In other words, after the card is stolen the kit selectively impersonates the victim's own bank to phish the live OTP — targeted, real-time card fraud.

## 5. Kit Identification — "Iron Man System" (钢铁侠)

Fingerprinting the client-side code and server responses:

- **Self-brand:** `Iron Man System` (config title), Chinese `Iron Man后台`; favicon `iron-man.png`.
- **Stack:** GoFrame (Go) backend behind Caddy; a Vue 3 + Element-Plus ("vue-pure-admin") operator SPA; real-time WebSocket channel; Bootstrap-based lure templates.
- **Commercial PhaaS:** it has a licensing subsystem — this is rented/sold software, not bespoke. The kit source is **not** on public GitHub (its unique strings return zero results), consistent with a closed, licensed product.
- **Operator language:** Chinese (config fields like `余额` = "balance", `填卡` = "card-fill", `红包` = "red-packet").

**Recovered kit assets** (non-sensitive; bank-branded assets deliberately withheld):

<p>
  <img src="/assets/img/2026-07-01-digital-egypt/operator-gov-glyph.jpg" alt="Generic government-building glyph uploaded by the operator" width="90">
  <img src="/assets/img/2026-07-01-digital-egypt/govsi-clone-placeholder.svg" alt="gov.si-clone template placeholder image" width="140">
  <img src="/assets/img/2026-07-01-digital-egypt/kit-cardloading.svg" alt="Kit card-processing loading spinner" width="90">
</p>

*Left→right: generic gov glyph the operator uploaded; the `gov.si`-clone template's default placeholder; the kit's card-processing spinner shown during the "verification" step.*

**Anti-analysis / evasion** built into the kit:
- **Cloaking** via an IP-intelligence API (flags datacenter/VPN/crawler/abuser IPs) plus **mobile-only** and **country** gating — researchers and sandboxes get a 404 while real mobile victims get the page.
- **Obfuscated, randomised URL base paths** and a decoy `/admin` that 404s.
- An **anti-blocklist ("防红") redirect layer** that spins up disposable clean-reputation redirect hops to launder link reputation before the landing domain.
- **BIN + country allow/deny** filtering.

## 6. Infrastructure

This is not two domains — it's a **rotating cluster** run by a single operator across two Chinese clouds. Passive DNS / urlscan pivoting off the hosting IPs surfaces a new throwaway domain roughly **every few days since 2026-06-22**, each burned as it gets blocklisted. `digitalgovs.fun` is just the newest node.

### 6.1 Hosting IPs

| IP | ASN / Provider | Country | Role |
|---|---|---|---|
| `43.160.201.83` | AS132203 Tencent Cloud (Aceville Pte) | Singapore | Primary lure + C2 |
| `43.130.228.129` | AS132203 Tencent Cloud | Tokyo | Sibling C2 (`digittal.sbs`) |
| `47.253.233.239` | AS45102 Alibaba Cloud US | USA (Charlottesville) | Secondary cluster node |

### 6.2 Domain registration history (WHOIS)

The two 2026-07-01 nodes were registered **the same morning, ~4.5 hours apart**, on different registrars — cheap disposable TLDs, privacy-protected, DNSSEC unsigned, no mail. Exact registry records:

| Field | `digitalgovs.fun` | `digittal.sbs` |
|---|---|---|
| Creation (UTC) | `2026-07-01T07:05:13Z` | `2026-07-01T11:34:01Z` |
| Registry expiry | `2027-07-01` | `2027-07-01` |
| Registrar | NameSilo, LLC (IANA 1479) | NameMart Pte. Ltd. |
| Registrar abuse | `abuse@namesilo.com` | (NameMart abuse) |
| Nameservers | `ns1/ns2/ns3.dnsowl.com` | `NS1/NS2.DOMAINNAMENS.COM` |
| DNSSEC | unsigned | unsigned |
| MX (mail) | none | none |

The **~4h29m registration gap**, dedicated IPs on the same ASN, and identical kit build across both = a single operator batch-provisioning the day's infrastructure. The wider Egypt cluster (below) shows the same pattern stretching back to **2026-06-22** — new domain every few days as older ones get blocklisted (`.ink .pics .rest .life .boats .skin .shop .fun`).

### 6.3 Egypt-targeting domain rotation (`/eg` clone)

| First seen | Domain | Landing | Hosting IP |
|---|---|---|---|
| 2026-06-22 | `digitalgov.ink` | `/` | `43.160.201.83` |
| 2026-06-22 | `digitalgov.pics` | `/eg` | `43.160.201.83` |
| 2026-06-24 | `digitalgov.rest` | `/eg` | `43.160.201.83` |
| 2026-06-24 | `digitalgov.life` | `/` | `43.160.201.83` |
| 2026-06-24 | `digitalgov.boats` | `/eg` | `47.253.233.239` |
| 2026-06-29 | `digitalgovs.skin` | `/eg` | `47.253.233.239` |
| 2026-06-30 | `digital.gov-eg.shop` | `/eg` | `47.253.233.239` |
| 2026-07-01 | `digitalgovs.fun` | `/eg` | `43.160.201.83` |
| 2026-07-01 | `digittal.sbs` | (sibling C2) | `43.130.228.129` |

`digitalgovs.fun` and `digittal.sbs` were both registered **2026-07-01, ~4.5 hours apart** (NameSilo + NameMart) — a clear same-day batch deploy. All nodes: no mail (MX) records, DNSSEC unsigned, disposable TLDs, dedicated IPs, `GoFrame HTTP Server` behind Caddy, Let's Encrypt (`CN=YE1`) certs.

### 6.4 Multi-country: same kit, different flag

The same codebase and the same `43.160.201.83` IP also served a **France**-targeting sibling — `amendes-justiecsc-gov.eu.cc` (first seen 2026-06-29), impersonating French "amendes justice" (justice/traffic fines). Naming-pattern candidates suggesting **Uzbekistan** (`digitalgovuz.top`) and Russia-language variants exist but are **unverified** — do not blocklist `digitalgov*` blindly; most other hits are legitimate orgs.

### 6.5 Operator dashboard / endpoint map

The kit hides its real routes behind per-deployment **obfuscated base paths** (a decoy `/admin` 404s to throw off scanners). For `digitalgovs.fun`:

| Route | Purpose |
|---|---|
| `https://digitalgovs.fun/eg` | Victim lure (mobile-cloaked) |
| `https://digitalgovs.fun/jCCtXobKrd/admin/` | Operator dashboard SPA (Vue, JWT/`gtoken`-gated) |
| `https://digitalgovs.fun/yYZIbqxAQy/api` | Victim-facing campaign-config API |
| `https://digitalgovs.fun/yYZIbqxAQy/api/input` | HTTP exfil endpoint (POST) |
| `wss://digitalgovs.fun/ws?token=<session>` | Live operator relay (real-time capture) |
| `https://digitalgovs.fun/admin/` | Decoy — Caddy 404, 0-byte body |
| `digittal.sbs` → `/BkKtetizxi/` | Sibling operator base path |

The operator dashboard itself is **hardened** — JWT-gated, 57 endpoints all require auth, `alg:none` rejected, IP rate-limit + captcha + TOTP/Telegram 2FA, role-isolated socket. No unauthenticated path into the stored victim database was found (that requires server/panel seizure by law enforcement).

### 6.6 Infrastructure at a glance

```
                 SMS lure (traffic fine)
                          │
                          ▼
   ┌──────────── mobile victim ────────────┐
   │  (desktop / datacenter / VPN → 404)   │
   └───────────────────┬───────────────────┘
                        ▼
        digitalgovs.fun/eg  (Caddy → GoFrame)
                        │  card + PII + OTP
          ┌─────────────┴─────────────┐
          ▼                           ▼
   POST /…/api/input           wss://…/ws?token=  ── live ──▶  Operator dashboard
   (HTTP fallback)             (real-time relay)              /jCCtXobKrd/admin/
                                                              + Telegram notify
          │
          ▼
   Anti-blocklist (防红) redirect hops ── launder link reputation
   ipregistry cloaking · BIN allow/deny · country gate
```

## 7. Notable Weaknesses in the Kit (high level)

Two design weaknesses are worth noting for defenders — described conceptually, without exploitation detail:

- **Unauthenticated configuration disclosure.** The victim-facing API returns the full campaign configuration (fake-page templates, scam text, targeting rules) to any client with a self-generated session identifier. This is what allowed safe, external documentation of the campaign's *structure* — no victim data involved.
- **Weak session-data confidentiality.** Captured session records are keyed only by a session identifier that travels in easily-logged locations (URL query string and a cookie). A kit that guards stolen data with nothing more than a log-leakable token is a data-breach risk in its own right — a reminder that criminal infrastructure is itself insecure, and that seized-server data is often trivially recoverable by investigators.

*(No proof-of-concept, request syntax, or captured records are published. Real victim data was never accessed.)*

## 8. Indicators of Compromise (IOCs)

Defanged. Everything below is safe to block/hunt on; no working exploitation request is included.

**Lure delivery**
- Sender number (this sample): `+212 7 79 79 47 62` (Morocco mobile — spoofable; treat as sample, not a fixed IOC)
- Lure link in message: `hxxps://digittal[.]sbs/AvVGcmoPXY`

**Domains — Egypt cluster (high confidence)**
- `digitalgovs.fun`
- `digittal.sbs`
- `digitalgov.ink`
- `digitalgov.pics`
- `digitalgov.rest`
- `digitalgov.life`
- `digitalgov.boats`
- `digitalgovs.skin`
- `digital.gov-eg.shop`

**Domains — sibling / other-country (same kit + IP)**
- `amendes-justiecsc-gov.eu.cc` (France — justice-fine lure)

**Candidate related — UNVERIFIED, do not blindly block**
- `digitalgovuz.top` (possible Uzbekistan), `digitalgov-safe.ru`, `digitalgov.ru`

**Hosting IPs / ASN**
- `43.160.201.83` — AS132203 Tencent Cloud, Singapore
- `43.130.228.129` — AS132203 Tencent Cloud, Tokyo
- `47.253.233.239` — AS45102 Alibaba Cloud, USA (Charlottesville)

**Nameservers (NameSilo)**
- `ns1.dnsowl.com`, `ns2.dnsowl.com`, `ns3.dnsowl.com`

**Lure / infrastructure paths**
- `hxxps://digitalgovs[.]fun/eg` (the fake traffic-fine lure)
- `hxxps://digitalgovs[.]fun/jCCtXobKrd/admin/` (operator dashboard base — per-deployment)
- `hxxps://digitalgovs[.]fun/yYZIbqxAQy/api` + `/api/input` (config + exfil — per-deployment)
- `wss://digitalgovs[.]fun/ws?token=<session>` (live operator relay)
- `digittal[.]sbs/BkKtetizxi/` (sibling operator base path)
- Template path artifact: `/we003_si_etc_gov/` (cloned from `gov.si`)

**Kit / TLS / server fingerprints**
- Kit self-brand `Iron Man System` (钢铁侠) **v6.0.0**; `iron-man.png` favicon; `serverConfig.json` title "Iron Man System"
- `server: GoFrame HTTP Server` + `via: 1.1 Caddy`
- TLS issuer `Let's Encrypt CN=YE1`
- `/eg` page body ETag `33746b336ed5dd450a5542c5d7ce4e55`
- SPA bundle `/assets/index-39cef7f8.js`, stylesheet `/assets/index-f18ae053.css`
- Hardcoded socket AES key string `your-secret-key` (kit default)
- Cloned Egyptian-gov branding + hotlinked `digital.gov.eg/logoDark.svg`

**Third-party services abused**
- BIN-lookup service (card routing), `ipregistry` IP-intelligence API (cloaking), Vercel/static host (anti-blocklist `防红` redirects), Telegram (operator exfil/notifications), vendor CDN `img.haoeryu.top` (Cloudflare; template assets `e1`–`e8`).

### 8.1 Consolidated blocklist (copy-paste)

```
# domains — Egypt cluster + FR sibling
digitalgovs.fun
digittal.sbs
digitalgov.ink
digitalgov.pics
digitalgov.rest
digitalgov.life
digitalgov.boats
digitalgovs.skin
digital.gov-eg.shop
amendes-justiecsc-gov.eu.cc
# hosting IPs
43.160.201.83
43.130.228.129
47.253.233.239
```

## 9. Guidance

**For the public**
- The Egyptian government does **not** collect traffic fines via card payment on `.fun` / `.sbs` links sent by SMS.
- **Never enter card details or an OTP** on a page you reached from an unsolicited SMS/link.
- This class of kit defeats **OTP and app-based 2FA**, not just SMS codes. If a "government" or "bank" page asks for an OTP right after you entered your card, stop.

**For defenders / banks / CERTs**
- Block and monitor the IOCs above; hunt the kit fingerprints (`iron-man.png`, GoFrame+Caddy combo, the `/admin%20`-style base-path quirk) to surface sibling domains — same-day registrations suggest more.
- Watch for real-time fraud patterns (card auth immediately followed by an OTP prompt) originating around these domains.
- Treat this as part of the broader Chinese live-interception PhaaS wave (Darcula / YYlaiyu / Haozi lineage).

## 10. IOC Table (SOC quick-reference)

Defanged, single-table view for triage / detection engineering. `[.]` = de-fanged dot; treat obfuscated base paths as **per-deployment** (fingerprint the pattern, not the exact string).

| # | Type | Indicator | Context | Confidence |
|---|---|---|---|---|
| 1 | domain | `digitalgovs[.]fun` | Active lure + C2 (this campaign) | High |
| 2 | domain | `digittal[.]sbs` | Sibling C2 / lure link in SMS | High |
| 3 | domain | `digitalgov[.]ink` | Cluster node (2026-06-22) | High |
| 4 | domain | `digitalgov[.]pics` | Cluster node (2026-06-22) | High |
| 5 | domain | `digitalgov[.]rest` | Cluster node (2026-06-24) | High |
| 6 | domain | `digitalgov[.]life` | Cluster node (2026-06-24) | High |
| 7 | domain | `digitalgov[.]boats` | Cluster node (2026-06-24) | High |
| 8 | domain | `digitalgovs[.]skin` | Cluster node (2026-06-29) | High |
| 9 | domain | `digital[.]gov-eg[.]shop` | Cluster node (2026-06-30) | High |
| 10 | domain | `amendes-justiecsc-gov[.]eu[.]cc` | France sibling (same kit + IP) | High |
| 11 | domain | `digitalgovuz[.]top` | Possible Uzbekistan variant | Low (unverified) |
| 12 | domain | `digitalgov-safe[.]ru`, `digitalgov[.]ru` | Possible RU variants | Low (unverified) |
| 13 | ipv4 | `43[.]160[.]201[.]83` | Tencent Cloud SG (AS132203) — primary C2 | High |
| 14 | ipv4 | `43[.]130[.]228[.]129` | Tencent Cloud Tokyo (AS132203) — sibling C2 | High |
| 15 | ipv4 | `47[.]253[.]233[.]239` | Alibaba Cloud US (AS45102) — cluster node | High |
| 16 | asn | `AS132203` / `AS45102` | Tencent / Alibaba hosting | Medium |
| 17 | url | `hxxps://digitalgovs[.]fun/eg` | Fake traffic-fine lure landing | High |
| 18 | url | `hxxps://digittal[.]sbs/AvVGcmoPXY` | Lure link delivered in SMS | High |
| 19 | url-path | `/jCCtXobKrd/admin/` | Operator dashboard base (per-deploy) | Medium |
| 20 | url-path | `/yYZIbqxAQy/api` + `/api/input` | Config + exfil endpoints (per-deploy) | Medium |
| 21 | url-path | `/BkKtetizxi/` | Sibling operator base (`digittal.sbs`) | Medium |
| 22 | url-path | `/we003_si_etc_gov/` | `gov.si`-clone template artifact | High |
| 23 | ws | `wss://<host>/ws?token=<uuid>` | Live operator relay / real-time exfil | High |
| 24 | msisdn | `+212 7 79 79 47 62` | SMS sender (spoofable — sample only) | Low |
| 25 | nameserver | `ns1/ns2/ns3.dnsowl.com` (NameSilo) | `digitalgovs.fun` NS | Medium |
| 26 | nameserver | `NS1/NS2.DOMAINNAMENS.COM` (NameMart) | `digittal.sbs` NS | Medium |
| 27 | http-header | `server: GoFrame HTTP Server` + `via: 1.1 Caddy` | Kit stack fingerprint | Medium |
| 28 | tls | `Let's Encrypt CN=YE1` | Cert issuer on cluster nodes | Low |
| 29 | http-etag | `33746b336ed5dd450a5542c5d7ce4e55` | `/eg` page body ETag | Medium |
| 30 | file | `/assets/index-39cef7f8.js`, `index-f18ae053.css` | Lure SPA bundle / stylesheet | Medium |
| 31 | favicon | `iron-man.png` | Kit self-brand "Iron Man System" v6.0.0 | Medium |
| 32 | string | `your-secret-key` | Hardcoded socket AES passphrase (kit default) | Medium |
| 33 | service | `bincheck.io` | BIN lookup on stolen cards | Medium |
| 34 | service | `api.ipregistry.co` | Cloaking / IP-intel gating | Medium |
| 35 | service | `img.haoeryu.top` (Cloudflare) | Kit-vendor CDN (templates `e1`–`e8`) | Medium |
| 36 | ttp | Card auth immediately followed by OTP prompt | Live-interception fraud pattern | — |

**Detection tips:** alert on card-BIN queries to `bincheck.io` from customer sessions you don't originate; hunt web logs for the `GoFrame`+`Caddy` header combo on `.fun`/`.sbs`/`.shop` hosts; watch CT logs + both Tencent IPs for the next rotated `digitalgov*` domain.

## 11. Responsible Disclosure

The impersonated national brand / CERT, the affected bank's fraud team, the hosting provider, and the registrars were notified via abuse channels prior to publication. This post intentionally omits exploitation steps and any captured data. Its purpose is defensive: to warn potential victims and help other responders identify and take down related infrastructure.

---

*Indicators are shared in good faith for defensive use. If you operate infrastructure referenced here and believe it is mislabeled, contact the author.*
