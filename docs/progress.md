# finance-codec execution ledger

Design: docs/design.md; plan: docs/plan.md. User approved scope and requested build.
Native execution in new requested directory on named feature branch.
Pre-flight: JSON/CSV share ordered (Draft, Option(reversal ID)) input to restore;
domain constructors and canonical scalar validation reused across formats.
Task 1: complete. Missing API RED; JSON/domain/history tests 5/5 GREEN on 0.5.0.
Task 2: complete. Missing CSV API RED; 10/10 combined tests GREEN seed42.
Task 3: complete. 18 codec tests including 4 generated properties pass debug
seed42/123 and release seed20261014. Separate four-library consumer passes debug
and release. Exact large token/native amounts, metadata, replay and reversals
survive both formats; malformed-history regression fixed through ledger API.
Dependency regression: ledger accepted non-adjacent duplicate account IDs because
List.dedup only removes adjacent elements. New ledger regression failed, then
construction and direct validation switched to shared full-chart unique_ids.
Only targeted ledger source/test files changed; pre-existing untracked tree preserved.
CSV stdlib inspected: its parser is file/IO based, so strict pure string parser
uses a local four-state implementation rather than adding IO capabilities.
Task 4: complete. README/CHANGELOG document schemas, APIs, validation, limits and
CSV formula risk. Independent read-only review found no blockers; noted scope
exclusions (authenticity/persistence/input limits) are documented caller duties.
Final March 0.5.0 verification: build/check pass; core 37/37, ledger 25/25, crypto
22/22 release regressions pass; existing 2+5+8 contract checks pass. The preexisting
crypto accessor refinement_unverified hint remains documented in that library.
Apple clang target-triple override warning is informational. Codec changes are
committed locally; targeted ledger fix remains in its pre-existing untracked tree.
