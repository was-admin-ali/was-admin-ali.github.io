# SnapNotes checkout site

A static landing + checkout page for SnapNotes' one-time "Lifetime" upgrade.
Hosted for free on GitHub Pages. Payment verification is now handled by a
small companion API in `../snapnotes-api/` (deployed on Vercel) — see that
folder's README for the backend setup. This README covers the site itself.

## How the whole system fits together

1. Extension gives everyone 10 free saved notes (tracked in `chrome.storage.local`).
2. When someone wants more, the extension opens this site in a new tab.
3. This site opens a **Paddle Checkout** overlay. Paddle handles the actual
   payment (cards, taxes, receipts, refunds — all of it).
4. The instant Paddle reports `checkout.completed` (client-side) **and**
   sends a webhook (server-side) to `snapnotes-api`, that API verifies the
   transaction and issues a real license key, stored in Redis.
5. This page fetches that key from the API and shows it on screen.
6. The person pastes the key into the extension ("Already purchased? Enter
   your code"). The extension calls the same API to confirm the key is real
   before unlocking unlimited notes — so, unlike a purely offline check,
   someone can't just make up a valid-looking code.

This replaces an earlier honor-system version of this project (format-only,
no server) — if you're upgrading from that version, you now need to deploy
`snapnotes-api` too; it isn't optional anymore for the checkout flow to work.

## Setup checklist

### 1. Paddle
1. Create a [Paddle](https://www.paddle.com) account (Paddle Billing, not Classic).
2. In **Catalog → Products**, create a product, e.g. "SnapNotes Lifetime".
3. Add a one-time **Price** to it (e.g. $19 USD). Copy its Price ID
   (`pri_...`).
4. In **Developer Tools → Authentication**, create a **client-side token**
   (`test_...` for sandbox, `live_...` for production). Copy it.
5. In **Checkout → Checkout settings → Approved domains**, add the domain
   this site will be hosted on, e.g. `yourusername.github.io`. Paddle.js
   silently refuses to open checkouts on domains that aren't approved here —
   this trips up almost everyone the first time, so don't skip it.
6. While testing, use **Sandbox** mode: a sandbox account, a sandbox client
   token, a sandbox price ID, and uncomment `Paddle.Environment.set("sandbox")`
   in `index.html`. Switch all three (token, price ID, and remove/comment out
   the sandbox line) when you go live.

### 2. Deploy `snapnotes-api` first
This site depends on it — do that deployment before coming back here. See
`../snapnotes-api/README.md`.

### 3. Edit `index.html`
Replace these three placeholders near the bottom of the file:
```js
const PADDLE_CLIENT_TOKEN = "YOUR_PADDLE_CLIENT_TOKEN";
const PADDLE_PRICE_ID = "YOUR_PADDLE_PRICE_ID";
const SNAPNOTES_API_BASE = "https://your-snapnotes-api.vercel.app"; // your deployed snapnotes-api URL
```
Also update the two `href="https://chrome.google.com/webstore"` placeholder
links once your extension has a real Chrome Web Store listing URL.

### 4. Edit the extension's `license.js`
Update these two lines to point at wherever you deployed this site and the API:
```js
const SNAPNOTES_CHECKOUT_URL = "https://YOUR_GITHUB_USERNAME.github.io/snapnotes-site/";
const SNAPNOTES_API_BASE = "https://your-snapnotes-api.vercel.app";
```
And in `manifest.json`, update `host_permissions` to match the same API URL.

### 5. Deploy to GitHub Pages
1. Create a new GitHub repo, e.g. `snapnotes-site`.
2. Push everything in this folder (`index.html`, `privacy.html`, `terms.html`,
   `style.css`, `icon128.png`, `screenshot.png`) to the repo's `main` branch.
3. In the repo, go to **Settings → Pages**, set **Source** to "Deploy from a
   branch", branch `main`, folder `/ (root)`. Save.
4. GitHub gives you a URL like `https://yourusername.github.io/snapnotes-site/`
   — that's the URL from steps 3 and 4 above.
5. Give it a minute or two after pushing/enabling Pages before it's live.

### 6. Test end to end
1. Open the deployed site, click **Unlock unlimited**, pay with a
   [Paddle sandbox test card](https://developer.paddle.com/concepts/payment-methods/test-payment-methods)
   if you're in sandbox mode.
2. Confirm the license key appears on screen after payment (this now depends
   on `snapnotes-api` responding — see that project's README if it hangs or
   errors).
3. Paste that key into the extension's "Enter your code" field and confirm
   it activates (pill in the popup header should change to "✓ Lifetime").
   This step now makes a real network call to `snapnotes-api` too.
4. Switch Paddle + this site + the API's env vars to production values
   before sharing the real link with users.

## Files
- `index.html` — the page itself (hero, pricing, checkout, FAQ)
- `privacy.html`, `terms.html` — legal pages required by both Chrome Web
  Store and Paddle's merchant review (see `../snapnotes/STORE_LISTING_CHECKLIST.md`)
- `style.css` — styling, matching the extension's navy/purple/amber palette
- `icon128.png`, `screenshot.png` — visual assets, swap for your own anytime
