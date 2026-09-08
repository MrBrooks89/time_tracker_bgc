# Migration source assessment

Reviewed: 2026-09-08. Status: static source review, not a runtime/security certification.

## Scope and provenance

| Source | Reviewed commit | Purpose |
| --- | --- | --- |
| [Python upstream](https://github.com/ebrooksbgc/time_tracker_bgc/tree/baae802434d92a92698878c14479bca6f21a54cd) | `baae802434d92a92698878c14479bca6f21a54cd` | Existing domain logic, persistence, UI, tests |
| [Node blueprint](https://github.com/MrBrooks89/time-tracker-nodejs/tree/ef8b37b3987b7760fc8d9d15b2b1d043d7a232c4) | `ef8b37b3987b7760fc8d9d15b2b1d043d7a232c4` | Independently implement agreed workflow behavior, not a line-by-line port |

The Python fork is [MrBrooks89/time_tracker_bgc](https://github.com/MrBrooks89/time_tracker_bgc).
Source and test files were read; dependencies were not installed, tests were not run, and no live database was inspected. Office documents and workbook contents were not independently validated. Counts asserted by tests are not verified production-data counts. Findings describe the pinned snapshots, not future repository revisions.

File/line references below are relative to the named repository at its pinned commit. They can be inspected through the commit links above.

## Corrections to the preliminary README-only review

- Streamlit can serve multiple users; SQLite can be centralized. Neither framework choice alone proves a deployment unsuitable. Actual missing identity/authorization, transactional controls, operational evidence, and accounting safeguards drive this migration.
- Both implementations use Wednesday–Tuesday fiscal weeks in code. Python README language describing Monday–Sunday entry is stale (`services.py:295–364`). Do not silently change the business week.
- Python has an MIT license (`LICENSE:1–21`); its README's no-license statement is stale. Preserve the license notice.
- No application license was found in the Node snapshot. Public visibility is not a license grant. Obtain rights-holder confirmation before copying source, tests, or documentation; use independently implemented business behavior meanwhile. Check workbook/asset provenance separately.
- Neither test suite has been executed in this planning session. No production-readiness percentage or performance claim is justified.

## Python: preserve the domain, change the boundaries

### Useful foundation

- Eight relational models: Project (also non-project category), Task, Employee, Timesheet, AccountingPeriod, ProjectAssignment, FavoriteAssignment, TimeEntry (`models.py:10–162`; FavoriteAssignment at `models.py:88–93`).
- SQLModel foreign keys, Numeric fields, unique employee/week and assignment/favorite indexes, partial unique running-timer indexes with PostgreSQL predicates (`models.py:117–162`).
- UTC conversion and local-day clipping; DST-aware midnight boundaries; elapsed-time calculations (`time_utils.py:6–31`, `services.py:825–826,1149–1155`).
- Decimal quarter-hour validation, task-driven classification, holiday-adjusted targets, archival intent, and published fiscal calendar (`services.py:295–364,573–602,874–880`).
- Test source contains 37 test functions across three files: services, Streamlit shell/grid checks, and workbook import. Useful examples cover timer constraints, foreign keys, clipping, one spring DST case, classification, locks, copy, notes, and daily limits (`tests/test_services.py`, `tests/test_app.py`, `tests/test_reference_import.py`). This is a test inventory, not measured coverage.

### Required redesign

| Finding | Evidence | Migration consequence |
| --- | --- | --- |
| Partner selection is not authentication; setup/history/close are broadly accessible | `app.py:237–299,1172–1181,1493–1547`; `services.py:1061–1069` | Actor-aware authorization at every backend operation; no trusted owner ID from the UI |
| Services commit internally; some getters create and commit records | `services.py:211–218,410–439,484–501` | Explicit unit of work per command; queries and validation cannot persist incidental changes |
| UI row editing invokes seven separately committed cell changes | `app.py:705–734`; `services.py:909–934` | Atomic row/week commands and stable entry/revision IDs |
| Read-check-write totals, submission, and close have no shared locking/version protocol | `services.py:442–460,634–675,874–934` | PostgreSQL concurrency tests and protected preconditions, not just transactional writes |
| Timesheet membership is inferred from employee and overlapping intervals; no timesheet FK | `models.py:56–62,96–109` | Explicit work-date allocations, weekly ownership, source interval provenance |
| Recall resets states including Locked/In Correction; submit does not fully recheck fiscal permission | `services.py:442–501` | Approved transition table shared by every entry path |
| Timer stop checks no affected-period locks; manual/delete validate only start day; clear/copy validate week start | `services.py:765–844,930–1069` | Validate every affected work date and period, including first-time creation |
| Save/delete destroys entry history; no actor/reason/revision ledger | `services.py:909–966` | Stable identity, append-only revisions/audit, explicit correction records |
| Startup performs ad hoc migrations/seeding on reruns | `app.py:61–62`; `database.py:175–236` | Deployment-controlled Alembic migrations, no production startup `create_all()` |
| Legacy added columns omit FKs; Boolean defaults use SQLite-style 0/1; checks/invariants are incomplete | `database.py:77–149,199–236`; `models.py:10–162` | A PostgreSQL URL is not a tested migration path; reconcile before applying stricter constraints |
| Manager is free text; email is not unique identity; importer assignments are synthetic | `models.py:41–53`; `reference_import.py:290–331` | HR/IdP-backed identity and relationship reconciliation; never seed production access from demo assignments |

### Data semantics that must survive

1. **Legacy storage is naive UTC**, while naive service inputs are interpreted as local wall time. The legacy migration marker prevents repeated local-to-UTC conversion (`database.py:31–74`). A target import must not convert already-UTC values twice.
2. Timers can retain microseconds; manual intervals can represent arbitrary minutes, while weekly cells require quarter-hours. Preserve raw endpoints and source precision. Do not round all historical work to quarter-hours or import from rounded CSV (`models.py:111–114`, `app.py:87–103`).
3. Weekly grouping is project/task/notes; classification and hands-on distinctions can collapse. Notes-only edits can fail on fractional timer hours or replace historical intervals/classification (`app.py:122–190,705–734`). New API changes must not silently reclassify or retime history.
4. Copying adds seven UTC days and can move local dates across DST; occupancy keys omit notes and can skip distinct rows (`services.py:1024–1057`). Specify calendar-date copying separately from interval duration.
5. Nullable employees/tasks and legacy first-employee/first-task fallback attribution are not authoritative. Preserve source IDs, record anomalies, and require explicit mapping approval (`database.py:103–112,175–195`).
6. Host `date.today()` and ambiguous/nonexistent local times require an explicit company-timezone/clock policy (`time_utils.py:14–18`, `services.py:495–501`).

## Node: workflow parity matrix

“Adopt” below means adopt an approved behavioral requirement, not inherit implementation correctness.

| Capability | Python today | Node source behavior | Proposed disposition |
| --- | --- | --- | --- |
| Identity and RBAC | Unrestricted partner selector | Authenticated sessions, six capability roles; `src/lib/permissions.ts`, `src/lib/session.ts` | Add enterprise OIDC; deny by default; separate principal from employee |
| Record scope | Caller-supplied employee filters | Self, global manager/leadership, current-project PM, finance aggregates; `src/lib/reports.ts:214–279,363–395,426–441` | Explicit approved scope matrix; do not assume managers need organization-wide access |
| Weekly entry | Cells, notes, timers, submit/recall | Whole-sheet replacement; save may withdraw submitted/approved sheets; `src/lib/actions/week.ts:297–437` | Stable entry revisions, explicit transitions, versioned submission |
| Delegated entry | Impersonation without attribution | Admin owner override and audit; `src/lib/actions/week.ts:50–68,321–352` | Actor and subject distinct; required reason and atomic audit |
| Approval | No decision workflow | Submitted → approved/rejected; no self-approval; `src/lib/approval.ts:29–55`, `src/lib/actions/approvals.ts:95–181` | Add immutable decisions linked to submitted revision; define fallback routing |
| Period close | Open → Correction → Locked | Exception window, finalization, reopen; `src/lib/actions/close.ts:181–442` | Durable close cycles; entry-date locks, immutable snapshots; explicit overrides |
| Locked correction | Rejected edits | Reasoned updates plus correction/audit/restated marker; `src/lib/actions/week.ts:599–775` | Immutable adjustment/revision; restate every affected period |
| Classification | Task snapshot and hands-on rules | Effective-date resolver and stored classification; `src/lib/classification.ts:10–44` | Stable rule version, deterministic resolution, explicit historical reclassification |
| Audit | No durable change ledger | Transaction-aware insertion; `src/lib/audit.ts:3–44` | Complete atomic audit with database-role protections and retention |
| Notifications | Not implemented | SMTP/log-only, scheduled/manual reminders; `src/lib/reminders.ts:217–315` | Transactional outbox, leases/retries, durable deduplication and attempt history |
| Exports/reporting | Clipped totals and CSV | Role-aware CSV/XLSX; `src/app/(app)/reports/export/route.ts:42–203` | Same scopes as API; distinguish draft/approved/posted; restatement metadata |
| Fiscal/calendar | Hard-coded fiscal periods and holidays | Generated years plus static/process caches; `src/lib/fiscal.ts:16–40,137–195` | One authoritative database-backed, versioned calendar |
| AI helper | Heuristic | Advisory external model integration | Defer; optional only after privacy, egress, cost, and authorization approval |

## Node hazards to turn into acceptance tests

These are static-code findings; no exploitation or runtime verification was performed.

- **Privileged role administration:** managers pass people-management checks and can create another admin/set another person's privileged role (`src/lib/actions/people.ts:26–35,49–123,159–205`). New design separates security administration from personnel management.
- **Finalized period bypass:** close updates existing sheets, while save/copy/holiday creation checks sheet state, not authoritative period state (`src/lib/actions/close.ts:348–364`, `src/lib/actions/week.ts:86–121,155–158`, `src/lib/week-data.ts:336–379`). Lock must cover missing sheets and all automated writers.
- **Reopen loses evidence:** deleting close records cascades distribution history; restoration guesses from stale submission/approval timestamps; later destructive saves can cascade correction history (`src/lib/actions/close.ts:417–441`, `src/db/schema.ts:395–418,475–491`). Reopen must append a cycle/event, never erase history or restore obsolete approval.
- **Read-check-write races:** approval/save/close validate outside the protected write transaction (`src/lib/actions/week.ts:155–166,297–321`, `src/lib/actions/approvals.ts:47–104`). Use revision checks and consistent locks covering both validation and mutation.
- **PM email disclosure:** one full exception report is sent to every selected manager/PM (`src/lib/actions/close.ts:120–150`, `src/lib/close-report.ts:149–230`). Send recipient-scoped content or authenticated links.
- **Incomplete append-only guarantee:** audit table is ordinary mutable storage; save summary cannot reconstruct entry revisions; other history has destructive cascades (`drizzle/0002_faulty_tattoo.sql:1–16`, `src/lib/actions/week.ts:297–338`). Define application-role restrictions and external retention separately from a claim of absolute immutability.
- **Reminder failure suppression:** a scheduled employee/week claim suppresses future sends even after failure, crash, or simulated delivery; it is not once-per-day retryable delivery (`src/lib/reminders.ts:237–318`). PostgreSQL needs a unique business key and recoverable claim/lease state.
- **Approval routing ambiguity:** admin queue and decision eligibility differ; direct reports of admins can appear in a queue they cannot approve (`src/lib/approval.ts:29–55`, `src/lib/week-data.ts:621–628`). Define one policy used by queues and actions.
- **Close rule/test mismatch:** pure `canFinalize` and its tests differ from the action about which states block early close (`src/lib/close.ts:87–104`, `src/lib/close.test.ts:134–162`, `src/lib/actions/close.ts:283–345`). Test the actual transaction/API, not only parallel helper logic.
- **Calendar and accounting assumptions:** week-start attribution only works because current fiscal boundaries align with weeks; generated future calendars are not consistently used in close/reports. Holiday OOO contributes to actuals while reducing expected hours (`src/lib/fiscal.ts:16–40,137–195`, `src/lib/reports.ts:59–125`, `src/lib/holidays.ts:74–82`, `src/lib/actions/close.ts:83–101`). Obtain finance sign-off before encoding policy.

## Consequences for implementation sequencing

1. Characterize useful existing rules, but label known defects instead of freezing them as desired behavior.
2. Confirm identity, calendar, schedule/holiday accounting, duration precision, approval routing, licensing, and recovery requirements.
3. Establish migrations, transaction ownership, audit, and authorization before a writable shared API pilot.
4. Add timesheets/approvals/close as incrementally tested components, with real PostgreSQL concurrency and failure tests.
5. Reconcile a dry-run import and rehearse a single-writer cutover before accepting production writes.

See [official documentation research](references.md) for the proposed backend foundations. The implementation roadmap will distinguish recommendations from decisions approved for execution.
