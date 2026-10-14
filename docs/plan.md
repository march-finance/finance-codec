# finance-codec Implementation Plan

> Execute inline with test-driven-development and one independent final review.

**Goal:** Implement docs/design.md using public core/ledger APIs on March 0.5.0.
**Architecture:** CodecWire strict JSON primitives -> CodecJson domain mapping;
CodecJournal restores history for JSON/CSV; CodecCsvText strict lexical CSV;
CodecCsv maps fixed rows to domain values. No crypto dependency in production.

## Constraints and review focus

Preserve exact large amounts, metadata optionality, order and reversal links.
Reject duplicate JSON keys and CSV entry/posting gaps; do not accept replay
coalescing as successful snapshot restoration. Verify malformed reversal contents
against the ledger-produced compensating entry, not just a user-supplied link.
Keep existing FinanceCodec quantity consumers working. CSV escape syntax and
metadata with commas/quotes/newlines require actual round trips.

## Task 1: Strict JSON domain codecs and journal restoration

Files: forge.toml, .march-version, .gitignore, lib/error.march, wire.march,
json.march, journal.march; test/support_test.march, json_test.march.

Produces:
CodecError.Error = InvalidEncoding | UnsupportedVersion | InvalidInteger |
  InvalidCsv | DuplicateEntry | InvalidReversal | Core(FinanceError.Error) |
  Ledger(LedgerError.Error).
CodecJson.encode/decode_quantity, encode/decode_account, encode/decode_draft,
encode/decode_journal; encode returns String, decode Result(domain, Error).
CodecJson.account_value/from_value, draft_value/from_value map Json values.
CodecJournal.restore(List(Account), List((Draft, Option(String)))) -> Result(Ledger, Error).

- [x] Write tests with a literal valid USD account and balanced draft, large exact
  quantity compatibility, bad keys/types/versions/integers and invalid balances.
  Assert `FinanceAmount.format(decode_quantity(encode_quantity(q)))` is literal
  `12345678901234567890.12`. Run forge test and observe missing API errors.
- [x] Implement strict field sets and canonical Int parsing using string_to_int
  followed by int_to_string equality. Use public constructors for domain values.
  Delegate quantities to core codec. Restore each entry with public post/reverse;
  require count to grow by one and compare resulting contents/link with input.
- [x] Run full tests and check; commit when green.

## Task 2: Strict CSV lexer and journal tables

Files: lib/csv_text.march, csv.march; test/csv_test.march.
Consumes Task 1 domain codecs/restoration. Produces CodecCsvText.parse(String)
-> Result(List(List(String)), Error), format(rows)->String;
CodecCsv.encode_journal(Ledger)->(String,String), decode_journal(String,String)
->Result(Ledger,Error).

- [x] Write lexer tests for quotes/commas/newlines and malformed quote syntax.
  Write account/posting table roundtrip and reject duplicate/gap/reordered indexes,
  malformed flags, metadata conflicts and extra columns. Run tests before code.
- [x] Implement four lexical states (start, bare, quoted, closed), explicit
  EOF/CRLF handling, and all-field quoted formatting. Fixed table headers:
  accounts version/account_id/kind/asset/precision;
  postings version/entry_index/entry_id/source/effective/recorded/has_external_id/
  external_id/reversal_of/posting_index/account_id/direction/asset/precision/amount.
  Reconstruct contiguous draft groups, then call the shared restore function.
- [x] Run full tests and check; commit when green.

## Task 3: Adversarial cases, properties and four-library integration

Files: test/properties_test.march, test/restore_test.march; integration/forge.toml,
integration/.march-version, integration/test/ecosystem_test.march.

- [x] Generate signed quantities and positive journal amounts; roundtrip JSON
  and CSV without changing atoms, precision or IDs. Generate reversal histories
  and ensure zero final balances plus unchanged original history.
- [x] Mutate snapshot JSON reversal link/content and duplicate event keys; prove
  rejection. Test empty accounts/empty history, multiple original/reversal entries,
  optional metadata containing escaped commas, quotes and multiline text.
- [x] In a separate consumer declare all four path dependencies. Build a crypto
  token transfer plus native gas, post it, serialize/restore via JSON and CSV,
  compare literal exact balances, replay the original event, then reverse it.
- [x] Verify debug seed42/123 and release seed20261014; independent review of
  implementation and scope; fix confirmed blockers with red-to-green tests.

## Task 4: Documentation and final verification

Files: README.md, CHANGELOG.md, docs/progress.md.

- [x] Document literal wire examples, exact headers, API signatures, compatibility,
  strict failure semantics, draft versus validated history, and CSV formula risk.
- [x] Run forge check, build --release, full tests debug/release, consumer tests,
  sibling regression suites. Record actual counts and compiler warnings.
- [x] Commit verified local branch codex/finance-codec-v0.1; no remote or publish.
