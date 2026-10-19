# Technology comparison for AITU Smart Schedule

Project: Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms (working name: AITU Smart Schedule). Team: Aizada Sembayeva, Aiya Zhakupova. Supervisor: Nadirov Nurtas.

This document compares alternatives for six stack decisions and justifies each choice against the requirements in [docs/05-requirements.md](05-requirements.md). The result is Go 1.27, PostgreSQL 18 (16 as the minimum), River for background jobs, Server-Sent Events opened through `fetch` with a polling fallback, the team's own solver with OR-Tools CP-SAT as an offline baseline, and React for the frontend.

## Summary of decisions

| # | Decision | Chosen | Main alternative | Deciding requirement |
|---|---|---|---|---|
| 1 | Backend language | Go 1.27 | Java 21+ with Spring Boot | Solver speed and parallel portfolio in the product language (NFR-PERF-01, NFR-PERF-02); one binary for api and worker (NFR-DEP-01) |
| 2 | Database | PostgreSQL 18 (minimum 16) | MySQL 8.4 | Overlapping room intervals rejected by the database itself (NFR-COR-02, FR-BOOK-03) |
| 3 | Background jobs | River (queue inside PostgreSQL) | Redis + Asynq | Job enqueued in the same transaction as the run record (FR-SCH-02, NFR-REL-01) |
| 4 | Real-time updates | Server-Sent Events, with polling as fallback | WebSocket | One-way server push of progress and seat counts (FR-SCH-03, FR-ENR-08) |
| 5 | Solver | Own two-phase solver in Go; OR-Tools CP-SAT as offline baseline only | OR-Tools CP-SAT inside the product | Minimal conflict explanation and own algorithmic contribution (FR-SCH-06, proposal objectives 4-5) |
| 6 | Frontend | React + TypeScript + Vite | Vue 3 + TypeScript | Runtime switching between kk/ru/en (NFR-USE-01) and available libraries; team familiarity to be confirmed by Aiya |

## 1. Method

### 1.1 Where the criteria come from

Each criterion is derived from a numbered requirement in docs/05-requirements.md, through the drivers in Section 2. A criterion that comes from a Must requirement is a knockout criterion (K). An alternative that fails a knockout criterion is rejected, whatever its other strengths, unless it can meet the requirement with a workaround, in which case the workaround's cost is counted. Criteria from Should or Could requirements, or from project constraints such as team size, are weighted criteria (W).

### 1.2 Scoring

Each cell uses the same symbols as the existing-systems comparison:

- ● meets the criterion with built-in, documented features;
- ◐ meets it only with a workaround, an extra component or extra code;
- ○ does not meet it.

The scores are not added up. Summing ordinal symbols would suggest a precision the evidence does not have, and one failed knockout outweighs any number of weighted advantages. Each decision is argued in prose from the knockout rows first and the weighted rows second.

### 1.3 Evidence and its limits

Evidence comes from official documentation, release notes and the project's own documents (proposal, literature review, existing-systems comparison). All web sources were accessed on 18.10.2026. Three limits apply:

1. No benchmarks were run for this document. Performance statements about languages and solvers are expectations based on documentation and the literature. The planned experiments (scalability, ablation, comparison with CP-SAT, load tests) will test the chosen stack, not the rejected ones.
2. Team familiarity is self-assessed and still has to be confirmed by both team members, Aiya in particular for the frontend.
3. Some sources are secondary (a vendor blog, a tutorial site). They are marked as such in Section 5 and are used only for facts that the official documentation states elsewhere or that are easy to check.

## 2. Requirement drivers

The drivers below collect the requirements that discriminate between technologies. Requirement IDs refer to docs/05-requirements.md.

