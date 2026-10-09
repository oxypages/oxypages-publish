---
name: oxypages-publish
description: Publish a static website (HTML, CSS, JS, images, or a ZIP) to OxyPages and hand the user a live HTTPS link. Works with no account for a 30-minute unclaimed link, or with an OxyPages API key for a permanent site the user owns.
version: 1.0.0
homepage: https://oxypages.com
metadata:
  openclaw:
    emoji: "\U0001F680"
    requires:
      bins:
        - curl
    primaryEnv: OXYPAGES_API_KEY
    envVars:
      - name: OXYPAGES_API_KEY
        required: false
        description: An OxyPages API key (starts with oxy_). Optional - without it the skill publishes an unclaimed 30-minute link. Create one at app.oxypages.com under Profile, API Keys. Free accounts include API keys.
---

# Publish a website to OxyPages

OxyPages is static website hosting. You hand it HTML (and any CSS, JS, images or fonts it uses) and it is live on an HTTPS link in seconds, on a `*.myoxypages.com` subdomain. No git, no build step, no CLI.

There are two ways to publish, and you must choose based on whether the user has given you an API key.

| | Unclaimed link (no key) | Owned site (API key) |
|---|---|---|
| Needs | nothing | `OXYPAGES_API_KEY` |
| Lives | 30 minutes, then gone unless claimed | permanently, on the user's account |
| Size | 3 MB, 10 files (the Free plan, so it is always claimable) | the account's plan |
| Rate | a small hourly cap per address | a far higher hourly cap |
| Use it for | a quick preview, "show me what this looks like live" | anything the user wants to keep |

## Consent rule - read this first

Publishing puts content on the public internet under a URL anyone can open. Before you publish:

1. **The user must have asked for the content to be published or hosted.** Writing a page is not the same as asking for it to go live. If they said "make me a landing page", show them the file and ask whether to publish it. If they said "put this online", "host this", "give me a link" or "publish it", that is consent.
2. **Never publish content that impersonates a brand or asks for credentials.** OxyPages runs a moderation pass on every publish and refuses or removes phishing, brand impersonation, gambling funnels and illegal content. Do not try to route around a refusal.
3. **Tell the user what you published and where.** Always relay the URL, and for an unclaimed link always relay the claim link and the 30-minute expiry.

## Path 1 - unclaimed link, no account

Use this when there is no `OXYPAGES_API_KEY`. One `POST` with the file as multipart form data. The file may be a single `.html` page or a `.zip` whose top level contains `index.html`.

```bash
# A single page
curl -sS -X POST https://api.oxypages.com/drops \
  -H "X-Oxy-Agent: openclaw" \
  -F "file=@index.html"

# A folder, zipped first (index.html must be at the top level of the zip)
zip -r site.zip index.html css/ js/ images/
curl -sS -X POST https://api.oxypages.com/drops \
  -H "X-Oxy-Agent: openclaw" \
  -F "file=@site.zip"
```

Send the `X-Oxy-Agent` header naming your agent. It buys nothing except attribution, and it is how OxyPages knows the publish came from an agent rather than a person.

A successful response:

```json
{
  "ok": true,
  "url": "https://unclaimed-k3j9x2m1qa.myoxypages.com",
  "token": "…",
  "claim_url": "https://app.oxypages.com/claim/…",
  "expires_at": "2026-09-12T13:05:00.000Z",
  "note": "Published. This unclaimed website expires in 30 minutes unless it is claimed via claim_url. …",
  "signup_url": "https://app.oxypages.com/signup",
  "login_url": "https://app.oxypages.com/login"
}
```

Relay to the user, in this order:

- the live `url`
- that it expires in 30 minutes unless claimed
- the `claim_url`, which moves the site into a free account permanently (signing up takes a minute and the link stays valid for the 30 minutes)
- that a free account also gives them an API key so you can publish permanent sites for them next time

A `warnings` array, when present, lists files that were left out (for example `README.md` or `.env`). The site is live; say which files were skipped.

Refusals come back as `{"ok": false, "error": "…"}` with an HTTP 4xx. The common ones:

