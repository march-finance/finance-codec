# finance-codec guide

Strict versioned JSON and CSV interchange for exact quantities and journals.
March 0.5.0; finance-codec 0.1.0. No file or network IO occurs inside this library.

## Install

Merge into your application's existing `[deps]` section:

```toml
[deps]
finance-core = "0.1.0"
finance-ledger = "0.1.0"
finance-codec = "0.1.0"
```

Run `forge deps`; commit your application's lockfile. Core and ledger are listed
explicitly for examples using their domain APIs. Pin `.march-version` to
`v0.5.0`, installing with `forge toolchain install v0.5.0`.
Crypto is not a production dependency and is not needed for this guide.

## Quickstart: JSON and CSV without losing exactness

Save this complete test under `test/codec_quickstart_test.march`, then run
`forge test --seed=42`. The first test round-trips a quantity; the second uses
an empty journal to demonstrate both tables without introducing posting setup.

```march
mod CodecQuickstart do
  needs IO.Clock, IO.Process

  fn example() : Result(String, CodecError.Error) do
    let input = "{\"version\":\"1\",\"asset\":\"fiat:USD\",\"precision\":\"2\",\"amount\":\"12.50\"}"
    let? quantity = CodecJson.decode_quantity(input)
    let? restored = CodecJson.decode_quantity(CodecJson.encode_quantity(quantity))
    Ok(FinanceAmount.format(restored))
  end

  test "documented exact JSON round trip" do
    assert (match example() do
      Ok(value) -> value == "12.50"
      Err(_) -> false
    end)
  end

  test "documented empty journal JSON and CSV round trip" do
    let journal = Result.unwrap(CodecJson.decode_journal(
      "{\"version\":\"1\",\"accounts\":[],\"entries\":[]}"))
    let (accounts_csv, postings_csv) = CodecCsv.encode_journal(journal)
    let restored = Result.unwrap(CodecCsv.decode_journal(accounts_csv, postings_csv))
    assert (LedgerJournal.count(restored) == 0)
    assert (CodecJson.encode_journal(restored) == CodecJson.encode_journal(journal))
  end
end
```

The unwraps are for fixed valid test fixtures only. Applications should
pattern-match decoder results and keep their previous journal on failure.
A successful decoder returns an entirely restored value, not a partial journal.

## Public entry points

| Data | Encode | Decode |
| --- | --- | --- |
| Quantity | CodecJson.encode_quantity(quantity) | CodecJson.decode_quantity(text) |
| Account | CodecJson.encode_account(account) | CodecJson.decode_account(text) |
| Draft | CodecJson.encode_draft(draft) | CodecJson.decode_draft(text) |
| Journal JSON | CodecJson.encode_journal(journal) | CodecJson.decode_journal(text) |
| Journal CSV | CodecCsv.encode_journal(journal) | CodecCsv.decode_journal(accountsText, postingsText) |

JSON encoders return String. CSV encode returns a pair of Strings, accounts first.
Decoders return `Result(value, CodecError.Error)`. Use your application's IO layer
to read/write those strings and provide atomic persistence.

`CodecJournal.restore(accounts, orderedDraftsAndReversalLinks)` accepts a list
of (Draft, Option(String)) pairs. It is the domain restoration boundary used by
the wire decoders. It is not a way to bypass ledger validation.

## JSON contract

Every schema carries version "1". Version, precision, timestamps, indexes where
applicable, and amounts use strings rather than JSON numbers. This preserves
arbitrary-size values. Optional external IDs and reversal links use null or strings
in the appropriate JSON fields.

Quantity JSON is compatible with core's FinanceCodec: exactly version, asset,
precision and amount. Signed quantities are permitted, but posting quantities
must be positive. Account kinds are asset, liability, equity, income and expense;
posting directions are debit and credit.

Decoders reject duplicate and unknown fields, wrong value types and unsupported
versions. Canonical integer fields reject leading zeros, plus signs and negative
zero. Amounts use core decimal parsing and normalize to native precision on output.
Do not use Float or a generic JSON-number coercion to prepare these payloads.

Decoding a draft checks its structural/domain constructors; it does NOT prove
balance against an account chart. Decode a full journal to validate the history,
or validate a decoded draft using your destination ledger.

See the [JSON v1 schemas](../README.md#json-v1) for full nested field lists.

## CSV contract

Journal interchange is TWO related tables, not a single transaction export:
accounts and postings. Keep both together as one snapshot.

Accounts header:
`version,account_id,kind,asset,precision`

Postings header:
`version,entry_index,entry_id,source,effective,recorded,has_external_id,external_id,reversal_of,posting_index,account_id,direction,asset,precision,amount`

Header order is fixed. Entry and posting indexes start at zero and must be
contiguous in file order. Each row repeats entry metadata; repetitions must
agree. has_external_id is 0 with an empty external_id for None, or 1 with a
nonempty value for Some. Empty reversal_of means no reversal.

Encoding quotes all fields, doubles embedded quotes and uses CRLF record endings.
Parsing accepts LF/CRLF record endings and quoted multiline fields, but rejects
bare quotes, unterminated fields, text after closing quotes and bare CR outside
quoted fields. Embedded newlines are data, not separate posting rows.

Empty histories use header-only tables; accounts can still be present.
Do not sort or independently reorder posting rows. Spreadsheet software can
coerce identifiers or amounts; use a text-preserving interchange workflow.
Formula-looking strings are preserved, not escaped for safe spreadsheet execution.

## Restoration and audit semantics

A journal is reconstructed in entry order through ledger posting and reversal
operations. Each imported entry must append exactly once. Duplicate IDs/events
are rejected rather than silently collapsed as harmless retries. Reversal targets
must already exist and supplied reversal content must match the compensating entry.

Never remove a failed row and claim that the rest is the same snapshot.
A new destination application's permissions, balances or business rules may add
constraints beyond what decoding checks.

## Errors and input safety

| CodecError | Meaning |
| --- | --- |
| InvalidEncoding | Invalid JSON, schema, fields or types |
| UnsupportedVersion | Unrecognized version |
| InvalidInteger | Noncanonical or out-of-range integer field |
| InvalidCsv | Invalid CSV syntax, table structure or row data |
| DuplicateEntry | Restoration would merge rather than append |
| InvalidReversal | Missing/forward target or altered compensating history |
| Core(error) | Asset, amount or metadata validation failed |
| Ledger(error) | Account, balance, posting or replay validation failed |

Error categories can depend on the first rejected layer; they are not per-field
diagnostics. Use Result matching rather than parsing human-readable messages.
`CodecWire` and `CodecCsvText` expose lower-level helpers, but domain callers
should prefer CodecJson and CodecCsv.

Bound input size and record counts before decoding untrusted payloads.
There is no streaming API, built-in resource budget, authenticity/signature check,
automatic migration, unknown-field preservation or durable transaction.
A round-trip proves internal consistency, not external truth or tamper resistance.

See [wire details and verification](../README.md) and the
[tested quickstart](../test/docs_quickstart_test.march).
