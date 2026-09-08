# Backend documentation research

Retrieved: 2026-09-08. Planning guidance only; no packages installed or compatibility tests run.

ExternalScout was attempted but blocked by its mandatory unavailable Write tool. A general research agent using the Context7 skill and official documentation supplied the findings below. This limitation concerns the research tool, not the proposed libraries.

## Version scope

SQLAlchemy documentation identifies 2.0.52; Alembic identifies 1.19.2; Pydantic guidance targets v2. FastAPI, SQLModel, and pydantic-settings references are rolling documentation. Psycopg documentation identifies a 3.3.6.dev1 build: use it for Psycopg 3 API concepts, **not as a production release pin**. PostgreSQL references target major 18. This is not a tested dependency lockfile. Resolve supported stable versions and test their combination during the approved foundation phase.

## Findings and sources

| Area | Architecture implication | Official source |
| --- | --- | --- |
| FastAPI sync/async | Start with synchronous `def` handlers/dependencies for blocking database operations. Async helpers do not automatically offload blocking calls. Revisit only with measurements. | [Concurrency](https://fastapi.tiangolo.com/async/) |
| Module boundaries | Domain routers and injected settings/session/identity dependencies; no domain rules embedded in UI or routing code. | [Bigger applications](https://fastapi.tiangolo.com/tutorial/bigger-applications/) |
| Transaction completion | Default yield dependency teardown may run after the response is sent. Commit the application command before returning success; use teardown for cleanup, not deferred commit. | [Dependencies with yield](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/) |
| Deployment | Supervise workers, terminate TLS appropriately, budget pools across replicas; execute migrations once as a deployment job. | [Deployment concepts](https://fastapi.tiangolo.com/deployment/concepts/) |
| Sessions | Fresh session per request/operation; engine per process. No shared mutable session, no reuse in independent background work. | [SQLModel dependency](https://sqlmodel.tiangolo.com/tutorial/fastapi/session-with-dependency/), [SQLAlchemy concurrency](https://docs.sqlalchemy.org/en/20/orm/session_basics.html#is-the-session-thread-safe-is-asyncsession-safe-to-share-in-concurrent-tasks) |
| Transactions | Begin the unit of work before reads that can autobegin. Mutation, decision, audit, and outbox insert commit atomically. | [Session transactions](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html) |
| Concurrency | Lock in consistent order; optimistic versions require expected client versions. ORM version counters do not protect arbitrary bulk DML. | [Version counters](https://docs.sqlalchemy.org/en/20/orm/versioning.html), [FOR UPDATE](https://docs.sqlalchemy.org/en/20/core/selectable.html#sqlalchemy.sql.expression.Select.with_for_update), [PostgreSQL locking](https://www.postgresql.org/docs/18/explicit-locking.html) |
| Model separation | Reuse SQLModel selectively; define separate create/update/public schemas, excluding actor, status, role and classification fields that clients may not control. | [Multiple models](https://sqlmodel.tiangolo.com/tutorial/fastapi/multiple-models/) |
| Pydantic v2 | Specify optional/nullability, `model_validate`/`model_dump`, `from_attributes` where applicable, and Decimal JSON contracts. | [Migration guide](https://docs.pydantic.dev/latest/migration/) |
| PostgreSQL driver | Explicit `postgresql+psycopg://`; evaluate Psycopg binary versus system-linked packaging for operational updates. Do not rely on the bare URL's default driver. | [SQLAlchemy Psycopg dialect](https://docs.sqlalchemy.org/en/20/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.psycopg), [Psycopg installation](https://www.psycopg.org/psycopg3/docs/basic/install.html) |
| Alembic | Autogenerate produces review candidates, not guaranteed safe migrations or data transfer. Review renames, checks, conversions, backfills, lock impact and sequences. | [Autogenerate](https://alembic.sqlalchemy.org/en/latest/autogenerate.html) |
| Settings | Validate before readiness, require real database/identity configuration, reject insecure production defaults, redact secrets. Strict mode alone is not fail-closed configuration. | [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/), [Strict mode](https://docs.pydantic.dev/latest/concepts/strict_mode/) |
| Data types | `date` for work dates; `timestamptz` for instants; preserve named timezone separately. Exact Numeric/Decimal or integer units; define finite values and rounding. | [Date/time](https://www.postgresql.org/docs/18/datatype-datetime.html), [Numeric](https://www.postgresql.org/docs/18/datatype-numeric.html) |
| Integrity | Database PK/FK/UNIQUE/NOT NULL/CHECK constraints complement API validation. CHECK alone permits NULL; referencing FK indexes need explicit consideration. | [Constraints](https://www.postgresql.org/docs/18/ddl-constraints.html) |
| Support lifecycle | Official policy supports a major for five years. Evaluate PostgreSQL 18, or 17 if hosting requires; 14 is close to November 2026 end-of-support. Verify current policy again at implementation. | [Version policy](https://www.postgresql.org/support/versioning/) |
| Recovery | PITR requires base backups and uninterrupted WAL archives; `pg_dump` is not a WAL replay base. Define recovery objectives and prove restores. | [Continuous archiving/PITR](https://www.postgresql.org/docs/18/continuous-archiving.html) |

## Identity boundary

Company OIDC provider and client topology remain undecided. FastAPI authentication examples are not an enterprise identity/provisioning system. Before implementation, obtain provider-specific official documentation and approve token/session validation, role mapping, deprovisioning, MFA enforcement through the IdP, CSRF/cookie boundaries where applicable, and identity propagation from Streamlit. Do not implement custom password/token issuance merely to match the prototype.
