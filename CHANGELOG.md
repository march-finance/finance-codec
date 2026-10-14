# Changelog

## 0.1.0

- Strict versioned JSON for exact quantities, accounts, drafts and journals.
- Quantity wire compatibility with finance-core's existing FinanceCodec.
- Exact two-table CSV journal export/import with strict quoting and row ordering.
- Full history restoration through public ledger operations, including reversals.
- Duplicate-event, ambiguous-chart and edited-reversal rejection.
- Generated round-trip properties and a separate four-library integration consumer.
- March 0.5.0 pin; local sibling dependencies, no publication or file IO.

Integration testing also corrected finance-ledger's non-adjacent duplicate account
validation, with a failing-then-passing regression in that sibling project.
