# Texas TDLR licence file (bulk CSV)

The Texas Department of Licensing and Regulation publishes all its licences as one CSV (`ltlicfile.csv`), updated daily according to `Last-Modified`. No key.

Measured on **2026-09-11**; the note on the data.texas.gov copy is from **2026-09-10**.

## 1. It is one 187 MB file, and some `Accept` headers are refused

- **Size:** 187,651,846 bytes, 996,129 rows, 82 licence types.
- **`Accept` (re-checked 2026-09-11):** `Accept: text/html` or `Accept: text/csv` returns 406. No `Accept` header, or `Accept: */*`, returns 206 for a range request.
- **Read time from a cloud datacenter:** 9.6 seconds with `Range: bytes=0-` (93 MB of memory while streaming), 26.4 seconds without `Range`.
- **Avoid:** stream it, and resume with `Range` from the last byte if the connection drops. Report "partial" if you never reach the end.

## 2. 30,881 barber licences appear twice

- **Got:** every Class A Barber licence (30,881) appears a second time with an empty licence type and the same number, sub-type, name and expiry.
- **Why it hurts:** each licence is counted or delivered twice.
- **Avoid:** if a typed row exists with the same (number, sub-type, name, expiry), drop the untyped one. Untyped rows without a typed twin also exist (1,677 barber shop rows; 18,509 with both type and sub-type empty). Keep those with type `null` rather than guessing a type.

## 3. (type, number) is not unique

- 334 (type, number) pairs repeat, because they are different credentials with different sub-types. Example: A/C Technician 353 exists as both REG and CER.
- **Avoid:** key on (type, number, sub-type). With that key there were 0 duplicates.

## 4. One licence type has an extra column

- The 272 Boiler Inspector rows have 24 columns; every other row has 23. The extra column comes after the continuing-education flag, so the first 19 columns stay in place.
- **Avoid:** read columns by position only up to the stable prefix, or parse with a CSV reader that tolerates ragged rows.

## 5. Individual licences sometimes carry a home address

- Most individual licences (electricians, A/C technicians, cosmetologists) have empty address and phone fields.
- Some don't. Water Well Driller individuals carry a BUSINESS PHONE, and Registered Accessibility Specialist individuals carry a "business address" that looks residential.
- **Avoid:** do not publish address or phone just because the column is filled. Decide by licence type.

## 6. "Last, First" does not catch every person

- Individuals are usually written `LAST, FIRST`. But 1,841 business licences (1,535 not yet expired) are held under a personal name written `FIRST LAST`, and a comma-pattern test does not see them.
- **Avoid:** if the distinction matters to you, decide by licence type first, name pattern second.

## 7. There is no status column and no issue date

- Expiry comes as `" 05/12/2027"`: a leading space, then MM/DD/YYYY. Expired licences stay in the file. There is no issue date.
- **Avoid:** trim, parse month-first, and compute "expired" from the date yourself.

## 8. Wide filters hold hundreds of thousands of rows

- **Sent:** "expiring within 365 days, any status" across the file.
- **Got:** 540,431 matching rows. Holding them all failed at 512 MB of memory.
- **Avoid:** keep only the top N by sort order (a heap) while streaming, and store rows compactly.

## Mirror

A copy of the file on data.texas.gov had stopped updating on 7/16 (checked 2026-09-10). Its metadata does not make that obvious, so read the TDLR original.
