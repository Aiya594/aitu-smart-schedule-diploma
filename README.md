# AITU Smart Schedule

Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms.

AITU Smart Schedule is a diploma project at Astana IT University. It is a web platform that generates a university timetable automatically, explains in plain language why a timetable cannot be built, lets teachers confirm their timetable, lets students enroll in streams without clashes, and lets all users book free rooms on a floor map.

**Status:** The analysis and design documents are in [`docs/`](docs/). Code has not been written yet; development starts with the scheduling algorithm in November 2026.

## Team

| Member | Responsibilities |
|---|---|
| Aizada Sembayeva | Scheduling algorithm (validator, generator, infeasibility report), OpenAPI contract, teacher confirmation and student enrollment backend, requirements, literature review on algorithms, experiments |
| Aiya Zhakupova | User research, database schema and migrations, Docker and CI, authentication, catalog and Excel import backend, React frontend, room map and booking, synthetic data generator, diagrams and mock-ups |

### Supervisor: Nadirov Nurtas.

## Key links

#### GitHub Repository: [AITU SmartSchedule](https://github.com/Aiya594/aitu-smart-schedule-diploma)
#### YouTrack: [YouTrack project SS](https://smartschedule.youtrack.cloud/issues/SS)
#### Figma: [SmartSchedule UX Prototype (Draft)](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp)

## Problem

University timetabling has to consider classes, instructors, student groups, rooms and time limits together. Preparing a timetable takes a lot of time and often still needs manual corrections when rules or preferences conflict, which leads to overlapping classes, poor room use and inconvenient schedules. The problem is NP-hard, constraints differ between institutions, and first-come-first-served stream enrollment must stay correct when many students try to take the last seats at once ([proposal §4](docs/01-proposal.md)).

## Users

| User | What the platform does for them |
|---|---|
| Scheduler (Registrar's Office) | Keeps reference data and imports it from Excel; generates a timetable with no hard-constraint violations; shows in plain language why generation fails; publishes and exports the timetable |
| Teacher | Records unavailable times and preferences; offers 1-3 valid timetable options to choose from |
| Student | Shows the streams of the student's own program; enrolls in a stream without clashes in the personal timetable; books a free room for self-study |

Source: [proposal §5](docs/01-proposal.md).

## Scope of the MVP

- Reference data (courses, streams, rooms, teachers, cohorts, time grid) and Excel import.
- Automatic timetable generation in two phases (backtracking, then simulated annealing) with an infeasibility report.
- Teacher options and confirmation; confirmed classes stay fixed in later runs.
- Student enrollment in streams of their own study program.
- Room booking and a basic interactive room map built from SVG floor plans.

Out of scope: integration with AITU's existing systems, exam timetabling, teacher substitutions during the semester and an open elective catalog ([proposal §10](docs/01-proposal.md)).

## Technology

| Layer | Choice |
|---|---|
| Backend | Go 1.27, modular monolith (one binary, `api` and `worker` modes) |
| Database | PostgreSQL 18 (16 minimum): exclusion constraints against double booking, conditional updates for seat counts |
| Background jobs | River (job queue inside PostgreSQL) |
| Real-time updates | Server-Sent Events, with polling as a fallback |
| Scheduling algorithm | Own solver in Go: backtracking with MRV, LCV and forward checking, then simulated annealing; QuickXplain for infeasibility reports |
| Baseline for experiments | Google OR-Tools CP-SAT, ITC-2007 Track 3 benchmark |
| Frontend | React, TypeScript, Vite, TanStack Query, Mantine |
| Deployment | Docker Compose (Caddy, api, worker, PostgreSQL), GitHub Actions |

Each choice and the alternatives considered are in the [technology comparison](docs/04-technology-comparison.md).

## Documentation

| # | Document | Contents |
|---|---|---|
| 01 | [Project proposal](docs/01-proposal.md) | Problem, users, research question, aim, objectives, scope |
| 02 | [Literature review](docs/02-lit_review.md) | Timetabling problem and benchmarks, solution methods, explaining infeasibility, course allocation, timetabling systems; 31 sources |
| 03 | [Existing-systems comparison](docs/03-existing_system_comparison.md) | UniTime, FET, aSc Timetables, Untis/WebUntis and the current AITU process against 8 criteria |
| 04 | [Technology comparison](docs/04-technology-comparison.md) | Six stack decisions with alternatives, criteria derived from requirements |
| 05 | [Requirements specification](docs/05-requirements.md) | Functional requirements for 7 modules with MoSCoW priorities and acceptance criteria; measurable non-functional requirements |
| 06 | [Project management and risks](docs/06-project-management.md) | Roles, milestones, risks, dependencies |
| 07 | [Architecture and diagrams](docs/07-architecture.md) | System context, use case and component diagrams, data model draft, key runtime scenarios, consistency check |
| 08 | [UX prototype and API examples](docs/08-ux-prototype.md) | Six key user scenarios with mock-ups in [Figma](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp); API, infeasibility report and `solverctl` examples |
| 09 | [Verification and evaluation plan](docs/09-verification-plan.md) | Test levels, test data, metrics, measurement conditions, baselines (CP-SAT, ITC-2007, ablation), experiments E1-E7 |
| 11 | [Supervisor feedback](docs/11-supervisor-feedback.md) | Supervisor's comments, decisions and resulting changes; open questions |
| — | [Diagram sources](docs/diagrams/src/) | Editable draw.io SVG sources of all diagrams (`.drawio.svg`, UML); they render as images on GitHub |
| — | [Questionnaire for scheduling staff](docs/questionnaire/questions.md) | Survey on the current timetabling process (in Russian) |
| — | [Articles](docs/articles/) | Full texts of the papers used in the literature review |

## Repository structure

```
.
├── README.md
├── docs/
│   ├── 01-proposal.md
│   ├── 02-lit_review.md
│   ├── 03-existing_system_comparison.md
│   ├── 04-technology-comparison.md
│   ├── 05-requirements.md
│   ├── 06-project-management.md
│   ├── 07-architecture.md
│   ├── 08-ux-prototype.md
│   ├── 09-verification-plan.md
│   ├── 11-supervisor-feedback.md
│   ├── diagrams/
│   │   └── src/      # editable draw.io SVG diagrams (.drawio.svg)
│   ├── questionnaire/
│   └── articles/
```

UX mock-ups are kept in Figma, not in the repository: [AITU Smart Schedule - UX prototype](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp).

Source code, database migrations, the API contract, deployment files and test data will be added to the repository as development starts.



## Project management

- Tasks and board: [YouTrack project SS](https://smartschedule.youtrack.cloud/issues/SS)
- Workflow: one branch per task (`SS-<n>-<short-name>`), commit messages start with the task id, every pull request is reviewed by the other team member before it is merged into `main`.
- Milestones and risks: [docs/06-project-management.md](docs/06-project-management.md)

| Milestone | Date |
|---|---|
| Pre-defense 1 | 24.10.2026 |