| Driver | What the stack must provide | Requirements |
|---|---|---|
| D1. Non-overlap enforced by the database | A room cannot hold two active occupations whose time intervals overlap, even if application code has a bug; this covers arbitrary booking intervals, not only fixed slots | NFR-COR-02, FR-BOOK-03, FR-SCH-11 |
| D2. Atomic seat reservation | 500 concurrent enrollment requests into one stream give 0 overflows, with p95 ≤ 300 ms | NFR-PERF-04, NFR-COR-06, FR-ENR-02 |
| D3. Multi-row atomic transactions | Confirming a teacher option, switching stream and publishing a version each change several rows in several tables, all or nothing | FR-CONF-03, FR-ENR-05, FR-SCH-11 |
| D4. Durable background jobs | Generation runs for seconds to minutes outside the request, can be cancelled, survives a worker crash, and is enqueued atomically with its run record | FR-SCH-02, FR-SCH-04, FR-CONF-04, NFR-REL-01 |
| D5. Solver speed and reproducibility | First feasible timetable ≤ 10 s and full run ≤ 120 s on 4 cores for 400 streams; local regeneration ≤ 3 s; same seed gives the same result | NFR-PERF-01, NFR-PERF-02, NFR-PERF-03, NFR-COR-03 |
| D6. Explanation and own contribution | Infeasibility reported as a minimal set of named constraint groups; the generation algorithm is the team's own work | FR-SCH-06, NFR-COR-05; proposal objectives 4-5; literature review §2.4 |
| D7. Server-to-client push | Generation progress and live seat counters reach the browser without a reload | FR-SCH-03, FR-ENR-08 |
| D8. Relational integrity | Program → cohort → course → stream → session links with foreign keys and checks | FR-CAT-06, FR-CAT-07, FR-CAT-08 |
| D9. Small operational footprint | `docker compose up` starts the whole system for a two-person team and the defense committee | NFR-DEP-01 |
| D10. Frontend needs | kk/ru/en with runtime switching, layout from 360 px, weekly grid, SVG floor map, TypeScript client generated from OpenAPI | NFR-USE-01, NFR-USE-02, FR-BOOK-01, FR-SCH-08, FR-X-03 |
| D11. Testability against a real database | Concurrency tests run against real PostgreSQL in a container with the race detector on | NFR-MNT-01 |

## 3. Decisions

### 3.1 Backend language: Go / Java (Spring Boot) / Python

The backend contains two very different workloads: a CPU-bound solver that must run hundreds of thousands of moves per second in the simulated annealing phase, and an ordinary transactional web API. The question is whether one language serves both.

| Criterion | Type | Go | Java + Spring Boot | Python (FastAPI) |
|---|---|---|---|---|
| Solver meets D5 in the product language (compiled code, cheap 64-bit bit-mask operations, parallel portfolio over 4 cores) | K (D5) | ● compiled; goroutines for the portfolio; `uint64` masks for 48 slots | ● JIT-compiled; threads or virtual threads for the portfolio | ○ interpreted inner loop; default CPython build keeps the GIL, the free-threaded build is optional [11]; meeting D5 needs C extensions, Numba or multiprocessing |
| Explicit SQL and visible locking (`FOR UPDATE`, conditional `UPDATE`) | K (D2, D3) | ● pgx + sqlc generate typed code from hand-written SQL | ◐ possible with JdbcTemplate or jOOQ; the Spring default (JPA/Hibernate) hides SQL and locks | ● psycopg or SQLAlchemy Core |
| One deployable for api and worker | W (D9) | ● one static binary, `--mode=api\|worker` | ◐ one fat JAR plus a JVM in the image | ◐ interpreter plus dependencies; worker usually a separate process model |
| Contract-first OpenAPI (server interfaces and TS client from one spec) | W (D10) | ● oapi-codegen | ● openapi-generator | ◐ FastAPI is code-first; contract-first needs extra tooling |
| PostgreSQL-backed job queue available | W (D4) | ● River [6] | ● several libraries exist | ● several libraries exist |
| OR-Tools CP-SAT callable directly for the baseline | W (D6) | ○ no Go API among the languages in the release notes [10] | ● Java API | ● Python API |
| Supervisor and team context | W | ● Go is the backend language named in the proposal (objective 6) | ◐ | ◐ |

