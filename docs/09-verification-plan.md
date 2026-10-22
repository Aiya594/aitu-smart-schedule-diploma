# Verification and evaluation plan for AITU Smart Schedule

Project: Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms (working name: AITU Smart Schedule).

Team: Aizada Sembayeva, Aiya Zhakupova.

Supervisor: Nadirov Nurtas.

Status: draft 0.1, 20.10.2026.

This plan describes how the team will show that the project aim has been achieved: what is tested at each level, which metrics are measured, under which conditions, and against which baselines. It introduces no new targets. Every target comes from the non-functional requirements and acceptance criteria in [05-requirements.md](05-requirements.md), and all of them still need the supervisor's agreement (requirements §6, item 1).

## 1. What has to be shown

The aim (P §7) is an algorithm that generates a university timetable automatically, and a web platform on it where teachers confirm their timetable, students enroll in streams and all users book rooms without time, teacher or room conflicts. The research question (P §6) splits this into claims that can be checked:

| # | Claim | Research question | Evidence | Blocks |
|---|---|---|---|---|
| C1 | The generator produces timetables with 0 hard-constraint violations within the time limits at AITU's assumed scale, and the same seed gives the same result | Main question; sub-question 2 | Experiments E1, E6 on synthetic data; independent validator | A |
| C2 | The two-phase algorithm is competitive with a state-of-the-art exact solver and each of its parts contributes | Sub-question 2 | Experiments E2, E3 on ITC-2007 Track 3 and synthetic data | A |
| C3 | When no timetable exists, the system names a minimal set of conflicting constraint groups that a person can act on | Main question | Experiment E4 on planted-conflict instances; usability task | B |
| C4 | Enrollment, booking and confirmation stay conflict-free under concurrent use at the target response time | Sub-question 3 | Integration tests and experiment E5 (load) | C |
| C5 | The three roles can complete their key tasks with the platform | Aim; sub-question 1 | Experiment E7 (usability), end-to-end tests | D |

Blocks: A, solver quality and speed. B, explainability. C, correctness under concurrency. D, usability and the platform.

## 2. Test plan

### 2.1 Test levels

The levels are those of requirements §2.2. Every acceptance criterion in requirements §3 and every NFR in §4 names one of them.

| Level | What it covers | Tools | When it runs | Pass rule |
|---|---|---|---|---|
| U: unit and property-based | Solver model, validator, prechecks, ΔF, QuickXplain; pure functions in modules | `go test`, rapid or `testing/quick` | Every pull request (CI) | All pass; `internal/solver` coverage ≥ 80% (NFR-MNT-03) |
| I: integration | Every API operation against real PostgreSQL in a container: roles and ownership, database constraints, transactions, idempotency, import, publish | `go test -race`, testcontainers-go | Every pull request (CI) | All pass with the race detector on (NFR-MNT-01) |
| L: load | Enrollment burst, mixed API load, concurrent bookings | k6 against Docker Compose | Before pre-defense 2 and the final pre-defense; after changes in enrollment or booking | NFR-PERF-04, NFR-PERF-05 targets met; invariant NFR-COR-06 holds |
| E: end-to-end and manual | Six scenarios of [08-ux-prototype.md](08-ux-prototype.md) in the browser; small screens; contrast | Playwright (Chromium, Firefox, WebKit), axe, recorded manual checks | Nightly in CI once the frontend exists; manual before each pre-defense | Scenarios complete; NFR-USE-02, NFR-USE-03, NFR-COMP-01 met |
| X: experiment | Solver speed and quality, comparison with CP-SAT, ablation, explanation quality | `solverctl`, experiment scripts, CP-SAT script | At the checkpoints in Section 6 | Results reported, whether or not the target is met |
| R: review and inspection | Code review, CI configuration, security scanners, generated code up to date | GitHub pull requests, golangci-lint, gosec, gitleaks, govulncheck, npm audit | Every pull request | Reviewer approval; 0 findings (NFR-SEC-06, 08, 10) |

