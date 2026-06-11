# zotero-webdav

A minimal WebDAV server on Cloudflare Workers backed by R2, built for
[Zotero file syncing](https://www.zotero.org/support/sync#alternative_syncing_solutions).
Implements exactly the WebDAV subset Zotero uses: `OPTIONS`, `PROPFIND`
(Depth 0/1), `MKCOL`, `HEAD`, `GET`, `PUT`, `DELETE`, with HTTP Basic auth.

Zotero stores each attachment as `<itemKey>.zip` + `<itemKey>.prop` under
`/zotero/`, which map 1:1 to R2 object keys. R2 has no directories, so
collections are implicit and `MKCOL` is a no-op success.

## Deploy

```sh
npm install
npx wrangler login                                  # if not already authed
npx wrangler r2 bucket create zotero-attachments
npx wrangler secret put WEBDAV_USERNAME             # pick a username
npx wrangler secret put WEBDAV_PASSWORD             # pick a strong password
npx wrangler deploy
```

## Configure Zotero

In **Settings → Sync → File Syncing**, set "Sync attachment files in My
Library using" to **WebDAV**:

- URL: `https://zotero-webdav.<your-subdomain>.workers.dev/` (Zotero appends
  `zotero/` itself)
- Username / password: the secrets you set above

Click **Verify Server**. Repeat on each device.

## Local development

```sh
npm run dev    # local server with simulated R2; creds in .dev.vars
```

## Notes & limits

- Workers caps request bodies (~200–500 MB depending on plan tier), which
  bounds the largest single attachment zip you can upload. Typical PDFs are
  nowhere close.
- Anyone with the URL can attempt Basic auth, so use a strong password;
  credentials are compared in constant time.
- Costs: R2 storage is $0.015/GB-month with zero egress fees; Workers
  requests at Zotero-sync volumes are effectively free on a paid plan.
- Zotero's "purge orphaned files" uses Depth-1 `PROPFIND`, which is
  supported (paginates the full bucket listing).