Decision: Go, pinned to 1.27. Go and Java both pass the knockout rows. Go wins on deployment (one binary in two modes, which a modular monolith with separate api and worker processes relies on), on explicit SQL by default, and on the project context, since the proposal names Go as the backend language. Python fails the solver knockout: the backtracking and annealing loops would need native extensions to reach the target speed, and the code that defines the algorithm would then be split between two languages.

Go's one weakness here is that CP-SAT has no Go API. This does not matter, because CP-SAT is only an offline baseline (Section 3.5) and runs as a separate Python script.

Version: Go 1.27.0 was released on 19.08.2026, and under Go's policy each major release is supported until two newer major releases exist [9], so Go 1.25 and older no longer receive security fixes. The `go.mod` file should declare `go 1.27`, and the version should be raised again only if Go 1.28 appears before the code freeze.

What would change the decision: if a supervisor requirement or team skill made Java mandatory, the design would carry over almost unchanged with Spring Boot, jOOQ and a PostgreSQL-backed queue. The solver would be rewritten, not redesigned.

### 3.2 Database: PostgreSQL / MySQL / MongoDB

| Criterion | Type | PostgreSQL | MySQL (InnoDB) | MongoDB |
|---|---|---|---|---|
| Database rejects overlapping intervals for the same room, for arbitrary booking times | K (D1) | ● `EXCLUDE USING gist (room_id WITH =, during WITH &&)` over `tstzrange` with `btree_gist` [2][3]; PostgreSQL 18 also adds `UNIQUE … WITHOUT OVERLAPS` [4] | ○ no exclusion constraints and no range types [5]; needs triggers or locking a parent row in application code | ○ no equivalent; a unique index covers fixed slots only, not arbitrary intervals |
| Atomic conditional update of a seat counter under concurrency | K (D2) | ● `UPDATE … WHERE enrolled < capacity`; under Read Committed the `WHERE` is re-checked on the newly committed row version [1] | ● same pattern with InnoDB row locks | ● single-document `findOneAndUpdate` with a `$lt` filter |
| Multi-row, multi-table transactions | K (D3) | ● | ● | ◐ multi-document transactions exist but need a replica set and have a 60-second default lifetime [7] |
| Foreign keys and CHECK constraints for curriculum data | W (D8) | ● | ● | ◐ schema validation, no foreign keys |
| Job queue and notifications in the same database | W (D4, D7, D9) | ● River; `LISTEN/NOTIFY` for SSE fan-out [12] | ○ no `LISTEN/NOTIFY` | ◐ change streams |
| JSON storage for run parameters and infeasibility reports | W | ● `jsonb` | ● `JSON` | ● native |

Decision: PostgreSQL. It is the only option that passes all three knockout rows. The exclusion constraint makes a double booking impossible by construction, and the literature review lists the same mechanism as the room-booking guarantee (§2.5). MySQL and MongoDB could implement the same rule only in application code or triggers, which is exactly the kind of check-then-write logic that NFR-COR-02 is meant to rule out.

A point the literature review left open is settled here. Lit review §2.5 noted that the safety of the seat counter under concurrent transactions depended on how Read Committed re-evaluates the condition, and that this had not been checked against the documentation. The PostgreSQL manual states that when `UPDATE` meets a row changed by a concurrent transaction, it waits for that transaction and then re-evaluates the `WHERE` clause against the updated row version [1]. So `UPDATE streams SET enrolled = enrolled + 1 WHERE id = $1 AND enrolled < capacity` cannot push `enrolled` past `capacity` at the default isolation level, and the `CHECK (enrolled <= capacity)` constraint is a second line of defence. The k6 and testcontainers tests in NFR-PERF-04 and NFR-COR-06 still verify it empirically.

Version: PostgreSQL 18 is the current major release [1], so the Docker image should use it, with 16 as the minimum supported version because the planned schema does not depend on version-18 features. The `WITHOUT OVERLAPS` syntax in PostgreSQL 18 is noted but not used: the planned constraint excludes cancelled bookings with a `WHERE (status = 'active')` predicate, which `EXCLUDE` supports, and the secondary source describing `WITHOUT OVERLAPS` [4] does not say whether a predicate is allowed.

### 3.3 Background jobs: River / Redis + Asynq / RabbitMQ