| Status | Meaning | What to do |
|---|---|---|
| 400 | no `index.html` at the top level, or a bad body | fix the file layout and retry once |
| 413 | over 3 MB or more than 10 files | trim the site, or ask the user for an API key (paid plans allow more) |
| 415 | a file type that is not hosted (binaries, executables) | remove it |
| 422 | the moderation pass refused the content | tell the user why; do not retry with small edits |
| 429 | the hourly cap is reached | relay `error` verbatim - it names the way past it, which is a free account and an API key. It does not state a number, so do not invent one |

## Path 2 - a permanent site on the user's account

Use this when `OXYPAGES_API_KEY` is set. Every request carries it as a bearer token. Keys start with `oxy_`; if the key is rejected with 401, tell the user to create a new one at app.oxypages.com, Profile, API Keys.

**1. Find or create the site.**

```bash
# Existing sites: id, name, subdomain, url, status
curl -sS https://api.oxypages.com/v1/sites \
  -H "Authorization: Bearer $OXYPAGES_API_KEY"

# A new site (the subdomain is generated; name is optional)
curl -sS -X POST https://api.oxypages.com/v1/sites \
  -H "Authorization: Bearer $OXYPAGES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme landing page"}'
```

Ask the user which site to deploy into when they already have one that fits. Do not create a new site for every publish; a redeploy replaces the site's files, which is what "update my site" means.

**2. Deploy the files.** Either a ZIP or a JSON file list. A deploy replaces the whole site, so send every file each time.

```bash
# ZIP (index.html at the top level)
curl -sS -X POST "https://api.oxypages.com/v1/sites/$SITE_ID/deployments" \
  -H "Authorization: Bearer $OXYPAGES_API_KEY" \
  -F "zip=@site.zip"

# JSON: text files as utf8, binaries as base64
curl -sS -X POST "https://api.oxypages.com/v1/sites/$SITE_ID/deployments" \
  -H "Authorization: Bearer $OXYPAGES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"files": [
        {"path": "index.html", "content": "<!doctype html>…"},
        {"path": "css/style.css", "content": "body{…}"},
        {"path": "logo.png", "content": "iVBORw0KGgo…", "encoding": "base64"}
      ]}'
```

The response carries `ok`, `url`, `deployment_number`, `pages`, `bytes`, `files` and, when some files were skipped, `warnings`. Relay the `url`. Version history is at `GET /v1/sites/$SITE_ID/deployments`.

## Limits, without guessing

Do not quote sizes from memory - they are tuned from the admin panel and this
file cannot keep up. Read them:

```bash
curl -sS https://api.oxypages.com/limits
```

It returns the per-page and per-asset ceilings, the ZIP upload ceiling, the
unclaimed-drop ceilings and each plan's storage. It does **not** carry the
hourly publish caps, and no number in this skill should either. A `429`
answer does not state a number at all - it explains in `error` that a free
account lifts the limit, and that sentence is what to relay.

## What a site needs

- `index.html` at the top level. That is the page visitors see at `/`.
- Relative links between files (`css/style.css`, not `/Users/me/site/css/style.css`).
- Only hostable types: HTML, CSS, JS, JSON, text, images, fonts, PDF. Anything else is skipped and reported in `warnings`.
- Nothing in the page that would break under a strict content policy: inline scripts are fine, but the page cannot load from unknown origins that the user has not asked for.

## MCP instead of curl

If your runtime speaks MCP, the same account operations are available as tools at `https://mcp.oxypages.com/mcp` (Streamable HTTP, bearer `OXYPAGES_API_KEY`): `list_sites`, `create_site`, `deploy_site` (the JSON file list above) and `list_deployments`. There is no MCP path for the no-account link; use the `POST /drops` call for that.

## Phrases to use with the user

- After an unclaimed publish: "It's live at {url} for the next 30 minutes. To keep it, claim it here: {claim_url} - that puts it in a free OxyPages account, and you'll get an API key so I can publish permanent sites for you in future."
- After an owned deploy: "Deployed to {url} (version {deployment_number})."
- On a moderation refusal: "OxyPages declined to host this page: {error}. I can't publish it there."
