# Project proposal: AITU Smart Schedule

## 1. Project title

Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms

Working name: AITU Smart Schedule.

Note: the formal title must match the wording of the officially approved topic. If the approved wording differs, this title will be changed to match it.

## 2. Team members

| Member | Role                                                                                                                                                                                                                                              |
| --- |---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Aizada Sembayeva | Scheduling algorithm (validator, generator, infeasibility report), OpenAPI contract, teacher confirmation and student enrollment backend, requirements, literature review on algorithms, experiments                                              |
| Aiya Zhakupova | User research (Registrar's Office and student interviews), database schema and migrations, Docker and CI, authentication, catalog and Excel import backend, React frontend, room map and booking, synthetic data generator, diagrams and mock-ups |

## 3. Supervisor

Nadirov Nurtas.

## 4. Problem statement

University timetabling is hard because classes, instructors, student groups, rooms and time limits all have to be considered together. Preparing a timetable takes a lot of time and often still needs manual corrections when rules or preferences conflict, which leads to overlapping classes, poor room use and inconvenient schedules. 


Published research explains why the problem is hard. University course timetabling is NP-hard, and exact algorithms are "only applicable to small sized problems" (Chen et al., 2021, p. 106516). Chen et al. (2021) also note that hard and soft constraints vary between institutions and that state-of-the-art benchmark methods "are not fully utilised in real-world" timetabling (p. 106522). A system therefore needs a configurable constraint model and an evaluation on a standard benchmark. Stream enrollment raises a separate issue. First-come-first-served registration is common, but it can cause "a competitive race that can overload systems and lead to inequitable outcomes" (Romero-Medina & Triossi, 2024, p. 3). Our MVP keeps first-come-first-served enrollment, so it must at least stay correct when many students try to take the last seats at the same time.

## 5. Target users

| User | Needs |
| --- | --- |
| Scheduler (Registrar's Office) | Keep reference data (courses, streams, rooms, teachers, cohorts, time grid) and import it from Excel; generate a timetable with no hard-constraint violations; see in plain language why generation fails; publish and export the timetable |
| Teacher | Set unavailable times and preferences; review 1 to 3 valid timetable options |
| Student | See the streams of their own study program; choose a teacher and time by enrolling in a stream without clashes in their personal timetable; book a free room for self-study |

## 6. Research question

Main question: How can a two-phase constraint-based algorithm generate a timetable with no hard-constraint violations for a university of AITU's assumed size within fixed time limits, explain why an input has no valid timetable, and support stream enrollment and room booking that stay conflict-free under concurrent use?

Sub-questions:

1. How is the timetable built at AITU today, and which constraints and problems matter most to the scheduler, teachers and students?
2. How does the proposed algorithm (backtracking with forward checking and MRV/LCV ordering, followed by simulated annealing) compare with Google OR-Tools CP-SAT on ITC-2007 Track 3 and on synthetic data of AITU's assumed size?
3. Which database-level mechanisms in PostgreSQL prevent stream overflow and double booking of rooms and teachers when hundreds of requests arrive at once, and at what response time?

## 7. Aim

To develop an algorithm that automatically generates a university timetable and a web platform built on it, in which teachers confirm their timetable, students enroll in streams and all users book free rooms without time, teacher or room conflicts.

## 8. Objectives

1. Identify the main academic constraints and candidate functional requirements in the reviewed literature.
2. Implement the core scheduling requirements in a working prototype and verify them with automated validation tests and synthetic scheduling scenarios.
3. Collect feedback from university scheduling staff on the practical importance of these requirements, as an additional validation layer.


## 9. Expected outcomes

- A timetable generator written in Go that meets the time and scale targets and gives the same result for the same random seed.

- A web platform for the scheduler, teachers and students, deployed with Docker Compose.
- PostgreSQL constraints (exclusion constraints on time ranges and conditional updates) that block double booking of a room or teacher and stream overflow.


## 10. Project scope

- Reference data (courses, streams, rooms, teachers, cohorts, time grid) and Excel import.
- Automatic timetable generation in two phases (backtracking, then simulated annealing) with an infeasibility report.
- Teacher options, confirmation; confirmed classes become fixed for later runs.
- Student enrollment in streams of their own study program.
- Room booking and an interactive room map (basic version) built from SVG floor plans.


Out of scope (future work):

- Integration with AITU's existing systems.
- Exam timetabling.
- Teacher substitutions during the semester.
- An open elective catalog across programs.


## References


Bonutti, A., De Cesco, F., Di Gaspero, L., & Schaerf, A. (2012). Benchmarking curriculum-based course timetabling: Formulations, data formats, instances, validation, visualization, and results. *Annals of Operations Research, 194*(1), 59-70. https://doi.org/10.1007/s10479-010-0707-0

Chen, M. C., Sze, S. N., Goh, S. L., Sabar, N. R., & Kendall, G. (2021). A survey of university course timetabling problem: Perspectives, trends and opportunities. *IEEE Access, 9*, 106515-106529. https://doi.org/10.1109/ACCESS.2021.3100613

Di Gaspero, L., McCollum, B., & Schaerf, A. (2007). *The Second International Timetabling Competition (ITC-2007): Curriculum-based course timetabling (Track 3)* (Technical Report QUB/IEEE/Tech/ITC2007/CurriculumCTT/v1.0). Queen's University Belfast. https://www.eeecs.qub.ac.uk/itc2007/curriculmcourse/report/curriculumtechreport.pdf

Junker, U. (2004). QUICKXPLAIN: Preferred explanations and relaxations for over-constrained problems. In *Proceedings of the 19th National Conference on Artificial Intelligence (AAAI-04)* (pp. 167-172). AAAI Press. https://cdn.aaai.org/AAAI/2004/AAAI04-027.pdf

Perron, L., & Didier, F. (2025). *CP-SAT* (Version 9.12) [Computer software]. Google. https://developers.google.com/optimization/cp/cp_solver/

Romero-Medina, A., & Triossi, M. (2024). Strategic priority-based course allocation. *Journal of Economic Behavior and Organization, 226*, Article 106701. https://doi.org/10.1016/j.jebo.2024.106701


