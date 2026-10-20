# Architecture and diagrams for AITU Smart Schedule

Project: Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms (working name: AITU Smart Schedule).

Team: Aizada Sembayeva, Aiya Zhakupova.

Supervisor: Nadirov Nurtas.

Status: draft 0.1, 20.10.2026. The component diagram and the data model are drafts and will be refined when development starts (checkpoints in [06-project-management.md §2.2](06-project-management.md)).

This document describes the architecture of the platform with three required diagrams (system context, use cases, components), a draft data model and four key runtime scenarios. Every element is traced to the requirements in [05-requirements.md](05-requirements.md) and to the stack chosen in [04-technology-comparison.md](04-technology-comparison.md), so that requirements, components, data and scenarios can be checked against each other (Section 8).

Diagram sources are in [`docs/diagrams/src/`](diagrams/src/) as draw.io SVG files (`.drawio.svg`) in standard UML notation. Each file is both an image that GitHub and browsers display directly and an editable draw.io diagram: open it at [app.diagrams.net](https://app.diagrams.net), in the draw.io desktop app or in VS Code with the Draw.io Integration extension. The Mermaid code blocks below show the same content as text; when a diagram changes, the `.drawio.svg` file and the matching block are updated in the same pull request.

| # | Diagram | Source (draw.io SVG) |
|---|---|---|
| 1 | System context | [01-context.drawio.svg](diagrams/src/01-context.drawio.svg) |
| 2 | Use cases | [02-use-case.drawio.svg](diagrams/src/02-use-case.drawio.svg) |
| 3 | Components (draft) | [03-component.drawio.svg](diagrams/src/03-component.drawio.svg) |
| 4 | Data model (draft) | [04-data-model.drawio.svg](diagrams/src/04-data-model.drawio.svg) |
| 5 | Sequence: generation run | [05-sequence-generation.drawio.svg](diagrams/src/05-sequence-generation.drawio.svg) |

## 1. Architectural style

The system is a modular monolith: one Go binary that runs either as `api` or as `worker`, one PostgreSQL database, and a React single-page application served through Caddy (TC §4). The choice follows from three drivers in the technology comparison:

- every critical operation (publish, confirm an option, enroll, switch stream) changes several tables in one transaction, which is simple in one database and hard across services (D3);
- the solver is CPU-bound and runs for seconds to minutes, so it runs in a separate `worker` process that takes jobs from the River queue in PostgreSQL, while the `api` process stays responsive (D4, NFR-PERF-06);
- a two-person team and the committee must be able to start everything with `docker compose up` (D9, NFR-DEP-01).

Modules inside the binary follow the seven functional modules of the requirements (auth, catalog, constraints, scheduling, confirmation, enrollment, booking + map). Modules talk to each other through Go interfaces, not through HTTP. The scheduling algorithm lives in the package `internal/solver`, which imports no database or HTTP package (NFR-MNT-02), so the same code is used by the worker, by the command-line tool `solverctl` and by the experiments.

## 2. System context


The platform has three kinds of users (P §5, requirements §1.1) and exchanges data with four kinds of external artefacts:

| External element | Direction | What is exchanged | Requirement |
|---|---|---|---|
| XLSX files | in / out | Reference data imported all-or-nothing with a per-row error report; published timetable exported by cohort, teacher and room | FR-CAT-04, FR-CAT-09, FR-SCH-12 |
| SVG floor plans | in | One plan per floor; room elements matched to rooms by `id` | FR-BOOK-07, NFR-SEC-11 |
| Calendar clients | out | Read-only `.ics` subscription with a revocable token | FR-ENR-09, NFR-SEC-12 |
| E-mail server | out | Option-ready and option-expired notices (Could) | FR-CONF-08 |
| ITC-2007 Track 3, UD2 validator, OR-Tools CP-SAT | offline | Benchmark instances and cost checks for `solverctl`; CP-SAT runs as a separate baseline script, never inside the product | FR-SCH-14, TC §3.5 |

AITU LMS and Microsoft 365 single sign-on are shown only to mark the boundary: both are Won't in this release (FR-CAT-10, FR-AUTH-08, P §10). The system therefore keeps its own accounts and imports data from Excel instead of synchronising with university systems.

## 3. Use cases


The UML version with stick-figure actors and use case ellipses is [02-use-case.drawio.svg](diagrams/src/02-use-case.drawio.svg). In the Mermaid rendering above, which has no UML use case notation, actors are drawn as rounded nodes, use cases as stadium shapes and the system boundary as a box. «include» and «extend» are labelled dashed arrows that point from the including or extending use case to its target. The "User" actor generalises the three roles for the use cases that every role has.

| UC | Use case | Actor | Requirements | Module |
|---|---|---|---|---|
| UC-01 | Manage reference data (programs, cohorts, courses, streams, teachers, rooms, buildings, floors) | Admin | FR-CAT-01, FR-CAT-05…08 | catalog |
| UC-02 | Import reference data from XLSX | Admin | FR-CAT-04, FR-CAT-09, FR-AUTH-06 | catalog |
| UC-03 | Configure term, time grid and holidays | Admin | FR-CAT-02, FR-CAT-03 | catalog |
| UC-04 | Set objective weights w1-w5 | Admin | FR-CON-03 | constraints |
| UC-05 | Generate timetable | Admin | FR-SCH-02, FR-SCH-04, FR-SCH-05, FR-SCH-13 | scheduling |
| UC-06 | Run prechecks | system (included) | FR-SCH-01 | scheduling / solver |
| UC-07 | View infeasibility report with repair hints | Admin | FR-SCH-06, FR-SCH-07 | scheduling / solver |
| UC-08 | Watch generation progress | Admin | FR-SCH-03 | scheduling, SSE |
| UC-09 | View timetable grid by cohort, teacher or room; move a session by hand | Admin | FR-SCH-08, FR-SCH-09 | scheduling |
| UC-10 | Send draft to teacher review | Admin | FR-CONF-01 | confirmation |
| UC-11 | Track teacher confirmations | Admin | FR-CONF-06, FR-CONF-07 | confirmation |
| UC-12 | Publish version | Admin | FR-SCH-11 | scheduling |
| UC-13 | Validate version | system (included) | FR-SCH-10, NFR-COR-01 | solver (validator) |
| UC-14 | Export timetable to XLSX | Admin | FR-SCH-12 | scheduling |
| UC-15 | Manage accounts | Admin | FR-AUTH-05, FR-AUTH-07 | auth |
| UC-16 | Set unavailability and preferences | Teacher | FR-CON-01, FR-CON-02, FR-CON-04…06 | constraints |
| UC-17 | Review 1-3 timetable options | Teacher | FR-CONF-01, FR-CONF-02 | confirmation |
| UC-18 | Confirm option | Teacher | FR-CONF-03, FR-CONF-05 | confirmation |
| UC-19 | Request regeneration with a comment | Teacher | FR-CONF-04 | confirmation |
| UC-20 | Browse streams of own program | Student | FR-ENR-01, FR-ENR-08 | enrollment |
| UC-21 | Enroll in stream | Student | FR-ENR-02, FR-ENR-03, FR-ENR-04, FR-ENR-07 | enrollment |
| UC-22 | Switch stream | Student | FR-ENR-05, FR-ENR-07 | enrollment |
| UC-23 | Drop stream | Student | FR-ENR-06, FR-ENR-07 | enrollment |
| UC-24 | Log in, refresh session, log out | All | FR-AUTH-01, FR-AUTH-02, FR-AUTH-04 | auth |
| UC-25 | View personal timetable, subscribe via iCal | All | FR-ENR-09 | enrollment |
| UC-26 | View floor map with room status | All | FR-BOOK-01, FR-BOOK-10 | booking + map |
| UC-27 | Search and book a free room | All | FR-BOOK-02, FR-BOOK-03, FR-BOOK-05, FR-BOOK-08 | booking + map |
| UC-28 | Cancel own booking | All | FR-BOOK-04, FR-BOOK-06 | booking + map |

Admin-only booking actions (upload floor plans FR-BOOK-07, cancel any booking FR-BOOK-09) are parts of UC-01 and UC-28 for the admin role and are not drawn separately. Role and ownership checks (FR-AUTH-03) apply to every use case.

## 4. Components (draft)

### 4.1 Component responsibilities

| Component | Responsibility | Requirements |
|---|---|---|
| React SPA | Screens for three roles; weekly grid; SVG floor map; language switch; keeps the access token in memory; opens SSE with `fetch` | FR-SCH-08, FR-ENR-01, FR-BOOK-01, NFR-USE-02, NFR-USE-03, NFR-SEC-03, TC §3.4, §3.6 |
| Generated TS client | Typed calls generated from `api/openapi.yaml`; CI fails if it is out of date | FR-X-03 |
| Caddy | HTTPS with HSTS, HTTP/2 (no six-connection limit for SSE), reverse proxy to `api`, static files of the SPA | NFR-SEC-01, TC §3.4 |
| HTTP layer | Routing, authentication middleware, role and ownership checks, RFC 9457 errors, `Idempotency-Key`, rate limits, request id for logs | FR-AUTH-03, FR-X-01, FR-X-02, NFR-SEC-04, NFR-SEC-05, NFR-OBS-01 |
| auth | Login, refresh-token rotation and revocation, password hashing (argon2id), accounts, `GET /me` | FR-AUTH-01…07, NFR-SEC-02, NFR-SEC-03 |
| catalog | CRUD of reference data; time grid and holidays; XLSX import with per-row report; deletion guard | FR-CAT-01…09 |
| constraints | Teacher unavailability and preferences, deadlines, objective weights | FR-CON-01…08 |
| scheduling | Generation runs (enqueue, cancel, store), versions, manual moves, validation before publish, atomic publish into `room_occupancy`, XLSX export | FR-SCH-01…13, FR-SCH-15 |
| confirmation | Options for each teacher, atomic confirm with `PROPOSAL_EXPIRED`, local regeneration jobs, status board, deadline | FR-CONF-01…08 |
| enrollment | Available streams, atomic seat reservation, clash check, switch, drop, enrollment window, personal timetable, iCal | FR-ENR-01…11 |
| booking + map | Room availability by interval, search, booking via the exclusion constraint, limits, SVG plan upload and sanitising | FR-BOOK-01…12, NFR-SEC-11 |
| SSE hub | Listens to PostgreSQL `NOTIFY` and pushes generation progress and seat counts to open clients; resumes from the last event id | FR-SCH-03, FR-ENR-08, NFR-PERF-07, NFR-REL-05 |
| audit log | One row per significant action with actor, action, entity, time and request id | NFR-SEC-07 |
| River job consumer (worker) | Takes `generate` and `regenerate-teacher` jobs, builds the solver problem from the database, writes the version, metrics or report, reports progress, supports cancel | FR-SCH-02…05, FR-CONF-04, NFR-REL-01, NFR-REL-02 |
| solver: model, prechecks | Sessions, domains (room type, capacity, availability), slot bit masks; four prechecks before search | FR-SCH-01, NFR-PERF-08 |
| solver: backtracking | First feasible timetable with MRV, LCV and forward checking; confirmed sessions fixed | FR-SCH-05, FR-CONF-05, NFR-PERF-01 |
| solver: simulated annealing | Improves soft-constraint cost F with incremental ΔF; seeded and reproducible | NFR-PERF-02, NFR-COR-03, NFR-COR-04 |
| solver: QuickXplain | Minimal set of conflicting constraint groups and repair hints | FR-SCH-06, FR-SCH-07, NFR-PERF-09, NFR-COR-05 |
| solver: validator | Independent recomputation of hard violations and F; used before publish, in tests and in experiments | FR-SCH-05, FR-SCH-10, NFR-COR-01 |
| PostgreSQL 18 | Domain data; exclusion constraint on room occupations; `CHECK enrolled <= capacity`; unique enrollment per course; River queue; idempotency keys | NFR-COR-02, NFR-COR-06, NFR-REL-04, TC §3.2 |
| solverctl | Runs the solver on JSON or ITC-2007 `.ctt` input, prints solution and metrics | FR-SCH-14 |
| CP-SAT baseline script | Same instances solved with OR-Tools CP-SAT for comparison; not deployed | P §6 sub-question 2, TC §3.5 |

### 4.2 Interfaces

| Interface | Between | Notes |
|---|---|---|
| REST over HTTPS, described in `api/openapi.yaml` | SPA → api | JSON; errors as `application/problem+json`; mutating requests carry `Idempotency-Key` (FR-X-01…03) |
| Server-Sent Events | api → SPA | Opened with `fetch` so the `Authorization` header can be sent; polling every 15 s as fallback (TC §3.4) |
| SQL through pgx and sqlc | api, worker → PostgreSQL | Parameterised queries only (NFR-SEC-06) |
| River `InsertTx` | scheduling, confirmation → queue | A job exists if and only if the run row was committed (FR-SCH-02, TC §3.3) |
| `LISTEN/NOTIFY` | worker and modules → SSE hub | Payload carries ids and counters only, no personal data (TC §3.4) |
| Go interface `Solve(Problem) → Solution / InfeasibilityReport` | worker, solverctl → solver | Lets CP-SAT be placed behind the same interface if risk R1 occurs ([06 §3](06-project-management.md)) |

## 5. Key runtime scenarios

Each scenario lists the components it passes through and the error codes it can return, so it can be checked against the use cases (Section 3) and the acceptance criteria.

### 5.1 Generation run (UC-05, UC-06, UC-07, UC-08)

Requirements: FR-SCH-01…06, NFR-PERF-01, NFR-PERF-02, NFR-PERF-06, NFR-PERF-09, NFR-REL-01.

### 5.2 Enrollment under concurrency (UC-21)

1. The student sends `POST /enrollments {streamId}` with an `Idempotency-Key`.
2. The enrollment module checks the enrollment window (`403 ENROLLMENT_CLOSED`) and the clash with the student's other streams (`409 SCHEDULE_CONFLICT`).
3. In one transaction it runs `UPDATE streams SET enrolled = enrolled + 1 WHERE id = $1 AND enrolled < capacity`. Zero updated rows means `409 STREAM_FULL`. Under Read Committed the `WHERE` is re-checked on the latest row version, so the counter cannot pass the capacity (TC §3.2).
4. It inserts the enrollment row. The unique constraint `(student, course, term)` rejects a second stream of the same course (`409 ALREADY_ENROLLED`).
5. On commit a `NOTIFY` with stream id and new count updates open pages through SSE (FR-ENR-08).

Requirements: FR-ENR-02…04, FR-ENR-07, FR-X-02, NFR-PERF-04, NFR-COR-02, NFR-COR-06.

### 5.3 Room booking (UC-27)

1. The user searches free rooms for an interval (FR-BOOK-02) or picks a room on the floor map (FR-BOOK-01).
2. `POST /bookings {roomId, from, to}`. The booking module checks role limits (`422 BOOKING_LIMIT`) and horizon (`422 OUTSIDE_HORIZON`).
3. It inserts a `room_occupation` row. The exclusion constraint `EXCLUDE USING gist (room_id WITH =, during WITH &&) WHERE (status = 'active')` rejects any overlap with a lesson or another booking. The SQLSTATE `23P01` is mapped to `409 ROOM_ALREADY_BOOKED` with the overlapping occupation.

Requirements: FR-BOOK-03, FR-BOOK-05, FR-BOOK-08, NFR-COR-02.

### 5.4 Teacher confirmation (UC-17, UC-18, UC-19)

1. When the admin sends a draft to review, the confirmation module enqueues one `regenerate-teacher` job per teacher to build options 2 and 3 (FR-CONF-01).
2. The teacher opens the options and confirms one. In one transaction the module locks the version's assignments for the option's slot-room pairs and checks that none was taken since the option was built. If one was taken it returns `409 PROPOSAL_EXPIRED` and enqueues new options (FR-CONF-03).
3. On success the teacher's assignments are marked `confirmed` and stay fixed in all later runs (FR-CONF-05).
4. "Request regeneration" stores the comment and enqueues a local re-solve with all other teachers fixed (FR-CONF-04, NFR-PERF-03).

## 6. Data model (draft)


Attributes and database constraints of the key tables are in [04-data-model.drawio.svg](diagrams/src/04-data-model.drawio.svg). The decisions that matter for correctness are:

- Weekly assignments are expanded into dated `room_occupation` rows when a version is published, so one exclusion constraint protects both lessons and bookings (TC §4 data model, FR-SCH-11).
- `streams.enrolled` has `CHECK (enrolled <= capacity)` as a second line of defence behind the conditional update (NFR-COR-02).
- Assignments of one version are unique per room-slot and per teacher-slot (H1, H2 in requirements §5).
- Deletion of referenced entities is blocked; entities are deactivated instead (FR-CAT-08).

The schema will be written as goose migrations by Aiya at the checkpoint "Database schema, migrations, synthetic data generator" (24.11.2026).

## 7. Deployment view

| Service (Docker Compose) | Image / binary | Exposed | Notes |
|---|---|---|---|
| `caddy` | Caddy | 80, 443 | TLS, HTTP/2, serves SPA build, proxies `/api` |
| `api` | project binary `--mode=api` | internal | Stateless; can be scaled, SSE fan-out through `LISTEN/NOTIFY` |
| `worker` | project binary `--mode=worker` | none | Runs solver jobs; 4 CPU cores in the reference setup (NFR-SCALE-02) |
| `postgres` | PostgreSQL 18 | internal | Volume for data; daily `pg_dump` (NFR-REL-03) |

Configuration comes from environment variables; the repository contains only `.env.example` (NFR-SEC-08). The whole system starts with `docker compose up` (NFR-DEP-01).

## 8. Consistency check

The table checks that every user-facing Must module has a use case, a component, a scenario or data structure, and a mock-up in [08-ux-prototype.md](08-ux-prototype.md).

| Module | Must FRs | Use cases | Components | Scenario / data | Mock-up (doc 08) |
|---|---|---|---|---|---|
| auth | FR-AUTH-01…05 | UC-15, UC-24 | HTTP layer, auth | `USER` | Login (§2.0) |
| catalog | FR-CAT-01…08 | UC-01, UC-02, UC-03 | catalog | `TERM`, `TIME_SLOT`, `STREAM`, `ROOM` | Import (§2.3) |
| constraints | FR-CON-01…03 | UC-04, UC-16 | constraints | `TEACHER_CONSTRAINT` | Teacher grid (§2.4) |
| scheduling | FR-SCH-01…03, 05, 06, 08, 10…12, 14 | UC-05…09, UC-12…14 | scheduling, worker, solver | §5.1; `GENERATION_RUN`, `SCHEDULE_VERSION`, `ASSIGNMENT` | Generation (§2.1), report (§2.2) |
| confirmation | FR-CONF-01, 03…06 | UC-10, UC-11, UC-17…19 | confirmation, worker | §5.4; `OPTION` | Teacher options (§2.4) |
| enrollment | FR-ENR-01…07, 09 | UC-20…23, UC-25 | enrollment, SSE hub | §5.2; `ENROLLMENT` | Student streams (§2.5) |
| booking + map | FR-BOOK-01…07 | UC-26…28 | booking + map | §5.3; `ROOM_OCCUPATION`, `BOOKING` | Floor map (§2.6) |
| cross-cutting | FR-X-01…03 | all | HTTP layer, TS client | API examples (doc 08 §3) | — |

## 9. Open points

These items come from requirements §6 and affect the architecture. They are to be agreed with the supervisor and recorded in [11-supervisor-feedback.md](11-supervisor-feedback.md).

| # | Item | Architectural impact |
|---|---|---|
| 1 | AITU's real time grid is unknown | None: the grid is data (FR-CAT-02) |
| 2 | Strictness of H4 for cohorts with several streams | Changes the solver's constraint set, not the components |
| 3 | No real AITU data or floor plans yet | Synthetic generator and team-drawn SVG plans use the same import paths |
| 4 | Whether the own solver meets NFR-PERF-01 at 400 streams | CP-SAT can be placed behind the same `Solve` interface (R1) |