### 2.2 Continuous integration

Every pull request to `main` runs: lint (golangci-lint with a depguard rule that keeps `internal/solver` free of database and HTTP imports, eslint), unit and integration tests with `-race`, a check that the code generated from `api/openapi.yaml` is up to date (FR-X-03), the security scanners, and the build. `main` accepts only green, reviewed pull requests (NFR-MNT-01). The validator also runs in CI on every test instance with a stored solution (NFR-COR-01).

### 2.3 Traceability rule

Each test name starts with the requirement id it checks (for example `TestFR_ENR_02_ConcurrentSeats`). A script lists all Must FR and all NFR ids from `05-requirements.md` and reports those without a test. Before each pre-defense this list must be empty for implemented modules; the remaining ids are shown as outstanding work.

## 3. Test data

| Data set | Content | Use | Source |
|---|---|---|---|
| ITC-2007 Track 3 | 21 public instances comp01-comp21 with the official validator | E2, E3; check of `solverctl` (FR-SCH-14) | Di Gaspero et al. (2007); LR §2.2 |
| Synthetic scale series | 50, 100, 200 and 400 streams; room load 60%, 80% and 90%; generated with a fixed generator seed | E1, E3, E6 | Team's synthetic data generator (checkpoint 24.11.2026) |
| R400 | 400 streams (about 1,200 sessions/week), 150 teachers, 120 rooms, 5,000 students, 80% room load | Reference instance for all performance targets | NFR-SCALE-01 |
| Planted-conflict set | At least 10 instances, each made infeasible by one known conflict (teacher unavailability, room-type capacity, cohort overload, …); some small, some R400-size | E4; FR-SCH-06, FR-SCH-07, NFR-PERF-09 | Built from the synthetic generator by editing one constraint group |
| Invalid-input set | Four data sets, one per precheck | FR-SCH-01 | Built by hand |
| Random small problems | 1,000 random problems small enough to solve exactly | Property-based tests NFR-COR-05 | Generated in the test |
| Integration seed | Small fixed university: 2 programs, 4 cohorts, 20 streams, 15 rooms, users of every role | Integration and end-to-end tests | `testdata/` in the repository |

If anonymised AITU data becomes available (risk R2), it is added as a further instance and reported separately; the synthetic sets remain the reference for comparability.

## 4. Metrics

| Metric | Definition | Target | Requirement |
|---|---|---|---|
| Hard violations | Violations of the domain rule, H1-H4 and confirmed sessions, counted by the independent validator | 0 in every published version and every reported solution | NFR-COR-01 |
| Time to first feasible (TTFF) | Wall time from job start to the first complete assignment with 0 hard violations | ≤ 10 s on R400, every run of seeds 1-10 | NFR-PERF-01 |
| Final cost F and S1-S5 | Objective value after the time limit, and its five terms | Reported as mean ± std over seeds; lower is better; no fixed target | NFR-PERF-02, requirements §5 |
| Full run time | Time to finish optimisation | ≤ 120 s on R400 | NFR-PERF-02 |
| Local regeneration time | Time to produce a teacher's new options | p95 ≤ 3 s over all teachers of R400 | NFR-PERF-03 |
| ITC-2007 soft cost | Official cost of the solution file | Reported per instance against CP-SAT and best known values | FR-SCH-14 |
| Reproducibility | Identical sorted assignment hash for same input, seed and workers | 10 of 10 repeats | NFR-COR-03 |
| ΔF accuracy | Incremental ΔF equals full recomputation | 10,000 random moves on each of 3 instances | NFR-COR-04 |
| Explanation correctness | On planted-conflict instances: precision and recall of the named groups against the planted ones; minimality (removing any named group makes the instance feasible) | Precision = recall = 1 on all ≥ 10 instances | FR-SCH-06, NFR-COR-05 |
| Repair-hint success | Share of planted conflicts made feasible by applying one suggested hint | 100% | FR-SCH-07 |
| Time to first explanation | Time from proven infeasibility to the first reported conflict set | ≤ 60 s on R400-size instances; timeout returns prechecks and "not determined" | NFR-PERF-09 |
| Overflow count | Enrollments above capacity; overlapping active room occupations; confirmations that both succeeded on one slot-room | 0 | NFR-PERF-04, NFR-COR-02, NFR-COR-06 |
| API latency | p95 response time per route under load | ≤ 300 ms | NFR-PERF-04, NFR-PERF-05 |
| Push latency | Commit time to SSE delivery | p95 ≤ 2 s | NFR-PERF-07 |
| Task success rate | Share of participants completing each of the three usability tasks without help | ≥ 80% | NFR-USE-04 |
| SUS score | System Usability Scale questionnaire after the tasks | Reported (no target) | NFR-USE-04 |

