# Requirements specification for AITU Smart Schedule

Project: Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms (working name: AITU Smart Schedule). 

Team: Aizada Sembayeva, Aiya Zhakupova. 

Supervisor: Nadirov Nurtas.

This document specifies the functional requirements of the platform's seven modules (auth, catalog, constraints, scheduling, confirmation, enrollment, booking and map), plus cross-cutting API rules, and the non-functional requirements with measurable target values. Every requirement has a MoSCoW priority and a source. Every Must requirement has an acceptance criterion that a test, an experiment or an inspection can check.

Requirements cite four project documents only (Section 2.3). No interviews are held, so requirements about AITU's current process rest on the problem statement in the proposal (P §4), which is based on public AITU sources, together with the literature review and the existing-systems comparison. No numeric target is agreed with the supervisor yet: Section 4 marks each target as stated in the proposal or set by the team in this document.

## 1. Scope

### 1.1 Users

| Role | Main goals | Modules |
|---|---|---|
| Admin / scheduler (Registrar's Office) | Keep reference data, import it from Excel, generate a timetable without hard-constraint violations, understand why generation fails, collect teacher confirmations, publish and export | auth, catalog, constraints, scheduling, confirmation, enrollment (statistics), booking |
| Teacher | Enter unavailability and preferences, review 1-3 valid options for own classes, confirm one or ask for regeneration, book a free room | auth, constraints, confirmation, enrollment (own timetable), booking |
| Student | See the streams of own program, enroll in streams without clashes, keep a personal timetable, book a free room for self-study | auth, enrollment, booking |

### 1.2 In and out of scope

In scope (MVP, P §10): reference data and XLSX import; two-phase generation with an infeasibility report; teacher options, confirmation and local regeneration; enrollment in streams of the student's own program; room booking with an SVG floor map; export to XLSX and iCal.

Out of scope (Won't in this release; listed as requirements so the boundary is explicit): integration with LMS, Microsoft 365 single sign-on, a mobile application, exam timetabling, teacher substitutions during the semester, the open elective catalog, waitlists and priority-based allocation rounds.

### 1.3 Glossary

| Term | Meaning |
|---|---|
| Course | A discipline in the curriculum, for example "Algorithms and Data Structures" |
| Stream | One offering of a course with one teacher and a seat capacity. A student chooses a stream, which fixes teacher and times. |
| Session | One weekly class of a stream (lecture, practice or lab), with a duration in pairs and a required room type. A stream with one lecture and two practices has three sessions. |
| Cohort | Students of one program in one year of study. Mandatory single-stream sessions of a cohort must not overlap (H4). |
| Slot | One pair on one weekday in the term's time grid, for example "Tuesday, pair 3" |
| Term | A semester with dates, holidays, time grid, objective weights and an enrollment window |
| Schedule version | A set of session assignments (session → slot, room) with status draft, review or published |
| Option (proposal) | One of 1-3 alternative placements of one teacher's sessions offered during review |
| Constraint group | A labelled set of constraints used by the infeasibility report, for example "unavailability of teacher X" or "capacity of computer labs" |
| Room occupation | A dated time interval during which a room is used by a lesson or a booking |

## 2. How requirements are prioritised, verified and traced

### 2.1 MoSCoW

- Must: the MVP fails, or a proposal objective cannot be met, without it.
- Should: planned for the MVP and important, but leaving it out does not break a Must requirement.
- Could: implemented only if time remains after all Must and Should requirements.
- Won't: out of scope for this release; recorded for the "further development" chapter.

### 2.2 Verification levels

Each acceptance criterion names how it is checked:

| Code | Level | Tool |
|---|---|---|
| U | Unit or property-based test | go test, rapid / testing/quick |
| I | Integration test against real PostgreSQL | testcontainers-go, `-race` |
| L | Load test | k6 |
| E | End-to-end or manual walkthrough of the UI | Playwright, or a recorded manual check |
| X | Experiment reported in the practical chapter | solverctl, experiment scripts |
| R | Review or inspection of code, configuration or CI | Pull request review, CI job |

### 2.3 Source codes

| Code | Source |
|---|---|
| P §n / P obj. n | Project proposal, docs/01-proposal.md (section or objective number) |
| LR §n | Literature review, docs/02-literature-review.md |
| ES §n | Existing-systems comparison, docs/03-existing-systems.md |
| TC §n | Technology comparison, docs/04-technology-comparison.md |
## 3. Functional requirements

Columns: ID; requirement ("the system shall …" is implied); priority (M, S, C, W); acceptance criterion; verification level (2.2); source.

### 3.1 Authentication and access (auth)

| ID | Requirement | P | Acceptance criterion | Ver. | Source |
|---|---|---|---|---|---|
| FR-AUTH-01 | Authenticate a user by email and password, and issue an access JWT valid for 15 minutes and a refresh token in a cookie with HttpOnly, Secure and SameSite=Strict | M | Valid credentials return 200 with an access token whose `exp − iat` = 900 s and set the refresh cookie with all three flags. A wrong password and an unknown email return the same 401 body (no account enumeration). | I | P §5; TC §4 |
| FR-AUTH-02 | Rotate the refresh token on every refresh, and revoke all refresh tokens of a user on logout or deactivation | M | After a refresh, the previous refresh token returns 401. After logout, refresh returns 401. After the admin deactivates a user, that user's next refresh returns 401. | I | TC §4 |
| FR-AUTH-03 | Check role (admin, teacher, student) and ownership on every endpoint: a teacher acts only on own constraints and options, a student only on own enrollments and bookings | M | An automated matrix test calls every operation in `api/openapi.yaml` as each role and with another user's resource id. Every case marked forbidden returns 403 and leaves the database unchanged. | I | P §5; TC §4 |
| FR-AUTH-04 | Return the current user's profile: role, linked teacher or student record, program and cohort for students, interface language | M | For one seeded user of each role, `GET /me` matches the seeded record field by field. | I | P §5 |
| FR-AUTH-05 | Let the admin create, edit and deactivate accounts and link them to teacher or student records | M | A created account can log in. A deactivated account cannot log in or refresh. Linking to a non-existent teacher returns 422. | I | P §5 |
| FR-AUTH-06 | Create accounts in bulk from the XLSX import; new accounts must change their password at first login | S | Importing 500 student rows creates 500 accounts; the first login of each redirects to a password change. | I | P §5, obj. 6 (data kept in Excel) |
| FR-AUTH-07 | Let a user change own password and let the admin reset a password, forcing a change at next login | S | After reset, the old password fails and the first login with the new one requires a change. | I | TC §4 |
| FR-AUTH-08 | Single sign-on with Microsoft 365 | W | — | — | P §10; ES §3 (AITU uses Office 365 sign-in); TC §4 |

### 3.2 Reference data (catalog)

| ID | Requirement | P | Acceptance criterion | Ver. | Source |
|---|---|---|---|---|---|
| FR-CAT-01 | Let the admin create, read, update and deactivate programs, cohorts, courses, streams, teachers, rooms, buildings and floors | M | One integration test per entity covers create, read, update, deactivate; list endpoints support pagination and search by code or name. | I | P §5, obj. 6 |
| FR-CAT-02 | Configure the time grid per term (working days, pairs per day, start and end time of each pair) as data, not code | M | Two terms, one with 5 days × 6 pairs and one with 6 days × 8 pairs, both generate without a code change, and the solver's slot count equals days × pairs in each. | I, U | P open items (AITU time grid unknown) |
| FR-CAT-03 | Store term start and end dates and a list of holidays, and expand published sessions only onto teaching days | M | In a 15-week term with one Monday holiday, publishing a weekly Monday session creates exactly 14 dated room occupations. | I | TC §4 (data model); P §10 |
| FR-CAT-04 | Import reference data from XLSX all-or-nothing, with an error report per row (sheet, row, column, message); re-importing updates by code instead of duplicating | M | A 200-row file with 3 invalid rows writes nothing and the report lists exactly those 3 rows. A valid file writes all rows. Importing the same valid file twice leaves the row counts unchanged. | I | P §4, §5, obj. 6 |
| FR-CAT-05 | Store for each room: code, building, floor, capacity, type (lecture, computer, laboratory, other), features, bookable flag and SVG element id | M | A room without capacity or type is rejected with 422. Room type and capacity are used in session domains (H-domain, Section 5). | I | LR §2.7 |
| FR-CAT-06 | Store for each stream: course, teacher, capacity, linked cohorts and session templates (type, duration in pairs, required room type, sessions per week) | M | Capacity ≤ 0 is rejected with 422. A stream with no room of the required type and sufficient capacity is reported by precheck (FR-SCH-01). | I | P obj. 3; LR §2.1 |
| FR-CAT-07 | Link program → cohort → courses for each term, and derive for each course whether it has a single stream for the cohort | M | The single-stream flag changes from true to false when a second stream of the course is added for that cohort. The student test in FR-ENR-01 relies on these links. | I | P §4, §5 |
| FR-CAT-08 | Block deletion of entities that a published version or an enrollment references; allow deactivation instead | M | Deleting a room used in a published version returns 409 with the list of references. Deactivating it succeeds and excludes it from later generation runs. | I | TC §4 (data model) |
| FR-CAT-09 | Provide XLSX templates matching the import format | S | A downloaded template filled with valid data imports with 0 errors. | I | P §5 |
| FR-CAT-10 | Synchronise with AITU LMS | W | — | — | P §10; ES §3 |

### 3.3 Teacher constraints and objective weights (constraints)

| ID | Requirement | P | Acceptance criterion | Ver. | Source |
|---|---|---|---|---|---|
| FR-CON-01 | Let a teacher mark unavailable slots on the term's weekly grid; unavailability is a hard constraint | M | A teacher marks Friday unavailable; in the next generation run the validator finds none of that teacher's sessions on Friday. | I, U | P §4, §5; ES §5.2 |
| FR-CON-02 | Let a teacher mark preferred and avoided slots with a weight from 1 to 3; these form soft constraint S1 | M | For a fixed solution, changing one preference changes the S1 term computed by the validator by the expected amount. A weight outside 1-3 returns 422. | U, I | P §5, obj. 3 |
| FR-CON-03 | Let the admin set the objective weights w1-w5 per term | M | A run started after a change records the new weights in its parameters. A negative weight returns 422. | I | P obj. 3 |
| FR-CON-04 | Warn a teacher on saving when the available slots are fewer than the slots the teacher's sessions need | S | A teacher with 10 pairs of load who leaves 8 slots available sees a warning naming 8 and 10; saving is still allowed. | I, E | ES §5.6 (diagnosis before generation) |
| FR-CON-05 | Let a teacher attach a reason to an unavailable slot, shown to the admin and in infeasibility reports | S | The reason appears in the admin view and in the report text when that slot is part of a reported conflict. | I | ES §6 (UniTime reason field) |
| FR-CON-06 | Let the admin set a deadline for constraint entry, after which teacher constraints are read-only unless reopened for a teacher | S | A change after the deadline returns 403 `CONSTRAINTS_LOCKED`; after the admin reopens that teacher, it returns 200. | I | ES §6 (UniTime survey lock) |
| FR-CON-07 | Let the admin edit constraints on behalf of a teacher, recorded in the audit log with the admin as author | S | The audit entry names the admin as actor and the teacher as target. | I | ES §5.2 |
| FR-CON-08 | Let the admin switch a constraint type between hard and soft | C | A switched type is treated as a penalty with the given weight by both solver and validator. | U | LR §2.1 |

### 3.4 Timetable generation (scheduling)

| ID | Requirement | P | Acceptance criterion | Ver. | Source |
|---|---|---|---|---|---|
| FR-SCH-01 | Run prechecks before search: a session with an empty domain; a teacher whose load exceeds available slots; demand for a room type above that type's slot capacity; cohort sessions above the slots in a week | M | Four prepared invalid datasets, one per check, each produce exactly the matching precheck error; a valid dataset produces none; run time within NFR-PERF-08. | U | P §4; ES §5.6, §6 (aSc pre-generation test) |
| FR-SCH-02 | Let the admin start a generation run for a term with a time limit, seed and number of workers; queue it and return its id at once; allow one active run per term | M | `POST` returns 202 with the run id. A second `POST` while a run is active returns 409 `RUN_IN_PROGRESS`. A test that forces a rollback after inserting the run leaves neither a run row nor a job. | I | P §5; TC §3.3 |
| FR-SCH-03 | Show run progress live: phase, elapsed time, best objective value F, unassigned sessions | M | During a run of at least 30 s the client receives at least one event every 2 s, and the last event equals the stored final metrics. | I, E | TC §3.4 |
| FR-SCH-04 | Let the admin cancel a run, keeping the best feasible solution found so far as a draft if one exists | S | Cancelling after the first feasible solution saves a draft with 0 hard violations; cancelling before it saves none; the worker stops within 2 s. | I | TC §3.3 |
| FR-SCH-05 | Store a finished run as a draft version with parameters, seed and metrics: time to first feasible, final F and its terms S1-S5, hard violations, room utilisation | M | All stored metrics equal the values the independent validator recomputes from the stored assignments. | I | P §9 (metrics) |
| FR-SCH-06 | When no feasible timetable is found, return a report naming a minimal set of conflicting constraint groups | M | On at least 10 test instances with one planted conflict each, the report names the planted groups and no others; removing any one named group makes the instance feasible; the text exists in all three languages. Time limit per NFR-PERF-09. | U, X | P §4, obj. 5; LR §2.4; ES §5.6 |
| FR-SCH-07 | Suggest at least one repair action for each conflict in the report (relax an unavailability, add a room of a type, split a stream, add a slot) | S | Every reported conflict has at least one hint; on each planted-conflict instance, applying the suggested hint makes it feasible. | X | LR §2.4 (Steiner et al., 2025) |
| FR-SCH-08 | Show a version as a weekly grid filtered by cohort, teacher or room, with soft-constraint penalties highlighted | M | For a seeded version, the grid for a chosen cohort, teacher and room shows exactly the sessions the API returns for that filter. | E | P §5 |
| FR-SCH-09 | Let the admin move a session by hand; reject moves that break a hard constraint and show the change in F before saving | S | Moving onto an occupied room returns `SCHEDULE_CONFLICT` naming the other session; a valid move shows a ΔF equal to the validator's recomputation. | I | ES §6 |
| FR-SCH-10 | Run the independent validator before publishing and block publishing if it reports any hard violation | M | A version with an H1 violation injected by SQL returns 409 on publish with the violation list; a clean version publishes. | I | P §4, obj. 4 |
| FR-SCH-11 | Publish atomically: set status to published, expand sessions into dated room occupations, open the enrollment window; if any occupation overlaps an active booking, publish nothing and list the bookings | M | After publish, lesson occupations = Σ (sessions × teaching weeks). With one overlapping booking present, publish returns 409 listing it and writes 0 occupation rows. | I | P §5, obj. 7; TC §4 (data model) |
| FR-SCH-12 | Export a published version to XLSX by cohort, teacher and room | M | The file opens in Excel and LibreOffice, and its lesson rows equal the sessions in the version. | I | P §5, §10 |
| FR-SCH-13 | Let the admin run a global regeneration that keeps all confirmed sessions fixed | S | Every confirmed session keeps its slot and room after regeneration. | I | P §10 |
| FR-SCH-14 | Provide the command-line tool `solverctl`, which runs the solver on a JSON problem or an ITC-2007 Track 3 `.ctt` file and outputs the solution and metrics | M | On comp01 the output has 0 hard violations, and the cost it reports equals the cost from the official UD2 validator. | X | P obj. 8; LR §2.2 |
| FR-SCH-15 | Compare two versions: moved sessions and difference in F | C | The list of moved sessions equals a SQL diff of the two versions. | I | ES §6 |

### 3.5 Teacher confirmation (confirmation)

| ID | Requirement | P | Acceptance criterion | Ver. | Source |
|---|---|---|---|---|---|
| FR-CONF-01 | When a draft is sent to review, give each teacher 1-3 valid options for own sessions (option 1 from the global solution, options 2-3 from re-solving only that teacher's sessions), shown on a weekly grid with the teacher's preferences marked | M | Each option passes the validator together with the rest of the version; other teachers' sessions are identical across all options; a teacher with no alternative sees option 1 and a message saying no alternative exists. | U, I | P §4, §5; ES §5.2 |
| FR-CONF-02 | Make options differ noticeably: each alternative differs from option 1 in the set of teaching days or in the slot of at least one third of the teacher's sessions (team value) | S | A unit test over generated options checks the rule for every teacher in the reference instance. | U | P §5 |
| FR-CONF-03 | Confirm an option in one transaction; if a slot-room pair in it was taken meanwhile, fail with `PROPOSAL_EXPIRED` and queue new options | M | Two teachers confirm, at the same time, options that share one free slot-room pair: exactly one gets 200 and the other `PROPOSAL_EXPIRED`, and the second teacher has new options within NFR-PERF-03. | I | P obj. 7; TC §2 (D3) |
| FR-CONF-04 | Let a teacher request regeneration with a comment; re-solve only that teacher's sessions with everything else fixed | M | After the request, a diff of all other teachers' sessions is empty and new options exist within NFR-PERF-03. | I | P §5, §10 |
| FR-CONF-05 | Treat confirmed sessions as hard constraints in every later local or global run of the version | M | Over 10 seeds, no later run changes the slot or room of a confirmed session. | U, I | P §10; LR §2.7 (Müller, 2005) |
| FR-CONF-06 | Show the admin each teacher's status (pending, confirmed, regeneration requested, expired) and totals | M | After a scripted sequence of confirmations and requests, the counts equal a direct database query. | I | P §10 |
| FR-CONF-07 | Set a confirmation deadline after which option 1 is accepted for teachers who did not respond | S | At the deadline, pending teachers become confirmed with option 1 and the audit log marks them "auto-confirmed". | I | P §10 |
| FR-CONF-08 | Notify teachers by email when options are ready or have expired | C | A test mail server receives one message per event. | I | P §5 |
| FR-CONF-09 | Let a teacher place sessions in any slot and room outside the offered options | W | — | — | P §7; ES §5.2 |

### 3.6 Stream enrollment (enrollment)

| ID | Requirement | P | Acceptance criterion | Ver. | Source |
|---|---|---|---|---|---|
| FR-ENR-01 | Show a student the streams of the courses in the student's cohort curriculum for the term, with teacher, times, free seats and a mark when a stream clashes with current enrollments | M | For a seeded student the list contains exactly the expected streams; the clash mark is true exactly for streams sharing a slot with an enrolled stream. | I | P §4, §5, §10 |
| FR-ENR-02 | Reserve a seat atomically on enrollment, and fail with `STREAM_FULL` when no seat is left | M | 200 concurrent requests from different students into a 30-seat stream give exactly 30 successes and 170 `STREAM_FULL`, and afterwards `enrolled` = 30 = number of enrollment rows. | I, L | P §4, obj. 7; LR §2.5; TC §3.2 |
| FR-ENR-03 | Fail with `SCHEDULE_CONFLICT` when the new stream shares a slot with any stream the student is enrolled in this term, naming the stream and slot | M | A clashing request returns 409 with the conflicting stream id and slot. Two concurrent requests by one student into two mutually clashing streams result in at most one enrollment. | I | P §4, §5 |
| FR-ENR-04 | Allow at most one stream per course per student per term | M | Enrolling in a second stream of the same course returns 409 `ALREADY_ENROLLED`; a direct SQL insert of such a row is rejected by the unique constraint. | I | P §5; TC §2 (D8) |
| FR-ENR-05 | Switch stream in one transaction, keeping the old enrollment if the new stream is full or clashes | M | Switching into a full stream returns `STREAM_FULL` and the student stays in the old stream; a successful switch lowers the old stream's `enrolled` by 1 and raises the new one's by 1. | I | P §4; TC §2 (D3) |
| FR-ENR-06 | Let a student drop an enrollment and release the seat | M | After the drop, `enrolled` falls by 1 and the stream shows one more free seat. | I | P §5 |
| FR-ENR-07 | Allow enrolling, switching and dropping only inside the enrollment window set by the admin | M | Requests before or after the window return 403 `ENROLLMENT_CLOSED`; inside it they behave normally. | I | P §4 |
| FR-ENR-08 | Update free-seat counts live on open pages | S | After an enrollment commits, a second open client shows the new count within NFR-PERF-07. | I, E | TC §3.4 |
| FR-ENR-09 | Show students and teachers a personal weekly timetable, and offer an iCal subscription URL with a revocable per-user token | M | The `.ics` file parses with an RFC 5545 library and its events equal the user's sessions on the term's teaching days; after the token is revoked the URL returns 404. | I | P obj. 6, §10 |
| FR-ENR-10 | Show the admin the fill rate per stream and the students who cannot build a clash-free set of streams for their mandatory courses | S | Both numbers equal reference SQL queries on seeded data. | I | P §9 (share of students with a clash-free set) |
| FR-ENR-11 | Let the admin enroll, move or remove a student, applying the same capacity and clash checks and writing an audit entry | S | An admin move into a full stream returns `STREAM_FULL`; a valid move creates one audit entry. | I | P §4 |
| FR-ENR-12 | Waitlist for full streams | W | — | — | ES §5.3 |
| FR-ENR-13 | Priority-based or batch allocation round before first-come-first-served enrollment | W | — | — | LR §2.5 (Romero-Medina & Triossi, 2024) |
| FR-ENR-14 | Open elective catalog across programs | W | — | — | P §10 |

### 3.7 Room booking and floor map (booking + map)

| ID | Requirement | P | Acceptance criterion | Ver. | Source |
|---|---|---|---|---|---|
| FR-BOOK-01 | Show any user a floor plan with each room's status (free, lesson, booked, not bookable) for a chosen date and time interval, marked by text or pattern as well as colour | M | For a seeded interval, the status of every room on the plan equals a reference query on room occupations; status is readable in a greyscale screenshot. | I, E | P §4, §10; ES §5.5 |
| FR-BOOK-02 | Search free rooms by interval, minimum capacity, type and features, as a list and highlighted on the map | M | Results equal a reference SQL query on seeded data, and no room with an overlapping active occupation appears. | I | P §4, §5 |
| FR-BOOK-03 | Book a free bookable room for an interval; an overlap with a lesson or another active booking is rejected by the database and returned as `ROOM_ALREADY_BOOKED` naming the overlapping occupation | M | 50 concurrent overlapping bookings of one room give exactly 1 success. An overlapping row inserted by direct SQL fails with `exclusion_violation` (SQLSTATE 23P01). | I | P §4, obj. 7; LR §2.5; TC §3.2 |
| FR-BOOK-04 | Let a user cancel own future booking, freeing the room at once | M | After cancelling, the availability query shows the room free for that interval; cancelling another user's booking returns 403. | I | P §5 |
| FR-BOOK-05 | Enforce configurable booking limits per role; student default: at most 2 hours per booking and at most 2 active future bookings | M | A student's third active booking and a student's 3-hour booking each return 422 `BOOKING_LIMIT`. | I | P §5; ES §6 |
| FR-BOOK-06 | List a user's own bookings | M | The list equals the user's active and past bookings in the seeded data. | I | P §5 |
| FR-BOOK-07 | Let the admin upload an SVG plan per floor; report bookable rooms on that floor with no matching element id, and element ids with no room | M | A plan missing one room's id produces a report naming that room. An SVG containing a `<script>` element or an `on*` attribute is rejected (NFR-SEC-11). | I | P §10; TC §4 |
| FR-BOOK-08 | Allow bookings only on dates inside a published term and within a booking horizon (default 14 days, team value) | S | A booking 20 days ahead returns 422 `OUTSIDE_HORIZON`. | I | P §5 |
| FR-BOOK-09 | Let the admin cancel any booking with a reason that the booker sees | S | The booker's list shows the booking as cancelled with the admin's reason. | I | ES §6 (Untis rights system) |
| FR-BOOK-10 | Show room details on click: capacity, features and the day's occupations | S | Details match the room record and the day's occupation rows. | E | ES §5.5 |
| FR-BOOK-11 | Suggest free rooms of the same type on the same floor or in the same building when the chosen room is taken | C | Suggestions are free for the interval and match the type. | I | ES §6 (UniTime "close by") |
| FR-BOOK-12 | Require admin approval for bookings of selected rooms; a pending request already blocks the room | C | A pending request makes a competing booking fail with `ROOM_ALREADY_BOOKED`. | I | ES §5.4 (UniTime events) |
| FR-BOOK-13 | Outdoor campus map or 3D building view | W | — | — | TC §4 |

### 3.8 Cross-cutting API rules

| ID | Requirement | P | Acceptance criterion | Ver. | Source |
|---|---|---|---|---|---|
| FR-X-01 | Return errors as RFC 9457 `application/problem+json` with a machine-readable code (`STREAM_FULL`, `SCHEDULE_CONFLICT`, `ROOM_ALREADY_BOOKED`, `PROPOSAL_EXPIRED`, …) and details of what conflicted | M | Every 4xx response in `openapi.yaml` uses the problem schema, and each listed code is produced by at least one integration test. | I, R | TC §4 (API conventions) |
| FR-X-02 | Accept an `Idempotency-Key` header on every mutating request; repeating a request with the same key returns the first response and creates nothing new | M | Sending `POST /enrollments` twice with the same key returns identical responses and leaves one enrollment row. | I | TC §4 (API conventions) |
| FR-X-03 | Describe every endpoint in `api/openapi.yaml` and generate the Go server interfaces and the TypeScript client from it | M | A CI job regenerates both and fails if `git diff --exit-code` shows changes. | R | TC §3.6, §4 |

## 4. Non-functional requirements

The Basis column says where a target comes from: "P obj. n" or "P §9" means the proposal states it; "team target" means this document sets it. All targets still need the supervisor's agreement.

### 4.1 Scale and reference conditions

| ID | Requirement | Target | Measurement | Basis | Source |
|---|---|---|---|---|---|
| NFR-SCALE-01 | Data scale the system must handle | Up to 400 streams (about 1,200 sessions per week), 150 teachers, 120 rooms, 5,000 students | Reference instance R400 from the team's synthetic data generator at 80% room load; all performance targets below are measured on R400 unless stated | P obj. 4 | P obj. 4 |
| NFR-SCALE-02 | Reference machine | 4 CPU cores; model, RAM and OS recorded in the experiment report | Every performance result states the machine | team target | LR §2.2 (results are machine-specific) |

### 4.2 Performance

| ID | Requirement | Target | Measurement | Basis | Source |
|---|---|---|---|---|---|
| NFR-PERF-01 | Time to first feasible timetable | ≤ 10 s | R400, 4 workers, seeds 1-10; met if every run is ≤ 10 s | P obj. 4 | P obj. 4 |
| NFR-PERF-02 | Full run with optimisation | ≤ 120 s on 4 cores | R400, seeds 1-10; final F reported as mean and standard deviation | P obj. 4 | P obj. 4 |
| NFR-PERF-03 | Local regeneration of one teacher's options | ≤ 3 s | p95 over all teachers of R400 | P obj. 4 | P obj. 4 |
| NFR-PERF-04 | Enrollment under a burst | 500 concurrent requests into one stream: 0 overflows, p95 ≤ 300 ms, no errors other than `STREAM_FULL` | k6 against the Docker Compose deployment on the reference machine | P obj. 7 | P obj. 7 |
| NFR-PERF-05 | Ordinary API latency | p95 ≤ 300 ms for available streams, personal timetable, room availability for one floor and booking creation, at 50 requests/s for 5 minutes | k6 mixed scenario | team target | P obj. 7 |
| NFR-PERF-06 | Generation request handling | `POST` of a generation run returns in ≤ 300 ms (work is queued) | Integration test timing | team target | TC §3.3 |
| NFR-PERF-07 | Push latency | An SSE event reaches an open client ≤ 2 s after the commit that caused it, p95 | Timestamp in event versus commit time, 100 events | team target | TC §3.4 |
| NFR-PERF-08 | Precheck time | ≤ 1 s on R400 | Go benchmark | team target | P obj. 4 |
| NFR-PERF-09 | Infeasibility report time | First report ≤ 60 s on an infeasible R400-size instance; on timeout the run returns the precheck results and a "not determined" status instead of hanging | Experiment on planted-conflict instances; time to first explanation reported | team target | LR §2.4 (asynchronous timeout; time to first explanation) |

### 4.3 Correctness

| ID | Requirement | Target | Measurement | Basis | Source |
|---|---|---|---|---|---|
| NFR-COR-01 | Hard constraints in published data | 0 violations of H1-H4 in every published version and 0 overlapping active room occupations | Independent validator (`validate.go`, separate code path from the solver) run on every published version and in CI on all test instances | P obj. 4 | P §4, obj. 4 |
| NFR-COR-02 | Database-level guarantees | Each of these is rejected by PostgreSQL when inserted by raw SQL that bypasses the application: overlapping active room occupations; two assignments in one room-slot of a version; two assignments of one teacher in one slot of a version; `enrolled > capacity`; two streams of one course for one student in a term | One integration test per constraint | P §9 | P §9; LR §2.5; TC §3.2 |
| NFR-COR-03 | Reproducibility | Same input, seed and worker count give an identical assignment in 10 of 10 repeats | Hash of the sorted assignment list | P §9 | P §9 (same seed, same result) |
| NFR-COR-04 | Incremental penalty | Incremental ΔF equals full recomputation for 10,000 random moves on each of 3 instances | Unit test | team target | LR §2.3 (delta evaluation) |
| NFR-COR-05 | Solver soundness (property-based) | For 1,000 random small problems: every returned solution has 0 validator violations; every reported conflict set is infeasible, and removing any one group from it makes the problem feasible | Property-based test | team target | LR §2.4 (Junker, 2004) |
| NFR-COR-06 | Seat counter invariant | After every load test, `enrolled` = number of enrollment rows ≤ `capacity` for every stream | SQL check at the end of the k6 and integration runs | team target | TC §3.2 |

### 4.4 Security

| ID | Requirement | Target | Measurement | Basis | Source |
|---|---|---|---|---|---|
| NFR-SEC-01 | Transport | HTTPS only; HTTP redirects to HTTPS; HSTS header set | Request over HTTP returns a redirect; response headers checked | team target | TC §4 (security baseline) |
| NFR-SEC-02 | Password storage | argon2id with at least 19 MiB memory, 2 iterations, parallelism 1 | Code review of parameters; stored hash prefix `$argon2id$` | team target | TC §4 (security baseline) |
| NFR-SEC-03 | Token handling | Access token kept in memory only (not in localStorage); refresh token rotated on each use; reuse of an old refresh token revokes the whole token family; no tokens in URLs or logs | Integration test of reuse; grep of logs after the test suite finds no token strings | team target | TC §3.4, §4 |
| NFR-SEC-04 | Authorisation | 100% of operations in `openapi.yaml` covered by the role and ownership matrix test (FR-AUTH-03) | Matrix test lists every operation; CI fails if an operation is missing | team target | P §5; TC §4 |
| NFR-SEC-05 | Rate limiting | Login: after 5 failed attempts per account in 15 minutes, 429 until the window ends. Enrollment and booking writes: 30 per minute per user | Integration test | team target | TC §4 (security baseline) |
| NFR-SEC-06 | SQL injection | Only parameterised SQL generated by sqlc; no SQL built with string formatting | gosec and a custom lint rule in CI with 0 findings | team target | TC §4 |
| NFR-SEC-07 | Audit | Actor, action, entity, timestamp and request id recorded for constraint edits, confirmations, regeneration requests, enrollments, switches, drops, bookings, cancellations, publishing and admin edits | Each action in a scripted scenario creates exactly one audit row | team target | TC §4 (security baseline) |
| NFR-SEC-08 | Secrets | No secrets in the repository; configuration from environment; `.env.example` only | gitleaks in CI with 0 findings | team target | TC §4 (security baseline) |
| NFR-SEC-09 | Personal data | Store only name, email, role, program, cohort, enrollments and bookings; demo data synthetic; if real AITU data is used, check against the Law of the Republic of Kazakhstan on personal data and its protection | Schema review; data inventory in the thesis | team target | TC §4 (security baseline) |
| NFR-SEC-10 | Dependencies | No known high or critical vulnerability at a release tag | govulncheck and npm audit in CI | team target | TC §4 (security baseline) |
| NFR-SEC-11 | Uploads | SVG sanitised (no scripts, event handlers or external references); XLSX ≤ 10 MB | Integration tests with malicious samples | team target | TC §4 (security baseline) |
| NFR-SEC-12 | iCal tokens | At least 128 random bits, revocable, read-only scope | Code review; revoked token returns 404 | team target | TC §4 (security baseline) |

### 4.5 Reliability

| ID | Requirement | Target | Measurement | Basis | Source |
|---|---|---|---|---|---|
| NFR-REL-01 | Worker crash during generation | No partial version is stored; the job is retried and completes | Integration test sends SIGKILL to the worker mid-run, restarts it and checks the result | team target | TC §3.3 |
| NFR-REL-02 | Graceful shutdown | On SIGTERM the API stops accepting new requests and finishes in-flight ones within 30 s; the worker returns its unfinished job to the queue | Integration test | team target | TC §4 (operations) |
| NFR-REL-03 | Backup | Daily `pg_dump`, 7 days kept; a restore is rehearsed once before pre-defense 3; RPO ≤ 24 h, RTO ≤ 1 h | Restore log in the repository | team target | TC §4 (operations) |
| NFR-REL-04 | Idempotency keys | Kept at least 24 h | Integration test with a key replayed after the API restarts | team target | TC §4 (API conventions) |
| NFR-REL-05 | Push resilience | The client reconnects automatically and resumes from the last event id; after an API restart the progress view recovers within 10 s without a page reload | E2E test that restarts the API | team target | TC §3.4 |
| NFR-REL-06 | Database restart | The API recovers its connections without a restart within 30 s of PostgreSQL coming back | Integration test | team target | TC §4 (operations) |

### 4.6 Usability, localisation and compatibility

| ID | Requirement | Target | Measurement | Basis | Source |
|---|---|---|---|---|---|
| NFR-USE-02 | Small screens | Student and teacher screens usable at 360 × 640 px with no horizontal scrolling; touch targets ≥ 44 px | Playwright screenshots at 360 px; manual check on one Android and one iOS phone | team target | TC §3.6 |
| NFR-USE-03 | Accessibility basics | Text contrast ≥ 4.5:1 (WCAG 2.1 AA); map status never by colour alone | Automated contrast check (axe) on main screens | team target | TC §3.6 |
| NFR-USE-04 | Usability check | At least 5 participants (scheduler, teachers, students) attempt three tasks: enroll in a stream, book a room from the map, name the cause of a failed generation from its report. Target: ≥ 80% task success; SUS score reported | Moderated sessions, results in the thesis | team target | LR §2.4, §2.6 (no reviewed source evaluates with users) |
| NFR-COMP-01 | Browsers | Latest two versions of Chrome, Edge, Firefox and Safari, desktop and mobile | Playwright on Chromium, Firefox and WebKit | team target | TC §3.6 |

### 4.7 Operation and maintainability

| ID | Requirement | Target | Measurement | Basis | Source |
|---|---|---|---|---|---|
| NFR-OBS-01 | Logging | Structured JSON logs with request id, user id, route, status and latency; no passwords or tokens | Log sample review; grep test (NFR-SEC-03) | team target | TC §4 (observability) |
| NFR-OBS-02 | Metrics | Prometheus: latency histogram per route; counts of `STREAM_FULL`, `ROOM_ALREADY_BOOKED`, `PROPOSAL_EXPIRED`; generation duration and outcome | `/metrics` scraped during the k6 run | team target | TC §4 (observability) |
| NFR-DEP-01 | One-command deployment | `docker compose up` on a clean machine with Docker starts the system with demo data within 5 minutes; no manual step except copying `.env.example` | Timed run on a clean VM before each pre-defense | team target | TC §4 |
| NFR-MNT-01 | Continuous integration | Every pull request runs lint (golangci-lint, eslint), unit and integration tests with `-race`, and the build; `main` accepts merges only through green pull requests | GitHub branch protection and Actions history | team target | TC §4 |
| NFR-MNT-02 | Solver independence | Package `internal/solver` imports no database or HTTP package | depguard rule in golangci-lint | team target | TC §3.5 |
| NFR-MNT-03 | Solver test coverage | Statement coverage of `internal/solver` ≥ 80% | `go test -cover` in CI | team target | TC §3.5 |

## 5. Scheduling rules used by the requirements

The solver, the validator and the infeasibility report share these definitions. The standard names follow the vocabulary mapped in LR §2.1, so that results stay comparable with ITC-2007 Track 3.

| Rule | Definition | Kind | Standard name |
|---|---|---|---|
| Domain | A session can be placed only in a slot-room pair where the room type matches, room capacity ≥ stream size, the teacher is available (FR-CON-01) and the session's pairs fit in one day | hard (unary, applied before search) | RoomCapacity (hard here; soft in ITC-2007), teacher Availability |
| H1 | A room holds at most one session per slot | hard | RoomOccupancy |
| H2 | A teacher teaches at most one session per slot | hard | Conflicts (teacher) |
| H3 | Sessions of one stream do not overlap | hard | Conflicts (stream) |
| H4 | Mandatory sessions of one cohort do not overlap when the course has a single stream for that cohort | hard (strictness open, to agree with the supervisor) | Conflicts (curriculum) |
| Confirmed | Confirmed sessions keep their slot and room (FR-CONF-05) | hard | fixed assignments |
| S1 | Teacher preferences (prefer / avoid, weight 1-3) | soft, weight w1 | — |
| S2 | Gaps ("windows") in a cohort's day | soft, weight w2 | CurriculumCompactness |
| S3 | Uneven load of a cohort across days | soft, weight w3 | MinWorkingDays / MaxDayLoad |
| S4 | Overlap of streams of different courses of one cohort, which reduces students' choice | soft, weight w4 | — |
| S5 | Empty seats: (room capacity − stream size) / room capacity | soft, weight w5 | capacity waste (Antunes-Batista et al., 2026) |

Objective: F(A) = w1·ΣS1 + w2·ΣS2 + w3·ΣS3 + w4·ΣS4 + w5·ΣS5 → min, subject to the domain rule, H1-H4 and confirmed sessions.

## 6. Assumptions, dependencies and open questions

| # | Item | Affects | Resolution |
|---|---|---|---|
| 1 | Target values in Section 4, both those from the proposal and the team targets, are not agreed | All NFR targets | Supervisor meeting; record in docs/11-supervisor-feedback.md |
| 2 | AITU's real time grid (pairs per day, Saturday, pair length, online classes without a room) is unknown | FR-CAT-02, NFR-SCALE-01 | Grid is data (FR-CAT-02); confirm the grid with the supervisor |
| 3 | Strictness of H4 for cohorts when a course has several streams | Section 5, FR-ENR-01 | Open question for the supervisor |
| 4 | Availability of anonymised AITU data and floor plans | NFR-SCALE-01, FR-BOOK-07 | Fallback: synthetic generator, ITC-2007, and schematic floor plans drawn by the team |
| 5 | Whether students at AITU may switch streams freely or only with permission (P §4 cites a 2020 guidebook) | FR-ENR-05, FR-ENR-07, FR-ENR-11 | The enrollment window and admin-only moves cover both answers |
| 6 | First-come-first-served enrollment is technically correct but not fair in the sense of LR §2.5 | FR-ENR-02, FR-ENR-13 | Stated as a limitation in the thesis; a priority round is future work |

## 7. Traceability

### 7.1 Proposal objectives to Must requirements

| Objective (P §8) | Must requirements |
|---|---|
| 3. CSP model with H1-H4, S1-S5 and an independent validator | Section 5; FR-SCH-05, FR-SCH-10; NFR-COR-01, NFR-COR-04, NFR-COR-05 |
| 4. Two-phase generator meeting time and scale targets | FR-SCH-01, FR-SCH-02, FR-SCH-03, FR-CONF-04; NFR-SCALE-01, NFR-PERF-01, NFR-PERF-02, NFR-PERF-03, NFR-COR-03 |
| 5. QuickXplain infeasibility report in plain language | FR-SCH-06; NFR-COR-05 |
| 6. Web platform for three roles | FR-AUTH-01 to 05, FR-CAT-01 to 08, FR-CON-01 to 03, FR-SCH-08, FR-SCH-11, FR-SCH-12, FR-CONF-01 to 06, FR-ENR-01 to 07, FR-ENR-09, FR-BOOK-01 to 07, FR-X-01 to 03;  NFR-USE-02, NFR-DEP-01 |
| 7. Database-level guarantees under concurrency | FR-ENR-02, FR-ENR-03, FR-BOOK-03, FR-CONF-03; NFR-PERF-04, NFR-COR-02 |
| 8. Evaluation on ITC-2007 and synthetic data, CP-SAT comparison | FR-SCH-14; NFR-SCALE-02 |

Objective 2 (literature and systems review) produces inputs to this document rather than requirements.

### 7.2 Requirements by module and priority

| Module | Must | Should | Could | Won't | Total |
|---|---|---|---|---|---|
| auth | 5 | 2 | 0 | 1 | 8 |
| catalog | 8 | 1 | 0 | 1 | 10 |
| constraints | 3 | 4 | 1 | 0 | 8 |
| scheduling | 10 | 4 | 1 | 0 | 15 |
| confirmation | 5 | 2 | 1 | 1 | 9 |
| enrollment | 8 | 3 | 0 | 3 | 14 |
| booking + map | 7 | 3 | 2 | 1 | 13 |
| cross-cutting | 3 | 0 | 0 | 0 | 3 |
| Functional total | 49 | 19 | 5 | 7 | 80 |

Non-functional requirements: 46, each with a numeric or pass/fail target and a measurement method.

## 8. Change log

| Date | Version | Change | Author |
|---|---|---|---|
| 18.10.2026 | 0.1 | First draft from the proposal, literature review, existing-systems comparison and technology comparison | Aizada Sembayeva |
| 18.10.2026 | 0.2 | Sources re-mapped to the proposal, literature, existing systems, technology comparison and team design decisions | Aizada Sembayeva |
| 19.10.2026 | 0.3 | Sources limited to P, LR, ES and TC;| Aizada Sembayeva |
