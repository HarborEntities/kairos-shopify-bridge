# Kairos Operator, Shopify session token bridge

Minimal static HTML page that runs inside the Shopify admin as an embedded app. It uses Shopify App Bridge to fetch a one-time session token, POSTs it to a Kairos webhook, and displays a success message. Kairos then exchanges the session token for a long-lived offline admin API token.

This is the workaround for the 2026 Shopify non-Plus auth limitation: client credentials grant is Plus-only, legacy custom apps are killed for new creation, OAuth code flow no longer applies to Dev Dashboard apps. Token exchange via App Bridge is the only remaining path.

## Setup

### 1. Edit `index.html`

Two placeholders to replace:

| Placeholder | Replace with |
|---|---|
| `REPLACE_WITH_CLIENT_ID` | The Client ID of the Kairos Operator app from dev.shopify.com (Settings tab). Looks like a 32-char hex string. |
| `REPLACE_WITH_WEBHOOK_URL` | The webhook.site URL you already control. Same one Kairos has been using. |

### 2. Push to GitHub Pages

```bash
cd /home/he-team/kairos-claude/shopify-bridge
git init
git add index.html README.md
git commit -m "Initial bridge page"
git branch -M main
git remote add origin https://github.com/<your-user>/<repo-name>.git
git push -u origin main
```

Then on github.com: repo Settings, Pages, set Source to "Deploy from a branch", branch `main`, folder `/ (root)`, save. Wait 1 minute for the deploy.

The bridge URL becomes `https://<your-user>.github.io/<repo-name>/`.

### 3. Update Dev Dashboard app config

dev.shopify.com, Kairos Operator app, Settings:

- **App URL:** the GitHub Pages URL above
- **Embed app in Shopify admin:** checked
- **Allowed redirection URLs:** add the same GitHub Pages URL (some App Bridge handshakes use this)

Release a new version.

### 4. DALLAS uninstalls, reinstalls

- Beachgroomer admin, Apps, Kairos Operator, Uninstall
- Click the original custom distribution install URL
- After install, click "Open app"
- Sees the bridge page run inside the admin iframe, captures token, displays success

### 5. Kairos picks it up

Tell Kairos the webhook has fired. Kairos reads the captured session token, exchanges it for an offline admin API token via:

```
POST https://beachgroomer.myshopify.com/admin/oauth/access_token
Content-Type: application/json
{
  "client_id": "<client-id>",
  "client_secret": "<client-secret>",
  "grant_type": "urn:ietf:params:oauth:grant-type:token-exchange",
  "subject_token": "<session-token-from-webhook>",
  "subject_token_type": "urn:ietf:params:oauth:token-type:id_token",
  "requested_token_type": "urn:shopify:params:oauth:token-type:offline-access-token"
}
```

Result: a long-lived `shpat_`-or-`atkn_`-format offline access token. Kairos saves it to `brain/secrets/shopify_admin_token.txt` and starts using it for Admin API calls.

## Security notes

- The bridge page is public-static. It does NOT contain any secret. The Client ID is public per Shopify's design (it identifies the app, not the merchant).
- The Client Secret stays only in Kairos's memory and is used server-side for the token exchange call.
- Session tokens shown in the bridge page are short-lived (60 seconds). Even if the page is screenshotted, the token expires before it can be abused.
- The webhook.site URL receives the session token. Treat that URL like a secret. Anyone who knows it could intercept the next install's token. Rotate after the install is complete.

## After the install works

The HTML page can stay deployed for future re-installs (e.g., if the token gets revoked or you onboard a new merchant). It is idempotent and safe to re-trigger.
