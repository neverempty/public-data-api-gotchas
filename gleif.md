# GLEIF LEI API

Endpoint: `https://api.gleif.org/api/v1` (no key; data under CC0).

Measured on **2026-09-11**; the renewal count is from **2026-09-10**.

```
curl -G 'https://api.gleif.org/api/v1/lei-records' \
  --data-urlencode 'filter[entity.legalAddress.country]=MC' \
  --data-urlencode 'page[size]=200' \
  --data-urlencode 'page[cursor]=*'
```

## 1. Page size tops out at 200

- `page[size]=201` returns 400 `must be between 1 and 200`.

## 2. Numbered paging stops at 10,000; use the cursor

- `page[number] * page[size] > 10000` returns 400.
- `page[cursor]=*` walks everything. Monaco (969) and Gibraltar (2,506) were read in full with 0 duplicates, matching `meta.total`.
- The last cursor page still carries `links.next`, and following it returns an empty page. Stop when you reach the total.

## 3. Name filters match whole words

- **Sent:** `filter[entity.legalName]=insur`
- **Got:** 0. Yet 63 Gibraltar legal names contain "insur".
- All the words have to be present, in any order; case and accents are ignored. `entity.legalName` searches the legal name only, while `entity.names` also covers other and transliterated names. Checked against full local counts, 16 of 17 words matched exactly.
- `fulltext` also matches addresses: `gibraltar` returned 2,416.
- **Avoid:** say "whole-word match" in your empty results, or users will read 0 as "no such company".

## 4. A date range ends at midnight of the end date

- **Sent:** `2026-09-07..2026-09-08`
- **Got:** 1,289. The correct count is 2,455, because the end date is read as 00:00.
- **Avoid:** send `a..bT23:59:59Z`. That matched per-day totals (09-08..09-08 gave 1,200). `>=a` and `<=b` are correct at day level.
- A reversed range returns 500; an impossible date returns 400.

## 5. Unknown country codes return 200 and nothing

- `ZZ` returns 200 with 0 records, not 400.
- Enumerated values (`registration.status` and similar) must be uppercase; lowercase returns 400.
- **Avoid:** validate country codes against GLEIF's own list before sending.

## 6. LEIs that do not exist are dropped silently

- `filter[lei]` takes 200 LEIs per call (a 4,275-character URL was fine). Unknown LEIs are left out of the response without an error.
- **Avoid:** diff requested against returned, and report the missing ones yourself. Validate the ISO 17442 mod-97 check digits first, so a typo is not reported as "not registered".

## 7. Parents come without names, and sometimes not at all

- `include=direct-parent,ultimate-parent` returns the parents' LEIs and any reporting exception in the same call, but not their names. The included records carry only `type` and `id`, so fetch names with a second `filter[lei]` call.
- Some RETIRED records have no parent fields at all (3 of 100). That means "not reported", not "no parent".

## 8. The direct-children list ignores filters

- `/lei-records/{lei}/direct-children` for Siemens AG returned 461 whether or not `country=DE` was added.
- `filter[ownedBy]` (ultimate parent) answers only at the top of a group: Siemens AG 573, Siemens Healthineers AG 0. Healthineers' direct children number 87.
- **Avoid:** do not combine direct-children with other filters and expect them to apply.

## 9. No rate limit showed up

- 90 sequential requests in 37 seconds and 200 requests at 10-way parallelism in 9 seconds all returned 200, with no rate-limit headers. Still back off on 429 and 5xx, and honour `Retry-After`.

## 10. "Issued" does not mean current

- 716 LEIs had status ISSUED but were past their next renewal date (2026-09-10).
- **Avoid:** compare `nextRenewalDate` with today rather than trusting the status column.
