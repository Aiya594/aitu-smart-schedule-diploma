# Comparison of existing timetabling systems for AITU Smart Schedule

Project: Development of a Web Platform for Planning Academic Streams with a Module for Selecting Courses, Instructors, and Booking Classrooms (working name: AITU Smart Schedule). Team: Aizada Sembayeva, Aiya Zhakupova. Supervisor: Nadirov Nurtas.

UniTime covers the most criteria of the four products reviewed: automatic generation, instructor self-entry of preferences through a web survey, clash-avoiding student sectioning, event room booking with approval, and an outdoor map. Russian and Kazakh are not among its shipped interface languages, and its diagnostics are built for expert schedulers. FET is a free single-operator desktop generator with a maintained Russian translation. aSc Timetables and Untis/WebUntis are commercial, school-first products whose diagnostic tools (aSc's Test with relaxation, Untis Diagnosis) are the strongest prior art for explaining failures, but neither documents teacher self-entry, a floor map or Kazakh. Publicly, AITU's current process is known only in outline: the Registrar's Office builds the schedule from students' individual curricula and delivers it through Microsoft Teams and Outlook; the method is not publicly documented. No option combines teacher self-service with confirmation of a generated option, live clash-checked stream enrollment, indoor floor-map booking, plain-language infeasibility explanations. AITU Smart Schedule targets that combination. It is a planned diploma prototype and does not try to match the scale or maturity of the products below.

## 1. Scope and method

The AITU diploma guideline asks the theoretical part of a diploma project to compare existing systems and position the product against them. Task names the candidates (UniTime, FET, aSc Timetables, Untis and the current AITU process) and requires a link for every claim.

The five options fall into three groups. UniTime and FET are open source and used at universities. UniTime was designed for universities, and FET is aimed mainly at schools but has university users. aSc Timetables and Untis/WebUntis are commercial products built mainly for schools. Untis has a university offering and documented university deployments; aSc has a student-based generator for course sections, but no university edition or named university customer was found. The current AITU process is the as-is baseline that AITU Smart Schedule would replace or support.

Evidence comes from official documentation, help centres, release notes and vendor sites, plus peer-reviewed papers and deployment documents from user institutions. All pages were accessed in October 2026. Some vendor pages render prices, help text or language lists with JavaScript, and that content could not be read.

Each cell in the matrix uses one symbol. ● means full support is documented. ◐ means the feature exists but misses part of the criterion definition. ○ means the sources state that the feature is absent. ? means the feature is not documented in the sources reviewed. No cell received ○, because the sources describe what products do and almost never state what they lack. For criterion 7 the symbols mean: ● no licence fee, ◐ paid licence, ? cost not public. 

The comparison has four limits. Vendor prices are not public: aSc's own price page showed no amounts and Untis is sold by quote. aSc and Untis do not publish their algorithms. AITU's internal process is not publicly documented, the main source is a 2020 student guidebook, and the 2023 Academic Policy could not be read; the AITU column must be checked against feedback from the university's scheduling staff (objective 3 of the proposal). Finally, no independent 2025-2026 usability reviews were found for any product.

## 2. Criteria

1. Automatic generation: the system builds a complete course timetable from input data without manual placement of each class.
2. Teacher preferences: teacher unavailability and soft preferences are inputs to generation; full support also requires that teachers can enter them themselves.
3. Student stream (section) selection without clashes: students choose among parallel streams of a course (teacher + time slot) and the system prevents or minimises time clashes in their personal timetable.
4. Room booking: users can reserve a free room for an ad hoc event or self-study outside the generated timetable, with protection against double booking.
5. Map / room location: the system shows where a room is on a campus or indoor map, and ideally lets users pick a room from the map.
6. Infeasibility diagnostics: when no complete timetable exists, the system identifies which constraints conflict, in terms the user can act on.
7. Licence/price: licence type and acquisition cost, as far as they are public.


## 3. System profiles

### UniTime

UniTime is a multi-user Java web application covering course timetabling, examination timetabling, student scheduling, instructor scheduling and event management, with distributed or centralised data entry [[4]](https://www.apereo.org/programs/software/unitime). It is licensed under Apache 2.0 and has been a sponsored project of the Apereo Foundation since March 2015 [[5]](https://www.unitime.org/). Its solver is an iterative forward search that keeps partial but consistent solutions and uses conflict-based statistics to escape local optima [[1]](https://icaps07-satellite.icaps-conference.org/demos/1/mms.pdf). The UniTime team co-organised the International Timetabling Competition 2019 [[5]](https://www.unitime.org/). Platform: Java on Apache Tomcat 10.1 or later since 4.9 build 82 (30 September 2025) [[4]](https://www.apereo.org/programs/software/unitime). Latest version: 4.9 build 152, dated 14 September on GitHub without a year; 2026 is inferred [[13]](https://github.com/UniTime/unitime/releases). Notable users: Purdue University, which has run its whole timetable on UniTime since January 2007 (about 9,000 classes, 570 rooms and 259,000 student class requests in 2007 figures) [[1]](https://icaps07-satellite.icaps-conference.org/demos/1/mms.pdf), and two faculties of Masaryk University [[2]](https://ideas.repec.org/a/spr/annopr/v239y2016i1d10.1007_s10479-014-1643-1.html).

### FET

FET is a free timetable generator first released in 2002 and licensed under the GNU AGPL version 3 [[16]](https://lalescu.ro/liviu/fet/). Since version 5.0.0 (2007) it has used a heuristic its author calls "recursive swapping": activities are sorted from most to least difficult, and an activity that has no free slot takes the slot with the fewest conflicts while the displaced activities are placed again recursively [[15]](https://lalescu.ro/liviu/fet/doc/en/generation-algorithm-description.html). It usually solves a complicated timetable in 5-20 minutes [[16]](https://lalescu.ro/liviu/fet/). Platform: C++/Qt desktop application for Linux, Windows and macOS, plus a command-line version [[14]](https://lalescu.ro/liviu/fet/features.html). Latest version: 7.12.0 (14 October 2026) [[19]](https://lalescu.ro/liviu/fet/news.html). Notable users: Near East University in North Cyprus has built university-wide timetables with FET for about six years, covering thousands of students and teachers [[21]](https://www.timeshighereducation.com/campus/how-create-universitywide-timetables-using-free-opensource-software).

### aSc Timetables

aSc Timetables is a commercial product from Slovakia, with EduPage as its web companion for publication, booking and registration. The vendor claims 200,000 schools worldwide and advertises an AI panel that can answer questions and, if permitted, change the data [[23]](https://www.asctimetables.com/). A cloud generator uses both the user's PC and the vendor's servers and can run until it finds a solution [[25]](https://help.edupage.org/?p=u1/u3/u58/t1294). A student-based generator handles sections of courses [[27]](https://help.edupage.org/?p=u1/u3/u63). Platform: a Windows desktop application that works offline and an online version for Mac, Windows, Linux and ChromiumOS that several users can edit at once [[24]](https://www.asctimetables.com/a/prices). Latest version: the desktop download is Version 2027, dated 10 June 2026 [[23]](https://www.asctimetables.com/). Notable users: no named university customers and no university edition page were found.

### Untis / WebUntis

Untis is made by Untis GmbH (Stockerau, Austria). The desktop planner has a basic package plus modules such as course scheduling (Kursplanung), department timetable, substitution planning and MultiUser. vUntis is a browser-based version, WebUntis is the web layer for publication and booking, and Untis Mobile runs on iOS and Android [[34]](https://www.untis.at/fileadmin/user_upload/Untis_Imagefolder_07.2025.pdf). The vendor's own figures disagree: the July 2025 brochure claims over 26,000 institutions [[34]](https://www.untis.at/fileadmin/user_upload/Untis_Imagefolder_07.2025.pdf) and the university page claims over 30,000 [[35]](https://www.untis.at/en/solutions/untis-for/educational-institutions). For universities Untis offers its standard modules plus consulting during introduction; no separate university module was found [[35]](https://www.untis.at/en/solutions/untis-for/educational-institutions). Latest version: not stated in the sources reviewed.

## 4. Comparison matrix

| Criterion | UniTime | FET | aSc Timetables | Untis/WebUntis |
|---|---|---|---|---|
| 1. Automatic generation | ● CPSolver, Purdue scale | ● recursive swapping heuristic | ● generator + cloud generator; algorithm unpublished | ● optimisation with course clusters; algorithm unpublished |
| 2. Teacher preferences (self-entry) | ● graded preferences; Instructor Survey for self-entry | ◐ weighted constraints; operator enters them | ◐ time-off entered in planner; self-entry not documented | ? not documented in pages read |
| 3. Stream selection without clashes | ● course requests, batch + online sectioning, wait-lists | ? fixed student sets; no self-enrollment documented | ◐ student-based generator from school-entered requests | ◐ WebUntis Student registration, imported into Kursplanung |
| 4. Room booking | ● event management with approval | ? not in documentation | ● EduPage ad hoc room booking | ● WebUntis Booking with rights system |
| 5. Map / room location | ◐ outdoor OpenStreetMap; floor plan not documented | ? not found | ? buildings as data only | ? list and drop-down search |
| 6. Infeasibility diagnostics | ◐ not-assigned reasons, suggestions, conflict statistics | ◐ advice, some pre-checks | ◐ Test with relaxation, Verification, Advisor | ◐ Diagnosis before/after, Reason column |
| 7. Licence / price | ● Apache 2.0, free, self-hosted | ● AGPL-3.0, free | ◐ one-time licence; reseller list price from $149 (Techjockey, undated) | ◐ paid, quote only |


### Evidence notes

UniTime. (1) Iterative forward search with conflict-based statistics; Purdue scale [[1]](https://icaps07-satellite.icaps-conference.org/demos/1/mms.pdf); solver phases [[3]](https://help.unitime.org/manuals/courses-solver). (2) "Stronger wins" preference rule [[6]](https://help.unitime.org/application-of-preferences); Instructor Survey with time and room preferences, a reason field for hard requirements, and lock after submission [[7]](https://help.unitime.org/instructor-survey). (3) Prioritised course requests with alternatives, batch solver, online Scheduling Assistant, time conflicts avoided, wait-lists, students imported from an SIS [[8]](https://help.unitime.org/manuals/student-scheduling). (4) Event requests, availability grid, pending/approve/reject, approved events block rooms for course and exam timetabling [[9]](https://help.unitime.org/manuals/events). (5) OpenStreetMap/Leaflet default since 4.4, no floor plan mentioned [[10]](https://help.unitime.org/openstreetmap). (6) Not-assigned classes, "no available placement", Suggestions, conflict-based statistics, advice to contact administrators [[3]](https://help.unitime.org/manuals/courses-solver). (7) Apache 2.0 [[12]](https://github.com/UniTime/unitime); "always free and always open" [[4]](https://www.apereo.org/programs/software/unitime). (8) Locales en, en_UK, cs, pl; translations through POEditor [[11]](https://help.unitime.org/localization).

FET. (1) Fully automatic generation [[14]](https://lalescu.ro/liviu/fet/features.html); algorithm [[15]](https://lalescu.ro/liviu/fet/doc/en/generation-algorithm-description.html). (2) Constraint weights 0-100%, teacher not-available periods, preferred times [[14]](https://lalescu.ro/liviu/fet/features.html); Near East University collected requests in a shared Google Sheet and staff entered them [[21]](https://www.timeshighereducation.com/campus/how-create-universitywide-timetables-using-free-opensource-software). (3) Students as years, groups and subgroups [[14]](https://lalescu.ro/liviu/fet/features.html). (4)(5) Room constraints only; no booking or map on the features page [[14]](https://lalescu.ro/liviu/fet/features.html). (6) Remove or weaken constraints, "impossible activity" report with locked activities [[17]](https://lalescu.ro/liviu/fet/doc/en/faq.html); generate without rooms first [[18]](https://lalescu.ro/liviu/fet/doc/en/instructions.html); "Remove constraints if FET fail to find a solution" [[20]](https://www.timetabling.de/manual/FET-manual.en.html); 7.1.6 outputs current and highest stage timetables, 7.8.3 adds a pre-generation check for one constraint type [[19]](https://lalescu.ro/liviu/fet/news.html). (7) AGPL-3.0 [[16]](https://lalescu.ro/liviu/fet/); "version 3 or later" [[22]](https://appstream.debian.org/sid/main/metainfo/fet.html). (8) About 40 languages including Russian, Kazakh not listed [[14]](https://lalescu.ro/liviu/fet/features.html); Russian updated in 7.9.1 [[19]](https://lalescu.ro/liviu/fet/news.html).

aSc Timetables. (1) "Over 5,000,000 possibilities" [[23]](https://www.asctimetables.com/); cloud generator [[25]](https://help.edupage.org/?p=u1/u3/u58/t1294). (2) Help articles on teacher time-off (titles only) [[26]](https://help.edupage.org/?p=u1%2Fu3); EduPage "Requests" is for leave of absence [[31]](https://edupage.org/). (3) School enters requests, sections defined, students placed into one section or none [[28]](https://help.edupage.org/?p=u1/u3/u63/t1231); EduPage Registration [[31]](https://edupage.org/); student-based generator only in Pro edition per reseller [[32]](https://www.techjockey.com/detail/asctimetables). (4) Ad hoc room booking [[31]](https://edupage.org/); room booking on the homepage [[23]](https://www.asctimetables.com/). (5) "Buildings" section, no map topic in help index [[26]](https://help.edupage.org/?p=u1%2Fu3). (6) Test options including "Test with RELAXATION" [[29]](https://help.edupage.org/?p=u1/u3/u59/t130); Verification lists broken or relaxed constraints [[30]](https://help.edupage.org/?p=u1/u3/u60/t540); Advisor checks (titles only) [[26]](https://help.edupage.org/?p=u1%2Fu3). (7) Unlimited free trial without export, no amounts visible [[24]](https://www.asctimetables.com/a/prices); Standard $149, Premium $390, Pro $699 "starting at", reseller list price (Techjockey, undated) [[32]](https://www.techjockey.com/detail/asctimetables). (8) aSc site in 11 languages without Russian [[24]](https://www.asctimetables.com/a/prices); EduPage help lists Russian but not Kazakh [[33]](https://help.edupage.org/?p=lang_id=1&p=u1/u3/u56/u610&lang_id=1).

Untis/WebUntis. (1) Course clusters as the basis of the algorithm [[36]](https://www.untis.at/en/products/untis-basic-software/course-scheduling-1); university timetables "generated and administered by several persons" [[35]](https://www.untis.at/en/solutions/untis-for/educational-institutions). (2) No teacher self-entry on the pages read [[37]](https://www.untis.at/en/solutions/untis-for/untis-for-all-school-types); teachers can change rooms and add events after generation [[39]](https://www.untis.at/en/products/webuntis/booking). (3) Online registration imported into Untis scheduling [[38]](https://www.untis.at/en/products/webuntis/student). (4) Free/occupied overview, equipment booking, rights system [[39]](https://www.untis.at/en/products/webuntis/booking). (5) No map on the Booking page [[39]](https://www.untis.at/en/products/webuntis/booking). (6) Diagnosis before and after scheduling [[40]](https://webhelp.untis.at/HTML/WebHelp/uk/untis/opdiagnose.htm); Overall diagnosis with points and a "Reason" column [[41]](https://webhelp.untis.at/HTML/WebHelp/uk/untis/optotaldiagnose.htm); WebUntis diagnosis flags clashes and unmet block requirements (2019 article) [[42]](https://help.untis.at/hc/en-150/articles/360009227740-Timetable-diagnosis). (7) No public price list, priced by module, separate university price list (secondary source) [[43]](https://www.ki-syndikat.de/tools/webuntis/). (8) Site in EN, DE, ES, FI; no CIS presence in the brochure [[37]](https://www.untis.at/en/solutions/untis-for/untis-for-all-school-types), [[34]](https://www.untis.at/fileadmin/user_upload/Untis_Imagefolder_07.2025.pdf).


## 5. Analysis by criterion

### 5.1 Automatic generation

All four products generate timetables; they differ in transparency and proven scale. UniTime's solver is described in published papers and has handled Purdue's 9,000 classes [[1]](https://icaps07-satellite.icaps-conference.org/demos/1/mms.pdf). FET's author documents the algorithm informally [[15]](https://lalescu.ro/liviu/fet/doc/en/generation-algorithm-description.html). aSc and Untis describe their generators only in marketing terms, as search with constraint relaxation [[25]](https://help.edupage.org/?p=u1/u3/u58/t1294) and as optimisation built on course clusters, with the German upper secondary school as Untis's example [[36]](https://www.untis.at/en/products/untis-basic-software/course-scheduling-1). We infer from the documentation that both model a university as a school, with semester groups as "classes" and sections as seminar lessons. For AITU the method is not public. AITU Smart Schedule can add a documented, reproducible solver tested on ITC-2007 Track 3 (UD2) against an OR-Tools CP-SAT baseline, at a planned scale far below Purdue's.

### 5.2 Teacher preferences and confirmation

UniTime's Instructor Survey is the closest prior art: instructors enter time and room preferences, give a reason for hard requirements, and cannot change the survey after submission unless a deputy reopens it [[7]](https://help.unitime.org/instructor-survey). The survey runs before scheduling. No step was found where a teacher reviews and approves the generated result. FET has fine-grained weighted constraints [[14]](https://lalescu.ro/liviu/fet/features.html), but only the operator enters them; Near East University collected requests in a shared spreadsheet [[21]](https://www.timeshighereducation.com/campus/how-create-universitywide-timetables-using-free-opensource-software). aSc documents teacher time-off in the planner, and EduPage "Requests" handles leave of absence [[31]](https://edupage.org/). The Untis pages read describe post-generation room changes by teachers [[39]](https://www.untis.at/en/products/webuntis/booking) but say nothing about preference entry. AITU Smart Schedule's loop, in which teachers enter constraints, see 1-3 valid options, confirm one, and have the confirmed classes fixed for later runs, is undocumented in all four products.

### 5.3 Student stream selection

UniTime has the most complete student sectioning: prioritised requests with alternatives, a batch solver that maximises the courses students get, an online assistant that keeps the existing schedule where possible, and wait-lists [[8]](https://help.unitime.org/manuals/student-scheduling). Students must be imported from an external SIS. Untis and aSc treat choice as a batch step. Students register in WebUntis Student, and the data is imported into Kursplanung [[38]](https://www.untis.at/en/products/webuntis/student). In aSc the school enters requests and the generator places each student into one section or none [[28]](https://help.edupage.org/?p=u1/u3/u63/t1231).

### 5.4 Room booking

UniTime, EduPage and WebUntis all support ad hoc booking. UniTime adds the most control: pending, approved and rejected states, per-room permission levels, and approved events that block rooms for course and exam timetabling [[9]](https://help.unitime.org/manuals/events). WebUntis has a free/occupied overview, equipment booking and a rights system [[39]](https://www.untis.at/en/products/webuntis/booking). No booking feature was found in FET's documentation. Whether students can book was not documented for aSc or Untis. AITU publishes no booking tool; the Registrar's Office lists control of classroom occupancy as its function [[47]](https://astanait.edu.kz/en/registrar-2). The project adds database-level protection against double booking and room booking for students' self-study.

### 5.5 Map and room location

UniTime stores room coordinates and shows buildings on OpenStreetMap or Google Maps. It also uses coordinates for travel distance and "close by" room search [[10]](https://help.unitime.org/openstreetmap), [[9]](https://help.unitime.org/manuals/events). No floor plan was found in its documentation. aSc treats buildings as data for constraints, and WebUntis offers list and drop-down room search. AITU's website links to an "AITU Map" whose function was not verified [[49]](https://sj-front.astanait.edu.kz/); feedback from the scheduling staff should check whether it already locates rooms. An interactive SVG floor map for finding and booking a free room is the clearest gap in the products reviewed. It is also the feature the project's own analysis calls secondary, so it carries scope risk.

### 5.6 Infeasibility diagnostics

aSc's Test is the closest prior art. When it fails, it shows the partial timetable and the card it could not place, and "Test with RELAXATION" reveals which constraint is too restrictive [[29]](https://help.edupage.org/?p=u1/u3/u59/t130). The Verification view lists broken or relaxed constraints and highlights the affected cards [[30]](https://help.edupage.org/?p=u1/u3/u60/t540). Untis Diagnosis checks input data before scheduling and reviews the result afterwards [[40]](https://webhelp.untis.at/HTML/WebHelp/uk/untis/opdiagnose.htm). Its overall diagnosis ranks the worst class and teacher timetables and gives a reason such as "Room not available" [[41]](https://webhelp.untis.at/HTML/WebHelp/uk/untis/optotaldiagnose.htm). UniTime offers load-time reasons, suggestions and conflict-based statistics down to the competing classes. Its manual still tells users to contact administrators when the cause is unclear [[3]](https://help.unitime.org/manuals/courses-solver). FET relies mostly on advice to remove or weaken constraints, with a few recent pre-generation checks [[17]](https://lalescu.ro/liviu/fet/doc/en/faq.html), [[19]](https://lalescu.ro/liviu/fet/news.html). All four report items, cards or constraint lists. None documents a computed minimal set of conflicting constraints written as a sentence the scheduler in the Registrar's Office can act on. AITU Smart Schedule plans that with QuickXplain. Whether the explanations are understandable has to be tested with users.



## 6. Positioning and implications

Where mature features exist, they are ahead of what a diploma prototype will build: UniTime on sectioning and booking, aSc and Untis on diagnostics. The project's case rests on combining five features for one university:

1. Teacher self-service with a confirmation loop: teachers enter unavailability and preferences, pick one of 1-3 valid options, and their confirmed classes stay fixed in later runs. UniTime covers the first step only.
2. Live, clash-checked enrollment into already-scheduled streams with a capacity guarantee under concurrent load. The commercial products run choice as a batch step.
3. Booking a free room from an indoor SVG floor map.
4. A plain-language explanation of infeasibility based on a minimal conflicting constraint set, going beyond lists of failed items.


The project does not try to match UniTime's Purdue scale (its target is 400 streams, 150 teachers and 120 rooms) or the maturity of FET (since 2002) and UniTime (at Purdue since 2007). The proposal leaves out integration with AITU's existing systems, examination timetabling, substitutions during the semester and the open elective catalog.

The existing systems offer concrete ideas to borrow:

- UniTime: the "stronger wins" rule for combining preferences, a reason field for hard requirements, locking the survey after submission, pending bookings that already block a room, and "close by" alternatives when a room is taken.
- FET: soft-constraint weights with guidance on choosing them, and the diagnostic step of generating without rooms first to separate room shortages from time conflicts.
- aSc: a pre-generation test that names the item it cannot place, showing the best timetable with and without relaxed constraints, and a verification view that highlights affected classes.
- Untis: diagnosis before and after generation, a ranking of the worst teacher and group timetables with a reason column, a rights system for booking, and iCal subscription for published timetables.

The main open question is the AITU baseline. Until feedback from the university's scheduling staff describes how the Registrar's Office builds the timetable, how students choose streams and how rooms are found, the claim that AITU needs this combination is an assumption, and the thesis should keep confirmed facts separate from assumptions.

## 7. Sources

All accessed October 2026.

1. K. Murray, T. Müller, H. Rudová. UniTime system demonstration paper, ICAPS 2007 satellite demos. https://icaps07-satellite.icaps-conference.org/demos/1/mms.pdf
2. T. Müller, H. Rudová. Paper on UniTime curriculum-based timetabling at Masaryk University, Annals of Operations Research 239(1), 2016. https://ideas.repec.org/a/spr/annopr/v239y2016i1d10.1007_s10479-014-1643-1.html
3. UniTime. Course Timetabling Solver Manual. https://help.unitime.org/manuals/courses-solver
4. Apereo Foundation. UniTime. https://www.apereo.org/programs/software/unitime
5. UniTime. Home page. https://www.unitime.org/
6. UniTime. Application of Preferences. https://help.unitime.org/application-of-preferences
7. UniTime. Instructor Survey. https://help.unitime.org/instructor-survey
8. UniTime. Student Scheduling manual. https://help.unitime.org/manuals/student-scheduling
9. UniTime. Events manual. https://help.unitime.org/manuals/events
10. UniTime. OpenStreetMap. https://help.unitime.org/openstreetmap
11. UniTime. Localization. https://help.unitime.org/localization
12. UniTime/unitime repository, GitHub. https://github.com/UniTime/unitime
13. UniTime releases, GitHub. https://github.com/UniTime/unitime/releases
14. FET. Features. https://lalescu.ro/liviu/fet/features.html
15. FET. Generation algorithm description. https://lalescu.ro/liviu/fet/doc/en/generation-algorithm-description.html
16. FET. Home page. https://lalescu.ro/liviu/fet/
17. FET. FAQ. https://lalescu.ro/liviu/fet/doc/en/faq.html
18. FET. Instructions. https://lalescu.ro/liviu/fet/doc/en/instructions.html
19. FET. News. https://lalescu.ro/liviu/fet/news.html
20. V. Dirr. FET manual. https://www.timetabling.de/manual/FET-manual.en.html
21. Y. Çerkez. Article on creating university-wide timetables with free open-source software (FET), Times Higher Education Campus, 1 March 2023. https://www.timeshighereducation.com/campus/how-create-universitywide-timetables-using-free-opensource-software
22. Debian AppStream. FET. https://appstream.debian.org/sid/main/metainfo/fet.html
23. aSc Timetables. Home page. https://www.asctimetables.com/
24. aSc Timetables. Prices and trial. https://www.asctimetables.com/a/prices
25. EduPage help. Cloud generator. https://help.edupage.org/?p=u1/u3/u58/t1294
26. EduPage help. TimeTables index. https://help.edupage.org/?p=u1%2Fu3
27. EduPage help. Student based timetable. https://help.edupage.org/?p=u1/u3/u63
28. EduPage help. Students can select/request courses at our school. https://help.edupage.org/?p=u1/u3/u63/t1231
29. EduPage help. Test reports a problem. https://help.edupage.org/?p=u1/u3/u59/t130
30. EduPage help. Verification of timetable. https://help.edupage.org/?p=u1/u3/u60/t540
31. EduPage. Home page. https://edupage.org/
32. Techjockey. aSc TimeTables (reseller listing, undated). https://www.techjockey.com/detail/asctimetables
33. EduPage help. Language selector. https://help.edupage.org/?p=lang_id=1&p=u1/u3/u56/u610&lang_id=1
34. Untis GmbH. Imagefolder 07.2025 (PDF). https://www.untis.at/fileadmin/user_upload/Untis_Imagefolder_07.2025.pdf
35. Untis. Untis for educational institutions. https://www.untis.at/en/solutions/untis-for/educational-institutions
36. Untis. Course scheduling. https://www.untis.at/en/products/untis-basic-software/course-scheduling-1
37. Untis. Untis for all school types. https://www.untis.at/en/solutions/untis-for/untis-for-all-school-types
38. Untis. WebUntis Student. https://www.untis.at/en/products/webuntis/student
39. Untis. WebUntis Booking. https://www.untis.at/en/products/webuntis/booking
40. Untis WebHelp. Diagnosis. https://webhelp.untis.at/HTML/WebHelp/uk/untis/opdiagnose.htm
41. Untis WebHelp. Overall diagnosis. https://webhelp.untis.at/HTML/WebHelp/uk/untis/optotaldiagnose.htm
42. Untis help. Timetable diagnosis (2019). https://help.untis.at/hc/en-150/articles/360009227740-Timetable-diagnosis
43. KI-Syndikat. WebUntis (secondary source, checked June 2026). https://www.ki-syndikat.de/tools/webuntis/
