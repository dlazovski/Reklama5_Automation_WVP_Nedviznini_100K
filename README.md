# Reklama5.mk — Real Estate Contact Extraction (n8n)

Automation that walks Reklama5.mk's real estate listings (category 157, for-sale, ≥100,000 €), pulls seller/agency contact info off each listing page, and appends new rows to a Google Sheet. Built for Moduvo.

## ⚠️ Important: the verification step described in the brief could not be run by me

The brief asked for a pre-build test fetch (currency check + subcategory check) before building the full scraper. I could not perform that: this environment's outbound network policy blocks `reklama5.mk` at the proxy level (confirmed 403 on the CONNECT tunnel — this is an organization-level egress restriction on my sandbox, not something wrong with the site or your account).

To compensate, I:
1. Built **both open questions as defensive, always-on guards in the main workflow** rather than one-time assumptions (see "Currency guard" and "Subcategory coverage" below).
2. Shipped a **separate, lightweight verification workflow** (`reklama5-verification.json`) that does the exact test the brief asked for — you run it once from your own machine/n8n instance (which has normal internet access), eyeball the results, and only then trust the full run.
3. Because I never saw the live HTML, all field extraction uses **anchor/regex-based text parsing** tied to the semantic landmarks you confirmed (the `<h1>` title, the "Се продава од:" label, the `AdDetails?ad=` URL pattern, image-presence for agency detection) rather than guessed CSS class names, which would have been much more likely to silently break. This is more resilient to markup you didn't show me, but you should still run the verification workflow first and skim the first real batch of rows before leaving this unattended.

## What's in this folder

```
workflows/
  reklama5-verification.json   ← run this FIRST, manually, no Sheets writes
  reklama5-scraper.json        ← the production scraper
README.md                      ← this file
```

## 1. Verification workflow — run this first

`workflows/reklama5-verification.json` fetches search results page 1, takes the first 10 listings, fetches each detail page, and reports (in the n8n execution output — nothing is written anywhere):

- `title`, `priceRaw`, `priceCurrency`, `isEurAndAbove100k`
- `breadcrumbGuess` — best-effort category breadcrumb text
- `sellerTypeDetection` — whether the image-presence agency/private detection found the "Се продава од" block at all

**Import it, connect no credentials (none needed), click "Test workflow", then open the `Parse & Report` node's output and check:**

1. **Currency**: is `priceCurrency` `€` for all 10? If any show `МКД`, that confirms the brief's suspicion — the main workflow already filters/flags these (see below), but let me know if it's more than a rare edge case, since it might mean `pricefrom=100000` isn't doing what we think.
2. **Subcategories**: does `breadcrumbGuess` vary across the 10 (houses, apartments, land, etc.), or is it all one type? If it's narrow, tell me what other listing types you'd expect to see and I'll check whether `cat=157` needs sibling `subcat=` values added to the search URL.
3. **Seller detection**: does `sellerTypeDetection` correctly say "Агенција (има слика)" vs "Приватно лице (нема слика)" for listings you recognize? Spot-check 2-3 against the live page.

If anything looks wrong, send me the raw output (or just describe the mismatch) and I'll adjust the parsing before you rely on the full scraper.

## 2. Main scraper — `workflows/reklama5-scraper.json`

### Setup

1. **Create the Google Sheet** with two tabs, headers exactly as below (row 1):

   **Listings**
   ```
   Наслов на оглас | Цена | Локација | Датум на објавување | Име на продавач/агенција | Телефон | Приватно/Агенција | Линк до оглас | Датум на извлекување
   ```

   **Errors**
   ```
   Чекор | Порака за грешка | Линк до оглас | Датум и време
   ```

2. **Import** `workflows/reklama5-scraper.json` into n8n (Workflows → Import from File).

3. **Google Sheets credential**: this workflow has **5** Google Sheets nodes (`Read Existing Listings`, `Append Listing Row`, `Append Excluded Row`, `Log Search Page Error`, `Log Listing Error`). Open each one and:
   - Select/create your Google Sheets OAuth2 credential.
   - Select your spreadsheet and the correct tab (`Listings` or `Errors` — already pre-filled by name, but click into the field once so n8n binds the actual sheet ID).

4. **First-import checklist** (standard for any hand-authored n8n template — these are one-click confirmations, not real problems):
   - Google Sheets nodes: confirm the "Operation" shows *Append* (write nodes) or *Get Row(s)* (Read Existing Listings). If n8n shows a version-mismatch banner, click "Update" — it's forward-compatible.
   - The two HTTP Request nodes (`Fetch Search Page`, `Fetch Listing Detail`): open the "Settings" tab and confirm **On Error** is set to *"Continue (using error output)"* — this is what makes the custom retry loop work instead of stopping the whole run on a timeout.
   - The `Config` node's values (see "Tuning" below) — defaults are conservative on purpose.

5. **Test run**: set `Config.maxPages` to `1` or `2` for a first manual run (click "Test workflow"), check the Sheet fills in correctly, then raise `maxPages` back up (default 50) for real runs.

6. Optionally enable the `Schedule Trigger` node (disabled by default, daily at 06:00) if you want this to run unattended. Leave it disabled if you'd rather trigger manually.

### How it works

