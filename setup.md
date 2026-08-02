# Denis Family Reunion — Setup Guide

A clear, step-by-step guide to publishing the site, turning on the **one‑person
editor login**, and connecting the RSVP form. No coding experience required — just
follow along. Total time: about 30–40 minutes, once.

---

## What's in this folder

| File / folder | What it is |
|---|---|
| `index.html` | The whole website. |
| `content.json` | The editable details — venues, dates, **announcements**, the **contribution amount & payment links**, and photos. The site reads this. |
| `admin/` | The private editor screen (`/admin/`) + its configuration. |
| `assets/` | Where your logo and photos live. |
| `SETUP.md` | This guide. |

The site works immediately if you just open `index.html` — but the **login editor**
and the **RSVP‑to‑Sheet** features only switch on once it's published (Parts 1 & 2).

---

## Part 1 — Publish the site + turn on the one‑person editor

This gives you exactly what you asked for: a website only **you** can change,
protected by **email + password**, where nobody else can edit unless you invite them.
It uses two free services — **GitHub** (stores the files) and **Netlify** (hosts the
site + handles the login). The editor itself is **Decap CMS** (a free, open‑source
content editor).

> Why GitHub + Netlify? The login/editor needs a place that can securely save your
> edits. Netlify's free "Identity" service provides the email‑and‑password login, and
> it saves changes back to GitHub. A simple drag‑and‑drop host can't do the login part.

### Step 1 — Put the files on GitHub
1. Create a free account at <https://github.com>.
2. Click **New repository**. Name it e.g. `denis-family-reunion`. You can make it
   **Private** (recommended) — the published website is still public, but the source
   files stay private.
3. On the new repo page, click **uploading an existing file**, then drag in
   **everything in this folder** (`index.html`, `content.json`, the `admin` folder,
   the `assets` folder). Click **Commit changes**.

### Step 2 — Host it on Netlify
1. Create a free account at <https://www.netlify.com> (choose "Sign up with GitHub" —
   easiest).
2. Click **Add new site → Import an existing project → GitHub**, and pick your
   `denis-family-reunion` repo.
3. Leave build settings empty (no build command; publish directory = root). Click
   **Deploy**. In under a minute you'll get a live link like
   `https://denis-family-reunion.netlify.app`. 🎉

### Step 3 — Turn on the login (Netlify Identity, invite‑only)
1. In your site's Netlify dashboard, open the **Identity** tab → **Enable Identity**.
2. Click **Settings and usage**. Under **Registration**, choose **Invite only**.
   *(This is the key setting — it means only email addresses you personally invite can
   ever log in. Nobody can self‑register.)*
3. Scroll to **Services → Git Gateway → Enable Git Gateway**. (This lets your saved
   edits flow back to GitHub.)

### Step 4 — Invite yourself (and only who you choose)
1. Still on the **Identity** tab, click **Invite users** and enter **your email**.
2. Check your inbox, click the invite link, and **set your password**.
3. That's it — you are now the single authorized editor. To add another person later,
   just invite their email the same way. Anyone not invited **cannot** edit.

### Step 5 — Point the editor at your site
1. In GitHub, open `admin/config.yml` → pencil icon to edit.
2. Change the two `YOUR-SITE.netlify.app` lines to your real Netlify address.
3. If your repo's default branch is `master` (not `main`), change `branch: main`
   accordingly. Commit.

