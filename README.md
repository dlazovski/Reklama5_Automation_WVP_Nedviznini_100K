# Reklama5.mk — Real Estate Contact Extraction (n8n)

Automation that walks Reklama5.mk's real estate listings (category 157, for-sale, ≥100,000 €), pulls seller/agency contact info off each listing page, and appends new rows to a Google Sheet. Built for Moduvo.

One workflow — `workflows/reklama5-scraper.json` — with a **Test Mode** toggle in the `Config` node for previewing results before anything is written.

## What the live pages actually look like

The original brief described the listing page as it appears in a logged-out browser: an `<h1>` title, a `Се продава од:` sidebar, and an image in that sidebar meaning "agency". Running the workflow against the real site showed all three of those assumptions do not hold for an automated client, so the parser is built against what the server actually returns:

- **The site serves the English UI to n8n**, even with an `Accept-Language: mk` header. `Се продава од` does not appear anywhere in the fetched HTML — the labels come back as "Sold by"/"Contact" style English text. Whatever selects Macedonian in your browser (a cookie set on first visit, or geo/IP defaulting) is not something the HTTP request reproduces.
- **There is no `<h1>` on the page at all.** The title lives in `<h5 class="card-title">` and the price in `<h5 class="mb-0 defaultBlue">`.
- **Every advertiser block contains an image** — a static `/Content/images/cert.png` "Verified Advertiser" badge — so "image present = agency" would have labelled every seller as an agency.

Because of this, every anchor in the parser is a **CSS class or icon class, not label text**, so it works regardless of which language the site decides to serve. Field-by-field:

| Field | Anchor |
|---|---|
| Title | `h5.card-title` (falls back to `<title>`) |
| Price | `h5.defaultBlue` |
| Date + location | `<span><small>DATE</small></span><br /><p>LOCATION</p>`, searched from the title onward |
| Seller name | `h5.my-0` |
| Phone | the `.fa-phone-alt` icon in the sidebar |
| Agency vs private | uploaded logo (a non-`/Content/images/` image) **or** an agency keyword in the name |

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

