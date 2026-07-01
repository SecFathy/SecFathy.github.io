---
layout: post
title: "Hunting IDOR and Broken Access Control: A Repeatable Methodology"
date: 2026-05-12 10:00:00 +0200
categories: offensive
tags: [access-control, idor, methodology, web]
---

Broken access control has topped the OWASP list for years, and for good
reason: it is cheap to introduce, hard to catch with scanners, and often
maps directly to real business impact. This post is the methodology I run on
every web target — sanitized, tool-agnostic, and framed around *authorization*
rather than any single bug class.

## The mental model

Every request answers three questions on the server:

1. **Who are you?** (authentication)
2. **What are you asking for?** (the object / action)
3. **Are you allowed?** (authorization)

IDOR and broken access control live entirely in question three. The bug is
never the identifier in the URL — it is the *missing check* behind it. So the
whole game is: find a request where the server trusts the client to scope its
own access, then prove it doesn't re-check on the server.

## Step 1 — Map the object model, not the endpoints

Before touching a single ID, list the *nouns* the app exposes: users, orders,
invoices, tickets, files, teams, API keys. For each noun, note:

- How it is referenced (sequential int, UUID, slug, hashid, composite key)
- Which roles *should* touch it (owner, team member, admin, support)
- Where it appears (path, query, body, header, cookie, JWT claim)

This inventory is what turns random ID-flipping into a checklist you can
actually complete.

## Step 2 — Two accounts, always

The single highest-signal setup is **two accounts you fully control** in the
same tenant tier:

- `victim` — create objects here, record their identifiers.
- `attacker` — replay `victim`'s requests using `attacker`'s session.

If `attacker` can read, mutate, or delete a `victim` object, you have a
horizontal access-control failure with a clean, ethical PoC — no third party
is ever touched.

## Step 3 — Swap one variable at a time

Take a known-good request from `victim` and, holding everything else constant,
substitute *only* the authorization context:

| Change | Tests for |
|---|---|
| Session cookie / token → `attacker` | Horizontal IDOR |
| Role claim / tier → lower privilege | Vertical escalation |
| Tenant / org id | Cross-tenant leakage |
| HTTP method (`GET`→`PUT`/`DELETE`) | Method-level gaps |
| `id` only, same session | Self-scoped enforcement |

Changing more than one variable at a time is how you get false results and
waste a triager's time.

## Step 4 — Beat the common "defenses"

Access-control fixes are frequently shallow. Things worth a second pass:

- **UUIDs are not authorization.** They raise the cost of guessing an ID, not
  of *using* one you already obtained (referrals, logs, exports, a shared link).
- **Obfuscated ids.** hashids/base64/short-codes are encodings; decode, iterate,
  re-encode.
- **Mass assignment.** The `GET` may hide a field the `PUT`/`PATCH` still
  accepts — try adding `role`, `owner_id`, `is_admin`, `team_id` to bodies.
- **Secondary contexts.** The bug often lives in the export, the webhook, the
  mobile API, or the GraphQL resolver — not the primary REST route the UI uses.
- **State-changing GETs.** `GET /account/42/delete` still exists in the wild.

## Step 5 — Prove impact, then stop

A finding without demonstrated impact is noise. For every candidate, answer:

> *What can a real attacker read, change, or destroy that they should not?*

Reading another user's PII, editing another org's billing, promoting your own
role — those are reports. "The ID is sequential" is not. Capture the minimal
request/response pair that proves the crossing, redact the victim data, and
write the impact in one sentence a non-engineer can act on.

## A note on discipline

Access-control testing is where scope discipline matters most, because a
working IDOR often *keeps working* on IDs you don't own. Stay inside accounts
you control, stop the moment impact is proven, and never enumerate real users'
data to "measure blast radius" — a single record is enough to demonstrate the
class.

That constraint is not a limitation on the finding. It's what makes the finding
reportable.
