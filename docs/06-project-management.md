# Project management and risks for AITU Smart Schedule

Project: Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms (working name: AITU Smart Schedule). Team: Aizada Sembayeva, Aiya Zhakupova. Supervisor: Nadirov Nurtas.

YouTrack: SS-15. Status: draft, 19.10.2026.

| Resource | Link |
|---|---|
| Repository | https://github.com/Aiya594/aitu-smart-schedule-diploma |
| YouTrack project (board and tasks) | `https://smartschedule.youtrack.cloud/projects/SS` |
| Documentation | [docs/](.) in the repository |


## 1. Roles and responsibilities

### 1.1 People

| Person | Role | Responsible for |
|---|---|---|
| Aizada Sembayeva | Developer, algorithm lead | Scheduling algorithm (validator, generator, infeasibility report), OpenAPI contract, teacher confirmation and student enrollment backend, requirements, literature review on algorithms, experiments ([proposal §2](01-proposal.md)) |
| Aiya Zhakupova | Developer, platform and frontend lead | User research (scheduling-staff questionnaire), database schema and migrations, Docker and CI, authentication, catalog and Excel import backend, React frontend, room map and booking, synthetic data generator, diagrams and mock-ups ([proposal §2](01-proposal.md)) |
| Nadirov Nurtas | Supervisor | Approves the topic wording and the diploma format, agrees the numeric targets and the scope, reviews documents before each milestone, helps the team reach the department and university staff |

### 1.2 Responsibility by work area

O = owner (does the work and answers for it), R = reviewer (reviews the pull request).

| Work area | Aizada | Aiya | |
|---|---|---|---|
| Proposal, requirements, technology comparison | O | R |  
| Literature review | O | R | 
| Existing-systems comparison | R | O |  
| Questionnaire for scheduling staff | R | O |  
| Project management and risks (this document) | O | R |  
| Diagrams and mock-ups | R | O | 
| Solver: model, validator, prechecks, backtracking, simulated annealing, QuickXplain | O | R | 
| Database schema, migrations, room-occupancy constraints | R | O | 
| OpenAPI contract | O | R |  
| Auth, catalog, Excel import backend | R | O |  
| Teacher confirmation and enrollment backend | O | R |  
| React frontend, room map and booking | R | O |  
| Docker Compose, CI | R | O |  
| Synthetic data generator | R | O | 
| Experiments and evaluation chapter | O | R | 
| Thesis text | O (algorithm, evaluation) | O (system, interface) | 

### 1.3 Way of working

- Task tracking: every piece of work is a YouTrack task in project SS with an owner, priority, due date and acceptance criteria. Board columns: Open → In Progress → Review → Done.
- Branches and commits: one branch per task, named `SS-<n>-<short-name>`. Commit messages start with the task id (`SS-15: …`) so that YouTrack links commits to tasks. Each team member commits from her own GitHub account.
- Pull requests: work reaches `main` only through a pull request reviewed by the other team member. A task moves to Done when its pull request is merged and its acceptance criteria are checked.
- Meetings: a short team sync twice a week, and a meeting with the supervisor every two weeks and before each milestone. Drafts go to the supervisor at least three days before a meeting. Decisions and comments are recorded in the repository with the date.
- Definition of done for code: tests pass in CI, the other member has reviewed it, and documentation affected by the change is updated in the same pull request.

## 2. Milestones

### 2.1 Department milestones

Pre-defense 1 is on 24.10.2026. The dates of pre-defense 2, the final pre-defense and the defense are taken from the department schedule when it is published, and the "Department date" column is updated in the same pull request. The "Ready by" column is the team's own deadline for the deliverables.

| Milestone | Department date | Ready by | What is shown | Status |
|---|---|---|---|---|
| Pre-defense 1 | 24.10.2026 | 23.10.2026 | Repository and YouTrack set up; proposal; literature review; existing-systems comparison; technology comparison; requirements specification; questionnaire for scheduling staff; this document; architecture and diagrams; UX mock-ups and API examples; verification and evaluation plan; supervisor feedback | Done |

### 2.2 Development checkpoints

Development follows the order set in the proposal (§10): the algorithm core comes first because it is the riskiest part, and the map and simulated annealing follow after it. Dates are targets and are reviewed at pre-defense 2.

