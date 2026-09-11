# Internet Archive Wayback Machine: the CDX server

Endpoint: `https://web.archive.org/cdx/search/cdx` (no key). Official description: `github.com/internetarchive/wayback/tree/master/wayback-cdx-server`.

Measured on **2026-09-10**, except trap 10's per-site timings (**2026-09-11**).

```
curl 'https://web.archive.org/cdx/search/cdx?url=example.com&output=json&limit=100&filter=statuscode:2..'
```

## 1. "Never archived" and "your query is broken" are the same answer

- **Sent:** a domain with no captures, and separately a valid domain with `from=abcd`.
- **Got:** `[]`, HTTP 200, for both. A range where `from` is later than `to` also returns nothing, with no error.
- **Why it hurts:** a typo in a date reads as "this site was never archived".
- **Avoid:** validate before sending. Accept `YYYY`, `YYYY-MM`, `YYYY-MM-DD` or an even-length prefix of `yyyyMMddhhmmss`, and check that `from <= to`.

## 2. A malformed URL is quietly read as a different URL

- **Sent:** `url=http://[bad`
- **Got:** 3 captures of a URL called `bad`.
- **Why it hurts:** you get plausible rows for something you did not ask about.
- **Avoid:** parse hostnames strictly and do not send what does not parse. The same class of bug exists on the client: `new URL('http://example.com:8080/').hostname` drops the port, so a lookup built from `hostname` returns captures of port 80, which is a different page. Reject non-standard ports rather than dropping them.

## 3. No limit means everything

- **Sent:** one page (`example.com`) with no `limit`.
- **Got:** 768,693 rows in 60 seconds.
- **Avoid:** always send `limit`.

## 4. About 13% of rows have no status code

- **Got:** status `"-"` with mimetype `warc/revisit` ("same content as before") on 377 of 3,000 rows under `apify.com`. Mimetype also appears as both `unk` and `unknown`.
- **Why it hurts:** `parseInt("-")` is `NaN`; `Number(x) || 0` invents a status of 0.
- **Avoid:** keep the status `null` and label the row as a revisit.

## 5. Asking the server to group a domain listing times out

- **Sent:** a domain-wide listing with `collapse=urlkey`.
- **Got:** 504 (the server gives up at 60 seconds), even for 20 rows (2026-09-10).
- `showNumPages` answered 403 (authorisation required) on 2026-09-10. On 2026-09-11 it answered 200 with a page count (`apify.com` as a domain: 101; `example.com`: 128). Handle both answers; do not assume either is permanent.
- **Avoid:** request raw captures with only the fields you need and group locally. Rows come in `urlkey` (SURT) order and in time order within a key, so a URL is finished as soon as the next key appears.

## 6. The resume key drops the row on the boundary

- **Sent:** `showResumeKey=true`, `limit=1`, on `apify.com/`, then the resume key.
- **Got:** at `20190711224257` there are two captures (a 200 and a 301). The first page returned one; the resumed page returned `[]`, so the 301 was lost.
- **With `collapse`:** paging moved the representative of a period, from 2011-02-10 to 2011-07-07 and from 2013-05-18 to 2013-05-25.
- **Avoid:** do not page snapshot queries; ask once with a `limit` large enough for the answer (we cap our own requests at 10,000; that cap is ours, not the API's). For URL listings, re-query the boundary second at each page break and add what the resume skipped.

## 7. Filter status on the server, not after collapsing

- **Sent:** monthly collapse (`collapse=timestamp:6`) on `apify.com`, 24 months, then removed non-200 rows locally.
- **Got:** 5 of the 24 monthly representatives were 301s, so those 5 months disappeared.
- **Server-side instead:** `filter=statuscode:2..` returned a 200 for all 24 months (18 seconds). `3..` and `[45]..` also work, but heavy pages produced more 503/504.

## 8. Newest-first "only when it changed" returns the end of a run

- **Sent:** a negative `limit` (newest first) with `collapse=digest` on `example.com`.
- **Got:** the content changed at the capture ending `021413`; the API returned the capture ending `094036`, the last of 17 identical captures.
- **Note:** `limit=-N` means "the last N captures", still returned oldest first.
- **Avoid:** for newest-first change detection, fetch the window without collapsing and pick the change points locally.

## 9. Some sites' URL listings are refused permanently

- **Sent:** a URL listing for `nytimes.com` by domain, host and prefix, with and without date ranges.
- **Got:** HTTP 403 `This type of CDX query requires authorization`, answered in about 0.1 seconds. The same query shape for `apify.com`, `python.org` and `wikipedia.org` returned 200, and a single-page snapshot of `nytimes.com` also worked.
- **Avoid:** treat this 403 as final. Do not retry it, and do not report it as a temporary failure.

## 10. 503 and 504 are routine, and some sites never finish

- Response times ranged from 0.4 to 60 seconds. In one window, 9 of 12 requests came back 503 or 504.
- `google.com`, `bbc.com` and `facebook.com` returned 504 repeatedly under default settings. One input on 2026-09-11 went 504, 504, 504, 503, 504, using 285 seconds of retries.
- **Avoid:** retry 503 later. For 504, narrow the date range or drop the collapse. Budget retries against your job's remaining time, and record which inputs were skipped when time runs out.

## Smaller things

- Filtering on a field that does not exist returns HTTP 400 with an empty body.
- Asking for `https://www.example.com/` returns captures stored as `http://example.com:80/`, because the server normalises scheme and `www`.

---

Actor that handles these traps: https://apify.com/neverempty/wayback-machine-scraper
