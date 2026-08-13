# Where things stand

Last updated: 2026-08-11, end of session.

## Working

The workflow runs end to end. A 5-page run completed successfully and wrote roughly 200 rows to the `Listings` tab with all fields populated — title, price, location, date, seller name, phone, and the Приватно/Агенција label.

Fixed and verified along the way:
- Parser rewritten against the real markup (no `<h1>`, English UI, CSS-class anchors) — see README
- Promoted/sponsored ads filtered on the results page before they cost a fetch
- Infinite pagination loop (page counter destroyed by the HTTP node) — page and retry state now live in workflow static data
- Price parsing for both `495.000 €` and `116,000 €` formats
- Phone fallback no longer matches embedded Google Maps coordinates

## Dedup — resolved 2026-08-12

Duplicate rows on the second run were caused by `Filter New Listings` looking up the exact header `Линк до оглас`; any drift in that header produced an empty "already seen" set, so every listing looked new. It now scans every cell of each row for an `ad=` pattern instead, so the column name no longer matters.

Confirmed working against the live sheet: 201 rows read, 201 ad IDs extracted, and the listings it flagged as new were verified absent from the sheet. The node reports `_existingRowsRead`, `_existingIdsFound`, `_collectedThisRun` and `_newAfterDedup` on its first output item if it ever needs re-checking — note the "new" count is measured after Test Mode's 10-item cap, so read it alongside the others.

Duplicates already in the sheet were removed manually, leaving 201 unique rows. A subsequent `maxPages = 10` run then took the sheet to 405 rows / 405 unique — the existing 201 untouched, 204 new appended, no duplicates. Dedup is verified across runs.

## Also outstanding

- **Agency detection's uploaded-logo branch is unverified.** The name-keyword path is confirmed working. An agency whose name contains no agency word would depend on the logo branch, which has never been seen firing against real markup. Check `sellerTypeSignal` on any row that looks miscategorised.
- **Subcategory coverage unconfirmed.** Whether `cat=157` alone returns every real-estate subtype was never verified.
- **`Schedule Trigger`** is still disabled. Enable it once the ramp is complete.

## Ramp progress

- `maxPages = 5` → 201 rows
- `maxPages = 10` → 405 rows
- `maxPages = 20` → completed
- `maxPages = 30` → aborted partway on a Google Sheets quota error; retries added, safe to re-run
- Continue raising until new rows stop appearing, then enable `Schedule Trigger`.

## Google Sheets rate limiting

A `maxPages = 30` run hit Google's Sheets API write quota (60 writes per minute per user) on `Append Listing Row`, which aborted the run. The workflow writes about 10 rows a minute, well inside quota, so this was either contention with something else using the same Google project or a throttled burst — transient either way.

All five Google Sheets nodes now have **Retry On Fail** (5 tries, 5s apart), and the four write nodes use **On Error = Continue**. A row that still fails after five tries is not lost: it simply is not in the sheet, so the next run's dedup treats it as new and re-fetches it.

If this recurs often, raise `Config.waitSeconds` from 4 to 6 — that slows the whole run but halves the write rate.

## Settings that break silently if lost on re-import

- `Fetch Search Page` and `Fetch Listing Detail` → On Error → *Continue (using error output)*, plus the error output wired to the matching retry handler
- `Read Existing Listings` → Always Output Data → on
- All five Google Sheets nodes → Retry On Fail on, Max Tries 5, Wait Between Tries 5000ms
- `Append Listing Row`, `Append Excluded Row`, `Log Search Page Error`, `Log Listing Error` → On Error → *Continue* (plain), so a Sheets failure cannot abort the whole run