The load is small, tens of generation runs per day plus local regenerations during the confirmation period. The requirements concern correctness of the job lifecycle, not throughput.

| Criterion | Type | River | Redis + Asynq | RabbitMQ |
|---|---|---|---|---|
| Job enqueued in the same transaction as the `generation_runs` row, so a run record never exists without its job or vice versa | K (D4) | ● `InsertTx`; a job is enqueued if and only if the transaction commits [6] | ○ Redis is a second store; needs a transactional outbox table in PostgreSQL | ○ same dual-write problem; needs an outbox |
| Job survives a worker crash and is retried | K (D4, NFR-REL-01) | ● jobs persist in PostgreSQL; failed jobs return to a retryable state [6] | ◐ depends on Redis persistence settings (AOF/RDB) | ● durable queues with manual acknowledgement |
| Cancel a running job | W (FR-SCH-04) | ● job cancellation API [6] | ◐ cancellation of active tasks through its inspector API (not checked for the current version) | ○ no job-level cancel; application protocol needed |
| At most one active run per term | W (FR-SCH-02) | ● unique jobs by args, period, queue and state [6] | ◐ unique-task option | ○ application logic |
| Extra infrastructure | W (D9) | ● none | ◐ Redis 4.0+ container [8] | ◐ broker container plus management |
| Maturity | W | ◐ pre-1.0 (v0.49, October 2026), MPL-2.0 [6] | ◐ pre-1.0 (v0.26, February 2026); API may change before v1 [8] | ● mature |

Decision: River. It is the only option that passes both knockout rows without a workaround, and it adds no container. The throughput advantage of Redis and RabbitMQ is irrelevant at tens of jobs per day.

Risk and fallback: River is pre-1.0. The version is pinned in `go.mod`, and the queue is used only from the `scheduling` and `confirmation` modules behind a small interface. If River blocks the project, the fallback is a plain `jobs` table polled with `SELECT … FOR UPDATE SKIP LOCKED`, which keeps the transactional-enqueue property and is a few hundred lines of code.

### 3.4 Real-time updates: SSE / WebSocket / polling

Two flows need push. Generation progress goes to one administrator, about one event per second for up to 120 s. Seat counters go to many students during an enrollment window. All writes already go through REST endpoints with `Idempotency-Key` (FR-X-02), so no flow needs client-to-server messages over the push channel.

| Criterion | Type | SSE | WebSocket | Polling |
|---|---|---|---|---|
| Server-to-client delivery of progress and counters | K (D7) | ● one-way by design [13] | ● two-way (one direction unused) | ● with delay up to the polling interval |
| Automatic reconnect and resume | W (NFR-REL-05) | ● built-in reconnect with server-set `retry` and event `id` [13] | ○ application code | ● trivial |
| Authentication with the access token | W (NFR-SEC-03) | ◐ the browser `EventSource` cannot send an `Authorization` header (W3C bug 25984, WONTFIX, fetch with streams recommended instead) [14] | ◐ the browser WebSocket API has the same limitation; token in the first message or a cookie | ● ordinary `fetch` with header |
| Load during an enrollment window (5,000 students) | W (NFR-PERF-04) | ● one idle connection per open page; Go handles this cheaply | ● same | ○ at a 5 s interval, 5,000 open pages make about 1,000 requests per second, competing with the enrollment requests the p95 target is measured on |
| Browser connection limits | W | ◐ 6 connections per browser and domain over HTTP/1.1; negotiated, by default 100 streams, over HTTP/2 [13] | ● | ● |
| Through the Caddy proxy and plain HTTP tooling | W (D9) | ● ordinary HTTP response | ◐ upgrade handling, ping/pong, separate testing | ● |
| Fan-out across api instances | W | ● `LISTEN/NOTIFY`, payload under 8,000 bytes [12] | ● same | n/a |

Decision: SSE for generation progress (FR-SCH-03) and seat counters (FR-ENR-08), with polling every 15 s as the fallback when the stream cannot be opened. WebSocket is rejected because two-way messaging is not needed and costs extra code for reconnection and proxying. Polling alone is rejected because of its load during the enrollment window.

