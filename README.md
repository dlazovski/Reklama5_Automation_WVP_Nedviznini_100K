# Reklama5.mk — Real Estate Contact Extraction (n8n)

Automation that walks Reklama5.mk's real estate listings (category 157, for-sale, ≥100,000 €), pulls seller/agency contact info off each listing page, and appends new rows to a Google Sheet. Built for Moduvo.

One workflow — `workflows/reklama5-scraper.json` — with a **Test Mode** toggle in the `Config` node for safely previewing results before it writes anything.

## ⚠️ Important: the live verification described in the brief could not be run by me

The brief asked for a pre-build test fetch (currency check + subcategory check) before building the full scraper. I could not perform that myself: this environment's outbound network policy blocks `reklama5.mk` at the proxy level (confirmed 403 on the CONNECT tunnel — an organization-level egress restriction on my sandbox, not something wrong with the site or your account).

To compensate, I:
1. Built **Test Mode** into the workflow itself (see below) so you can run the exact check the brief asked for — from your own n8n instance, which has normal internet access — without needing a second file or writing anything to your sheet.
2. Built both open questions as **defensive, always-on guards**, not one-time assumptions: a currency/price filter that runs on every listing, every run (see "Currency guard" below), regardless of what Test Mode shows you.
3. Because I never saw the live HTML, all field extraction uses **anchor/regex-based text parsing** tied to the semantic landmarks you confirmed (the `<h1>` title, the "Се продава од:" label, the `AdDetails?ad=` URL pattern, image-presence for agency detection) rather than guessed CSS class names, which would have been much more likely to silently break.

## 1. Setup

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

## 2. Run it in Test Mode first

`Config.testMode` is `true` by default. In this mode:
- Pagination stops after search-results page 1 (doesn't walk page 2, 3, ...).
- Only the first 10 new listings are processed.
- **Nothing is written to Google Sheets.** Rows that would have been appended instead flow into `Preview Listing Output` / `Preview Search Error` (plain no-op nodes) so you can inspect them in n8n's execution log.

Click "Test workflow", then open `Preview Listing Output` and check the items that landed there for:

1. **Currency**: is `priceCurrency` `€` for all of them? If any show `МКД`, that confirms the brief's suspicion about `pricefrom=100000` — the currency guard (see below) already keeps these out of `Listings` regardless, but tell me if it's more than a rare edge case.
2. **Fields look right**: title, location, date, seller name, phone, and the Агенција/Приватно лице label — spot-check 2-3 against the live listing pages.
3. **Subcategory coverage**: skim the titles — do they cover different real-estate types (houses, apartments, land, etc.), or all one type? If it's narrow, tell me what other types you'd expect and I'll check whether `cat=157` needs sibling `subcat=` values added to the search URL (Test Mode doesn't check this directly, but 10 real titles is usually enough to tell).

If anything looks wrong, describe the mismatch and I'll adjust the parsing before you turn Test Mode off.

## 3. Go live

Open the `Config` node, set `testMode` to `false`, and run again (or enable the `Schedule Trigger` node — disabled by default, daily at 06:00 — for unattended runs). With Test Mode off, pagination walks all pages up to `Config.maxPages` (default 50), all new listings are processed (not just 10), and rows are actually appended to the sheet.

## How it works

**Pagination loop** (`Fetch Search Page` → `Parse Search Page` → `More Pages?` → `Pagination Wait` → back to `Fetch Search Page`):
- Walks `page=1,2,3...` of the confirmed search URL (`cat=157&sell=1&pricefrom=100000`) until a page returns zero `AdDetails?ad=` links, or `Config.maxPages` is hit (or, in Test Mode, after page 1).
- Only collects ad IDs + absolute URLs on this pass (`https://reklama5.mk/AdDetails?ad={id}`) — every displayed field (title, price, location, date, seller, phone) is read from the **detail page**, not the results card. This was a deliberate simplification: the detail-page field locations are the ones you confirmed via manual inspection; the results-card markup wasn't, so leaning on the confirmed source reduces the chance of silently wrong data.
- 3-4s wait between search-page requests (`Config.waitSeconds`).

**Dedup**: before fetching any detail pages, `Read Existing Listings` reads the `Линк до оглас` column of the whole `Listings` tab, extracts the `ad=` ID from each existing link, and `Filter New Listings` drops anything already present. Only genuinely new ad IDs get fetched (then further capped to 10 if Test Mode is on).