### Step 6 — Edit any time, from anywhere
- Go to **`https://your-site.netlify.app/admin/`** (or click **"Family organizer
  login"** at the very bottom of the site).
- Log in with your email + password.
- Change venues, dates, the RSVP deadline, **post announcements**, **set the
  contribution amount and payment links**, add photos to the gallery, etc. Press
  **Publish**. Your change goes live in about a minute. (Posting announcements is in
  Part 5; adding payment links is in Part 4.)

✅ **Result:** one login, one editor (you), invite‑only. Visitors can read the site
but can never change it.

> **Lightweight alternative:** if you'd rather skip the CMS, you (and only you) can also
> edit `content.json` directly on GitHub — that's already protected by your GitHub
> password. The `/admin/` editor is just the friendlier, no‑code way to do the same thing.

---

## Part 2 — Connect the RSVP form to a Google Sheet

So every RSVP + T‑shirt order collects in one spreadsheet you own.

### Step 1 — Create the Sheet
Go to <https://sheets.google.com>, create a **blank** spreadsheet, name it
**Denis Reunion RSVPs**. Leave it empty (headers are added automatically).

### Step 2 — Add the script
In that sheet: **Extensions → Apps Script**. Delete anything there, paste the code
below, and **Save**.

```javascript
// Denis Family Reunion — RSVP collector
function doPost(e) {
  var lock = LockService.getScriptLock();
  lock.waitLock(30000);
  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var sheet = ss.getSheetByName('RSVPs') || ss.insertSheet('RSVPs');
    var data = JSON.parse(e.postData.contents);
    var headers = ['Timestamp','Family / Household','Contact Name','Email','Phone',
                   'RSVP Status','Adults','Children','Events Attending',
                   'T-Shirt Orders','Total Shirts','Notes'];
    if (sheet.getLastRow() === 0) {
      sheet.appendRow(headers);
      sheet.getRange(1,1,1,headers.length).setFontWeight('bold');
      sheet.setFrozenRows(1);
    }
    var shirtText = (data.shirts || []).map(function(s){
      return s.qty + '× ' + s.size + (s.who ? ' (' + s.who + ')' : '');
    }).join('; ');
    sheet.appendRow([ new Date(), data.familyName||'', data.contactName||'',
      data.email||'', data.phone||'', data.attending||'', data.adults||'',
      data.kids||'', (data.events||[]).join(', '), shirtText,
      data.totalShirts||0, data.notes||'' ]);
    return ContentService.createTextOutput(JSON.stringify({result:'success'}))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({result:'error',message:String(err)}))
      .setMimeType(ContentService.MimeType.JSON);
  } finally { lock.releaseLock(); }
}
function doGet(){ return ContentService.createTextOutput('Denis Reunion RSVP endpoint is live.'); }
```

### Step 3 — Deploy it
1. **Deploy → New deployment** → gear ⚙️ → **Web app**.
2. **Execute as: Me** · **Who has access: Anyone** → **Deploy** → authorize with your
   Google account. Copy the **Web app URL** (`https://script.google.com/macros/s/…/exec`).

### Step 4 — Paste the URL into the site
In `index.html`, find:
```javascript
const SHEET_ENDPOINT = "PASTE_YOUR_GOOGLE_APPS_SCRIPT_URL_HERE";
```
Replace the placeholder with your URL (keep the quotes), commit. RSVPs now flow into
your Sheet. Until you do this, the form runs in a friendly **preview mode**.

> The Sheet lives in **your** Google Drive — only you can see the responses unless you
> choose to share it.

---

## Part 3 — Make it yours

### Your logo
The site currently uses a hand‑drawn tree emblem that echoes your logo. To use your
**exact** artwork:
1. Add your file to the `assets` folder as `logo.png` (a transparent PNG works best).
2. In `index.html`, find the line beginning `<svg class="hero__emblem"` and replace
   that whole `<svg …>…</svg>` block with:
   `<img class="hero__emblem" src="assets/logo.png" alt="Denis Family Reunion logo">`
   *(Or just send the file over and I'll wire it in everywhere — hero, footer, favicon.)*

### Photos
Easiest: open **`/admin/`**, go to **Memories photos**, and upload — they appear in the
gallery automatically. (Or drop files in `assets/` and list them in `content.json`.)

### Everything marked "to be determined"
The host hotel, brunch spot, cost, and RSVP deadline are all editable in **`/admin/`**
under **Reunion Details** — no code needed.

---

## Part 4 — Collect contributions (payments)

The new **Contribute** section shows a suggested amount and a row of payment buttons.
**You choose which options to offer** — fill in the ones you want and leave the rest
blank (blank ones simply don't appear). Until you add at least one, the section shows a
friendly "coming soon" placeholder.

Crucially, **no card details are ever entered on or stored by this site.** Each button
hands off to the payment provider's own secure, hosted checkout — so the family and the
committee stay entirely out of card-handling (in payments terms, PCI-DSS **SAQ-A**).

Add these in **/admin/** under **💳 Family Contribution → Payment options** (or edit
`content.json` directly). You can also change the **amount** ("$150 per adult") there
anytime.

### 4A — Card payments (Stripe Payment Link) — best for far-flung family
1. Create a free account at <https://stripe.com>.
2. In the Stripe Dashboard, open **Payment Links** (under **Product catalogue**, or
   just search "Payment links") and create a **new link**.
3. Either set a fixed price (e.g. $150) or choose **"let customers pay what they want"**;
   name it "Denis Reunion Contribution" and create it.
4. Copy the link (it looks like `https://buy.stripe.com/…`) and paste it into the
   **Card link** box. A **Pay by card** button appears — cards, Apple Pay and Google
   Pay, worldwide. Stripe deducts a small per-payment processing fee.

### 4B — PayPal
1. Create your **PayPal.Me** link at <https://paypal.me> (e.g. `https://paypal.me/DenisReunion`).
2. Paste it into the **PayPal link** box. A **PayPal** button appears.

### 4C — Zelle / Venmo / Cash App (US family, usually no fees)
- Enter the **email or phone** for Zelle, the **@handle** for Venmo, or the **$Cashtag**
  for Cash App in the matching boxes.
- Each shows as a neat **tap-to-copy** chip — the payer opens their own bank or app and
  sends to that handle. (Because the published site is public, these handles are visible
  to any visitor. That's normal for a reunion site; if you'd prefer to keep them
  family-only, see the access-code note in Part 6.)

### 4D — Letting family pay in installments

Two routes, both run entirely by the provider — you never store cards or send the
reminders yourself:

**PayPal "Pay in 4" (PayPal fronts you the money).** A family member can split their
contribution into four payments over six weeks; PayPal pays **you the full amount up
front**, then collects the four installments from them and sends all the reminders
itself. To offer it:
1. You need a **PayPal Business** account with **PayPal Checkout** enabled (Pay in 4 is
   included at no extra cost beyond standard PayPal checkout fees).
2. Use a **PayPal Checkout** button/link for the PayPal option — generate one with
   PayPal's no-code **Smart Payment Button**, or send a **PayPal invoice**. A plain
   **PayPal.Me** link generally will **not** surface Pay in 4.
3. Paste that checkout link into the **PayPal link** box in `/admin/`. Eligible US family
   then see "Pay in 4" automatically at checkout — nothing else for you to do.

**Stripe payment plan (a schedule you set, with automatic reminders).** Stripe can split
a $200 invoice into dated installments and email the payer an automatic reminder before
each due date:
1. In the Stripe Dashboard, create an **Invoice** for the household, add the $200 line
   item, and choose **Payment plan** — set the number of installments and due dates
   (e.g. 4 × $50, monthly).
2. Turn reminders on once, for all invoices: **Settings → Billing → Invoices → "Manage
   advanced invoicing features" → "Send reminders if a one-off invoice hasn't been
   paid"**, then **Add reminder** for each nudge (e.g. 3 days before due, on the due
   date, and a few days after).
3. Send the invoice. The payer gets a hosted invoice and taps to pay each installment;
   Stripe chases them automatically and you watch the balance in the Dashboard. *(Note:
   payment plans don't auto-charge a saved card — the payer pays each part themselves,
   which most families prefer to a recurring card charge.)*

> **Want this to scale without creating invoices by hand?** I can write you a small
> script that reads your RSVP Google Sheet and creates a Stripe payment-plan invoice for
> each attending household automatically — just ask, and I'll walk you through running it
> safely with your Stripe key (the key never goes near the website).

The **Contribute** section already tells family both options exist; reword or hide that
panel under **💳 Family Contribution → Installments messaging** in `/admin/`.

### Keeping track of who paid
Everyone is asked, on the site, to put their **family / household name** in the payment
note. Reconcile from each provider's own record: Stripe and PayPal list every payment in
their dashboards; Zelle/Venmo/Cash App appear in your banking or app history. If you'd
later like contributions matched to RSVPs automatically (a "contribution sent" field on
the RSVP form feeding your Google Sheet), just ask and I'll wire it in.

---

## Part 5 — Post an announcement

1. Go to **/admin/** and log in.
2. Open **📣 Announcements (news board)**.
3. Click **Add Announcement**, set the **date**, write a **title** and **message**, and
   tick **Pin to top** if it should stay at the top (e.g. a deadline reminder or the
   hotel room-block link).
4. Press **Publish**. It appears in the **News** section within about a minute, newest
   first — pinned posts always sit above the rest. Edit or delete posts anytime the same
   way.

> The starter **"Welcome to the reunion website"** post is just an example — edit it or
> delete it whenever you like.

---

## Part 6 — Security notes (the short version)

- **Who can edit:** only email addresses you invite in Netlify Identity (set to
  *Invite only*). Everyone else gets read‑only — the published site.
- **Who can view:** the site is public by default (normal for a reunion site). If you
  want it family‑only, Netlify offers site‑wide password protection on paid plans, or I
  can add a lightweight access code — just note a front‑end code is convenience, not
  strong security.
- **RSVP data:** stored in your own Google Sheet, visible only to you unless shared.
- **Payments:** the site never collects or stores card details. Card and PayPal buttons
  link out to the provider's own secure checkout; Zelle/Venmo/Cash App show a handle you
  pay from your own app. No financial secrets live in the site (PCI-DSS SAQ-A).
- **Transport:** Netlify serves everything over HTTPS automatically.
- **No secrets in the code:** the RSVP endpoint only accepts new rows; it can't read
  the sheet back. The editor login is handled by Netlify, not stored in the site.

---

## Questions or changes
This is a living draft. Send me your logo, photos, confirmed venues, costs, and
committee contact and I'll fold them in — plus anything else (a family‑tree page, music
requests, automatic matching of contributions to RSVPs, automatic RSVP reminders, a
custom domain like `denisfamilyreunion.com`, and more). I'm happy to walk you through
Netlify — or the Stripe/PayPal setup — live, too.
