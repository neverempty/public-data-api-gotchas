# SEC EDGAR full-text search index: Form D

Endpoint: `https://efts.sec.gov/LATEST/search-index` (the index behind EDGAR full-text search). No key; public domain. Each hit links to the filing's `primary_doc.xml`.

Measured on **2026-09-11**.

```
curl -H 'User-Agent: YourApp/1.0 (you@example.com)' \
  'https://efts.sec.gov/LATEST/search-index?forms=D&startdt=2026-09-01&enddt=2026-09-10&from=100'
```

## 1. Deep paging stops at 10,000

- 100 hits per page, paged with `from=`. `from + size` may not exceed 10,000; `from=10000` returns `Result window is too large`.
- Form D filings for January to September 2026 exceed 10,000 (`relation: "gte"`), so paging a wide range silently stops at 10,000.
- **Avoid:** split the date range in half until each window holds fewer than 10,000. Over one range the windows added up to 44,314 filings, which matched the reported total.

## 2. `forms=D` returns amendments too

- 2026-09-01 to 09-10, first count: 1,647 filings, 980 `D` and 667 `D/A`. All accession numbers were different; a D and its D/A share a file number (`021-...`).
- The same range counted again at 00:55 UTC on 2026-09-11 gave 1,854. Late filings keep landing in past windows.

## 3. The CDN does not key on `items=`

- **Sent:** the same query with `items=06C` (Rule 506(c)).
- **Got:** 204 hits the first time. Sent again, 1,441, which was the answer to an earlier `06B` request. `items=ZZZ` returned 417, the unfiltered count.
- **Why it hurts:** the filter you sent is not the filter you get; you get whatever someone else asked for first.
- **Avoid:** do not send `items=`. Read each hit's items locally (this matched the filing XML in 520 of 520 cases).

## 4. Page one can be a stale cached copy

- **Got:** the first page (no `from=`) was served by the CDN from an older version and lacked the newest 7 filings. In one run, 7 of 13 were missing.
- The cache key includes `forms`, `startdt`, `enddt`, `from`, `q` and `locationType`. Arbitrary parameters (`zz=`, `_=`, `size=`, `page=`) are not in the key, so they do not bust it.
- **Avoid:** send a unique `locationType` value on every request (it is ignored when `locationCodes` is absent). With that change, a run matched a direct count of 326 exactly.
- **Caveat:** this depends on EDGAR continuing to ignore unknown `locationType` values.

## 5. `locationCodes` is the registered state, not the issuer's state

- It differed from the issuer address in the filing for 14 of 409 filings (3.4%), and it was empty for 95 of the 1,647 filings from 2026-09-01 to 09-10 (5.8%; foreign issuers whose filings say E9, N4, X0...). Filtering by state drops every empty one.
- The parameter is case-sensitive (`ca` returns 0), and an unknown code (`ZZ`) returns 200 with 0 hits.
- **Avoid:** filter on the issuer address inside the filing.

## 6. `entityName=` does not match partial names, and its behaviour changed within a day

- Earlier on 2026-09-11, `Skin`, `skinaware` and two-word names all returned 500.
- Re-checked at 19:36 JST the same day: `entityName=Skin` returned 200 with 0 hits, and lowercase `entityName=skinaware` returned 200 with 1 hit.
- **Why it hurts:** 0 hits for part of a name looks like "no filings by that issuer".
- **Avoid:** match on `display_names` in the hits instead.

## 7. `ciks=` needs 10-digit zero padding

- `ciks=2154546` returned 0 hits; `ciks=0002154546` returned 1.

## 8. "Indefinite" is not zero

- 256 of 429 filings (60%, mostly funds) state the offering amount as `Indefinite`.
- **Avoid:** return `null` with a flag. Never coerce it to 0.

## 9. The first sale date may not exist yet

- 46 of 429 filings carry `<yetToOccur>` instead of a date.

## 10. An impossible date is accepted at one end and fails quietly at the other

Re-checked on 2026-09-11 at 19:36 JST (earlier that day, `2026-09-31` returned 500):

- `enddt=2026-09-31` returns 200 with 1,901 hits, the same count as `enddt=2026-09-30`. The bad date is accepted without a word.
- `startdt=2026-09-31` returns HTTP 200 with an error in the body: `date_time_exception: Invalid date 'SEPTEMBER 31'`.
- **Why it hurts:** the first case silently answers a different question; a client that checks only the status reads the second case as "no filings".
- **Avoid:** validate dates before sending, and treat a body carrying `errorType` as a failure even under 200. A 500 also turns up on ordinary requests (a third page returned 500, then 200 on retry), so retry it.

## 11. Electronic Form D starts in December 2008

- 2008: 442 filings. 2005: 0.

## Personal data in the filing

Related persons (officers, directors, promoters) are listed with full street addresses, which can be home addresses. In 611 filings that were not pooled investment funds, the issuer's street address matched a related person's address in 440 (72%), and the issuer phone on small companies can be a personal mobile. Decide what to publish before you store it.

The SEC publishes a limit of 10 requests per second and asks for a `User-Agent` with contact details.