Two implementation details follow from this choice:

- Authentication: the access token lives in memory (NFR-SEC-03), and `EventSource` cannot send it. Putting it in the URL is ruled out because tokens in query strings end up in logs. The frontend therefore opens SSE streams with `fetch` and a readable stream (for example the `@microsoft/fetch-event-source` package), which sends the normal `Authorization` header and keeps one authentication path for all endpoints.
- HTTP/2: Caddy serves HTTPS over HTTP/2, which removes the six-connection limit. The local development setup should also go through Caddy, or developers will hit the limit with several tabs open.

Notifications carry only identifiers and counters (stream id and `enrolled`), never personal data, which keeps them well under the `NOTIFY` payload limit.

### 3.5 Solver: own implementation / OR-Tools CP-SAT / ILP solver

The ILP option is represented by HiGHS, an open-source MIT-licensed MIP solver [15], and by Gurobi as the commercial reference used in several reviewed papers (Rappos et al., 2022; Antunes-Batista et al., 2026).

| Criterion | Type | Own: backtracking + SA + QuickXplain in Go | OR-Tools CP-SAT | ILP (HiGHS / Gurobi) |
|---|---|---|---|---|
| Algorithmic contribution of the diploma is the team's own work | K (D6; proposal objectives 4-5) | ● | ○ the team would write only the model | ○ same |
| Infeasibility reported as a minimal set of named constraint groups | K (FR-SCH-06) | ● QuickXplain over constraint groups, using the own solver as the consistency check (Junker, 2004) | ◐ returns a subset of assumptions that makes the model infeasible [16]; minimality not documented, so QuickXplain would still run on top | ◐ IIS support depends on the solver; Steiner et al. (2025) used it with a commercial solver |
| First feasible ≤ 10 s, full run ≤ 120 s for 400 streams | K (D5) | ◐ target, to be shown by the experiments | ◐ no published CP-SAT results on ITC-2007 or ITC-2019 in the reviewed literature (lit review §2.3) | ◐ exact MIP on hard instances can take hours and large memory (Rappos et al., 2022; Müller et al., 2025) |
| Proven optimality or lower bounds | W | ○ heuristic, no bounds | ● | ● |
| Local regeneration with all other classes fixed, 2-3 distinct options | W (FR-CONF-04, FR-CONF-02) | ● designed for it (FR-CONF-01, FR-CONF-04) | ● fix variables, block earlier solutions | ● same |
| Reproducible with a fixed seed | W (NFR-COR-03) | ● seeded per goroutine, deterministic tie-break | ◐ with a fixed seed; multi-worker search needs extra settings to be deterministic | ◐ solver-dependent |
| Called from the Go backend | W (D9) | ● same package | ◐ no Go API [10]; sidecar service or cgo | ◐ HiGHS C API through cgo |
| Licence | W | ● own code | ● Apache 2.0 | ● HiGHS MIT; ◐ Gurobi commercial (free academic licence) |

Decision: the product uses the team's own two-phase solver: backtracking with MRV, LCV and forward checking, then simulated annealing, with QuickXplain for infeasibility (lit review §2.7). OR-Tools CP-SAT is the baseline in the experiments only (proposal objective 8), run from a separate Python script with one worker to follow the ITC-2007 single-processor rule and with its default parallel setting reported alongside (lit review §2.2). An ILP solver is not used; HiGHS can be added as a second baseline if time allows.

The own solver passes the two knockout rows that CP-SAT and ILP fail: the diploma needs an algorithm the team designed, and the explanation module needs a consistency check it can call on any subset of constraint groups. Both exact solvers score better on bounds, which is why CP-SAT is the yardstick. The speed row is open for all three. The literature supports the two-phase design for interactive budgets (lit review §2.3), but whether the own solver meets 10 s at 400 streams is a result of the project, not an assumption.

Fallback: the `solver` package exposes `Problem → Solution | InfeasibilityReport` and has no database or HTTP dependency (NFR-MNT-02). If the own solver misses NFR-PERF-01 at the backtracking milestone (01.12.2026), the shortfall is reported, the target is renegotiated with the supervisor, and CP-SAT behind the same interface stays an option for the demo. The explanation and confirmation logic would still be the team's own.

