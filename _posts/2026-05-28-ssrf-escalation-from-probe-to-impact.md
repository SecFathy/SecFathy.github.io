---
layout: post
title: "SSRF Escalation: From a Blind Probe to Concrete Impact"
date: 2026-05-28 11:30:00 +0200
categories: offensive
tags: [ssrf, cloud, methodology, web]
---

Server-Side Request Forgery is easy to *find* and easy to *undersell*. A lot
of reports stop at "the server fetched my Burp Collaborator URL" — which proves
the primitive but not the impact. This post walks the escalation ladder I climb
once SSRF is confirmed, without naming any real target.

## What SSRF actually gives you

SSRF turns the vulnerable server into a proxy that speaks from *inside* the
trust boundary. The value is not "it made a request" — it's *who the request
appears to come from*. Internal services, metadata endpoints, and allow-listed
peers trust that origin in ways they never trust the internet.

So escalation is a question of: **what does the network trust about this
server that it wouldn't trust about me?**

## Step 0 — Classify the SSRF

Before escalating, know what you're holding:

- **Full-response SSRF** — you see the fetched body. Highest value.
- **Blind SSRF** — no body, but observable side effects (timing, DNS, out-of-band).
- **Semi-blind** — status codes, response lengths, or error strings leak.

Blind doesn't mean low impact; it means you escalate through *oracles* (timing
diffs, DNS callbacks) instead of reading bodies directly.

## Step 1 — Reach the internal surface

Enumerate what the server can reach that you can't:

- `127.0.0.1` and `::1` on common service ports (admin panels, debug endpoints,
  unauthenticated internal APIs bound to localhost).
- RFC1918 ranges and the container/pod network.
- Service-discovery names (`*.internal`, `*.svc.cluster.local`, DB hostnames).

A localhost-bound admin console with no auth (because "it's only reachable from
the box") is a classic full-response SSRF jackpot.

## Step 2 — Cloud metadata, carefully

Cloud metadata services are the highest-yield target, but modern deployments
harden them. Know the difference:

- **IMDSv1** (unauthenticated `GET`) — if reachable, credentials fall out
  directly. Increasingly rare.
- **IMDSv2** (session-token, `PUT` first, hop-limit) — usually blocks naive
  SSRF because you can't issue the `PUT` or the hop limit is 1. Don't claim
  impact you can't demonstrate; test whether the primitive can actually mint
  the token.

If you *can* reach role credentials, the report writes itself — but stop at
proving retrieval. Do not use the credentials to pivot into production.

## Step 3 — Bypass the filter, don't fight it

Allow/deny lists around SSRF fail in predictable ways:

- **Parser confusion** — `http://attacker.com#@internal/`, embedded
  credentials, `\` vs `/`, URL vs host-header mismatch.
- **DNS rebinding** — a name that resolves to a public IP on validation and a
  private IP on fetch (TOCTOU on resolution).
- **Redirects** — allow-listed host issues a `302` to an internal target; many
  fetchers follow it with the original trust.
- **Alternate schemes** — `gopher://`, `file://`, `dict://` when the client is
  a raw socket library rather than an HTTP client. `gopher://` in particular can
  forge arbitrary TCP payloads (e.g., line-based protocols) and turn SSRF into
  interaction with non-HTTP services.
- **IP encodings** — decimal, octal, IPv6-mapped, and zero-compression forms of
  `127.0.0.1` slip past naive string checks.

## Step 4 — Turn reach into impact

The ladder, roughly in ascending severity:

1. **Internal recon** — port/service mapping from inside. Useful, rarely a
   headline on its own.
2. **Sensitive read** — internal dashboards, config endpoints, unauth APIs.
3. **Credential access** — metadata roles, internal token services.
4. **SSRF → RCE** — `gopher` into an unauthenticated Redis/queue that permits
   command or job injection, or an internal CI/deploy hook.

Each rung needs its own PoC. Don't report the top rung and demonstrate the
bottom one.

## Writing it up

A strong SSRF report shows three things: the **primitive** (the fetch you
control), the **crossing** (the internal thing you reached that the internet
can't), and the **impact** (what that specifically lets an attacker do). Redact
any real internal hostnames or credentials to their shape — `AKIA…`, not the
value — and describe blast radius without exercising it.

The difference between a P4 and a P1 SSRF is almost never the bug. It's how far
you responsibly walked the escalation ladder — and how clearly you proved each
step.