**Detail fetch loop** (`Loop Listings` batch-of-1 over new listings):
- `Fetch Listing Detail` → `Parse Listing Detail` extracts title (`<h1>`), price (first €/МКД-style number after the title), date + location (near a "Денес/Вчера HH:MM" pattern), and the seller block anchored on the "Се продава од" label — image present → Агенција, text-only → Приватно лице. Phone is read from that block first; if not found there, it falls back to scanning the full description text for a Macedonian phone pattern (`07X XXX XXX` / `070/787-127` / `07XXXXXXX`), same as the brief specified.
- Phone is normalized to digits-only alongside the original: e.g. `070/787-127 (070787127)`.
- 3-4s wait between every detail-page request (`Config.waitSeconds`).

**Retry / exponential backoff**: both HTTP fetch points use `onError: continueErrorOutput`, so a failed request (timeout, 5xx) routes to a retry handler instead of stopping the workflow. It retries up to `Config.maxRetries` (default 3) times, waiting `baseRetryWaitSeconds × 2^attempt` between tries (≈3s, 6s, 12s, 24s with defaults). After retries are exhausted, the failure is logged to the `Errors` tab (step, error message, URL, timestamp) — or, in Test Mode, to `Preview Search Error` instead — and the workflow moves on to the next listing/page.

**Currency guard**: `pricefrom=100000` on the search URL should mean everything returned is above threshold in EUR, but per the brief's own concern, some results can show in МКД (which would put the real EUR value under threshold). `Currency & Price Filter` only lets a row through to `Listings` if the parsed currency is `€` **and** the numeric price is ≥ `Config.priceFrom` **and** the URL host is `reklama5.mk`. Anything that fails this — МКД-priced, unparseable price, or (as a defensive belt-and-suspenders check) a non-reklama5.mk URL — is logged to `Errors` with the reason instead of silently dropped. This guard runs regardless of Test Mode.

**Domain safety**: the search URL is hardcoded to `reklama5.mk` (never the `m.` subdomain), all detail URLs are built by prepending `https://reklama5.mk` to the ID pulled from `AdDetails?ad=`, and the currency guard double-checks the final URL host before writing a row.

## Tuning (`Config` node)

| Field | Default | Meaning |
|---|---|---|
| `testMode` | `true` | Page-1-only, 10-listing cap, no Sheets writes — see "Run it in Test Mode first" |
| `priceFrom` | 100000 | Minimum EUR price kept (also baked into the search URL) |
| `waitSeconds` | 4 | Wait between every outbound request (rate limit) |
| `baseRetryWaitSeconds` | 3 | Base for exponential backoff (× 2^attempt) |
| `maxRetries` | 3 | Max retry attempts per failed request |
| `maxPages` | 50 | Safety cap on pagination depth (ignored in Test Mode) |

## Columns written to `Listings`

`Наслов на оглас, Цена, Локација, Датум на објавување, Име на продавач/агенција, Телефон, Приватно/Агенција, Линк до оглас, Датум на извлекување` — in that order, matching the spec exactly. `Линк до оглас` is always the full absolute URL.

## Respectful-scraping notes

- Only `reklama5.mk` is ever fetched — the workflow never touches `m.reklama5.mk` (which disallows automated access via robots.txt).
- No login/session/cookies — contact info is confirmed visible while logged out.
- 3-4s between every single request, both for pagination and for detail pages; failed requests back off exponentially rather than hammering a slow server.
- Nothing here bypasses any access control — it reads what's already publicly rendered on the page.
- Re-check the Terms of Service periodically; the brief noted no explicit automated-collection clause as of this build, but ToS pages change.

## Known limitations / what to watch on first real runs

- **Selectors are text-anchor/regex based, not exact CSS selectors** — because I never saw the live HTML. This is deliberately more resilient to class-name churn, but if Reklama5 changes wording (e.g. renames "Се продава од"), extraction breaks silently for that field until updated. Skim the first day or two of output even after Test Mode looks clean.
- **Subcategory coverage isn't auto-checked** — Test Mode gives you 10 real titles to skim, but doesn't systematically verify `cat=157` covers every real-estate subtype. If you spot a gap, tell me and I'll extend the search loop with `subcat=` values.
- **The exact JSON shape of the HTTP node's error-branch item** (used by the retry handlers) is based on my best knowledge of current n8n behavior, not a live test. If the `Errors` tab (or `Preview Search Error` in Test Mode) ever shows a blank/wrong `Линк до оглас` on a retry-exhausted row, open `Fetch Listing Detail`'s error output in n8n, check what fields are actually present on that item, and tell me — it's a one-line fix in `Listing Retry Handler` / `Search Page Retry Handler`.
- Google Sheets node internals (operation names, resource-locator format) are based on current n8n source, not a live import test in your instance — the "First-import checklist" above covers the handful of things worth a 30-second glance.
