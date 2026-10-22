# UX prototype and API examples for AITU Smart Schedule

Project: Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms (working name: AITU Smart Schedule).

Team: Aizada Sembayeva, Aiya Zhakupova.

Supervisor: Nadirov Nurtas.

Status: draft 0.1, 20.10.2026.

This document shows how the three roles and external tools will interact with the platform. Section 2 describes six key user scenarios with mid-fidelity mock-ups. Section 3 gives examples of API requests and responses, the infeasibility report and the `solverctl` input and output. Requirement ids refer to [05-requirements.md](05-requirements.md); use case ids (UC-nn) refer to [07-architecture.md §3](07-architecture.md).

## 1. Prototype

The mock-ups are designed in Figma: **[AITU Smart Schedule — UX prototype](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp)**. The file has three pages: "Cover & notes", "Screens" (the scenarios below) and "Components" (Button, Tag, Alert, Lesson, Stream card, Top bar, Mock note, built on local colour and spacing variables and text styles). Every screen carries a yellow note with its scenario and requirement ids. The screens contain invented data and no logic. Their purpose is to agree on screens, flows and error messages before the frontend is built 


| # | Scenario | Role | Figma screen | Use cases |
|---|---|---|---|---|
| 0 | Sign in | All | [00 · Sign in](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=3-2) | UC-24 |
| 1 | Generate a timetable and review the grid | Admin | [01 · Admin · Generation](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=3-36) | UC-04, UC-05, UC-08, UC-09, UC-12, UC-14 |
| 2 | Understand why generation failed | Admin | [02 · Admin · Infeasibility report](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=4-54) | UC-06, UC-07 |
| 3 | Import reference data, track confirmations | Admin | [03 · Admin · Import and confirmations](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=4-173) | UC-02, UC-11 |
| 4 | Set availability, choose a timetable option | Teacher | [04 · Teacher · Availability and options](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=5-96) | UC-16…UC-19 |
| 5 | Enroll in streams on a phone | Student | [05a](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=6-162), [05b](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=6-215), [05c](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=6-254) · Student (360 px) | UC-20…UC-23, UC-25 |
| 6 | Find and book a free room on the floor map | All | [06 · All users · Room map and booking](https://www.figma.com/design/wPqTQwWE7WJQ4iXm5mzazp?node-id=5-366) | UC-26…UC-28 |

Design rules applied in every mock-up:

- Every conflict the backend can return (`STREAM_FULL`, `SCHEDULE_CONFLICT`, `ROOM_ALREADY_BOOKED`, `PROPOSAL_EXPIRED`, `RUN_IN_PROGRESS`, `ENROLLMENT_CLOSED`, `BOOKING_LIMIT`) has a visible message in plain language that says what happened and what to do next (FR-X-01).
- Status is never shown by colour alone: rooms on the map use colour, a line pattern and a text label; penalties in the grid use colour and a ⚠ mark (NFR-USE-03).
- Student screens are drawn at 360 × 640 px (NFR-USE-02).
- The interface language (EN / RU / KZ) can be switched on the login screen and in the header.

## 2. Key user scenarios

### 2.0 Sign in (UC-24)

Goal: any user enters the system with an e-mail and password.

Steps: enter e-mail and password → Sign in → land on the role's home screen. A wrong password and an unknown e-mail show the same message, so accounts cannot be enumerated (FR-AUTH-01). First login after a bulk import asks for a new password (FR-AUTH-06).

### 2.1 Admin: generate a timetable (UC-04, UC-05, UC-08, UC-09, UC-12)

Goal: the scheduler gets a timetable with no hard-constraint violations for a term and publishes it.

Steps:
1. Open the term's Generation page; check or change the objective weights w1-w5 (FR-CON-03).
2. Set time limit, seed and number of workers; press Generate. The run is queued and the page shows its id at once (FR-SCH-02). If a run is already active, `RUN_IN_PROGRESS` is shown.
3. Watch live progress: prechecks, backtracking (time to first feasible), simulated annealing (best F, unassigned sessions). The admin may cancel; the best feasible solution is kept as a draft (FR-SCH-03, FR-SCH-04).
4. Review the result: validator-checked metrics (hard violations, F and S1-S5, room utilisation) and the weekly grid filtered by cohort, teacher or room, with penalised sessions highlighted (FR-SCH-05, FR-SCH-08).
5. Send the draft to teacher review, or publish. Publishing is blocked with a list of violations if the validator finds any (FR-SCH-10, FR-SCH-11). Export to XLSX (FR-SCH-12).


### 2.2 Admin: understand why generation failed (UC-06, UC-07)

Goal: when no valid timetable exists, the scheduler sees which rules conflict and what to change, without reading solver logs. This is the screen that answers the proposal's problem statement about manual corrections when rules conflict (P §4).

Steps:
1. The run ends with "No valid timetable".
2. The page lists each conflict as a small set of named constraint groups (e.g. "Unavailability of teacher A. Ivanova", "Capacity of computer labs", "Stream DB-02"), together with the reasons teachers gave (FR-SCH-06, FR-CON-05).
3. Under each conflict, repair hints link to the data to change: relax an unavailability, add a room, split a stream, add a slot (FR-SCH-07).
4. If a precheck failed, its message is shown instead (FR-SCH-01).


### 2.3 Admin: import reference data and track confirmations (UC-02, UC-11)

Goal: load the term's data from Excel without partial imports, then follow teachers' confirmations.

Steps:
1. Download the template, fill it, upload it (FR-CAT-09).
2. If any row is invalid, nothing is written and the report lists sheet, row, column and message for each error. After fixing, the import shows counts of created and updated rows (FR-CAT-04).
3. During review, the confirmation board shows counts and a per-teacher list: confirmed, pending, regeneration requested with the teacher's comment, expired (FR-CONF-06), and the deadline after which option 1 is accepted (FR-CONF-07).



### 2.4 Teacher: set availability and choose an option (UC-16…UC-19)

Goal: the teacher influences the timetable before generation and accepts a concrete weekly timetable after it.

Steps:
1. On the weekly grid, mark cells as preferred, avoided (weights 1-3) or unavailable, with an optional reason (FR-CON-01, FR-CON-02, FR-CON-05). A warning appears when fewer slots are available than the teacher's sessions need (FR-CON-04).
2. When the draft is sent to review, compare up to three options side by side. Each shows days used and how many preferred and avoided slots it hits (FR-CONF-01, FR-CONF-02).
3. Confirm one option. If a room in it was taken in the meantime, `PROPOSAL_EXPIRED` is shown and new options are prepared (FR-CONF-03).
4. Or request regeneration with a comment; only this teacher's sessions are re-solved (FR-CONF-04).


### 2.5 Student: enroll in streams (UC-20…UC-23, UC-25)

Goal: the student picks a stream (teacher and time) for each course of the program without clashes, on a phone.

Steps:
1. Open a course; see its streams with teacher, times, free seats and a clash mark (FR-ENR-01). Seat counts update live (FR-ENR-08).
2. Enroll, switch or drop. Messages: stream full (`STREAM_FULL`), time clash with an enrolled stream (`SCHEDULE_CONFLICT`), switch kept the old seat on failure, window closed (`ENROLLMENT_CLOSED`) (FR-ENR-02…07).
3. Open "My week" with lessons and own room bookings, and copy the iCal link (FR-ENR-09).



### 2.6 All users: find and book a free room (UC-26…UC-28)

Goal: a student or teacher finds a free room for self-study or a meeting and books it.

Steps:
1. Choose date, time interval, minimum capacity, type and floor; free rooms are highlighted on the SVG plan and listed (FR-BOOK-01, FR-BOOK-02).
2. Click a room to see capacity, features and the day's occupations (FR-BOOK-10).
3. Book. If someone booked an overlapping interval a moment earlier, `ROOM_ALREADY_BOOKED` is shown with free alternatives on the same floor (FR-BOOK-03, FR-BOOK-11). Limits per role are enforced (`BOOKING_LIMIT`, FR-BOOK-05).
4. See and cancel own bookings; bookings cancelled by the admin show the reason (FR-BOOK-04, FR-BOOK-06, FR-BOOK-09).



### 2.7 Coverage of Must requirements by mock-ups

| Module | Must FRs with a user interface | Shown in |
|---|---|---|
| auth | FR-AUTH-01 | 2.0 |
| catalog | FR-CAT-04 (FR-CAT-01…03, 05…08 are standard CRUD forms, not drawn) | 2.3 |
| constraints | FR-CON-01, 02, 03 | 2.4, 2.1 |
| scheduling | FR-SCH-01…03, 05, 06, 08, 10…12 | 2.1, 2.2 |
| confirmation | FR-CONF-01, 03, 04, 06 | 2.4, 2.3 |
| enrollment | FR-ENR-01…07, 09 | 2.5 |
| booking + map | FR-BOOK-01…06 | 2.6 |

## 3. API and data examples

These examples are the first draft of `api/openapi.yaml` (FR-X-03). Paths and field names may change when the contract is written (checkpoint 05.12.2026), but the error format, status codes and error codes are fixed by the requirements. All errors use RFC 9457 `application/problem+json` (FR-X-01). Every mutating request carries an `Idempotency-Key` header (FR-X-02). Ids are shortened for readability.

### 3.1 Sign in (FR-AUTH-01)

```http
POST /api/v1/auth/login
Content-Type: application/json

{ "email": "student@example.edu", "password": "••••••••" }
```

```http
HTTP/1.1 200 OK
Set-Cookie: refresh_token=…; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth
Content-Type: application/json

{ "accessToken": "eyJhbGciOi…", "expiresIn": 900 }
```

A wrong password and an unknown e-mail both return:

```json
{ "type": "https://smartschedule/errors/invalid-credentials", "title": "Invalid e-mail or password", "status": 401, "code": "INVALID_CREDENTIALS" }
```

### 3.2 Start a generation run (FR-SCH-02)

```http
POST /api/v1/terms/t-2027s/runs
Authorization: Bearer eyJhbGciOi…
Idempotency-Key: 6f1c2e0a-0b7d-4c55-9a51-2d6f8d1f3e11
Content-Type: application/json

{ "timeLimitSec": 120, "seed": 7, "workers": 4 }
```

```http
HTTP/1.1 202 Accepted
Location: /api/v1/runs/r-48

{ "runId": "r-48", "status": "queued" }
```

When another run of the term is active:

```json
{
  "type": "https://smartschedule/errors/run-in-progress",
  "title": "A generation run is already active for this term",
  "status": 409,
  "code": "RUN_IN_PROGRESS",
  "activeRunId": "r-47"
}
```

### 3.3 Generation progress over SSE (FR-SCH-03)

```http
GET /api/v1/runs/r-48/events
Authorization: Bearer eyJhbGciOi…
Accept: text/event-stream
```

```text
retry: 3000

id: 41
event: progress
data: {"runId":"r-48","phase":"annealing","elapsedMs":70210,"bestF":1184,"unassigned":0,"hardViolations":0}

id: 58
event: finished
data: {"runId":"r-48","status":"feasible","versionId":"v-12","timeToFirstFeasibleMs":6800,"finalF":1021}
```

### 3.4 Infeasibility report (FR-SCH-06, FR-SCH-07)

`GET /api/v1/runs/r-49/report` returns the report shown in mock-up 2.2. The `text` fields are produced from templates in the user's language.

```json
{
  "runId": "r-49",
  "status": "infeasible",
  "explanationTimeMs": 14200,
  "prechecks": [],
  "conflicts": [
    {
      "groups": [
        { "kind": "teacher_unavailability", "ref": "teacher:T-0031",
          "text": "Teacher A. Ivanova is not available Mon–Wed", "reason": "teaching at another faculty" },
        { "kind": "room_type_capacity", "ref": "room_type:computer",
          "text": "Computer labs with ≥ 30 seats are full on Thu–Fri, pairs 1–4" },
        { "kind": "stream_sessions", "ref": "stream:DB-02",
          "text": "Stream DB-02 needs 2 lab sessions per week for 30 students" }
      ],
      "hints": [
        { "action": "relax_unavailability", "ref": "teacher:T-0031", "text": "Free 2 pairs on Mon–Wed" },
        { "action": "add_room", "ref": "room_type:computer", "text": "Add a computer lab with ≥ 30 seats" },
        { "action": "split_stream", "ref": "stream:DB-02", "text": "Split DB-02 into two streams of 15" },
        { "action": "add_slot", "ref": "grid", "text": "Add pair 7 on Thu–Fri" }
      ]
    }
  ]
}
```

A precheck failure has the same shape with `conflicts: []` and, for example:

```json
"prechecks": [
  { "code": "TEACHER_OVERLOAD", "ref": "teacher:T-0044",
    "text": "Teacher K. Kim teaches 14 pairs but marked only 10 slots available" }
]
```

### 3.5 Enroll in a stream (FR-ENR-02, FR-ENR-03, FR-X-02)

```http
POST /api/v1/enrollments
Authorization: Bearer eyJhbGciOi…
Idempotency-Key: 0d9e5a77-4b8a-4c7f-8f0e-1a2b3c4d5e6f
Content-Type: application/json

{ "streamId": "WEB-02" }
```

Success:

```http
HTTP/1.1 201 Created

{ "enrollmentId": "e-9081", "streamId": "WEB-02", "seatsLeft": 0 }
```

Stream full:

```json
{
  "type": "https://smartschedule/errors/stream-full",
  "title": "Stream is full",
  "status": 409,
  "code": "STREAM_FULL",
  "streamId": "WEB-02",
  "capacity": 30
}
```

Time clash:

```json
{
  "type": "https://smartschedule/errors/schedule-conflict",
  "title": "Time clash with an enrolled stream",
  "status": 409,
  "code": "SCHEDULE_CONFLICT",
  "conflictingStreamId": "DB-01",
  "slot": { "day": "MON", "pair": 1 }
}
```

Repeating the same request with the same `Idempotency-Key` returns the first response and creates no second enrollment.

### 3.6 Book a room (FR-BOOK-03)

```http
POST /api/v1/bookings
Authorization: Bearer eyJhbGciOi…
Idempotency-Key: 3a7c…
Content-Type: application/json

{ "roomId": "C1.3.310", "from": "2027-02-15T14:00:00+05:00", "to": "2027-02-15T16:00:00+05:00", "purpose": "Group project meeting" }
```

Overlap rejected by the exclusion constraint (SQLSTATE 23P01):

```json
{
  "type": "https://smartschedule/errors/room-already-booked",
  "title": "Room is already occupied in this interval",
  "status": 409,
  "code": "ROOM_ALREADY_BOOKED",
  "roomId": "C1.3.310",
  "overlapping": { "kind": "booking", "from": "2027-02-15T14:30:00+05:00", "to": "2027-02-15T15:30:00+05:00" },
  "alternatives": ["C1.3.302", "C1.3.312"]
}
```

### 3.7 Teacher confirms an option (FR-CONF-03)

```http
POST /api/v1/versions/v-12/options/o-31/confirm
Authorization: Bearer eyJhbGciOi…
Idempotency-Key: 8b21…
```

```json
{
  "type": "https://smartschedule/errors/proposal-expired",
  "title": "This option is no longer available",
  "status": 409,
  "code": "PROPOSAL_EXPIRED",
  "takenBy": { "roomId": "C1.2.218", "slot": { "day": "TUE", "pair": 2 } },
  "newOptionsPending": true
}
```

### 3.8 `solverctl` command-line tool (FR-SCH-14)

`solverctl` runs the same solver package outside the web platform, for tests and experiments.

```text
solverctl solve --input comp01.ctt --format itc2007 --time-limit 60 --seed 1 --workers 1 --out comp01.sol
```

Output (stdout, JSON):

```json
{
  "instance": "comp01",
  "seed": 1,
  "workers": 1,
  "status": "feasible",
  "timeToFirstFeasibleMs": 210,
  "totalMs": 60000,
  "hardViolations": 0,
  "softCost": 5,
  "softByConstraint": { "RoomCapacity": 0, "MinWorkingDays": 5, "CurriculumCompactness": 0, "RoomStability": 0 }
}
```

The solution file `comp01.sol` uses the ITC-2007 format (`course room day period` per line), so the official validator can check that its cost equals `softCost`. The numbers above are illustrative, not results.

Input in the project's own JSON format (shortened):

```json
{
  "grid": { "days": 5, "pairsPerDay": 6 },
  "rooms": [ { "id": "C1.2.218", "type": "computer", "capacity": 30 } ],
  "teachers": [ { "id": "T-0031", "unavailable": [[0,1],[0,2]], "preferences": [{ "slot": [1,1], "weight": 3 }] } ],
  "streams": [ { "id": "DB-02", "teacherId": "T-0031", "size": 30, "cohorts": ["SE-2301"],
                 "sessions": [ { "type": "lab", "durationPairs": 1, "roomType": "computer", "perWeek": 2 } ] } ],
  "weights": { "w1": 3, "w2": 2, "w3": 1, "w4": 2, "w5": 1 },
  "fixed": []
}
```
