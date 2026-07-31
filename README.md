# kinerix-app-links

Serves **`https://app.kinerixathletics.com/.well-known/`** via GitHub Pages, so
iOS and Android open Kinerix deep links in the app instead of the browser.

Two files, both **public by design** — Apple's CDN and Google's digitalassetlinks
API fetch them anonymously:

| File | Purpose |
|---|---|
| `.well-known/apple-app-site-association` | iOS Universal Links (`/invite/*`, `/vpc/*`) |
| `.well-known/assetlinks.json` | Android App Links (`autoVerify`) |

Neither contains a secret. The Android **SHA-256 certificate fingerprint is a
public fingerprint of a public certificate** — every Android app using App Links
publishes it. Nothing here grants access to anything; these files only *assert*
that a named app may claim links on this domain, and the OS verifies that
assertion against the app's actual signature.

## Also served here: `/auth/reset-password`

`auth/reset-password/index.html` is the landing page for the Supabase
password-reset email (W-WEB-RESET-PASSWORD-PAGE-01, gov 4609). Unlike the
association files it is a page a **human** reads, which is precisely why it
cannot be an edge function: the supabase.co gateway rewrites `text/html` →
`text/plain` as an anti-phishing measure, so an edge-function route renders as
raw HTML source in a browser. Static Pages is the fix — the same reason the
legal documents live in `kinerix-legal`.

It contains no secret. The embedded Supabase key is the **publishable** key
(public by design, independently rotatable); the control is the single-use
recovery token in the emailed link. The page makes **no third-party requests** —
no CDN script, no web font, no analytics — because a page whose only job is
accepting a password should not load code we do not control.

⚠ **If the publishable key is rotated, this page breaks silently.** Re-mirror it
as part of any rotation.

⚠ It stays inert until two dashboard settings land: the redirect URL in
**Auth → URL Configuration**, and an **Auth → Email Templates → Reset Password**
link of the form `…/auth/reset-password/?token_hash={{ .TokenHash }}&type=recovery`.
Both are dashboard-only, so no gate in either repo can see them drift.

**Use the trailing slash.** Verified 2026-07-31: Pages 301s the extensionless
path to the trailing-slash one and *does* preserve the query string, so both
work — but a redirect is one more hop where a mail scanner or link-rewriting
proxy could drop the token, and the token is single-use, so anything that
consumes or mangles it burns the reset.

## Also served here: `/invite/` + `404.html`

`invite/index.html` is the public roster-invite landing page
(W-WEB-INVITE-LANDING-PAGE-01, gov 4612). It closed a live dead end: both server
emitters send `app.kinerixathletics.com/invite/<token>`, and this repo served
only `.well-known/`, so that path 404'd. App Links verification cannot succeed
while the two placeholders below are unfilled, so **every** emailed invite opened
without the app installed hit it.

`404.html` is **load-bearing and permanent** — do not delete it as boilerplate.
The invite token arrives as a *path segment*, GitHub Pages has no rewrite rules,
and a root `404.html` is the only hook available: it forwards `/invite/<token>`
to `/invite/?token=<token>`, where the real page answers with a 200. Every invite
already emailed carries the path form, so this is not a transitional shim. It
only ever rewrites to a fixed same-origin path, so it cannot become an open
redirect; unmatched non-invite paths still render an ordinary not-found.

The page holds no secret — same publishable key as the reset page — and calls two
anon RPCs: `api_preview_invite_v1` (read-only; it never burns the claim handle)
and `api_get_invite_retention_notice_v1` (the counsel-binding notice, which is
bound from the database and never hardcoded).

## ⚠ Source of truth is the app repo, not this one

Canonical copies live at `web/.well-known/` and `web/auth/` in
**`mjsa1967-ux/kinerix_beta`**, where `tools/deeplink_domain_audit.py`
(DL-06, DL-09, DL-10) gates them against the declared host and app identifier.
**Edit there, then mirror here** — the same standing rule the `kinerix-legal`
mirror carries.

## ⚠ `.nojekyll` is load-bearing — do not delete it

GitHub Pages runs Jekyll, which **excludes dot-directories**. Without
`.nojekyll` at the repo root, `/.well-known/` returns 404 and both link families
fail verification with **nothing reported anywhere** — no error, no log, no
warning. The links just quietly open the browser instead of the app.

## Setup recorded here so it can be reproduced

* DNS — `app` CNAME → `mjsa1967-ux.github.io` (nameservers: BusinessIdentity LLC).
  Note `kinerixathletics.com` has a **wildcard A record**, so this subdomain
  resolved *before* the CNAME existed and returned 404 from the website host.
  **"Does it resolve?" is therefore not a valid success test** — fetch the files.
* Pages — source `main` / root, custom domain `app.kinerixathletics.com`,
  Enforce HTTPS.
* `CNAME` file must match the custom domain exactly.

## Verifying (fetching the files is the only real test)

```bash
curl -sSI https://app.kinerixathletics.com/.well-known/assetlinks.json     # want 200, application/json, no redirect
curl -sSI https://app.kinerixathletics.com/.well-known/apple-app-site-association
curl -sS  https://app-site-association.cdn-apple.com/a/v1/app.kinerixathletics.com
adb shell pm verify-app-links --re-verify com.kinerixathletics.app
adb shell pm get-app-links com.kinerixathletics.app                        # want "verified"
```

Apple's CDN fetching the file is the real iOS test — your own server answering is not.

## Status — infrastructure PROVEN, secrets still pending

Verified live 2026-07-24, immediately after setup:

| Check | Result |
|---|---|
| `assetlinks.json` over HTTPS | **200**, `application/json` |
| `apple-app-site-association` over HTTPS | **200**, `application/octet-stream` |
| **Apple CDN** fetch + parse | **200, parsed** — see note below |
| **Google digitalassetlinks** fetch | **fetched OK**; only error is the placeholder fingerprint |
| `.nojekyll` / dot-directory publishing | working |
| HTTPS certificate | approved, Enforce HTTPS on |

**The `application/octet-stream` question is ANSWERED.** GitHub Pages cannot set
`application/json` on an extensionless file, and Apple's docs ask for it — so
this was flagged as a real risk before setup. Apple's CDN fetched and parsed the
file anyway, returning it as valid JSON. **No Cloudflare proxy or paid custom
domain is needed.** Recorded here because the next person will re-ask this.

Both files still carry **placeholders** (`TEAMID_PENDING`,
`SHA256_FINGERPRINT_PENDING`), so verification cannot yet succeed — Google says
so explicitly: `malformed cert fingerprint: SHA256_FINGERPRINT_PENDING`. That
error is the proof the pipeline works: Google reached the file, parsed the JSON,
and objected only to the value.

Publishing in this state was deliberate. It proved DNS → Pages → `.nojekyll` →
HTTPS → content-type → Apple/Google fetch end to end, independently of the
signing secrets, so filling them in is a one-line edit rather than a debugging
session with five unknowns.

**Remaining:** replace the two placeholders, re-run the two verification URLs
above, then `adb shell pm get-app-links com.kinerixathletics.app`.
