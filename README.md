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

## ⚠ Source of truth is the app repo, not this one

Canonical copies live at `web/.well-known/` in **`mjsa1967-ux/kinerix_beta`**,
where `tools/deeplink_domain_audit.py` (DL-06, DL-10) gates them against the
declared host and app identifier. **Edit there, then mirror here** — the same
standing rule the `kinerix-legal` mirror carries.

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

## Status

Both files currently carry **placeholders** (`TEAMID_PENDING`,
`SHA256_FINGERPRINT_PENDING`). Verification cannot succeed until they are
replaced with the Apple Developer Team ID and the release signing SHA-256.
Publishing them in this state is harmless — they assert nothing — and it lets
the DNS, Pages, HTTPS and `.nojekyll` chain be proven independently, so filling
the secrets later is a one-line edit rather than a debugging session.
