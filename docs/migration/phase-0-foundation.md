# Phase 0: design and foundation contracts

Date: 2026-09-08. **Proposed next component; all tasks unstarted. Documentation only.**

[Master roadmap](implementation-plan.md) · [Static source assessment](source-assessment.md) · [Official references](references.md) · [Task record](../../.tmp/tasks/fastapi-migration/task.json)

## Navigation

- [Objective and approval boundary](#objective-and-approval-boundary)
- [Inputs and context boundary](#inputs-and-context-boundary)
- [Atomic task plan](#atomic-task-plan)
- [Contract design checklist](#contract-design-checklist)
- [Acceptance-test design](#acceptance-test-design)
- [Review gates and implementation handoff](#review-gates-and-implementation-handoff)
- [Tracking and validation limitations](#tracking-and-validation-limitations)

## Objective and approval boundary

Produce a reviewable foundation decision/contract packet that can support a **separate** approval for M1 PostgreSQL/API skeleton implementation. Do not implement M1 under a task named “foundation.” This phase's deliverables are Markdown design artifacts only; no Python, SQL migrations, executable fixtures, CI configuration or dependency lockfiles are created by these tasks.

The current authorization permits these two roadmap documents and the initial task definitions. The 12 future tasks below remain pending; they do not assert that stakeholder interviews, decisions, designs, implementation or tests have happened. Seek confirmation before starting the next design work. Completion of the packet is distinct from owner approval, and owner approval of a design is distinct from authorization to install/run/change software.

Out of scope: application changes, database reads/writes, local/remote deployment, dependency installation/repair, source-code test execution, commits/pushes/PRs/issues, changes to the original app or unrelated checkouts, new frontend styling/implementation, and copying Node code. Later runtime tests are specified here for future approval, not executed now.

## Inputs and context boundary

- Use the supplied session context at `.tmp/sessions/2026-09-08-fastapi-migration/context.md`; do not modify or create session bundles.
- Use the pinned Python `baae802434d92a92698878c14479bca6f21a54cd` and Node `ef8b37b3987b7760fc8d9d15b2b1d043d7a232c4` evidence already captured in the source assessment. Do not redo discovery or inspect the unrelated Node checkout.
- `context_files` in task JSON are standards paths only, selected from the session's Context Files. `reference_files` are existing source/docs only, relative to this approved checkout; new task outputs are named separately in `deliverables`.
- Apply documentation, code-quality, security, test-coverage, component-planning, feature-breakdown, code-review, external-library and API-design standards. JavaScript/custom-JWT examples in generic standards are not this project's implementation requirements. The approved enterprise OIDC boundary and sync Python recommendation govern this proposal.
- Source findings are static and research versions unverified. No live counts, production mappings, test coverage or performance claims may be substituted for missing evidence.

## Atomic task plan

Proposed artifact directory: `docs/migration/phase-0/`. These artifact files **do not exist yet** and are not part of this run's creation. Each task owns only its listed document. JSON paths are `.tmp/tasks/fastapi-migration/subtask_NN.json`.

Each task is a bounded **1-2 hour authoring/review pass**, assuming supplied inputs. Twelve tasks imply 12-24 authoring hours, not elapsed delivery time or a full migration estimate. Stakeholder availability, new research approval, unresolved decisions and implementation are excluded. If a question cannot be answered in the bound, record its owner/blocker rather than inventing a decision; split further work only with approval.

| Seq | Single outcome / future deliverable | Depends on | Parallel? | Binary acceptance for the design artifact |
| --- | --- | --- | --- | --- |
| 01 | Baseline and decision intake: `phase-0/01-baseline-decisions.md` | None | No | Pinned evidence, D01-D11 owner/status/evidence fields, scope and missing inputs recorded; no guessed approval |
| 02 | Identity/scope matrix: `phase-0/02-identity-scope.md` | 01 | Yes | Actor/target, OIDC/topology alternatives, effective-date routing and negative scope cases specified; no admin/self-approval bypass |
| 03 | Calendar/precision policy: `phase-0/03-calendar-precision.md` | 01 | Yes | Nonstandard finance dates, Wed-Tue, 53-week/cross-period, UTC-marker/DST, exact duration and leave decisions mapped to blocking modules |
| 04 | Persistence foundation proposal: `phase-0/04-persistence-foundation.md` | 01 | Yes | Sync session/UoW, SQLModel sufficiency checks, stable-version verification plan, Alembic/config/role/pool requirements stated without installs |
| 05 | Relational invariant catalog: `phase-0/05-relational-invariants.md` | 02, 03, 04 | No | Domain relations, IDs/FKs/constraints, immutable revisions/snapshots, effective dates and guard coverage trace to approved-or-pending policies |
| 06 | State/transaction protocol: `phase-0/06-state-transactions.md` | 05 | No | Separate weekly/period transitions, global lock order and every affected date, race outcomes and audit/outbox/commit boundary specified |
| 07 | HTTP command contract: `phase-0/07-api-contracts.md` | 02, 06 | Yes | Versioned noun routes, wire/error/pagination/version/idempotency examples and denied-field/scoping rules provided |
| 08 | Durable delivery contract: `phase-0/08-outbox-contract.md` | 02, 06 | Yes | Unique keys, leases/retries/crashes, dry-run isolation, at-least-once and recipient scopes specified |
| 09 | Migration reconciliation specification: `phase-0/09-migration-reconciliation.md` | 03, 05, 06 | Yes | Manifest/mapping/quarantine/state and exact totals, timer freeze, single writer and both rollback boundaries specified |
| 10 | Operations foundation requirements: `phase-0/10-operations-gates.md` | 04 | Yes | Owner-required RPO/RTO/SLO, role/TLS/secrets/backup/restore/load/support gates specified without invented SLA |
| 11 | Executable-test acceptance catalog (document only): `phase-0/11-test-catalog.md` | 07, 08, 09, 10 | No | Each critical invariant has fixture/input, actual boundary, expected result and future milestone; no test-pass claims |
| 12 | Approval and M1 handoff packet: `phase-0/12-approval-packet.md` | 11 | No | All 01-11 artifacts cross-referenced; blockers and owner disposition recorded; bounded M1 proposal and explicit implementation approval request included |

Dependency graph:

```text
01 -> {02,03,04}
{02,03,04} -> 05 -> 06 -> {07,08,09}
04 -> 10
{07,08,09,10} -> 11 -> 12
```

The table/JSON includes additional explicit 02/03/05 dependencies where useful. Tasks 02/03/04 and later 07/08/09/10 may overlap **only** in separate owned documents once their listed dependencies are complete. They read earlier artifacts but do not edit them. Shared roadmap integration and cross-document conflict resolution belong to the final review, not simultaneous writers. The final packet records conflicts and requests approval for changes; it does not silently overwrite upstream decisions. No backend/frontend implementation is hidden in these parallel flags.

### Task-specific review focus

1. **Baseline:** distinguish known static findings, recommendations, missing production inputs and approved decisions. Map every D01-D11 to owner role, evidence, resolution deadline/gate and affected module. Confirm Python MIT preservation, absent Node app license and workbook rights as separate issues.
2. **Identity:** draft allowed/denied role × record × operation matrix. Include finance aggregation anti-inference, manager role-administration denial, admin no-self-approval, delegated preparer separation, current authority versus historic relationships, fallback-route choices, and Streamlit per-user credential/session proof. Provider-specific proof remains pending until provider chosen.
3. **Calendar:** provide illustrative decision tables, not invented finance dates. Define half-open effective ranges, classification resolution, raw/local/UTC provenance, DST ambiguous/nonexistent input, exact interval-to-date allocation, quarter-hour manual input versus fractional legacy/timer storage and schedule/holiday expected-work quantities.
4. **Persistence:** propose module layout and interfaces without creating app files. Identify reviewed-stable version selection evidence still needed, SQLModel capability checks, engine/session lifecycle, no teardown commit, migration/runtime/worker role separation, settings fail-closed checklist and whole-deployment connection formula.
5. **Relations:** describe keys/cardinalities and database constraints, archival/no destructive cascades, immutable raw/revision/classification/submission/close membership, source-ID maps and sequence handling. Coverage of published work dates and independent period guards must not rely on timesheet existence.
6. **Transactions:** state machines plus sequence diagrams for save versus close (missing sheet too), decision versus amendment, cross-period move, active timer versus close and retroactive grant/calendar change. Use one lock order for every writer. Snapshot old/new dates then recheck under locks; abort on changed affected set. Specify retries/timeouts without treating stale versions as retryable success.
7. **API:** document request/response examples with offset instants, string microseconds/Decimals and opaque IDs. Include actor/role fields rejected, expected-version conflict, idempotency same/mismatched payload, authorization on replay, error sanitization, cursor scoping and original/restated report selectors.
8. **Outbox:** document message/delivery-attempt lifecycle, lease expiry, transactional claim/send/result boundaries and recipient changes. Include crash after SMTP acceptance, simulation/live namespaces, unique occurrence business key, dead-letter/replay controls and alert ownership.
9. **Migration:** design manifest and reconciliation output schemas in Markdown, including raw precision and marker; map source Not Started/In Progress to draft with original status retained, quarantine weekly Open pending approved legacy disposition, and map workflow/period states separately. Preserve correction deadlines (expired windows remain noneditable), locked legacy exceptions and unknown preceding workflow/approval/audit provenance. Define quarantine/signoff, full-repeat default unless reliable deltas proven, timer disposition, restore rehearsal and post-write preservation/forward-fix choice.
10. **Operations:** define evidence/owner slots for hosting, TLS, secrets, least privilege, migration rollout, pool/load budget, PITR restore, queue/auth/close incidents, logs without PII, retention, dependency scans/patch cadence/EOL and go/no-go authority.
11. **Tests:** consolidate examples into cases that later engineers can implement using disposable PostgreSQL and injected clocks; distinguish unit rules from DB constraints, API transactions, OIDC integration and deployment/restore acceptance. No tests or test fixtures are created/executed in this task.
12. **Handoff:** reconcile traceability and decisions, identify blocked slices, document review outcomes accurately, and present a narrow M1 plan for explicit approval. Missing approvals remain missing; recording a packet as drafted does not approve implementation.

## Contract design checklist

Proposed conceptual interfaces below guide the document authors; these are **not code or final signatures**:

| Boundary | Inputs | Required output / failure semantics |
| --- | --- | --- |
| Identity verifier | Approved OIDC credential/session context | Verified principal with issuer/subject/current auth context; fail closed, no target selected by auth input |
| Scope policy | Principal, capability, target/project, work dates, current grants and historical relationship versions | Allowed scope or denial; same predicate for queues, actions, report/export/email |
| Calendar resolver | Work date, approved company zone/calendar version | Exactly one week/period plus schedule/rule references, or unmapped/ambiguous error |
| Classification resolver | Work date, approved task/assignment inputs and rule version | Immutable classification result and version; no fallback or historical mutation |
| Application command/UoW | Actor, authorized target, expected versions, payload, idempotency key, injected clock | Committed result or full rollback; server-owned actor, audit and outbox; no commit in cleanup |
| Lock coordinator | Candidate old/new dates, people, sheets and entries | Ordered guards then under-lock revalidation; changed lock set aborts; period guards exist without sheets |
| Entry/revision writer | Explicit allocation/input mode and raw provenance | Stable ID, immutable next revision, exact totals; conflict instead of overwrite |
| Submission/decision policy | Exact revision set, frozen route and current authority | Immutable submission/decision, no-self approval and explicit recall/amend/reapproval |
| Close/correction policy | Period versions, cutoff/exceptions, reason and finance capability | Immutable snapshot/restatement/reopen lineage; no deletion or admin finality override |
| Outbox repository/worker | Unique event/recipient/occurrence key and authorized payload/reference | Atomic enqueue; leased retryable at-least-once delivery outside locks; simulation does not consume live claim |
| Import mapping/reconciler | Consistent manifest, exact raw rows, approved mappings and state map | Stable mapped IDs + exact reconciliation + explicit quarantines/differences, never invented audit/approval |

Future M1 must demonstrate with real PostgreSQL that the persistence/UoW shape can support these boundaries. Do not require an ORM rewrite, distributed services or new UI to satisfy them. Numerical limits, authentication topology, routing exceptions and finance policies are explicit decisions, not defaults hidden in types.

## Acceptance-test design

The catalog produced by task 11 must contain named cases with arrange/act/assert expectations, a future milestone, evidence owner, and source-finding/decision link. Minimum cases:

| Case family | Future execution and binary expected result |
| --- | --- |
| Foundation/config | M1: missing DB/IdP config or insecure production defaults prevent readiness; session isolation holds; commit failure cannot produce success; Alembic migration/constraints verified in disposable Pg |
| Scope denial | M2/M5: cross-person/project reads, filters, counts, exports and email denied; finance only approved aggregates; manager cannot grant admin; admin cannot approve own work; deprovisioned/stale grants fail closed |
| Ledger rollback/permissions | M2: injected failure leaves no business/audit/outbox partial commit; runtime role UPDATE/DELETE of audit/revisions/decisions/close evidence denied; no claim about unrestricted DBA |
| Calendar/effective dates | M3: Wed-Tue and approved nonstandard periods/53-week fixtures; all cross-period allocations resolved by date; overlapping/missing manager/assignment/schedule/rule ranges rejected or explicitly routed |
| Duration/DST | M3: raw microseconds conserved; legacy migration marker prevents double conversion; gaps rejected, folds explicit; date-copy has no UTC-week shift; holiday/OOO and expected work stay distinct |
| Real concurrent writers | M3/M4: two connections/barriers; save/save stale conflict; save/submit and approve/amend serialize; first-sheet save versus actual close includes committed write or rejects it; multi-period old/new dates both guarded |
| Timer and finality | M3/M4: unique active timer constraint; stop spanning closed date rejected/routed without loss; close blocks unresolved timer; no helper-only close simulation |
| Weekly and fiscal state | M4: lock never implies approval; recall/amend require explicit transitions; stale approval not restored; corrections require reason and scope; original snapshots/reopen lineage remain intact |
| Idempotency/response failure | M3/M5: duplicate concurrent same key produces one effect/result; mismatched payload conflicts; lost response after commit replays same result; revoked actor cannot retrieve replayed protected data |
| Worker crash/retry | M5: expired lease recovered; SMTP accept/result-record crash may duplicate but loses no durable intent; dry run does not poison delivery; recipient scope revoked before send cannot leak payload |
| Migration | M6: fresh-target repeat import matches manifest, IDs/sequences/FK counts/exact multi-dimensional totals; Not Started/In Progress become draft with source status retained; weekly Open awaits approved legacy disposition; expired correction deadlines stay noneditable; unknown/null/duplicate/state anomalies quarantined; locked legacy stays locked with unknown preceding workflow preserved; no fabricated historical approvals |
| Recovery/cutover | M6: authorized timer resolution, verified single writer/freeze, full snapshot or proven delta; timed PITR restore meets owner-approved RPO/RTO; post-write rollback preserves/reconciles new work and external effects |

Fixtures must be synthetic/disposable or explicitly authorized sanitized copies, never the repository database/workbook. Use an injected clock and named zone rather than host time. Tests of actual API/application transaction paths complement pure rule tests; they cannot be replaced by mocks or a second helper implementation. CI will later run lint/type/unit/real-Pg/contract/migration/dependency and license/security scans after execution approval.

## Review gates and implementation handoff

### Phase-0 artifact acceptance

- All 12 documents are drafted in their assigned files, linked from the approval packet, and marked proposed/approved/deferred/blocked accurately.
- Each D01-D11 has a named owner or an explicit unassigned-owner blocker, evidence, affected milestone and disposition. “TBD” is allowed as an honest recorded blocker, not a passed implementation gate.
- Domain/state/transaction/API/outbox/import contracts agree on actor/target, exact dates/units, stable revisions, finality and atomicity. Conflicts are reported for approval, not silently patched across documents.
- Every critical invariant has a future acceptance test and milestone. No runtime results, source counts, stable-version compatibility or stakeholder signoff are claimed without evidence.
- Operations and migration plans specify who must approve risk/budgets/restore/cutover; no invented RPO/RTO, SLO or production date.
- Packet distinguishes document completion from architecture approval and from authorization to implement M1.

### Separate M1 approval request

After review, request explicit approval that identifies: approved architecture and D01 choices; allowed checkout/environment/actions; bounded skeleton/settings/session/UoW/Alembic/CI deliverables; dependency selection and installation permission; disposable PostgreSQL/test execution permission; acceptance evidence; and stop/report handling. M1 excludes business workflow implementation and live data migration unless separately authorized.

M2 waits for D02/D03 and the M1 gate. M3 waits for calendar/precision and scopes. M4 waits for finance routing/finality approval. M5 writable Streamlit waits for proven per-user authentication; otherwise request only a controlled read-only pilot. M6 requires all reconciliation/restore/security/finance/support gates. A future frontend design task requires its own UI standards/workflow and approval; no UI implementation tasks exist in this phase.

## Tracking and validation limitations

Initial record: parent `active` for **planning**, 12 `pending`, 0 `in_progress`, 0 `completed`, 0 assigned agents and no start/completion timestamps. The task objective explicitly says implementation is not approved. No later M1-M6 implementation subtasks are created; they will be planned at their component gate.

The task CLI failed previously in cached ts-node configuration (`fileExists` TypeError) before status output. The user approved direct reads/manual JSON schema and dependency validation without dependency repair. Do not retry the CLI or run npm/npx/ts-node. Current validation reads each JSON/document and checks required fields, ID/sequence/count consistency, acyclic dependencies, pending/unassigned state, standards/source separation and distinct parallel deliverables. It is not an automated JSON-parser/CLI certification or an application test run.

Any actual validation failure must be reported and work stopped; no automatic fix without approval. Pending owner decisions are recorded blockers for future execution, not hidden implementation progress. Next proposed task is **01, baseline and decision intake**, subject to permission to begin the next design work.
