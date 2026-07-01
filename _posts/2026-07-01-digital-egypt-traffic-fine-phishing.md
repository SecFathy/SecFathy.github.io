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

## 1. The Lure

Victims land on a mobile-only page that renders as an Egyptian government portal:

- **Branding:** "مصر الرقمية" (Digital Egypt) + the Ministry of Communications & Information Technology logos.
- **Pretext:** *"Your vehicle was detected by the Smart Traffic Monitoring system,"* citing **قانون المرور رقم 66 لسنة 1973** (Traffic Law No. 66 of 1973).
- **Urgency hook:** *"Early-payment feature — 50% discount within 24 hours."*
- **Authenticity tricks:** the page **clones real `digital.gov.eg` content** (footer links, and it even hotlinks the genuine government `logoDark.svg`) plus real government social-media links.

Interestingly, the underlying HTML template was **cloned from an unrelated government portal (`gov.si`, Slovenia)** — a residual `lang="sl"` and template path give it away — then rebranded into Arabic. Template reuse across countries is a hallmark of this PhaaS family.

## 2. The Harvest Flow

The victim is walked through a multi-step funnel:

```
Lure (fake traffic-fine)  →  enter vehicle plate ("inquire about the violation")
   →  card entry (number, name, expiry, CVV)   → BIN lookup for card routing
   →  OTP / authenticator / QR "verification"   (live interception step)
   →  "payment" + address details               (name, address, phone, email)
   →  "success"
```

Data captured across the flow includes: card number, cardholder name, expiry, CVV, BIN-derived metadata, OTP, PIN, full name, address, city, phone, and email — plus a running history of every card attempt.

Crucially, this is a **live-interception** kit: submissions surface on the operator's console in **real time**, so an operator can trigger a genuine transaction on the victim's card and then present a fake OTP page to capture the one-time code seconds before it expires — defeating OTP-based 2FA and even app/QR verification. This is the same technique recent reporting attributes to Chinese PhaaS families (Darcula/Magic Cat, YYlaiyu, Lighthouse, Haozi/"Magic Mouse", ByteDance Live Panel).

## 3. Bank-Targeted OTP Interception

The campaign configuration contains a **dedicated, bank-branded OTP-interception page** for **a major Egyptian bank** (identity withheld). It:

- displays the bank's genuine logo,
- shows an Arabic prompt asking for the *"6-digit dynamic PIN sent to your registered mobile,"*
- echoes the victim's own (masked) card number for authenticity,
- mimics the real "Confirm / Resend" OTP UI.

In other words, after the card is stolen the kit selectively impersonates the victim's own bank to phish the live OTP — targeted, real-time card fraud.

## 4. Kit Identification — "Iron Man System" (钢铁侠)

Fingerprinting the client-side code and server responses:

- **Self-brand:** `Iron Man System` (config title), Chinese `Iron Man后台`; favicon `iron-man.png`.
- **Stack:** GoFrame (Go) backend behind Caddy; a Vue 3 + Element-Plus ("vue-pure-admin") operator SPA; real-time WebSocket channel; Bootstrap-based lure templates.
- **Commercial PhaaS:** it has a licensing subsystem — this is rented/sold software, not bespoke. The kit source is **not** on public GitHub (its unique strings return zero results), consistent with a closed, licensed product.
- **Operator language:** Chinese (config fields like `余额` = "balance", `填卡` = "card-fill", `红包` = "red-packet").

**Anti-analysis / evasion** built into the kit:
- **Cloaking** via an IP-intelligence API (flags datacenter/VPN/crawler/abuser IPs) plus **mobile-only** and **country** gating — researchers and sandboxes get a 404 while real mobile victims get the page.
- **Obfuscated, randomised URL base paths** and a decoy `/admin` that 404s.
- An **anti-blocklist ("防红") redirect layer** that spins up disposable clean-reputation redirect hops to launder link reputation before the landing domain.
- **BIN + country allow/deny** filtering.

