# FastAPI and PostgreSQL migration: proposed implementation roadmap

Date: 2026-09-08. **Status: planning only; architecture proposed, NOT approved.**

The approved work is the fork, completed static source review, and planning documents. No application implementation, installation, test execution, database access, deployment, commit, push, PR, or new GitHub issue is authorized by this roadmap. Preserve the original Streamlit application and database. A future task becoming dependency-ready is not permission to execute it.

## Navigation

- [Evidence and scope](#evidence-and-scope)
- [Proposed architecture](#proposed-architecture)
- [Domain boundaries and relational model](#domain-boundaries-and-relational-model)
- [Identity and authorization](#identity-and-authorization)
- [Calendar, duration, and classification](#calendar-duration-and-classification)
- [Workflow versus fiscal finality](#workflow-versus-fiscal-finality)
- [Transaction and concurrency policy](#transaction-and-concurrency-policy)
- [Proposed API contracts](#proposed-api-contracts)
- [Outbox, reporting, and interim UI](#outbox-reporting-and-interim-ui)
- [Milestones and dependency gates](#milestones-and-dependency-gates)
- [Migration, cutover, and rollback](#migration-cutover-and-rollback)
- [Verification and production support gates](#verification-and-production-support-gates)
- [Unresolved decisions](#unresolved-decisions)
- [Planning validation and next approval](#planning-validation-and-next-approval)

## Evidence and scope

Read the [source assessment and behavior parity matrix](source-assessment.md) and [official documentation research](references.md) together with this proposal. Evidence is static, not runtime-verified or a security certification:

| Baseline | Immutable revision | Treatment |
| --- | --- | --- |
| Python fork [MrBrooks89/time_tracker_bgc](https://github.com/MrBrooks89/time_tracker_bgc) | `baae802434d92a92698878c14479bca6f21a54cd` | Preserve useful domain behavior and MIT notice; do not preserve known defects as requirements |
| Node behavioral blueprint | `ef8b37b3987b7760fc8d9d15b2b1d043d7a232c4` | Independently specify behavior; no Node application license found, so no source/test/documentation copying without rights-holder approval |

The review identifies missing identity boundaries, destructive edits, state/period coupling, and concurrency gaps; it does not prove SQLite or Streamlit intrinsically unsuitable. No production records, workbook contents, actual test results, performance, or deployment health were verified. Workbook/assets require provenance and finance validation; demo credentials, demo assignments, and a repository workbook are not production bootstrap data. Other user checkouts remain out of scope.

## Proposed architecture

```text
Company OIDC (provider/client topology TBD)
                 |
Per-user authenticated Streamlit adapter / optional future UI
                 | HTTPS, actor credential, untrusted target/filter inputs
FastAPI modular monolith: routers -> application commands/queries -> domain policies
                 | explicit unit of work; sync blocking DB path
PostgreSQL: business data + immutable ledgers + audit + durable outbox
                 |
One supervised outbox worker -> recipient-scoped SMTP / approved integrations

Separate deployment migration job: Alembic, elevated migration role
Separate recovery/monitoring controls: secrets, backups/WAL, restore drill, alerts
```

- Start with synchronous FastAPI `def` handlers and dependencies, request-scoped SQLModel/SQLAlchemy sessions, and Psycopg 3 via `postgresql+psycopg://`. One engine per process; never share a mutable session between requests, users, threads, or worker operations.
- Retain SQLModel where it can express the required mappings; use SQLAlchemy constraints, locking, and explicit SQL where needed. An ORM rewrite is not a prerequisite. Separate persistence models from Pydantic v2 input/output schemas; use pydantic-settings with fail-closed validation.
- Application commands own transaction boundaries and commit **before returning a success response**. Dependency teardown rolls back/cleans up, never performs the success commit. Read endpoints cannot create records as a side effect.
- Alembic migrations run once under deployment control, not on application startup; review generated candidates, including renames, data conversions, checks, indexes, and lock impact. No production startup `create_all()` or ad hoc seeding.
- Exact stable Python, FastAPI, SQLModel, SQLAlchemy, Psycopg, Alembic, Pydantic/settings and PostgreSQL versions are TBD. PostgreSQL 18/17 are candidates, subject to hosting/support and a verified compatibility matrix. Research identifiers, especially the Psycopg development documentation version, are not release pins.
- Default topology is one deployable backend plus one durable PostgreSQL outbox worker. No default microservices, Redis, Kubernetes, or async rewrite. Hosting, replicas and worker count require measurements and operations approval.
- Budget maximum connections as the sum of `replicas × API processes × (pool_size + max_overflow)`, worker pools, migration/reporting/recovery tools, and reserved administration capacity. Keep this below the approved database limit with headroom; include thread concurrency, timeouts, backpressure and long reports. Never size each worker as though it owns the entire database.

## Domain boundaries and relational model

Proposed logical tables, not approved migrations. Modules own their writes; cross-module commands coordinate through one application unit of work, not independent commits. Reporting consumes scoped projections and immutable snapshots, not UI-specific business rules.

| Module | Proposed relations and keys | Boundary/invariants |
| --- | --- | --- |
| Identities / people / assignments | `principal` unique `(issuer, subject)`; `person`; explicit principal-person link; capability grants; effective-dated manager, project assignment and delegation records | Email/free-text manager is not identity. Separate security grants from personnel maintenance; retain effective-date and change provenance |
| Calendar / schedules | `calendar_version`, `work_date` unique `(company_id, date)`, `fiscal_period`, `work_week`, `schedule_version`, dated holiday/leave policy | Every writable date maps to exactly one published period/version and Wed-Tue week. Validate complete coverage/non-overlap; freeze published versions referenced by history |
| Entries / timers / timesheets | `timesheet` unique `(person_id, week_id)` with version/state; stable `entry`; append-only `entry_revision`; date allocations; timer session/raw interval | Entry identity survives edits/voids. Revision and allocation FKs bind explicit person/week/date; timer source interval may allocate across dates/weeks/periods |
| Classification version ledger | `classification_rule_version`, effective rule ranges, immutable classification snapshot referenced by allocation/revision | Rule resolution uses work date and approved inputs. Historical rows do not change when rules change |
| Approval | immutable `submission`, submission-revision membership, routing snapshot, `approval_decision`, recall/amend events | Decision references the exact submitted version; no approval inferred from timestamp, lock, or administrator role |
| Fiscal close / restatement | `period_guard`, `close_cycle`, immutable close snapshot + lines, `correction`, versioned restatement, reopen event | Guard exists independently of sheets. No deleting cycles, distributions or corrections; original and restated reporting remain addressable |
| Audit + outbox | `audit_event`, `outbox_message`, delivery attempts, idempotency record | Business mutation, audit and outbox insertion are atomic; delivery attempts are durable and recoverable |
| Reports / settings | versioned policy settings, report/export requests and snapshot references where needed | Shared authorization policy for API/export/email; explicit draft, approved and posted views |

Use opaque stable public IDs; preserve original source IDs separately with unique `(source_system, source_table, source_id, import_batch)` mapping/provenance. Surrogate type is a phase-0 decision; do not regenerate IDs on each import or revision. Review PK/FK/NOT NULL/UNIQUE/CHECK constraints, FK indexes, nonoverlapping effective-date ranges, finite numeric values, valid interval ordering, positive versions and one running timer per person. A CHECK does not replace NOT NULL. Avoid cascading deletion of financial/audit evidence. Person/project/task archival preserves references.

Allocations link each immutable revision to work dates and the corresponding sheet(s); cross-week intervals cannot be attributed solely to their start date. A note-only change preserves interval, allocation and classification unless an explicit authorized change requests otherwise. A tombstone revision hides a voided entry from current views without deleting its history. Snapshot membership records the exact revision IDs and classification/calendar versions, not a query against today's mutable state.

## Identity and authorization

Company OIDC is required; provider, provisioning and browser/Streamlit/API topology remain undecided. No custom password store, JWT issuer, or prototype-derived login system. Validate provider-specific issuer, audience, signature/keys, expiry, allowed algorithms and session/token type; enforce MFA through company policy. Specify deprovisioning, group mapping, key rotation, stale grants, logout/revocation latency, and cookie/CSRF/CORS boundaries before implementation. Never treat an ID token as a general API access token without an approved provider contract.

The backend derives the **actor** from verified authentication. An employee/target ID is untrusted input requiring record-level authorization. Audit actor, target, delegated authority and reason separately. Default deny; capability and scope are both required. Proposed least-privilege matrix:

| Capability | Allowed scope | Explicit denials |
| --- | --- | --- |
| Employee entry/read/submit | Own authorized dates/sheets and assigned project/task choices | Another person's data; setting actor, role, approval, classification snapshot or period state |
| Manager personnel/read/decision | Approved effective-dated report relationships; decision must also match frozen submission route and current capability | Organization-wide access by default; privileged role administration; self-approval |
| Project manager | Approved assigned-project reporting slice; entry detail only if separately granted | Other projects, whole-person totals through joins, broad exception email, automatic timesheet approval |
| Finance reporting | Approved aggregates only, including export/email | Employee rows, individual entry notes, drill-down/filter combinations that defeat aggregate-only scope |
| Finance close/correction | Separate close/restatement capability within finance scope | Generic admin override of finality; obtaining raw employee queries merely by holding aggregate reporting capability |
| Delegated entry | Explicit grant, effective dates, authorized target and mandatory reason | Impersonation without attribution; delegation implying approval |
| Security administration | Explicit separate security capability and audited role changes | Personnel-manager role alone granting roles; implicit financial edit/close privileges |

No-self-approval applies to **all actors, including administrators**; compare authenticated actor-person linkage with the subject, not a caller-supplied identity. Proposed additional separation: an actor who materially prepared a delegated submission cannot approve that revision; owner confirmation is required. Queues and decision APIs use the same eligibility predicate. No eligible manager, self-route, missing relationship or reassignment ambiguity fails closed into a pending exception route, not an admin auto-approval.

Effective-date proposal: schedule, assignment and classification eligibility resolve per work date using half-open date ranges `[valid_from, valid_to)`. Report visibility requires current permission plus explicit historical scope; a previous assignment alone does not grant access forever. Approval routing snapshots the eligible relationship for the submitted dates, then rechecks current authority at decision time. Midweek manager changes may require split routing or a designated approver: D03 must choose one. Retroactive relationship changes append history and an explicit reroute event; they cannot silently rewrite old decisions. Fallback approvers, delegation expiry, overlapping relationships, and historic manager visibility are approval blockers.

Apply scope before aggregation, pagination, counts, joins and export generation, not after fetching unrestricted data. Finance aggregate endpoints use an approved dimension allowlist; small-cell suppression and anti-inference policy require finance/privacy approval. An independently granted employee capability must not turn a finance report into a detailed query. Permission to administer security is not a blanket bypass to finance finality.

## Calendar, duration, and classification

- Preserve current **Wednesday-Tuesday** work weeks. The published period pattern is **not a standard repeating 4-4-5 calendar**; do not replace it with a generic generator. Finance must supply/version the authoritative dates, 53-week treatment, future years, and cross-period week rules. Calendar gaps or ambiguous mappings reject writes/imports; no fallback to the week-start period.
- Allocate to periods by each **work date**, even when a week straddles periods. Preserve named company timezone alongside UTC instants. Model dates as `date`, instants as `timestamptz`, and elapsed duration as exact integer microseconds (serialized as a decimal string) or an explicitly approved equivalent exact type.
- Timer durations retain seconds/microseconds; weekly manual allocations accept integer quarter-hour units. Preserve legacy arbitrary-minute manual intervals and raw endpoints with source precision. Do not blindly round, convert via float, reconstruct from rounded CSV, or force legacy timer values into quarter-hour cells. Reporting may format values, but reconciliation uses exact underlying units.
- Legacy persisted timestamps are **naive UTC**; naive legacy service inputs had different local-wall-time semantics. Preserve raw values, the source timezone, migration-marker value/presence, conversion rule and batch provenance. No second timezone conversion. Unknown provenance is quarantined rather than guessed.
- Proposed new instant inputs require offsets; local-wall-time convenience inputs require a named zone and explicit fold selection when ambiguous. Reject nonexistent DST times instead of shifting them. Copy by local calendar dates and approved allocation semantics, not by adding seven UTC days; copying timer intervals needs an explicit policy. Clock injection replaces host `date.today()` in decisions/tests.
- Define scheduled paid hours, expected **worked** hours, holiday OOO and other leave as distinct quantities. Finance/HR must decide whether holiday allocation appears in actual worked totals or only paid/leave totals; do not simultaneously reduce expected work and count holiday as worked without an approved definition. Daily caps, overlapping intervals, part-time changes and approved exceptions require explicit policy.
- Classification resolver returns a rule-version ID and immutable result per dated allocation. Notes-only edits and unrelated task changes do not reclassify history. Historical reclassification requires reason, new revision and affected-period restatement if posted. Missing/overlapping rules fail closed; no first-task/first-person attribution fallback.

Unresolved calendar, timezone, schedule/leave and precision policies block the affected entry, classification, approval, reporting and migration modules, not merely documentation polish.

## Workflow versus fiscal finality

Weekly workflow and period control are separate state machines. **Locked is not approved.** A sheet may intersect open and closed periods; APIs return both workflow state and date-level editability/reasons.

| Command | Weekly transition proposal | Preconditions and retained evidence |
| --- | --- | --- |
| Ordinary edit/copy/void | `draft -> draft` or `in_correction -> in_correction` | Expected version, scope, all affected periods writable; immutable revision and audit |
| Submit | `draft/in_correction -> submitted` | Revalidate all dated allocations, totals and route; freeze revision membership; no active timer |
| Recall | `submitted -> draft` | Explicit recall command, expected submission/sheet version, before a valid decision and only while affected periods permit; preserve submission/recall history |
| Reject | `submitted -> in_correction` | Authorized non-self approver; immutable reasoned decision against exact submission |
| Approve | `submitted -> approved` | Same eligibility as queue; exact revision match and non-self check |
| Amend approved work | `approved -> in_correction` | Explicit reasoned amendment, authorized scope and open periods; old approval remains tied to old revisions, never applies to edited content |
| Resubmit/reapprove | `in_correction -> submitted -> approved` | New submission and decision; no stale approval restoration |

Proposed period lifecycle is `open -> correction_window -> closed`; correction windows require explicit scoped permissions, not universal edit access. Close validates actual transaction state and records exceptions, cutoff and snapshot. Proposed finalization rejects unresolved drafts/reviews/running timers unless finance approves a documented exception rule; never change those sheets to approved merely to close.

Closed-period edits use a **separate correction/restatement command**, mandatory reason and finance correction capability, not ordinary save or an admin flag. Record adjustment/replacement revision, prior/new amounts, affected dates, original snapshot reference, approval requirements and restatement version. Every affected closed period needs authorization and a new restatement; cross-period operations are all-or-nothing. Whether correction approval is two-person and how reapproval interacts with posted deltas must be settled in D06.

Reopen appends an authorized reasoned event and a new cycle referencing the prior close; it never deletes snapshots/distribution history or resurrects old approvals. Original closed reports remain immutable and selectable. Ordinary writes remain prohibited until the approved reopen/correction protocol enables the relevant dates. No blanket administrator bypass.

## Transaction and concurrency policy

Proposed baseline favors correctness over fine-grained throughput: authoritative period rows serialize writers within an affected period. Optimize only after real PostgreSQL race/load evidence and a reviewed replacement protocol.

1. Authenticate and perform any necessary provider/network work **before** obtaining business locks. Open an explicit application unit of work before ORM reads that might autobegin. Resolve candidate old/new dates from persisted revision plus request; this is a planning read, not final validation.
2. Lock in one global order: company authorization-policy guard (shared for commands, exclusive for grant/relationship changes); company calendar guard (shared for commands, exclusive for calendar publication); **all affected period guards ordered by company and period ID**; affected people ordered by ID; sheets ordered by person/week; entries/timers ordered by ID. Use `FOR UPDATE` for period/person/sheet/entry write guards. Relevant effective-date policy changes take the appropriate exclusive guard in the same order. Never acquire earlier-level locks after later-level locks.
3. Every affected old **and** new work date must map through the locked published calendar to a durable period guard, even if no entry or timesheet exists. Precreate guards with calendar publication; missing dates/guards reject the command rather than create an unlocked substitute. Close/reopen takes the same period guards. Writes cannot bypass close by creating the first sheet, holiday allocation or timer result.
4. Under those locks re-read current grants, mappings, revisions, versions, workflow and period permissions; revalidate ownership, assignments, totals/caps and all transition preconditions. If the candidate affected-date set changed while waiting, abort/retry from a fresh transaction; do not add locks out of order. Use the employee guard to serialize running-timer uniqueness and person-level totals; DB constraints remain a second defense.
5. Require `expected_version`/`If-Match` on mutable aggregates; increment sheet/entry versions with compare-and-swap checks. Missing precondition is rejected, stale version is an edit conflict, never last-write-wins. Bulk changes, deletes, copies, imports, holiday jobs, timer stop, submission, decisions and actual close APIs use this protocol; ORM automatic versioning alone does not cover arbitrary bulk DML.
6. Write immutable revisions/decisions/snapshots, mutable current pointers, audit and outbox in the **same transaction**. Roll back all on any failure. Commit before returning success; teardown only closes/rolls back. A socket failure can still occur after commit: clients recover through idempotency, not blind repetition.
7. Release DB locks before external HTTP/SMTP or expensive rendering. Bound statement/lock timeouts and retry only recognized deadlock/serialization failures with bounded backoff and unchanged idempotency identity. Do not retry stale-version or authorization errors as successes.

Close holds its period guard while validating all relevant rows and committing the snapshot. Since all writers need the same guard, a save either commits first and is included in close validation, or observes closed state and fails. Missing sheets are not a loophole. Submission versus save and decision versus amendment serialize through the same sheet/version protocol. Multi-period moves lock old and new periods together; never unlock one side to complete the other.

Running timers have no known final end date. Proposed close rejects any active timer whose open interval could affect the period; authorized stop must allocate every affected date under the same protocol first. Start/stop must respect person guards, dates and period guards; a timer stopped after a period closes cannot silently post into it. Long-running/abandoned timer resolution and exception route are D05/D06 decisions. Broad guard contention and close duration are explicit M1/M4 load risks, not assumed solved.

## Proposed API contracts

Examples are contract proposals, not implemented routes. `/api/v1`, noun resources, explicit schemas and generated OpenAPI after approval. GETs are side-effect-free.

| Example endpoint | Contract boundary |
| --- | --- |
| `GET /api/v1/weeks?employee_id=...&start_date=...` | Scoped Wed-Tue sheets, versions, weekly state and per-date period controls |
| `POST /api/v1/entries`, `PATCH /api/v1/entries/{entry_id}` | Stable entry identity, separate revisions; employee target authorized, actor server-derived; no replace-all history destruction |
| `POST /api/v1/weeks/{week_id}/entry-batches` | Atomic multi-cell/copy command with expected sheet/entry versions; all changes pass or none persist |
| `POST /api/v1/timer-sessions`, `POST /api/v1/timer-sessions/{id}/stops` | Exact raw instants and durable stop result; no implicit rounding |
| `POST /api/v1/weeks/{week_id}/submissions` | Freeze exact revision set and routing snapshot |
| `POST /api/v1/submissions/{id}/recalls`, `POST /api/v1/submissions/{id}/decisions` | Explicit recall or reasoned approve/reject; no-self-approval and optimistic precondition |
| `POST /api/v1/weeks/{week_id}/amendments` | Explicit reasoned amendment; requires subsequent submission/reapproval |
| `POST /api/v1/close-cycles`, `POST /api/v1/close-cycles/{id}/finalizations` | Scoped finance capability, period guard/version and immutable snapshot |
| `POST /api/v1/close-cycles/{id}/reopenings`, `POST /api/v1/corrections` | Reasoned new event/cycle or correction, never delete a close |
| `GET /api/v1/reports`, `POST /api/v1/report-exports` | Same record/aggregate scopes; explicit date range, draft/approved/posted view and snapshot/restatement version |

Wire rules to finalize in phase 0:

- Opaque IDs and large integer microsecond values are strings; Decimal hours are canonical finite base-10 strings, never JSON floats. Quarter-hour count is a bounded nonnegative integer for that input mode only. Signed deltas are confined to authorized corrections. Fix maximum magnitudes and rounding-at-display rules before implementing validators.
- ISO `YYYY-MM-DD` work dates, offset-required RFC 3339 instants, up to six fractional second digits, and separate IANA zone/calendar version. State whether interval endpoints are half-open; proposed `[start, end)`. No silent truncation of excess precision. Null, omitted and zero are distinct.
- Updates require ETag/`If-Match`; command bodies may carry `expected_version` when multiple aggregates are involved. If both are present they must agree. Proposed errors: 428 missing precondition, 412 stale version, 409 invalid transition/closed period/idempotency mismatch, 422 invalid fields, 401 unauthenticated, 403 denied capability, 404 inaccessible record where existence must be hidden, 429 rate limit and retryable 503 infrastructure contention. Never leak another employee's version or data in an error.
- Consistent problem response: `type`, `title`, `status`, stable `code`, `request_id`, and sanitized field errors. Successful envelopes include `data` and applicable `meta` (version/calendar/snapshot); fixed allowlisted filters/sorts and bounded cursor pagination ordered by stable tie-breaker. Cursor is opaque and scoped to actor/query; do not expose unauthorized counts. Export size/runtime limits require an approved budget.
- Require idempotency keys for command POSTs and retryable batch mutations. Unique scope includes company, actor, operation and key; compare a canonical payload hash including target and expected versions. Same key/same payload returns the original status/result/IDs, without duplicate revision/audit/outbox; same key/different payload returns 409. Recheck present authorization before disclosing a replayed result. Concurrent claims serialize on a DB uniqueness constraint.
- Store the successful result atomically with business commit. An in-flight duplicate waits within a bound or receives a retryable pending response; it must not execute twice. Document retryable failures and retention: financially material command deduplication must outlive the approved retry/recovery window; key expiry/reuse must never silently duplicate a financial action. Retention and maximum payload sizes are D08 decisions. A response can be lost **after commit**.

## Outbox, reporting, and interim UI

One supervised worker consumes the PostgreSQL outbox using short transactions and recoverable leases, with unique business keys such as `(event_type, subject_id, revision_or_cycle, recipient_scope, scheduled_occurrence)`. Lease claims may use `FOR UPDATE SKIP LOCKED`; commit claim, deliver outside locks, then persist attempt/outcome. Expired leases recover after crashes; retries have bounded backoff, attempt history, alerts and authorized replay/dead-letter handling. New sessions per worker operation.

SMTP is **at-least-once**, not exactly-once: a crash after acceptance but before recording success may cause duplicates. Stable message IDs can help but do not prove deduplication. Dry-run/simulation uses separate nondelivery records/namespace and never consumes a live success/deduplication claim. Suppression, retry and scheduling keys distinguish intended occurrences; a failed reminder must not suppress all future reminders.

Recipient scope is derived and rechecked before rendering/sending. Prefer authenticated links over sensitive email attachments; never distribute a full company exception report to every PM. If grants changed, cancel/restrict delivery or require a new authorized export, rather than leaking a queued payload. Report/export artifacts need scoped access, expiry, retention and formula-injection-safe CSV/XLSX output. Posted reports specify original or restated snapshot; current draft reports must be visibly different.

Streamlit may remain an interim UI **only through a per-user authenticated API**, with the backend as sole writer. A global administrator service token plus an employee dropdown is forbidden. Prove actor propagation, session isolation, token renewal/logout, deprovisioning and CSRF/cookie boundaries before a shared writable pilot. Do not cache sessions or user-specific results globally. If topology cannot be proven, permit only a separately approved, controlled read-only pilot on an isolated snapshot with reviewed access; do not expose the old unrestricted selector as a safe shared UI. A future UI replacement is an independent decision, not a migration prerequisite. AI/external advisory features are deferred pending separate privacy/egress approval.

## Milestones and dependency gates

All milestones below are **unstarted**. Completed source review is an input to M0, not evidence that M0 or implementation is complete. Each milestone gets a detailed component plan and explicit execution approval when prerequisites are met. Later phases are deliberately not fabricated one-hour subtasks.

| Milestone | Scope / dependencies | Gate: evidence required before advancing |
| --- | --- | --- |
| **M0 Decisions/baseline** | [Phase-0 design/foundation plan](phase-0-foundation.md); uses pinned evidence | Owned decision register, contracts, schema/locking/state proposals, reconciliation/test catalogs, operations requirements and approval packet. Blocking decisions resolved for the next authorized slice; no runtime claim |
| **M1 PostgreSQL + API skeleton, migrations/CI** | M0 foundation/architecture approval; stable compatibility choices | Later real-Pg migration up/forward-compatibility tests, constraint smoke tests, request-session isolation, fail-closed settings, commit-before-response/rollback test, dependency lock and CI lint/type/test/migration/scan evidence. Pool/timeout budget reviewed |
| **M2 Identity/scope/audit** | M1; approved OIDC topology and scope matrix | Provider integration and revoked/invalid identity tests; negative role × record × endpoint/export/email matrix; actor/target audit; audit rollback and runtime-role denied UPDATE/DELETE. No writable shared pilot yet |
| **M3 Entries + classification** | M2; signed calendar/schedule/precision rules | Exact timer/manual allocations, stable revisions, effective-date rules and assignment checks; real-Pg save/save, first-sheet/close guard, timer/period and DST copy tests; no destructive history |
| **M4 Approvals + close** | M3; approved routing/finality/correction policies | Actual submission/decision/close API race tests, no-self-approval including admin, missing-sheet close protection, immutable original/restated snapshots, reasoned correction/reopen/reapproval tests |
| **M5 Reporting/jobs/UI adapter** | M4 for final integration; M2 scope + M3 stable contracts for isolated preparation | Aggregate-only finance, scoped exports/email, idempotency and worker crash/retry/simulation tests; authenticated Streamlit adapter isolation proof or explicitly restricted read-only pilot. UI choice independently approved |
| **M6 Migration rehearsal/pilot/production** | M1-M5 gates; finance/security/operations approvals | Repeated reconciled dry runs, restore drill, authorized timer resolution, single-writer rehearsal, approved pilot and support ownership; finance signs exact totals/expected differences and go/no-go |

Critical path: `M0 -> M1 -> M2 -> M3 -> M4 -> M5 -> M6`. Limited overlap after stable contracts: operations/runbook design; migration mapping design; isolated outbox/report contract drafting. Do not parallel-edit shared schema/UoW/authorization modules. No reporting or UI integration bypasses M2 scopes, M4 finality, or the pilot gate. M0 JSON tasks describe document ownership and their own dependency DAG.

Full delivery estimate is unknown until team capacity, IdP/HR/finance availability, data volume/quality, hosting and support requirements are known. Phase-0 authoring estimates are bounded in the component plan; they exclude decision wait time and all implementation. Re-estimate each later milestone after its gate, not with an invented fixed completion date.

## Migration, cutover, and rollback

### Staged, repeatable dry runs

1. Obtain authorized access to a **consistent read-only source copy**, not the live DB or rounded exports. Record acquisition method, snapshot boundary, file/schema hashes, row counts, source commit, SQLite consistency/WAL handling, timezone and legacy migration-marker value/presence in an immutable manifest. Hash sensitive evidence without logging raw records; store copies under approved retention/access controls.
2. Version the mapping specification: employees/principals/managers/projects/tasks/assignments/calendars/classification, source-state interpretation, source IDs to stable target IDs and import-batch lineage. Preserve raw timestamp endpoints and exact duration precision before any conversion. Never bootstrap access from synthetic assignments or fallback first rows.
3. Inventory invalid, null, duplicate, orphaned and ambiguous rows in quarantine with source identity, reason, exact raw values and review disposition. Do not silently drop, merge or impute. Finance/HR/security approve mappings and expected differences; unresolved material anomalies block cutover. Preserve original data and document where stricter target constraints intentionally reject source defects.
4. Load an isolated disposable target in reviewed FK order. Preserve classification snapshots; when a rule version cannot be established, mark an approved legacy snapshot/provenance rather than invent history. Preserve source IDs/mappings and initialize/check target sequences above imported values. Record row/FK/index/constraint results and repeatability on a fresh target.
5. Explicit source-state map: Python **Not Started -> draft** and **In Progress -> draft**, retaining the original status as import provenance (`models.py:60`, `services.py:469–474`). Submitted -> submitted with an import-marked submission snapshot, **not a fabricated historic approver**; In Correction -> in_correction with imported-state provenance. The services also accept weekly **Open** (`services.py:24–30`), but its persistence provenance is unestablished: intentionally quarantine that legacy value pending an approved mapping to draft, and do not confuse it with accounting-period Open. Locked -> date-level period lock with imported lock provenance, **not approved**. Period Open/Correction/Locked map separately to open/restricted correction/closed controls after finance review (`services.py:640–670`). Preserve each `correction_deadline` and its raw date/timezone interpretation (`models.py:75`, `services.py:495–501`); expired windows remain noneditable, and missing or ambiguous deadlines require review rather than a fresh window. Any other source values are quarantined; Draft is not an asserted source status. Period transitions overwrite weekly states (`services.py:661–670`), so a locked sheet's preceding workflow cannot be reconstructed reliably. Missing approval/audit history is recorded as unknown; only the import event has the real migration actor/time. Legacy closed periods stay locked; draft/review sheets within them remain explicit exceptions, not automatically approved/unlocked.
6. Reconcile source/target PK and FK counts, quarantined counts, exact duration totals by employee, date, Wed-Tue week, period, project and classification, plus combined dimensions capable of exposing compensating errors. Include allocation conservation against each raw interval, classification snapshot equality, active-timer inventory, state/lock maps and sequences. Record expected differences separately with reason/owner; no unexplained tolerances or net-total-only signoff.
7. Exercise DST gaps/folds, midnight/cross-week/cross-period boundaries, naive-UTC already-converted markers, fractional timers, null/duplicate fixtures, missing calendar/rules and locked legacy draft exceptions. Rehearse restart/repeat import behavior and signed manifest identity. A successful dry run does not authorize live migration.

### Single-writer cutover

- Agree freeze window, owners and communication; test backup/PITR restore first. Stop active timers through authorized, recorded actions **before freeze**, or approve an explicit pending-timer migration/recovery state preserving start, owner and raw provenance. Never silently discard or synthesize stop times.
- Disable every legacy writer (UI, jobs, imports, manual paths), not just the new UI; verify the freeze. Take the final consistent snapshot and manifest. A final delta import is allowed **only if** reliable change/delete capture and replay were demonstrated during rehearsal; otherwise repeat a full snapshot import into a clean target. No dual-write.
- Reconcile again, have finance approve exact totals/state exceptions and mappings, security approve scope/identity, and operations approve restore/runbooks. Document go/no-go before enabling target writes. Keep old application/data preserved and read-only with a visible frozen-data warning, not an alternative writable system.
- Enable target as the sole writer for the approved pilot/production cohort only when access isolation is proven. Pilot rollout must not split writers over overlapping employees/dates or shared finance periods. Observe integrity, queue, error and reconciliation checks; name the authority that can halt writes.

### Rollback boundary

**Before target business writes:** keep target disabled, preserve migration evidence, verify source freeze boundary and restore/re-enable the unchanged legacy writer through an authorized switch. Readiness checks must confirm only one writer. Never automatically unlock legacy closed periods.

**After target business writes:** stop target writes, preserve a consistent target backup/WAL, new revisions, approvals, audit/outbox delivery history and idempotency records. Inventory and reconcile every post-cutover change and delivered side effect. Blindly restoring the old SQLite snapshot loses business records and is forbidden. Reverse mapping/replay requires a separately rehearsed, approved process and finance signoff; if legacy cannot represent new workflow/history, keep both preserved and choose a forward fix or controlled read-only outage. Already-sent SMTP cannot be rolled back. Restore/replay must account for pending/delivered outbox claims to avoid uncontrolled duplicates. Never erase closes or unlock periods as a rollback shortcut.

## Verification and production support gates

These are future acceptance requirements, **not tests run in this planning session**.

| Gate | Required evidence |
| --- | --- |
| Database integrity/concurrency | Disposable real PostgreSQL, real constraints, independent sessions/barriers for save/save, save/submit, approve/amend, close/save on missing sheets, timer stop/close, multi-period moves and duplicate commands. Invoke actual command/API close path, not a divergent helper |
| Precision/effective dates | Injected clock/zone; DST gaps/folds, no shifted seven-day copies, overlapping/gapped effective dates, holiday/expected-work separation, exact microsecond conservation and approved legacy differences |
| Security scopes | Negative role × record × API/export/email tests, cross-person/project filters and counts, finance aggregate-only, privileged role escalation denied, admin self-approval denied, revoked identity/grants fail closed |
| Transaction evidence | Forced failure rolls back business + audit + outbox + idempotency together; success commits before response; network loss after commit replays same result; no external delivery under DB lock |
| Ledger protection | Runtime and worker DB roles cannot UPDATE/DELETE audit/revision/decision/close evidence; no destructive cascade paths. Separate controlled migration/retention role. Tamper-evidence and protected backups improve assurance; no claim of absolute DBA-proof immutability |
| Worker recovery | Duplicate claims, crash before/after SMTP, expired lease, retries/dead-letter/replay, revoked recipient scope and dry-run nonpoisoning; at-least-once documented |
| CI/deployment | Approved stable lockfile; lint, type checks, unit/real-Pg integration/contract tests, migration checks, dependency/security/license scans; reviewed migration job and compatible deployment/forward-fix strategy |
| Configuration/secret handling | Production fails readiness for missing/unsafe identity/DB/calendar settings; no demo credentials or workbook seeds; secrets manager/approved injection, rotation, TLS client/proxy/DB, strict origins and least-privilege runtime/migration/worker roles |
| Recovery | Encrypted base backups + continuous WAL/PITR with monitored failures, retention/access controls and a timed restore drill. Named owner approves RPO/RTO; no invented SLA and no claim pg_dump alone provides PITR |
| Load/observability | Owner-approved SLO, concurrency/latency/export-size/queue-age budgets and representative load tests. Pool exhaustion, lock waits, failed close/import and queue lag alerting. Logs contain opaque correlation IDs, not PII, entry notes, tokens or email payloads |
| Support | Named service/data/security/finance/on-call owners, incident and escalation routes, runbooks for failed migration, restore, stale lease, duplicate email, close correction and auth outage; owner-approved dependency patch cadence and periodic EOL review |

No production gate passes on screenshots, unit tests alone, a helper-only close test, or static source review. Database owners retain privileged capabilities; audit protection claims must state the threat model and retention controls. Support signoff includes training, accessibility/workflow acceptance for any later UI, and a supported response to data-integrity incidents.

## Unresolved decisions

All owners below are required **roles**, not assigned people. Record named owners and dated approvals in phase 0. Until resolved, defaults deny unsafe behavior rather than guessing.

| ID | Decision and required owner | Blocks |
| --- | --- | --- |
| D01 | Architecture/topology, stable dependency matrix, SQLModel sufficiency, hosting/PostgreSQL major and pool budget — engineering + operations | M1 implementation approval |
| D02 | Company OIDC provider/client topology, identity-person mapping, provisioning/deprovisioning and Streamlit actor propagation — security/IdP owner | M2; any shared writable pilot |
| D03 | Effective-dated manager/assignment visibility, midweek routing, fallback approver, delegated preparer separation, role administration — HR + security + workflow owner | M2 scopes, M3 eligibility, M4 decisions |
| D04 | Authoritative fiscal calendar (nonstandard pattern), Wed-Tue/53-week/cross-period rules, timezone, publication control — finance | Calendar, entries, close, migration |
| D05 | Schedule/holiday/OOO worked-vs-paid semantics, daily caps/overlap, quarter-hour vs raw precision, DST/copy and active-timer handling — HR + finance | M3, reporting, timer freeze/migration |
| D06 | Close blockers/exceptions, recall/amend, closed correction/reapproval, two-person control, restatement/reopen and legacy state exceptions — finance + security | M4 and cutover |
| D07 | HR/source mapping authority, quarantine resolutions, legacy provenance and asset/license rights — data owner + finance + rights holder | Import and any Node/asset copying |
| D08 | API limits/decimal bounds, idempotency retention, outbox occurrence/retry policy, recipient scopes and email/export retention — API + security + operations | M3/M5 contracts and recovery |
| D09 | Aggregate dimension allowlist/small-cell suppression, original/restated reporting and distribution — finance + privacy | M5 finance queries/export/email |
| D10 | Hosting/secrets/TLS, named support owners, RPO/RTO, SLO/load budget, retention/patch cadence/EOL reviews — operations + business owner | Production and restore signoff |
| D11 | Interim authenticated adapter vs controlled read-only pilot, optional future UI, cohort and single-writer cutover/rollback authority — product + security + finance | Shared pilot and M6 |

## Planning validation and next approval

Next component: [Phase 0: design and foundation contracts](phase-0-foundation.md). Tracking: [task.json](../../.tmp/tasks/fastapi-migration/task.json), with 12 future design-only subtasks, all pending and unassigned. Parent `active` means an active **planning record**, not approved implementation or a completed milestone.

The required task CLI status attempt failed in cached ts-node configuration with `TypeError: Cannot read properties of undefined (reading 'fileExists')` before it reported status. The user explicitly approved continuing with direct file/manual JSON validation and no tooling repair. No CLI retry, npm/npx execution, installation, application test or database execution is part of this continuation. Direct directory reads found no existing task/planning/contract output directories or ADR directory in this checkout; no planning-agent decisions were invented.

Validation is limited to reading the documents/JSON against the supplied task schema, checking fields/counts/dependency order, reference separation and scope. It does not establish parser/CLI certification, runtime correctness, source-data reconciliation, unchanged-worktree status via Git, or production readiness. Stop and report any actual validation failure rather than automatically fixing it without approval.

Request review of this proposal and authorization for the phase-0 design tasks. **A separate explicit approval of the bounded M1 component plan, named decisions and permitted environment/actions is required before any implementation, installation, tests, migration or application execution.** No approval is implied by this document or task completion.
