# finance-codec

[![Tests](https://github.com/march-finance/finance-codec/actions/workflows/ci.yml/badge.svg)](https://github.com/march-finance/finance-codec/actions/workflows/ci.yml)

Licensed under the [MIT License](LICENSE).

## Documentation

Start with the [library guide](docs/guide.md) for installation, a complete tested
quickstart, API usage, error handling and operational limits. The
[quickstart source](test/docs_quickstart_test.march) runs in the normal test suite.
This README provides the module overview and detailed reference notes below.

## GitHub checkout and CI

Clone this repository and resolve its published dependencies:

```sh
git clone https://github.com/march-finance/finance-codec.git
cd finance-codec
forge deps
forge test --seed=42
```

GitHub Actions runs on pushes, pull requests and manual dispatches. It installs
the pinned March 0.5.0 release, checks/builds this package, and runs debug/release
tests and properties. Dependencies use published version 0.1.0 with committed lockfiles.
The workflow also tests all four libraries together via the integration consumer.

Strict, exact JSON and CSV interchange for March finance data. Version 0.1.0,
pinned to March 0.5.0. Production dependencies: published `finance-core` and
`finance-ledger`. No dependency on finance-crypto; its ledger event references
already travel in metadata.

## API

| Module | Operations |
| --- | --- |
| CodecJson | `encode_quantity` / `decode_quantity` |
| CodecJson | `encode_account` / `decode_account` |
| CodecJson | `encode_draft` / `decode_draft` |
| CodecJson | `encode_journal` / `decode_journal` |
| CodecCsv | `encode_journal` / `decode_journal` |
| CodecJournal | `restore(accounts, orderedDraftsAndReversalLinks)` |
| CodecWire | Shared strict JSON field/type/version/integer validation |
| CodecCsvText | Pure strict `parse(text)` / `format(rows)` helpers |

JSON encoders return String; JSON decoders return `Result(value, CodecError.Error)`.
CSV encode returns `(accountsCsv, postingsCsv)`; decode takes these two strings
and returns `Result(LedgerJournal.Ledger, CodecError.Error)`. No file IO occurs.

```march
let json = CodecJson.encode_journal(journal)
let? restored = CodecJson.decode_journal(json)
let (accounts_csv, postings_csv) = CodecCsv.encode_journal(restored)
CodecCsv.decode_journal(accounts_csv, postings_csv)
```

These calls belong inside a function returning the codec Result type. Existing
`FinanceCodec.encode/decode` in finance-core remain unchanged. CodecJson quantity
functions wrap that exact wire format and map errors to `CodecError.Core`.

## JSON v1

All amounts, precisions, timestamps and versions are JSON **strings**, not Number
or Float. Negative quantity values are supported; posting amounts must be positive.
Precision is native 0..255, validated by core. Exact decimal parsing never rounds.
Amounts use core's decimal grammar and normalize on output to full precision;
integer fields must already be canonical (`0`, `2`, `-100`, not `02`, `+2`, `-0`).

Quantity (compatible with core):

```json
{"version":"1","asset":"fiat:USD","precision":"2","amount":"12345678901234567890.12"}
```

Account:

```json
{"version":"1","id":"cash","kind":"asset","asset":"fiat:USD","precision":"2"}
```

Kinds are exactly `asset`, `liability`, `equity`, `income`, `expense`.

Draft:

```json
{
  "version":"1","id":"entry-1","source":"import",
  "effective":"100","recorded":"101","external_id":null,
  "postings":[
    {"account":"cash","direction":"debit","quantity":{"version":"1","asset":"fiat:USD","precision":"2","amount":"1.23"}},
    {"account":"equity","direction":"credit","quantity":{"version":"1","asset":"fiat:USD","precision":"2","amount":"1.23"}}
  ]
}
```

Directions are exactly `debit` and `credit`. Timestamps are caller-supplied UTC
Unix seconds within March's Int range; effective must not exceed recorded.
`external_id` is null or a nonblank string. No ambient clock or timezone inference.

Journal root has exactly `version`, `accounts`, `entries`. Accounts use the above
account shape. Every entry is exactly:

```json
{"draft": {"version":"1","id":"entry-1","source":"import","effective":"100","recorded":"101","external_id":null,"postings":[]}, "reversal_of":null}
```

This last snippet illustrates an entry's shape only: an empty posting list is a
valid draft, but is **rejected** when restoring a posted journal. `reversal_of` is
null for originals or the exact ID of an earlier entry for a reversal.

Every schema object rejects missing, unknown, duplicate or wrongly typed fields,
including nested objects. Unknown versions fail. JSON object field order does
not matter; array order does. Empty account charts and empty histories are allowed.

## Restoration is validation, not opaque deserialization

`decode_draft` reconstructs a draft using public smart constructors. It does not
certify account membership or balance. Use the ledger's validation/posting API.

`decode_journal` reconstructs a **fresh** journal via `LedgerJournal.new`, `post`
and `reverse`. It preserves account order, entry order, posting order, metadata,
IDs and reversal links. It rejects:

- Duplicate account IDs anywhere in the chart.
- Unknown accounts, asset/precision mismatches, invalid amounts or unbalanced entries.
- Duplicate entry IDs or duplicate source-scoped event keys—even exact retries.
- Missing, self or forward reversal links; repeated reversal of the same original.
- Edited compensating postings, including changed amounts, accounts, direction or order.

Every entry must increase history length by one. Ledger replay coalescing is not
accepted as snapshot restoration. Reversal-of-reversal histories are supported.
No state is published on failure; partial intermediate journals remain local
immutable values. This is snapshot restoration, not append/merge into a live ledger.

## CSV v1

Use two tables together: the account chart and one row per ordered posting.
Headers are mandatory, exact, case-sensitive and in this order:

```text
version,account_id,kind,asset,precision
```

```text
version,entry_index,entry_id,source,effective,recorded,has_external_id,external_id,reversal_of,posting_index,account_id,direction,asset,precision,amount
```

Every data row has version `1`. Account rows use the same kind/asset/precision
rules as JSON. Posting rows repeat all entry metadata. `entry_index` starts at 0;
each new contiguous group increments by one. `posting_index` starts at 0 within
each group and increments by one. Missing, repeated or reordered indexes fail.
Repeated metadata must agree exactly across the group's rows; no sorting or
silent regrouping occurs.

`has_external_id` is `0` with an empty external_id cell for None, or `1` with a
nonblank external_id for Some. `reversal_of` is empty for None; otherwise an
earlier entry ID. These conventions do not conflate None with a blank valid ID.
Header-only tables represent empty data; completely empty files are rejected.

The lexer accepts LF or CRLF record endings and doubled quotes inside quoted
fields. Commas, quotes and embedded CR/LF inside quoted fields are preserved.
Unclosed quotes, quotes inside bare fields, garbage after a closing quote, bare
CR outside quotes, wrong widths and blank data records fail. Output always quotes
every field, doubles internal quotes and uses CRLF including the final row.
The low-level `format` helper expects at least one field in each row; schemas
always have fixed nonzero widths.

CSV is exact machine interchange, **not a spreadsheet-sanitized export**. Sources
or IDs starting with `=`, `+`, `-`, or `@` are preserved. Do not open untrusted CSV
as executable spreadsheet formulas; sanitize separately for presentation, without
altering archival input used for restoration.

## Errors and limits

`CodecError.Error` distinguishes InvalidEncoding, UnsupportedVersion,
InvalidInteger, InvalidCsv, DuplicateEntry, InvalidReversal, and wrapped Core or
Ledger errors. Errors are coarse domain categories, not row/path-rich reports.
Quantity compatibility errors remain wrapped core errors, including unknown
quantity versions. No panic is used to reject malformed caller data.

All parsing/restoration is eager and in memory. Callers must bound input bytes,
nesting and record counts before accepting untrusted data. No authentication,
tamper evidence, signature verification, persistence, atomic database replacement,
streaming, automatic migrations, unknown-field retention or crypto-envelope codec
is provided. Restored balances do not prove transactions happened on a chain.

## Verify

Resolve the published dependencies; install the pinned compiler
with `forge toolchain install v0.5.0` if necessary.

```sh
forge toolchain which
forge deps
forge check
forge build --release
forge test --seed=42
forge test --seed=123
forge test --release --seed=20261014
cd integration
forge deps
forge test --seed=42
forge test --release --seed=20261014
```

The separate integration project declares all four libraries and exercises a
large six-decimal token transfer with 18-decimal native gas, JSON/CSV restoration,
idempotent replay, then reversal and another round trip. Unit/property suites
cover signed quantity compatibility, strict structural validation, quoting,
metadata optionality, invalid histories, and exact restored balances.