## 5. Measurement conditions

- Machine: one reference machine with 4 CPU cores; CPU model, RAM, OS, Go version and PostgreSQL version are recorded in every result table (NFR-SCALE-02). No other heavy process runs during a measurement.
- Seeds: solver experiments use seeds 1-10 for each instance and configuration; results are given as mean, standard deviation, minimum and maximum.
- Workers: the own solver is measured with 4 workers for the product targets, and with 1 worker for the ITC-2007 comparison, following the competition's single-processor rule (LR §2.2).
- Time limits: 10 s and 120 s on synthetic data (NFR-PERF-01, NFR-PERF-02); on ITC-2007 the limit is the competition's benchmark time computed on the reference machine, and the same limit is given to CP-SAT.
- Load tests: the full Docker Compose deployment (Caddy, api, worker, PostgreSQL) on the reference machine, with k6 on the same machine or a second machine on the same network (recorded). Each scenario runs after a warm-up of 30 s and is repeated 3 times.
- Correctness after load: at the end of every load or concurrency test, SQL checks verify `enrolled = count(enrollments) ≤ capacity` for every stream and no overlapping active room occupations (NFR-COR-06).
- Recording: raw outputs (JSON from `solverctl`, k6 summaries) are committed under `experiments/results/` with the commit hash of the code that produced them, so every table in the thesis can be regenerated.

## 6. Baselines

| Baseline | Compared with | Why it is a fair comparison | Used in |
|---|---|---|---|
| Google OR-Tools CP-SAT | Own solver on the same instances and time limits; CP-SAT run with 1 worker and, separately, with its default parallel setting | A state-of-the-art general solver; the model is written by the team from the same constraint definitions (requirements §5) | E2 |
| Best known ITC-2007 Track 3 results | Own solver's soft cost per instance | Published reference values for the same instances and validator (LR §2.2) | E2 |
| Ablated own solver | Full solver vs. (a) backtracking without MRV/LCV, (b) without forward checking, (c) backtracking only, without simulated annealing | Shows which parts of the design matter; same code, same seeds | E3 |
| Naive concurrency implementation | Database guarantees vs. "read then write" in application code without the conditional update, exclusion constraint or unique index | Shows that the guarantees are needed: the naive version is expected to overbook under the same load | E5 |
| Current manual process at AITU | Platform as a whole | Qualitative only: answers to the scheduling-staff questionnaire ([questionnaire](questionnaire/questions.md)) on time spent and typical conflicts; no numeric comparison is claimed | E7, discussion chapter |

## 7. Experiments

