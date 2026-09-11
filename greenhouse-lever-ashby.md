# Greenhouse, Lever, Ashby, Workable, Workday: dates, duplicates and caps

This chapter covers counting open roles and new roles from the public job board endpoints. Field coverage for the same five systems is in [ats-job-board-api-notes](https://github.com/neverempty/ats-job-board-api-notes), which this chapter does not repeat.

Measured on **2026-09-11**, from a cloud datacenter; all five returned 200.

## 1. Greenhouse `updated_at` is a bulk-update date

- **Got:** on Stripe's board, earlier on 2026-09-11, `updated_at` fell within the last 7 days for all 613 postings. `first_published` put 64 of them in the last 7 days.
- **Why it hurts:** "new this week" computed from `updated_at` says the whole board is new.
- **Avoid:** use `first_published`.

## 2. Greenhouse: one job id, several postings, often different jobs

- One `internal_job_id` can carry several postings, and most of the time they are different jobs, not copies. Stripe's board, re-checked on 2026-09-11: 626 postings under 605 job ids. 17 ids carried more than one posting (38 postings in all), and in 15 of those 17 the titles differed. Grouping on id + title gives 624 roles.
- So grouping on `internal_job_id` alone merges different jobs, which undercounts new roles by about 10%. Checked against direct counts: Elastic 78 vs 88 new in 30 days, Databricks 148 vs 158.
- **Avoid:** group on `internal_job_id` + title. For Elastic that gave 352 postings, 202 roles, 13 new in 7 days and 88 new in 30 days, all matching a count made directly from the API by a separate script.
- `content=true` is heavy: skipping it shrank one response from 4.4 MB to 0.35 MB.

## 3. Lever: one job per location

- On Palantir's board (re-checked 2026-09-11), 141 of 310 postings fell into 57 groups with the same title, department, team and description, differing only in location. Merging them removes 84 postings.
- Some descriptions live only in `lists`: on Spotify's board, 4 of 75 postings had empty opening, description and additional text.
- `createdAt` is stable. Palantir's oldest is 2009-12-05, which a field that moves on every edit would not keep.
- **Avoid:** to compare bodies, hash title + department + team + all text including `lists`. Without a body, do not merge on title alone.

## 4. Department may be empty or constant

- Ashby, Plaid: all 99 postings have department `All Departments`, while `team` carries the real split (Engineering 28, Product 12, ...).
- Lever, Palantir: department is empty; `team` is filled.
- **Avoid:** if department has at most one value and team has several, break down by team.

## 5. Workable: which date?

- On Blueground's board, `published_on` was later than `created_at` for 14 of 21 postings, by up to about 10 months. Workable does not document whether re-posting moves `published_on`.
- **Avoid:** count new roles from `created_at`.

## 6. Workable returns 200 and an empty board for common names

- For 16 of 20 well-known companies whose real boards are on other systems, the Workable name lookup returned 200 with 0 jobs.
- **Why it hurts:** if you guess the ATS from a company name and stop at the first 200, you report "not hiring".
- **Avoid:** try every system, prefer the one with rows, and never treat an empty board found by name alone as evidence. A name that does not exist returns 404 on all four of Greenhouse, Lever, Ashby and Workable.

## 7. Workday repeats itself past offset 2,000

- **Sent:** 200 pages of 20 on Nvidia's board, 4,000 rows.
- **Got:** only 2,000 distinct requisition ids. `total` also stops at 2000, while the Time Type facet sums to 2,614.
- `total` and `facets` come only on the first page; later pages report `total: 0`.
- Each page took about 1.4 seconds (200 pages in 277 seconds).
- **Avoid:** stop at offset 2,000, drop repeated ids, and when `total` is 2000 use a single-valued facet (Time Type) to count past it.

## 8. Workday facets can overlap

- On Red Hat's board, Job Function sums to 152 against a total of 143, because one posting can carry several values.
- **Avoid:** sum only facets whose values are exclusive.

## 9. Trailing whitespace splits locations

- On Airbnb's board, `United States` and `United States ` (with a trailing space) counted as two locations.
- **Avoid:** normalise whitespace before grouping.

---

Actor that handles trap 1 (posting date from `first_published`): https://apify.com/neverempty/greenhouse-jobs
