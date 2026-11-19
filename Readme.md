# treasury-invite

Cloudflare Worker behind `invite.getzaldo.com` that backs Zaldo's invite /
deep links. It runs on the getzaldo.com zone as the route `invite.getzaldo.com/*`
(see `wrangler.jsonc`), so the DNS record's target does not matter.

History: this was a Deno Deploy app until November 2026, when Deno failed to
renew the custom domain's certificate and unlinked it. The legacy
`treasury-invite.meditec.deno.net` host (invite links shared before the
getzaldo.com migration) is still served by the old Deno deployment from the
pre-Worker `main.ts` and goes away when Deno Deploy Classic shuts down in
April 2027.

Given an invite URL like:

```
https://invite.getzaldo.com?group=<groupId>&name=<groupName>
```

it routes the visitor by who they are:

- **Link-preview crawlers** (WhatsApp, Facebook, Telegram, Slack, Discord, …)
  receive HTML with Open Graph / Twitter card tags for a rich preview. The card
  image is served from this same service at `/og-banner.png`.
- **Android browsers** get a landing page whose button fires an `intent://` deep
  link into the installed app, with `browser_fallback_url` pointing at the web
  app, plus an APK-download link and a deferred-invite clipboard handoff.
- **Everyone else — desktop and iPhone** — is 302-redirected straight to the
  Flutter web app with `?group=&name=` intact.

iPhones used to get the mobile landing page, which tried `treasury://invite?…`
first and only fell through to the web app after 1.5 s. There is no iOS build
(`ios/Runner/Info.plist` registers no `CFBundleURLTypes`), so Safari answered
with a modal "Cannot Open Page — the address is invalid" that blocked the
fallback. Nothing outside Android claims the scheme, so nothing outside Android
should be offered it.

## Configuration

The Flutter web app origin is the `WEB_APP_URL` **constant at the top of
`src/index.ts`**, not an env var. It was an env var set in the Deno Deploy dashboard
and it silently drifted to a stale Cloudflare Pages preview host during the
getzaldo.com migration; that host 301s to getzaldo.com and drops the query
string, so every invite arrived with no group to join.

Plain values live under `vars` in `wrangler.jsonc`; the anon key is a secret
(`npx wrangler secret put SUPABASE_ANON_KEY`).

| Var                          | Purpose                                                        |
| ---------------------------- | -------------------------------------------------------------- |
| `SUPABASE_URL`               | APK download URL + short-code resolution RPC.                  |
| `SUPABASE_ANON_KEY`          | Auth for the `get_invite_info` RPC (resolves `/<code>` links). |
| `OG_IMAGE_URL`               | Optional. Overrides the default self-hosted `/og-banner.png`.  |
| `ANDROID_SHA256_FINGERPRINT` | App Links cert fingerprint for `/.well-known/assetlinks.json`. |

Link formats accepted: `/<code>` (8-char base62, resolved via Supabase) and the
legacy `?group=<uuid>&name=<name>` (still live in already-shared chats).

## Local development and deploy

```sh
npm install
npm run dev      # wrangler dev, http://localhost:8787
npm run deploy   # wrangler deploy (needs `npx wrangler login`)
```

For local runs, put `SUPABASE_ANON_KEY=...` in `.dev.vars` (gitignored).
Test the crawler path with:

```sh
curl -A "WhatsApp/2" "http://localhost:8787/?group=00000000-0000-0000-0000-000000000000&name=Ski%20Trip"
```

`public/og-banner.png` is served by Workers Static Assets before the Worker
runs; its cache header is in `public/_headers`.

Roll back by removing the `routes` entry and redeploying (or deleting the route
in the dashboard).