| # | Experiment | Data | Metric | Success criterion | Requirements | Planned by |
|---|---|---|---|---|---|---|
| E1 | Scalability of the own solver | Synthetic 50-400 streams at 60/80/90% room load, seeds 1-10, 4 workers | TTFF, full run time, final F, hard violations | 0 hard violations in all runs; TTFF ≤ 10 s and full run ≤ 120 s on R400 | NFR-PERF-01, NFR-PERF-02, NFR-COR-01 | 01.12.2026 (backtracking), 20.01.2027 (with SA) |
| E2 | Comparison with CP-SAT and best known results | ITC-2007 comp01-comp21; synthetic 100-400 | Feasibility rate, soft cost, TTFF | Reported per instance; the claim is "competitive" only if the own solver is within the CP-SAT result on a majority of instances, otherwise the gap is reported | FR-SCH-14; P §6 sub-question 2 | 01.04.2027 |
| E3 | Ablation | Synthetic 100 and 200; ITC-2007 subset | TTFF, final F, share of runs feasible | Each removed component makes at least one metric worse, or the result is reported as "no measurable effect" | LR §2.3 | 01.04.2027 |
| E4 | Explainability | ≥ 10 planted-conflict instances, small and R400-size | Precision, recall, minimality, hint success, time to first explanation | Precision = recall = 1, all sets minimal, every hint repairs its conflict; time ≤ 60 s | FR-SCH-06, FR-SCH-07, NFR-COR-05, NFR-PERF-09 | 01.02.2027 |
| E5 | Concurrency and load | (a) 500 concurrent enrollments into one 30-seat stream; (b) 50 concurrent overlapping bookings of one room; (c) two teachers confirming options that share a slot-room; (d) mixed load 50 req/s for 5 min; each with the naive baseline | Overflow count, p95 latency, error codes | 0 overflows and only `STREAM_FULL` / `ROOM_ALREADY_BOOKED` / `PROPOSAL_EXPIRED` as errors; p95 ≤ 300 ms; naive baseline shows overflow | NFR-PERF-04, NFR-PERF-05, NFR-COR-02, NFR-COR-06, FR-ENR-02, FR-BOOK-03, FR-CONF-03 | 15.03.2027 |
| E6 | Reproducibility | R400 and 3 ITC-2007 instances, 10 repeats per seed | Assignment hash | 10 of 10 identical | NFR-COR-03 | 20.01.2027 |
| E7 | Usability | ≥ 5 participants (scheduler, teachers, students) on the deployed system; tasks: enroll in a stream, book a room from the map, name the cause of a failed generation from its report | Task success, time on task, SUS, comments | ≥ 80% success per task | NFR-USE-04 | Before the final pre-defense |

Dates follow the development checkpoints in [06-project-management.md §2.2](06-project-management.md).

## 8. When the aim counts as achieved

The aim is achieved when:

1. Every Must functional requirement of the implemented modules has a passing test at the level named in its acceptance criterion.
2. Claims C1, C3 and C4 hold: 0 hard violations in all reported solutions, correct and minimal explanations on all planted conflicts, and 0 overflows under load.
3. Experiments E1-E7 have been run under the conditions of Section 5 and their results are in the repository.

Claim C2 is a research result rather than a pass/fail condition. If the own solver is slower than CP-SAT or misses a time target, the result is reported as it is, the target is discussed with the supervisor, and the thesis explains the cause (risk R1 in [06-project-management.md](06-project-management.md)). The same applies to a usability task below 80%: the result and the changes it leads to are reported.

## 9. Limitations of the evaluation

- Synthetic data may be easier or harder than AITU's real data; the generator's parameters are documented and, where possible, set from questionnaire answers.
- ITC-2007 Track 3 does not contain streams, enrollment or confirmed sessions, so it tests only the core timetabling part of the solver.
- The CP-SAT model is written by the team; a better model might perform better. The model is published in the repository so it can be checked.
- The usability check has a small sample and is qualitative evidence, not a statistical result.
- All performance numbers are specific to the reference machine.

## 10. Change log

| Date | Version | Change | Author |
|---|---|---|---|
| 20.10.2026 | 0.1 | First draft: claims, test levels, data, metrics, conditions, baselines, experiments E1-E7 | Aizada Sembayeva |