**Pagination loop** (`Fetch Search Page` → `Parse Search Page` → `More Pages?` → `Pagination Wait` → back to `Fetch Search Page`):
- Walks `page=1,2,3...` of the confirmed search URL (`cat=157&sell=1&pricefrom=100000`) until a page returns zero `AdDetails?ad=` links, or `Config.maxPages` (default 50) is hit.
- Only collects ad IDs + absolute URLs on this pass (`https://reklama5.mk/AdDetails?ad={id}`) — every displayed field (title, price, location, date, seller, phone) is read from the **detail page**, not the results card. This was a deliberate simplification: the detail-page field locations are the ones you confirmed via manual inspection; the results-card markup wasn't, so leaning on the confirmed source reduces the chance of silently wrong data.
- 3-4s wait between search-page requests (`Config.waitSeconds`).

**Dedup**: before fetching any detail pages, `Read Existing Listings` reads the `Линк до оглас` column of the whole `Listings` tab, extracts the `ad=` ID from each existing link, and `Filter New Listings` drops anything already present. Only genuinely new ad IDs get fetched.

**Detail fetch loop** (`Loop Listings` batch-of-1 over new listings):
- `Fetch Listing Detail` → `Parse Listing Detail` extracts title (`<h1>`), price (first €/МКД-style number after the title), date + location (near a "Денес/Вчера HH:MM" pattern), and the seller block anchored on the "Се продава од" label — image present → Агенција, text-only → Приватно лице. Phone is read from that block first; if not found there, it falls back to scanning the full description text for a Macedonian phone pattern (`07X XXX XXX` / `070/787-127` / `07XXXXXXX`), same as the brief specified.
- Phone is normalized to digits-only alongside the original: e.g. `070/787-127 (070787127)`.
- 3-4s wait between every detail-page request (`Config.waitSeconds`).

**Retry / exponential backoff**: both HTTP fetch points use `onError: continueErrorOutput`, so a failed request (timeout, 5xx) routes to a retry handler instead of stopping the workflow. It retries up to `Config.maxRetries` (default 3) times, waiting `baseRetryWaitSeconds × 2^attempt` between tries (≈3s, 6s, 12s, 24s with defaults). After retries are exhausted, the failure is logged to the `Errors` tab (step, error message, URL, timestamp) and the workflow moves on to the next listing/page — nothing stops the overall run.

**Currency guard**: `pricefrom=100000` on the search URL should mean everything returned is above threshold in EUR, but per the brief's own concern, some results can show in МКД (which would put the real EUR value under threshold). `Currency & Price Filter` only lets a row into `Listings` if the parsed currency is `€` **and** the numeric price is ≥ `Config.priceFrom` **and** the URL host is `reklama5.mk`. Anything that fails this — МКД-priced, unparseable price, or (as a defensive belt-and-suspenders check) a non-reklama5.mk URL — is logged to `Errors` with the reason instead of silently dropped, so you can audit how often it happens.

**Domain safety**: the search URL is hardcoded to `reklama5.mk` (never the `m.` subdomain), all detail URLs are built by prepending `https://reklama5.mk` to the ID pulled from `AdDetails?ad=`, and the currency guard double-checks the final URL host before writing a row.

### Tuning (`Config` node)

| Field | Default | Meaning |
|---|---|---|
| `priceFrom` | 100000 | Minimum EUR price kept (also baked into the search URL) |
| `waitSeconds` | 4 | Wait between every outbound request (rate limit) |
| `baseRetryWaitSeconds` | 3 | Base for exponential backoff (× 2^attempt) |
| `maxRetries` | 3 | Max retry attempts per failed request |
| `maxPages` | 50 | Safety cap on pagination depth |

### Columns written to `Listings`

`Наслов на оглас, Цена, Локација, Датум на објавување, Име на продавач/агенција, Телефон, Приватно/Агенција, Линк до оглас, Датум на извлекување` — in that order, matching the spec exactly. `Линк до оглас` is always the full absolute URL.

## Respectful-scraping notes

- Only `reklama5.mk` is ever fetched — the workflow never touches `m.reklama5.mk` (which disallows automated access via robots.txt).
- No login/session/cookies — contact info is confirmed visible while logged out.
- 3-4s between every single request, both for pagination and for detail pages; failed requests back off exponentially rather than hammering a slow server.
- Nothing here bypasses any access control — it reads what's already publicly rendered on the page.
- Re-check the Terms of Service periodically; the brief noted no explicit automated-collection clause as of this build, but ToS pages change.

## Known limitations / what to watch on first real runs

- **Selectors are text-anchor/regex based, not exact CSS selectors** — because I never saw the live HTML. This is deliberately more resilient to class-name churn, but if Reklama5 changes wording (e.g. renames "Се продава од"), extraction breaks silently for that field until updated. Skim the first day or two of output.
- **Breadcrumb/subcategory data is not carried into the main scraper** — it's only in the verification workflow, used purely to answer the brief's "does cat=157 cover all subtypes" question. If you determine additional `subcat=` values are needed after running verification, tell me and I'll extend the search-URL loop (it currently only paginates `page=`, not subcategory).
- **The exact JSON shape of the HTTP node's error-branch item** (used by the retry handlers) is based on my best knowledge of current n8n behavior, not a live test. If the `Errors` tab ever shows a blank/wrong `Линк до оглас` on a retry-exhausted row, open `Fetch Listing Detail`'s error output in n8n, check what fields are actually present on that item, and tell me — it's a one-line fix in `Listing Retry Handler` / `Search Page Retry Handler`.
- Google Sheets node internals (operation names, resource-locator format) are based on current n8n source, not a live import test in your instance — the "First-import checklist" above covers the handful of things worth a 30-second glance.