4. **Check these two node settings after import** (both are already correct in the JSON, but n8n's import can be finicky and both cause silent, confusing failures if wrong):
   - `Fetch Search Page` and `Fetch Listing Detail` → **Settings** tab → **On Error** must be *"Continue (using error output)"*. This is what routes failures into the retry loop instead of killing the run on the first timeout.
   - `Read Existing Listings` → **Settings** tab → **Always Output Data** must be **on**. n8n skips any node that receives zero input items, so on a fresh sheet with no data rows this node returns nothing and the entire pipeline silently stops right there.

## 2. Run it in Test Mode first

`Config.testMode` is `true` by default. In this mode:
- Pagination stops after search-results page 1.
- Only the first 10 new listings are processed.
- **Nothing is written to Google Sheets.** Rows that would have been appended instead flow into `Preview Listing Output` / `Preview Search Error` (plain no-op nodes) so you can inspect them in n8n's execution log.

Click "Test workflow", then open `Preview Listing Output` and check the items for:

1. **All fields populated** — title, price, location, date, seller name, phone. An empty column across every row means that field's anchor has drifted and needs updating.
2. **`Приватно/Агенција` looks right** — spot-check against the live pages. The parser also emits a `sellerTypeSignal` field (`name-keyword` / `uploaded-logo` / `no-logo-no-keyword`) so you can see *why* it decided what it did.
3. **Currency** — every `Цена` should end in `€`. Anything in МКД is filtered out before it reaches the sheet, but it's worth knowing if it's common.

## 3. Go live

Open the `Config` node, set `testMode` to `false`, and run. Rows now append to the sheet, and pagination walks up to `maxPages`.

**Mind the runtime.** At ~6 seconds per listing (4s rate-limit wait plus fetch time) and roughly 40+ ad links per results page, a 50-page run is a 3+ hour single execution — fragile against browser disconnects and execution timeouts. Ramp up instead:

| Run | `maxPages` | Result |
|---|---|---|
| 1 | 5 | ~200 listings, ~20 min |
| 2 | 10 | Re-scans 1-10, skips what's stored, adds pages 6-10 |
| 3 | 20 | Adds pages 11-20 |

Every run starts at page 1, so **re-running with the same `maxPages` adds nothing new** — you have to raise it. Re-scanning is cheap though: dedup happens *before* the detail fetches, so re-covering old pages costs only the search-page requests, not 6 seconds per already-stored listing.

Once you're at full depth, enable the `Schedule Trigger` node (disabled by default, daily 06:00). Ongoing runs mostly dedup away and pick up new arrivals, which is the steady state you want.

## How it works

**Pagination loop** (`Fetch Search Page` → `Parse Search Page` → `More Pages?` → `Pagination Wait` → back):
- Walks `page=1,2,3...` of the search URL (`cat=157&sell=1&pricefrom=100000`) until a page returns zero result cards, or `maxPages` is hit (or, in Test Mode, after page 1).
- Reads each result card's own price and keeps only those matching the filter (see the promoted-ads note under Known limitations). Cards with no visible price are kept for the detail page to judge.
- Collects only ad IDs and absolute URLs here — every displayed field is read from the detail page, since that markup is the part we've verified.

**Dedup**: `Read Existing Listings` reads the `Линк до оглас` column, extracts the `ad=` ID from each stored link, and `Filter New Listings` drops anything already present. Only new ad IDs reach the fetch loop.

**Detail fetch loop** (`Loop Listings`, batch size 1): fetch → parse (see anchor table above) → currency/price guard → sheet. 4s wait between every request.

Two parsing details worth knowing:
- **Relative dates are resolved.** The site shows "Today 13:00"; the sheet stores `11.08.2026 13:00`, since "Today" is meaningless in a row you read next week. The raw string is kept in `datePostedRaw`.
- **The phone fallback strips URLs first.** The page embeds Google Maps coordinates (`center=42.0069480002806,...`) that match a naive phone pattern — an early version pulled `0069480002` as a phone number. The fallback now strips URLs and only accepts 07X mobile numbers. The primary source remains the sidebar icon anchor.

**Retry / exponential backoff**: both HTTP nodes use `onError: continueErrorOutput`, so a timeout or 5xx routes to a retry handler rather than stopping the workflow. Retries up to `maxRetries` (default 3), waiting `baseRetryWaitSeconds × 2^attempt` (≈3s, 6s, 12s, 24s). After that the failure is logged to `Errors` and the run moves on.

**Currency guard**: `Currency & Price Filter` only passes a row if currency is `€`, the numeric price is ≥ `priceFrom`, and the host is `reklama5.mk`. Failures are logged to `Errors` with the reason rather than silently dropped. Note the price parser handles both `495.000 €` and `116,000 €` formats by checking the length of the last separator group — an earlier version read "180,000 €" as 180.

**Domain safety**: the search URL is hardcoded to `reklama5.mk` (never the `m.` subdomain), detail URLs are rebuilt from the ad ID, and the guard re-checks the host before any write.

## Tuning (`Config` node)

| Field | Default | Meaning |
|---|---|---|
| `testMode` | `true` | Page-1-only, 10-listing cap, no Sheets writes |
| `priceFrom` | 100000 | Minimum EUR price kept |
| `waitSeconds` | 4 | Wait between every outbound request |
| `baseRetryWaitSeconds` | 3 | Base for exponential backoff (× 2^attempt) |
| `maxRetries` | 3 | Max retry attempts per failed request |
| `maxPages` | 50 | Pagination depth cap (ignored in Test Mode) |

## Columns written to `Listings`

`Наслов на оглас, Цена, Локација, Датум на објавување, Име на продавач/агенција, Телефон, Приватно/Агенција, Линк до оглас, Датум на извлекување` — in that order. `Линк до оглас` is always the full absolute URL.

## Respectful-scraping notes

- Only `reklama5.mk` is fetched — never `m.reklama5.mk`, which disallows automated access via robots.txt.
- No login, session, or cookies — contact info is public on the page.
- 4s between every request; failures back off exponentially rather than hammering a slow server.
- Nothing bypasses access control; it reads what's already publicly rendered.
- Re-check the Terms of Service periodically. No explicit automated-collection clause was found when this was built, but ToS pages change.

## State that lives in workflow static data

The HTTP Request node **replaces the incoming item with the response body**. Any counter passed along as an item field is therefore destroyed by every fetch, which is not obvious until a loop refuses to end. Three things are kept in `$getWorkflowStaticData('global')` for exactly this reason:

| Key | Purpose |
|---|---|
| `listings` | Ad IDs collected across all scanned pages |
| `currentPage` / `searchAttempt` | Pagination position and per-page retry count |
| `listingAttempt` / `retryAdId` | Per-listing retry count, reset when the loop moves to a new ad |

`Init Pagination` resets all of them at the start of every run. If you edit these nodes, do not switch them back to reading `page` or `attempt` off the incoming item: `page` becomes `undefined`, `1 + 1` is recomputed forever, and the workflow paginates the same page indefinitely without ever reaching the detail loop. The same applies to `attempt` — it would stay at 1 and retry a failing request forever.

## Known limitations

- **Promoted ads are filtered on the results page, not after fetching.** The results page carries a sponsored strip at the top whose cards use identical markup to real results (`div.ad-desc-div`, `a.SearchAdTitle`) but ignore the search filters — page 1 served a 350 € rental and a 480 € commercial rental despite `sell=1&pricefrom=100000`. Since no container class separates them, `Parse Search Page` reads each card's own price and drops non-matching ones before they cost a fetch. Cards showing no price are kept and left for the detail page to judge. Watch `skippedOnPage` / `skippedSamples` in that node's output if the numbers look off.
- **Agency detection is confirmed only via the name keyword.** A listing named "Прима Каза - Агенција за недвижности" is correctly labelled Агенција. The uploaded-logo branch (for agencies whose name lacks an agency word) has not been verified against a known example — check `sellerTypeSignal` if a row looks miscategorised.
- **Parsing is anchored on CSS classes**, which are stable against language changes but not against a site redesign. If a column goes uniformly empty, that anchor moved.
- **Subcategory coverage is unverified.** Whether `cat=157` alone returns every real-estate subtype was never confirmed; the titles seen so far include apartments, business premises, and land, which suggests broad coverage but is not proof.