| Checkpoint | Target date | Owner | Done when |
|---|---|---|---|
| Solver data model and independent validator | 10.11.2026 | Aizada | The validator finds every hard-constraint violation on prepared test sets |
| Prechecks for invalid input | 17.11.2026 | Aizada | Each of the four precheck errors is caught on a prepared invalid dataset |
| Database schema, migrations, synthetic data generator | 24.11.2026 | Aiya | Migrations apply from scratch; an overlapping booking is rejected by the database; data sets of 50-400 streams are generated |
| Backtracking generator | 01.12.2026 | Aizada | Solutions on synthetic sets pass the validator with 0 violations |
| OpenAPI contract, Docker Compose, CI | 05.12.2026 | Aizada (API), Aiya (Docker, CI) | Server interfaces and TypeScript client are generated; `docker compose up` starts the system; CI is green |
| Simulated annealing | 20.01.2027 | Aizada | Incremental penalty equals full recomputation; F falls over time |
| Auth, catalog and Excel import backend | 31.01.2027 | Aiya | Integration tests for roles and import pass |
| QuickXplain infeasibility report | 01.02.2027 | Aizada | On infeasible sets with a known cause, the report names that cause |
| Frontend shell, admin screens | 15.02.2027 | Aiya | The admin imports data and starts generation from the interface |
| Teacher confirmation and local regeneration | 28.02.2027 | Aizada | Concurrent confirmation test: one teacher wins, the other gets `PROPOSAL_EXPIRED` |
| Student enrollment | 15.03.2027 | Aizada | 200 concurrent requests into a 30-seat stream give exactly 30 enrollments |
| Room map and booking | 15.03.2027 | Aiya | 50 concurrent overlapping bookings give exactly 1 success; the map shows correct statuses |
| Experiments and comparison with OR-Tools CP-SAT | 01.04.2027 | Aizada | Tables and charts for the practical chapter are in the repository |

## 3. Risks

Probability and impact are rated Low (1), Medium (2) or High (3); the score is their product. Risks with a score of 6 or more are reviewed at every team sync. The owner watches the trigger and starts the mitigation.

| # | Risk | Probability | Impact | Score | Trigger | Mitigation | Owner |
|---|---|---|---|---|---|---|---|
| R1 | The team's own solver does not meet the time targets at the planned scale (400 streams) | Medium | High | 6 | On 01.12.2026 backtracking cannot find a feasible timetable for 200 streams within the time limit | Validator and backtracking are built first, so the problem shows early. Run the heuristic ablation on small sets before scaling. If the target is still missed, agree a smaller scale with the supervisor and keep OR-Tools CP-SAT behind the same solver interface for the demo ([technology comparison §3.5](04-technology-comparison.md)). Report the shortfall honestly in the thesis | Aizada |
| R2 | No anonymised data from the university (rooms, streams, teaching load) | High | Medium | 6 | No data received by 24.11.2026 | Synthetic data generator with adjustable size and room load; ITC-2007 Track 3 instances as a public benchmark ([literature review §2.2](02-lit_review.md)); questionnaire answers used to set realistic parameters | Aiya |
| R3 | No floor plans for the room map | Medium | Medium | 4 | No plans received by 15.01.2027 | The map is a basic version in the scope ([proposal §10](01-proposal.md)) and is built after the core. The team draws schematic SVG plans of one or two floors itself. Booking also works from a room list without the map | Aiya |
| R4 | Few or no answers to the questionnaire for scheduling staff, which weakens objective 3 ([proposal §8](01-proposal.md)) | High | Medium | 6 | Fewer than 3 answers two weeks after the questionnaire is sent | Send the short form ([docs/questionnaire/questions.md](questionnaire/questions.md)) through the supervisor and the department early; send one reminder. Report a small sample as qualitative feedback. Objectives 1 and 2 (literature and automated validation) do not depend on it | Aiya |
| R5 | Workload: two people, a large scope and coursework and exams at the same time | High | High | 9 | Two or more checkpoints in section 2.2 late by more than one week | Must requirements first, then Should, then Could ([requirements](05-requirements.md)). Scope freeze at pre-defense 2. Simulated annealing, the map and Could features are cut before any Must feature. The board is reviewed at each team sync | Both |
| R6 | Supervisor feedback arrives late, or targets and scope are not agreed in time | Medium | Medium | 4 | No answer to a draft within one week | Fixed meeting every two weeks; drafts sent three days before; questions sent as a short numbered list; decisions recorded with dates in the repository | Aizada |
| R7 | Concurrency bugs in enrollment, booking or teacher confirmation cause overbooking or double booking | Medium | High | 6 | A concurrency test fails or is flaky in CI | Database constraints are the last line of defence (exclusion constraint, unique indexes, CHECK on the seat counter). Concurrency tests run against real PostgreSQL in CI. Load test with k6 before the final pre-defense ([technology comparison §3.2](04-technology-comparison.md)) | Aizada |
| R8 | One team member is unavailable for a long time, or knowledge of a module sits with one person | Medium | High | 6 | A member is absent for more than one week, or a module has had no review by the other member | Every pull request is reviewed by the other member, so both know all modules. Module boundaries are fixed by the OpenAPI contract. Tasks are reassigned in YouTrack and the plan is adjusted with the supervisor | Both |
| R9 | The infeasibility report is too slow or too hard to read | Medium | Medium | 4 | The report takes longer than its time limit or the text is unclear in the usability check | Run prechecks first; group constraints at the level the scheduler understands; a timeout returns the precheck result; text from templates in three languages ([literature review §2.4](02-lit_review.md)) | Aizada |
| R10 | A pre-1.0 library (the River job queue) changes or breaks | Low | Medium | 2 | An upgrade breaks the build or the tests | Pin versions; keep the queue behind a small interface; fallback is a plain jobs table polled with `SELECT … FOR UPDATE SKIP LOCKED` ([technology comparison §3.3](04-technology-comparison.md)) | Aizada |
| R11 | Personal data or secrets end up in the repository (questionnaire answers, `.env` files) | Low | High | 3 | A secret scanner finding or a name in committed data | Questionnaire answers stored anonymised and only as summaries; demo data synthetic; `.env` in `.gitignore`; secret scanning in CI | Aiya |
| R12 | The approved topic wording or the diploma format differs from the current plan | Low | Medium | 2 | Supervisor or department changes the wording or format | Confirm both at the first supervisor meeting; the proposal states that the title follows the approved wording ([proposal §1](01-proposal.md)) | Aizada |

