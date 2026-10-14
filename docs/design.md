# finance-codec v0.1

Approved scope: versioned JSON for exact quantities, accounts, drafts and journals;
two-table CSV journal exchange; strict validation; restore ordered history through
public finance-ledger operations including linked reversals. March 0.5.0.

Dependencies are sibling finance-core and finance-ledger only. The existing
FinanceCodec quantity representation stays compatible; this package uses Codec*
module names to avoid collision. Crypto envelopes are not serialized: their
source/event keys already survive in ledger metadata.

JSON numbers are strings, never Float. v1 objects reject unknown/missing/duplicate
fields, wrong types, unknown versions and noncanonical integer text. Quantities
use core's native-precision decimal parser; no rounding. A decoded draft remains
a draft, not proof of balance. Restored journals validate accounts, posting
balance, unique entry/event IDs, and exact compensating reversal contents.

JSON shapes:
- Quantity: existing FinanceCodec version/asset/precision/amount object.
- Account: version/id/kind/asset/precision strings; kind asset, liability, equity,
  income or expense.
- Draft: version/id/source/effective/recorded strings, external_id string or null,
  postings array of account/direction/quantity objects. direction debit or credit.
- Journal: version/accounts/entries; each entry has draft and reversal_of string
  or null. Accounts/entries/postings preserve their input order.

CSV is two strings: accounts table and postings table. Headers are exact and
ordered; every row carries version 1. Each account occurs once. Every posting row
repeats its entry metadata, entry_index and posting_index; indexes start at zero,
must be contiguous and agree with row order. No lossy regrouping or sorting.
External optional ID has an explicit flag; None is not confused with text.
Reversal None is an empty cell, since ledger entry IDs cannot be empty.

CSV parsing accepts LF or CRLF outside quoted fields and doubled quote escapes;
rejects unclosed quotes, quotes in unquoted fields, garbage after closing quote,
bare CR outside quotes, bad widths/headers and inconsistent repeated metadata.
Emit every field quoted and CRLF rows. CSV is exact machine interchange, not a
spreadsheet-safe presentation export; no formula sanitization that changes data.

Restoration never forges opaque constructors. Use new, post or reverse into a
fresh local immutable journal. Reject replay-coalesced duplicates, missing/forward
reversal links, multiple reversals, and edited compensating postings. On any
error return no journal; no IO, writes, network, or ambient clocks.

Limits: no cryptographic authenticity, persistence, migration, unknown-field
retention, streaming or input-size limits. Callers bound untrusted inputs before
parsing. Schema validation is not authorization or protocol validation.
