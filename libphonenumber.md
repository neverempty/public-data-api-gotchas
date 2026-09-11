# Phone numbers with Google libphonenumber (via libphonenumber-js)

No network API here. This is Google's numbering-plan metadata as ported by `libphonenumber-js` 1.13.13, checked against `google-libphonenumber` 3.2.46 and Google's public demo.

Measured on **2026-09-11**. Example numbers below are fictional (Ofcom drama range `020 7946 0xxx`, and libphonenumber's own example numbers).

**What "valid" means:** the number fits a range allocated in that country's numbering plan. It does not tell you whether the line is in service, who uses it, or which carrier it moved to under number portability.

## 1. The default metadata is the wrong one

- With the default `min` metadata, `getType()` returns `undefined` for most countries and `isValid()` checks length only: `+49 10123456` is "valid" under `min`.
- **Avoid:** import from `libphonenumber-js/max`.

## 2. Against Google's own examples

- 1,377 of Google's example numbers across 245 regions: validity agreed on all 1,377, and type differed on 1 (TA `+2908999`: Google says FIXED_LINE, the port says FIXED_LINE_OR_MOBILE).
- 16,524 random inputs compared with Google's demo showed 0 disagreements on validity, type and E.164.

## 3. +1 numbers cannot be split into mobile and fixed

- US, Canada and the other +1 regions return FIXED_LINE_OR_MOBILE, the same as Google.

## 4. An invalid number has no country

- If `isValid()` is false, `country` is `undefined` even for `+1` or `+44`; `+1` alone is shared by more than 20 regions.
- **Avoid:** return country `null` and keep the calling code.

## 5. Without a `+`, the default country decides

- `020 7946 0000` read with default country US becomes `+102079460000`, which is invalid.
- `011 44 20 7946 0000` read as US, and `0044 20 7946 0000` read as GB or DE, both become `+44`. `0044` does not work under US rules.
- **Avoid:** record, on every row, which country the number was read as.

## 6. Full-width prefixes are not read; bracketed ones depend on `extract`

- A full-width `＋８１` (common in Japanese and Chinese input) fails, and is then read as a national US number (`+1819012345678`, invalid).
- With `extract: false`, `(+44) 20 7946 0000` returns NOT_A_NUMBER. With default parsing it reads as `+442079460000`, valid (checked locally on 2026-09-11).
- **Avoid:** apply NFKC normalisation. If you parse with `extract: false`, also rewrite a leading `(+44)` to `+44` (`+(44)` already parses). En dash, em dash and full-width hyphen separators parse without any rewriting.

## 7. Parsing does not extract from text

- With `extract: false`, `Tel: +44 ...`, or two numbers in one field, returns NOT_A_NUMBER.

## 8. Mobile time zones are per country, not per prefix

- Google's rule: only geographic types (fixed line, fixed-or-mobile, and mobile in countries 52, 54 and 55) use prefix lookup. Mobile and toll-free numbers get every time zone of the calling code.
- Australian mobiles get 8 zones. A prefix lookup returns Sydney only, which disagreed with Google on 16% of numbers.
- A Jamaican mobile number returned 42 time zones (all of NANP), and so does Google's demo.
- Adding countries to the geographic-mobile list from memory (62, 86) created 2 disagreements with Google. Take the list from the library source.

## 9. Dedupe on E.164

- The same number written two ways gives the same E.164 (plus extension), so dedupe on that.
- Scale: 100,000 distinct numbers took 92 seconds in a 512 MB container, peaking at 220 MB.
