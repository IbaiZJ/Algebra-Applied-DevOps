# 8. Sprint backlogs

The project is planned as four two-week sprints, following [05-Process-and-ceremonies.md](05-Process-and-ceremonies.md). Each sprint has one goal and an ordered set of stories and platform tasks. Points include both product stories and E6 enablers, as described in [07-Estimation.md](07-Estimation.md). A sprint backlog is a forecast that is reviewed against actual capacity at each Sprint Review.

## Sprint overview

| Sprint | Goal | Stories | Enablers | Story points | Enabler points | Total |
| :----- | :--- | :------ | :------- | :----------: | :------------: | :---: |
| 1 - Walking skeleton | Sign in, register a doctor and build/test the containerized application. | US-01, US-02 | T-E6.1, T-E6.2 | 10 | 4 | **14** |
| 2 - Core records | Complete patient and doctor administration and publish the image. | US-03, US-07, US-08, US-12, US-13, US-14, US-15 | T-E6.3, T-E6.4 | 14 | 5 | **19** |
| 3 - Appointments and clinical records | Deliver the appointment-to-visit flow with reliable data recovery. | US-04, US-05, US-06, US-09, US-10, US-11 | T-E6.5, T-E6.7 | 16 | 4 | **20** |
| 4 - Harden and observe | Complete account recovery, authorised export, audit visibility and operational checks. | US-16, US-17, US-18, US-19, US-20 | T-E6.6, T-E6.8 | 11 | 6 | **17** |
| **Total** |  | **20 stories** | **8 enablers** | **51** | **19** | **70** |

The first sprint's 14-point forecast is the initial capacity assumption. Later totals are provisional until actual velocity is known. US-19 and US-20 are **Could** stories; if a sprint forecast must be reduced, defer them before any Must story, security check or Definition of Done item.

## Sprint 1 - Walking skeleton

**Sprint goal:** A staff member can sign in, an administrator can register a doctor, and the project builds and runs through its first automated pipeline.

| Backlog item | Points | Planned tasks and increment |
| :----------- | :----: | :-------------------------- |
| US-01 - Sign in and receive role-appropriate access | 8 | T-01.1 to T-01.7: credential/session setup, login endpoint and form, deny-by-default route checks, automated API and browser tests, and manual keyboard/error review. |
| US-02 - Register a doctor | 2 | T-02.1 to T-02.6: doctor model/migration, administrator-only API, form/list, automated authorization tests and manual validation review. |
| T-E6.1 - Containerized development setup | 2 | Add Dockerfiles and Docker Compose; confirm the documented local startup path. |
| T-E6.2 - Pull-request CI baseline | 2 | Run code checks, available tests and a container build on every pull request. |
| **Sprint total** | **14** | **Story subtotal: 10; enabler subtotal: 4.** |

## Sprint 2 - Core records

**Sprint goal:** Receptionists can manage patient information, administrators can maintain doctors and staff accounts, and the main-branch pipeline publishes a traceable image.

| Backlog item | Points | Planned tasks and increment |
| :----------- | :----: | :-------------------------- |
| T-E6.4 - Shared audit-event writer | 3 | Provide the event structure for actor/action/timestamp, instrument login and doctor registration, and exclude credentials, tokens and clinical payloads. |
| US-03 - Register a patient | 3 | T-03.1 to T-03.5: design/build form, patient table/migration, API endpoint, automated tests (including audit-event checks), then merge through CI and verify the main-branch image. |
| US-07 - Search for and view a patient | 2 | T-07.1 to T-07.5: scoped search API, administrative detail UI, automated access/data tests and manual no-results/accessibility check. |
| US-08 - Edit patient information | 2 | T-08.1 to T-08.5: validated update API, edit form, automated history-preservation tests and manual form check. |
| US-12 - Update doctor details and availability | 2 | T-12.1 and T-12.2: administrator update flow plus automated validation and relationship tests. |
| US-13 - Deactivate a patient and preserve history | 1 | T-13.1 and T-13.2: soft deactivation and automated historical-link checks. |
| US-14 - Deactivate a doctor and preserve history | 1 | T-14.1 and T-14.2: prevent new bookings and verify historical links with automated tests. |
| US-15 - Create a staff account and assign a role | 3 | T-15.1 and T-15.2: account/role creation and automated role-assignment/access tests. |
| T-E6.3 - Publish versioned image to GHCR | 2 | Build and publish from `main`, tag releases with `vMAJOR.MINOR.PATCH` and record the source commit. |
| **Sprint total** | **19** | **Story subtotal: 14; enabler subtotal: 5.** |

