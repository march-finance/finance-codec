# finance-codec execution ledger

Design: docs/design.md; plan: docs/plan.md. User approved scope and requested build.
Native execution in new requested directory on named feature branch.
Pre-flight: JSON/CSV share ordered (Draft, Option(reversal ID)) input to restore;
domain constructors and canonical scalar validation reused across formats.
Task 1: complete. Missing API RED; JSON/domain/history tests 5/5 GREEN on 0.5.0.
Task 2: complete. Missing CSV API RED; 10/10 combined tests GREEN seed42.
Task 3: in progress. Property/adversarial tests added; integration consumer next.
Dependency regression: ledger accepted non-adjacent duplicate account IDs because
List.dedup only removes adjacent elements. New ledger regression failed, then
construction and direct validation switched to shared full-chart unique_ids.
Only targeted ledger source/test files changed; pre-existing untracked tree preserved.
CSV stdlib inspected: its parser is file/IO based, so strict pure string parser
uses a local four-state implementation rather than adding IO capabilities.