## 5. Infrastructure

Two throwaway command-and-control domains, **registered the same day, ~4.5 hours apart**, on the same cloud ASN — a clear single-operator batch deployment:

| Domain | Role | Registered | Registrar | Hosting |
|---|---|---|---|---|
| `digitalgovs.fun` | Lure + C2 (this campaign) | 2026-07-01 | NameSilo | Tencent Cloud (Singapore) |
| `digittal.sbs` | Sibling C2 (same kit) | 2026-07-01 | NameMart | Tencent Cloud (Tokyo) |

Both: no mail (MX) records, DNSSEC unsigned, disposable TLDs, dedicated IPs, `GoFrame HTTP Server` behind Caddy.

## 6. Notable Weaknesses in the Kit (high level)

Two design weaknesses are worth noting for defenders — described conceptually, without exploitation detail:

- **Unauthenticated configuration disclosure.** The victim-facing API returns the full campaign configuration (fake-page templates, scam text, targeting rules) to any client with a self-generated session identifier. This is what allowed safe, external documentation of the campaign's *structure* — no victim data involved.
- **Weak session-data confidentiality.** Captured session records are keyed only by a session identifier that travels in easily-logged locations (URL query string and a cookie). A kit that guards stolen data with nothing more than a log-leakable token is a data-breach risk in its own right — a reminder that criminal infrastructure is itself insecure, and that seized-server data is often trivially recoverable by investigators.

*(No proof-of-concept, request syntax, or captured records are published. Real victim data was never accessed.)*

## 7. Indicators of Compromise (IOCs)

**Domains**
- `digitalgovs.fun`
- `digittal.sbs`

**Lure / infrastructure paths**
- `hxxps://digitalgovs[.]fun/eg` (the fake traffic-fine lure)
- Randomised obfuscated base paths for the operator panel and victim API (per-deployment)
- Template path artifact: `/we003_si_etc_gov/` (cloned from `gov.si`)

**Hosting**
- `43.160.201.83` (Tencent Cloud, Singapore)
- `43.130.228.129` (Tencent Cloud, Tokyo)
- ASN AS132203 (Tencent Cloud)

**Third-party services abused**
- A BIN-lookup service (card routing), an IP-intelligence API (cloaking), a static-hosting provider (anti-blocklist redirects), and Telegram (operator exfil/notifications).

**Fingerprints**
- Kit self-brand `Iron Man System` / 钢铁侠; `iron-man.png` favicon; `serverConfig.json` title "Iron Man System"
- `server: GoFrame HTTP Server` + `via: 1.1 Caddy`
- Cloned Egyptian-gov branding + hotlinked `digital.gov.eg/logoDark.svg`

## 8. Guidance

**For the public**
- The Egyptian government does **not** collect traffic fines via card payment on `.fun` / `.sbs` links sent by SMS.
- **Never enter card details or an OTP** on a page you reached from an unsolicited SMS/link.
- This class of kit defeats **OTP and app-based 2FA**, not just SMS codes. If a "government" or "bank" page asks for an OTP right after you entered your card, stop.

**For defenders / banks / CERTs**
- Block and monitor the IOCs above; hunt the kit fingerprints (`iron-man.png`, GoFrame+Caddy combo, the `/admin%20`-style base-path quirk) to surface sibling domains — same-day registrations suggest more.
- Watch for real-time fraud patterns (card auth immediately followed by an OTP prompt) originating around these domains.
- Treat this as part of the broader Chinese live-interception PhaaS wave (Darcula / YYlaiyu / Haozi lineage).

## 9. Responsible Disclosure

The impersonated national brand / CERT, the affected bank's fraud team, the hosting provider, and the registrars were notified via abuse channels prior to publication. This post intentionally omits exploitation steps and any captured data. Its purpose is defensive: to warn potential victims and help other responders identify and take down related infrastructure.

---

*Indicators are shared in good faith for defensive use. If you operate infrastructure referenced here and believe it is mislabeled, contact the author.*