### 3.6 Frontend: React / Vue / Angular

Requirements discriminate less here than in the other decisions. All three frameworks can meet the Must requirements, so the choice rests on the i18n model, library availability and team skill.

| Criterion | Type | React + TS + Vite | Vue 3 + TS + Vite | Angular |
|---|---|---|---|---|
| TypeScript client generated from OpenAPI 3.1 | K (FR-X-03) | ● openapi-typescript (framework-independent) | ● same | ● same, or openapi-generator |
| kk/ru/en with switching at runtime and plural rules | K (NFR-USE-01) | ● i18next / react-i18next | ● vue-i18n | ◐ built-in `@angular/localize` produces one build per locale and needs a reload to switch [17]; runtime switching needs Transloco or ngx-translate |
| Weekly grid | W (FR-SCH-08, FR-ENR-09) | ● FullCalendar React connector | ● FullCalendar Vue connector | ● FullCalendar Angular connector |
| SVG floor map with per-room status and zoom on phones | W (FR-BOOK-01, NFR-USE-02) | ● inline SVG + react-zoom-pan-pinch | ● inline SVG + a framework-independent pan-zoom library | ● same |
| Responsive component library | W (NFR-USE-02) | ● Mantine | ● PrimeVue or Vuetify | ● Angular Material |
| Server-state caching with invalidation from SSE events | W (D7) | ● TanStack Query | ● TanStack Query (Vue adapter) | ● RxJS services |
| Team familiarity (frontend owner: Aiya) | W | to confirm | to confirm | to confirm |

Decision: React with TypeScript, Vite, TanStack Query and Mantine, on the condition that Aiya confirms she can work in it efficiently. If she is more productive in Vue, Vue 3 meets every row equally and the switch costs nothing at this stage, since no frontend code exists yet. Angular is the weakest fit because its built-in i18n assumes one build per language, while the requirement is a language switch inside one session, and the extra framework structure adds little for a three-role SPA.

## 4. Other stack choices

These choices were not compared in depth. Each follows from the decisions above or from a single requirement.

| Area | Choice | Rejected | Reason, with requirement |
|---|---|---|---|
| Architecture | Modular monolith, one binary with `api` and `worker` modes | Microservices | One database and one transaction per critical operation (D3); two-person team (D9) |
| HTTP layer | net/http + chi middleware, server interfaces from oapi-codegen | Gin, Echo | Contract-first API (FR-X-03); generated code fits the standard library |
| Data access | pgx/v5 + sqlc | GORM | Every lock and conditional update stays visible in SQL (D2, D3); parameterised queries only (NFR-SEC-06) |
| Migrations | goose | — | Versioned SQL migrations run from the binary (NFR-DEP-01) |
| Authentication | Access JWT 15 min + rotating refresh token in httpOnly cookie, argon2id, RBAC | OIDC / Microsoft 365 SSO | Standard SPA scheme (FR-AUTH-01, FR-AUTH-02); SSO is out of MVP scope (FR-AUTH-08) |
| Room map | SVG floor plans with `id` = room code, coloured by React | Leaflet/Mapbox, three.js | Indoor plan without geocoordinates (FR-BOOK-01, FR-BOOK-07); street-map libraries and 3D add nothing to the requirement |
| Deployment | Docker Compose (Caddy, api, worker, postgres), GitHub Actions | Kubernetes | One command for the committee (NFR-DEP-01); Caddy provides TLS and HTTP/2 (Section 3.4) |
| Testing | go test with `-race`, testify, testcontainers-go, k6 | SQLite in tests, mocks | Locking semantics must be those of real PostgreSQL (D11, NFR-PERF-04) |
| Data model | Weekly sessions expanded into dated `room_occupancy` rows on publish; foreign keys and CHECK constraints along program → cohort → course → stream | Recurrence rules evaluated at query time | One exclusion constraint then covers both lessons and bookings (D1, D8; FR-CAT-03, FR-CAT-08, FR-SCH-11) |
| API conventions | RFC 9457 problem details with machine-readable codes; `Idempotency-Key` header on every mutating request | Ad hoc JSON errors, no idempotency | The client must tell `STREAM_FULL` from `SCHEDULE_CONFLICT` and show what conflicted; a repeated click during an enrollment burst must not create a second record (FR-X-01, FR-X-02, NFR-REL-04) |
| Security baseline | HTTPS with HSTS through Caddy; argon2id at the OWASP minimum (19 MiB, 2 iterations, parallelism 1); per-account login lockout and per-user write limits; audit log table; secrets only in environment variables; gitleaks, gosec, govulncheck and npm audit in CI; uploaded SVG sanitised; random, revocable iCal tokens; personal data limited to what scheduling needs | Security left to defaults | Three roles act on each other's data, and the system holds student records (NFR-SEC-01 to NFR-SEC-12) |
| Observability | slog JSON logs with request id; Prometheus metrics | Plain-text logs | Load-test results and conflict counts must be measurable for the evaluation (NFR-OBS-01, NFR-OBS-02, NFR-PERF-04) |
| Operations | Graceful shutdown on SIGTERM; daily `pg_dump` with one rehearsed restore; connection pool that reconnects after a database restart | Managed cloud database | Self-hosted demo on one machine (NFR-REL-02, NFR-REL-03, NFR-REL-06) |