Risk overview by score:

| Impact \ Probability | Low | Medium | High |
|---|---|---|---|
| High | R11 | R1, R7, R8 | R5 |
| Medium | R10, R12 | R3, R6, R9 | R2, R4 |

## 4. Dependencies

| Dependency | What the team needs | From whom | Needed by | Status | If it does not arrive |
|---|---|---|---|---|---|
| University data | Anonymised lists of rooms (capacity, type), streams, teaching load and the real time grid (pairs per day, days per week) | Registrar's Office, through the supervisor | 24.11.2026 | Not requested yet | Synthetic data and ITC-2007 (R2); the time grid is configurable, so a later change needs no code change |
| Floor plans | Plans of the floors with room numbers, in any format the team can redraw as SVG | University facilities or administration, through the supervisor | 15.01.2027 | Not requested yet | Schematic plans drawn by the team (R3) |
| Supervisor | Topic wording and format confirmed; numeric targets and scope agreed; feedback on documents before each milestone | Nadirov Nurtas | Topic and targets: 23.10.2026; feedback: before each milestone | Meeting to be scheduled | Escalate through the department; keep working on the current plan and record the open decision (R6, R12) |
| Scheduling staff | Answers to the questionnaire | Staff who build or approve timetables, through the supervisor and department | 15.11.2026 | Questionnaire ready in the repository | Report the answers received as qualitative feedback (R4) |
| Department schedule | Dates of pre-defense 2, the final pre-defense and the defense | Department | As soon as published | Pre-defense 1 date known (24.10.2026) | Plan with the "Ready by" dates in section 2.1 |
| Public benchmark | ITC-2007 Track 3 instances and validator | Public competition website | 01.12.2026 | Publicly available | — |
| Tools | GitHub repository, YouTrack project, CI minutes | GitHub, YouTrack | Now | Set up | — |

## 5. Documents

| Document | Path |
|---|---|
| Project proposal | [docs/01-proposal.md](01-proposal.md) |
| Literature review | [docs/02-lit_review.md](02-lit_review.md) |
| Existing-systems comparison | [docs/03-existing_system_comparison.md](03-existing_system_comparison.md) |
| Technology comparison | [docs/04-technology-comparison.md](04-technology-comparison.md) |
| Requirements specification | [docs/05-requirements.md](05-requirements.md) |
| Project management and risks | docs/06-project-management.md  |
| Architecture and diagrams | [docs/07-architecture.md](07-architecture.md) |
| UX prototype and API examples | [docs/08-ux-prototype.md](08-ux-prototype.md), mock-ups in [Figma](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp) (HTML fallback in [prototype/](../prototype/index.html)) |
| Verification and evaluation plan | [docs/09-verification-plan.md](09-verification-plan.md) |
| Diagram sources | [diagrams/src/](../diagrams/src/) |
| Questionnaire for scheduling staff | [docs/questionnaire/questions.md](questionnaire/questions.md) |