## Sprint 3 - Appointments and clinical records

**Sprint goal:** A receptionist can schedule and maintain appointments without double-booking; doctors can complete visits and manage clinical notes and history.

| Backlog item | Points | Planned tasks and increment |
| :----------- | :----: | :-------------------------- |
| US-04 - Schedule an appointment without double-booking | 5 | T-04.1 to T-04.6: appointment model, atomic conflict rule, booking form, automated boundary/concurrency and browser tests, and manual validation review. |
| US-05 - Record diagnosis and visit notes | 3 | T-05.1 to T-05.6: clinical record schema, doctor-only API and form, access/persistence tests, and manual usability review. |
| US-06 - List and filter appointments | 2 | T-06.1 to T-06.5: role-scoped list query, filters, automated API/browser coverage and manual empty-state check. |
| US-09 - Update or cancel an appointment | 2 | T-09.1 and T-09.2: reschedule/cancel behavior and automated conflict/status tests. |
| US-10 - Mark own appointment as completed | 1 | T-10.1 and T-10.2: doctor-only completion and automated ownership/status tests. |
| US-11 - Read previous medical records | 3 | T-11.1 and T-11.2: doctor-scoped history view and automated allowed/denied access tests. |
| T-E6.5 - Health checks and privacy-safe logs | 2 | Expose service health and structured operational logs without credentials or unnecessary patient data. |
| T-E6.7 - Encrypted backup and verified restore | 2 | Configure database backup, document the restore procedure and verify it against the project database. |
| **Sprint total** | **20** | **Story subtotal: 16; enabler subtotal: 4.** |

## Sprint 4 - Harden and observe

**Sprint goal:** Complete secure account recovery and authorised export, expose audit/operational information safely, and verify the release against the non-functional requirements.

| Backlog item | Points | Planned tasks and increment |
| :----------- | :----: | :-------------------------- |
| US-16 - Deactivate an account or reset its password | 3 | T-16.1 and T-16.2: secure account-state/reset flow and automated disabled-login/non-enumeration tests. |
| US-17 - Export authorised data to CSV | 3 | T-17.1 and T-17.2: doctor-scoped export with automated CSV-content, permission and audit tests. |
| US-18 - Review security audit events | 3 | T-18.1 and T-18.2: restricted audit view and automated checks for required fields and sensitive-data exclusion. |
| US-19 - View a patient's appointment statuses | 1 | T-19.1 and T-19.2: upcoming/past status view with automated privacy and result tests. **Could** priority; defer if capacity is short. |
| US-20 - Filter own appointments by status | 1 | T-20.1 and T-20.2: doctor-scoped status filter with automated cross-doctor denial tests. **Could** priority; defer if capacity is short. |
| T-E6.6 - Monitoring dashboard | 3 | Display health, latency, error rates and container restarts. |
| T-E6.8 - NFR quality gates | 3 | Add automated role-security and performance checks plus accessibility checks for core workflows to CI. |
| **Sprint total** | **17** | **Story subtotal: 11; enabler subtotal: 6.** |

## Sprint selection and review rules

- Select only stories whose dependencies in [06-Task-decomposition.md](06-Task-decomposition.md) are complete or included earlier in the same sprint.
- Each story must have acceptance criteria, a point estimate, a test task and a clear Definition of Done before planning.
- Create GitHub issues from the task IDs, link each issue to its story, and use the sprint, epic, type, status and story-point fields on the project board.
- During the review, count only completed points when updating velocity. If actual capacity is below the next forecast, re-order the backlog and defer Could work first; do not skip automated tests or security and privacy checks.
- Follow the Definition of Done in [04-Product-backlog.md](04-Product-backlog.md) for every story and enabler.
