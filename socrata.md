# Socrata SODA API: city building permits, state licence data

No key. Measured on **2026-09-10** (building permits) and **2026-09-11** (licence datasets).

| Portal | Dataset |
|---|---|
| New York City (DOB NOW) | `rbx6-tga4` |
| Los Angeles | `pi9x-tg5x` |
| Austin | `3syk-w9eu` |
| Chicago | `ydr8-5enu` |
| San Francisco (data.sf.gov) | `i98e-djp9` |
| New Orleans | `rcm3-fn58` |
| Cincinnati | `uhjb-xac9` |
| Seattle | `76t5-zqzr` |
| Washington L&I contractor licences | `m8qx-ubtq` (160,752 rows) |
| Oregon CCB active licences | `g77e-6bhs` (56,171 rows, 45,508 licence numbers) |

## 1. Dollar amounts stored as text compare as text

- **Sent:** permits issued in the last 30 days with a value over $1,000,000.
- **Got:**

| City | text compare `> '1000000'` | with `::number` | bare numeric compare |
|---|---|---|---|
| New York City | **9,997** | **699** | 400 type-mismatch |
| Los Angeles | **4,479** | **63** | 400 |
| San Francisco | **1,122** | **35** | 400 |
| Cincinnati | **767** | **12** | 400 |
| Austin / Chicago / Seattle | 19 / 81 / 27 | same | same (numeric columns) |

- **Avoid:** always cast. Casting a column that is already numeric gives the same result, so cast every time.

```
curl -G 'https://data.cityofnewyork.us/resource/rbx6-tga4.json' \
  --data-urlencode "\$where=estimated_job_costs::number > 1000000" \
  --data-urlencode '$select=count(*)'
```

## 2. One permit, several rows

All rows from the last 7 days:

| City | rows | extra rows (rows minus distinct permits) | why |
|---|---|---|---|
| New York City | 2,168 | 339 (15.6%) | 229 of 235 groups differ only in `work_type`; 6 have different `tracking_number` (separate applications) |
| San Francisco | 251 | 20 (8.0%) | several addresses; each of the 231 permits has exactly one `primary_address_flag="Y"` |
| Cincinnati | 107 | 6 (5.6%) | line items with different type and amount (all 6 groups had different amounts) |
| LA, Austin, Chicago, New Orleans, Seattle | | 0 | |

New York re-checked on 2026-09-11 (issued 2026-09-03 to 09-09): 2,842 rows, 2,417 distinct permits, 425 extra rows (15.0%). Keyed on (work_permit, tracking_number), there were 418 extra rows.

- **Why it hurts:** if you count or bill per row, about one New York row in seven is a second copy of the same permit.
- **Avoid:** group before counting, using `(work_permit, tracking_number)` for NYC, the primary address for SF, and a summed amount for Cincinnati. NYC renewals reuse the permit number with a new issuance (5,083 of 14,100 rows in 30 days, 36%). Those are separate issuances, not duplicates.

## 3. The same column name means different things

- **Chicago:** `contact_1` is mostly the OWNER, usually a private person. Contractor contacts use types such as `CONTRACTOR-*`, `GENERAL CONTRACTOR` and `TENT CONTRACTOR`. Matching "contains CONTRACTOR, does not start with OWNER" raised contractor-name coverage from 29% to 98%.
- **Placeholders:** Austin's `total_job_valuation` is filled on 12% of rows, often with `"1"` or `"0"`. San Francisco uses `"1.0"`.
- **Cincinnati** contractor names arrive wrapped in literal quote characters (shape: `"\"EXAMPLE ROOFING\""`).
- **New York's** `work_permit` column contains the text `Permit is not yet issued` on some rows (4 since July 1).
- **Austin's** street address is in `original_address1`; `permit_location` holds the project name.

## 4. A keyword has to search several columns

- `upper(job_description) like '%ROOF%'` works: 919 rows in New York, 1,280 in Los Angeles.
- Cincinnati's `description` holds only category names such as `Alteration`. Over 365 days, ROOF, SOLAR, POOL and DECK each matched 0 rows, while HVAC matched 2,871.
- **Avoid:** search every text column the city fills, and when a city cannot match a word, say why in the empty result.

## 5. Errors are honest

- A column that does not exist or a malformed date returns 400 with the reason in the body. San Francisco's `message` says only `Invalid SoQL query`. On 2026-09-11 the same body also carried `"errorCode":"query.soql.no-such-column"` and the column name under `data`, so read `errorCode` and `data`, not just `message`.

## 6. Paging

- `$order=<issue date> DESC, <permit number>` produced 0 duplicates across `$offset` pages.
- `$limit` above 50,000 returns in one call (NYC: 51,561 rows).
- A deep `$offset` took 56 seconds, so split by date instead.
- If the data can change while you page, append `:id` to `$order`. (`$select=:id,*` returns 400, but `:id` in `$order` returns 200.)

## 7. Publication lags by 1 to 5 days

- Latest issue date on 2026-09-10: NYC 9/8, Cincinnati 9/8, LA 9/5. A "last 1 day" query can be empty for reasons unrelated to activity.

## 8. Equality is case-sensitive (Washington)

- `upper(city)='SEATTLE'` returned 8,173 rows; `city='Seattle'` returned 616.

## 9. Dates stored as text (Oregon)

- `lic_exp_date` holds strings like `"02/06/2028"`. `lic_exp_date < '10/15/2026'` returned 46,036 rows because it compares month-first strings. `to_floating_timestamp` raises a type error.
- **Avoid:** pull candidate rows with a per-month `like`, convert dates locally, then compare.

## 10. One Oregon licence, several rows and several expiry dates

- One row per endorsement: licence 100283 has two rows (CSC2 and RSC) with bonds of 25,000 and 20,000.
- 4,251 licences carry different dates on different endorsements. For example, licence 3550 expires 2027-10-08 on CGC1 and RGC but 2026-10-08 on LBPR (lead-based paint). Outside LBPR, 0 licences disagreed; 91 licences have only LBPR.
- Home inspector (OCHI) and locksmith (OCLS) numbers collide with CCB licence numbers (1022, 119, 13 and others), so the key has to be (licence family, number).
- The "active licences" dataset also contains rows whose expiry month has already passed in 2026.
- **Avoid:** group per licence first, then apply the expiry filter. Filtering rows can return one endorsement of a licence and report the licence as expiring.

## 11. Status and expiry disagree (Washington)

- Licence numbers are unique. `status` has 10 values (ACTIVE 75,811 / EXPIRED 61,213 / SUSPENDED 9,759 / ...), and 34 ACTIVE licences are past their expiry date.
- Bonds (`bzff-4fmt`) and insurance (`ciwg-agsx`) are separate datasets with historical rows per licence. Bond expiry is text, either `Until Canceled` or `MM/DD/YYYY`.

---

Actor that handles these traps: https://apify.com/neverempty/us-building-permits-scraper
