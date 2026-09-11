# SEC EDGAR submissions API

Endpoints: `https://data.sec.gov/submissions/CIK##########.json` and `https://www.sec.gov/files/company_tickers.json`. No key; public domain.

Measured on **2026-09-10**.

```
curl -H 'User-Agent: YourApp/1.0 (you@example.com)' \
  https://data.sec.gov/submissions/CIK0000320193.json
```

## 1. `filings.recent` is not the full history

- **Got:** for Apple, `recent` held 1,000 filings, and another 1,246 sat in the separate files listed in `filings.files[]`, 2,246 in total. The oldest filing in `recent` was dated 2015-07-22.
- **Why it hurts:** reading only `recent` drops 55.5% of Apple's history, and nothing in the response says so.
- **Avoid:** fetch every file in `filings.files[]` and append it.

## 2. `recent` is not always exactly 1,000

- Apple: 1,000. Cheniere: **1,007**.
- **Avoid:** do not hard-code 1,000 as a sign of truncation.

## 3. Tickers are not companies

- The official list had 10,407 tickers for **8,013** companies, and 1,441 companies had more than one ticker (share classes, e.g. `UHAL` and `UHAL-B`).
- **Why it hurts:** counting tickers as companies overstates the count by about 30%.

## 4. Class separators are hyphens

- 544 tickers use a hyphen as the class separator; one uses a dot.
- **Why it hurts:** people type `BRK.A`, and a lookup that does not also try `BRK-A` reports "no such ticker".

## 5. CIKs must be zero-padded to 10 digits

- `CIK320193.json` returns 404. `CIK0000320193.json` works.
- Note that the archive folder path (`/Archives/edgar/data/<cik>/<accession without hyphens>/`) takes the CIK **without** padding.

## 6. A missing CIK is a 404 with an XML body

- **Got:** HTTP 404, with an XML body rather than JSON.
- **Why it hurts:** a client that parses every response as JSON crashes instead of reporting "not found".

## 7. Most filings are Form 4

- Of Apple's 1,000 recent filings, 589 were Form 4 (insider transactions) and 34 were 10-Q.
- **Avoid:** filter by form type before doing anything else.

## 8. One company can arrive twice

- `AAPL` and `320193` resolve to the same CIK. If input can contain both, dedupe on CIK or the same filings are delivered twice.

## Access policy

The SEC asks for a `User-Agent` with contact details and publishes a limit of 10 requests per second. Ten requests fired within 316 ms all returned 200, but stay under the published limit anyway.

---

Actor that handles these traps: https://apify.com/neverempty/sec-edgar-filings
