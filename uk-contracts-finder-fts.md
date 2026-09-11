# UK Contracts Finder and Find a Tender (OCDS)

Endpoints (no key; OGL v3.0):

- Contracts Finder (CF): `https://www.contractsfinder.service.gov.uk/Published/Notices/OCDS/Search`
- Find a Tender (FTS): `https://www.find-tender.service.gov.uk/api/1.0/ocdsReleasePackages`

Measured on **2026-09-11**.

```
curl 'https://www.find-tender.service.gov.uk/api/1.0/ocdsReleasePackages?updatedFrom=2026-09-09T00:00:00&updatedTo=2026-09-09T23:59:59&limit=100'
```

## 1. 100 per page; follow `links.next`

- Both services cap `limit` at 100 (101 returns 400), and both page with a cursor in `links.next`.
- One day (2026-09-09): CF 136 releases over 2 pages; FTS 444 over 5.

## 2. CF stops at 100 without a next link

- When a CF window holds more than 100 releases, it can return exactly 100 and no `links.next`, so the remainder is dropped silently.
- **Avoid:** if a page comes back full with no next link, halve the time window and re-read. For 2026-09-10, four 6-hour windows gave 108, the correct count.

## 3. The rate limit: about 12 requests per 2 minutes

- **Got:** HTTP 429 with `Rate limit of 12 exceeded. Please retry after 120 seconds.` after about 20 quick requests to either service.
- Gaps of 5.5 seconds still hit it. At 6.5 seconds, 7-day runs hit it once or twice, first at the 16th to 20th request.
- **Avoid:** space requests about 10.5 seconds apart across both services, and on 429 wait the number of seconds the body gives. CF's documentation also describes a 403 with a 5-minute block.

## 4. CF ignores `keyword=`; FTS rejects it

- CF `keyword=software` returned the same 100 rows in the same order as no keyword; only 13 of them mention software.
- FTS `keyword=` returns 400.
- **Avoid:** filter keywords locally.

## 5. FTS `stages=` returns only part of the set

| stage | returned with `stages=` | counted locally |
|---|---|---|
| tender | 11 | 95 |
| award | 45 | 283 |
| planning | 3 | 55 |

- CF's `stages=` matched local counts (16/16, 119/119).
- **Avoid:** do not send `stages=` to FTS; filter locally.

## 6. Date filters run in UK time

- CF `publishedFrom/To` and FTS `updatedFrom/To` are read as UK local time. A release in the middle of a window sent in UTC was missed by both services.

## 7. The two services share no identifiers

- CF OCIDs start `ocds-b5fd17-` and FTS OCIDs start `ocds-h6vhtk-`. Across one full day there were 0 shared OCIDs and 0 cross-references.
- The same notice can be linked only by title + buyer (3 of CF's 128 that day). Within one service, different OCIDs can share title + buyer (route lots, 2 to 5 at a time).
- Titles are not written identically on both sides (a leading procurement number such as `CA<digits>`, a trailing ` - AWARD`, buyer names spelled differently). A strict title + buyer match missed 20 real pairs.
- **Avoid:** merge only one-to-one title + buyer matches, normalise titles first, and say in the output how the two services were joined.

## 8. One OCID, several releases

- Tender, correction and award arrive as separate releases under one OCID (CF 8, FTS 23 that day). Award notices often lack the value or deadline.
- **Avoid:** compile per OCID with the OCDS merge rules: objects merge deeply, arrays with `id` merge by `id`, later releases win.

## 9. Values, deadlines and regions

- Placeholder values of 1 and 0 occur (3 that day). The currency was GBP only. FTS filled value on 26% of rows and deadline on 18%.
- CF gives region names (`London`); FTS gives NUTS codes (`UKI`, `UKI32`).
- Notice URLs can be built: CF uses `/Notice/<release id without the trailing -digits>` (136/136); FTS uses `/Notice/<release id>` (217/217).

## 10. A deadline is not a status

- Treating "deadline in the future" as "open" marks planning, awarded and cancelled notices as open (6 of 100 rows).
- **Avoid:** read the stage and tender status, not only the deadline.
