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

## Open — start here tomorrow

**Duplicate rows appeared on the second run.** The run was stopped partway. Dedup should have prevented this: `Read Existing Listings` reads the sheet, `Filter New Listings` drops ad IDs already stored.

`Filter New Listings` has already been rewritten (and is committed here) to fix the most likely cause — it previously looked up the exact header `Линк до оглас`, so any drift in that header name would silently produce an empty "already seen" set and every listing would look new. The new version scans every cell of every row for an `ad=` pattern instead, so the column name no longer matters.

**This fix is committed but has not been tested yet.**

### To pick up

1. Paste the current `Filter New Listings` code from `workflows/reklama5-scraper.json` into that node (or re-import the workflow if you would rather redo the Sheets config).
2. Run with `Config.testMode = true` — fast, writes nothing.
3. Open `Filter New Listings` and read the diagnostic counts on the first output item:
   - `_existingRowsRead` — rows returned from the sheet
   - `_existingIdsFound` — ad IDs extracted from them
   - `_collectedThisRun` — ad IDs found on the scanned pages
   - `_newAfterDedup` — what survived the filter

   `_existingRowsRead` around 200 with `_existingIdsFound` at 0 means the column-name problem, and this fix resolves it. `_existingRowsRead` at 0 means the Sheets read itself is returning nothing, which is a different problem in that node's configuration.
4. Clean the duplicates already in the sheet: Google Sheets → Data → Data cleanup → Remove duplicates, keyed on `Линк до оглас`.
5. Once dedup is confirmed, resume the ramp: `testMode = false`, `maxPages` to 10, then 20, then 40.

## Also outstanding

- **Error-logging nodes can abort a run.** `Append Excluded Row`, `Log Search Page Error` and `Log Listing Error` stop the whole workflow if they fail, so a problem writing to the `Errors` tab takes down the scrape. Setting their On Error to "Continue" would make runs survive it. Not yet done.
- **Agency detection's uploaded-logo branch is unverified.** The name-keyword path is confirmed working. An agency whose name contains no agency word would depend on the logo branch, which has never been seen firing against real markup. Check `sellerTypeSignal` on any row that looks miscategorised.
- **Subcategory coverage unconfirmed.** Whether `cat=157` alone returns every real-estate subtype was never verified.
- **`Schedule Trigger`** is still disabled. Enable it once the ramp is complete.

## Settings that break silently if lost on re-import

- `Fetch Search Page` and `Fetch Listing Detail` → On Error → *Continue (using error output)*, plus the error output wired to the matching retry handler
- `Read Existing Listings` → Always Output Data → on