## 5. Sources

Project documents: proposal (docs/01-proposal.md), literature review (docs/03-literature-review.md), existing-systems comparison (docs/04-existing-systems.md). Papers cited by author and year are listed in the literature review.

All web sources accessed 18.10.2026.

1. PostgreSQL Global Development Group. 13.2 Transaction Isolation, section 13.2.1 Read Committed (version 13 page; the site lists 18 as the current version). https://www.postgresql.org/docs/13/transaction-iso.html
2. PostgreSQL Global Development Group. 5.5 Constraints (exclusion constraints). https://www.postgresql.org/docs/current/ddl-constraints.html
3. PostgreSQL Global Development Group. 8.17 Range Types. https://www.postgresql.org/docs/current/rangetypes.html
4. Neon. PostgreSQL 18 temporal constraints with `WITHOUT OVERLAPS` (secondary source). https://neon.com/postgresql/18/temporal-constraints
5. SQL for Devs. Time ranges without overlapping (MySQL marked as not supported; secondary source). https://sqlfordevs.com/non-overlapping-time-ranges
6. River package documentation, v0.49.0, pkg.go.dev. https://pkg.go.dev/github.com/riverqueue/river
7. MongoDB. Performance best practices: transactions and read/write concerns (vendor blog). https://mongodb.com/blog/post/performance-best-practices-transactions-and-read-write-concerns
8. Asynq package documentation, v0.26.0, pkg.go.dev. https://pkg.go.dev/github.com/hibiken/asynq
9. The Go Project. Release history. https://go.dev/doc/devel/release
10. Google. OR-Tools release notes (v9.15, January 2026). https://developers.google.com/optimization/support/release_notes
11. pydevtools. What is PEP 779? (free-threaded Python 3.14; secondary source). https://pydevtools.com/handbook/explanation/what-is-pep-779/
12. PostgreSQL Global Development Group. NOTIFY. https://www.postgresql.org/docs/current/sql-notify.html
13. MDN Web Docs. Using server-sent events. https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
14. W3C Bugzilla. Bug 25984: EventSource custom headers (RESOLVED WONTFIX, 2015). https://www.w3.org/Bugs/Public/show_bug.cgi?id=25984
15. HiGHS. Repository and README. https://github.com/coin-or/HiGHS
16. OR-Tools OCaml bindings, `Sat.Response` (field `sufficient_assumptions_for_infeasibility`), v9.15. https://ocaml.org/p/ortools/9.15.0-1/doc/ortools/Ortools/Sat/Response/index.html
17. SimpleLocalize. Angular i18n guide, 2026 (secondary source). https://simplelocalize.io/blog/posts/angular-i18n-guide/
