# 6. Task decomposition

An epic groups related work, a user story describes value for a hospital role, and a task names a small change to the repository. Tasks are sized so one owner can finish each in a short work session, open a branch and merge it without leaving the application broken.

## Epics and story map

| Epic | Scope | User stories |
| :--- | :---- | :----------- |
| E1 - Authentication and access | Login, account lifecycle and role enforcement. | US-01, US-15, US-16 |
| E2 - Patient management | Register, search, view, edit and deactivate patients. | US-03, US-07, US-08, US-13, US-19 |
| E3 - Doctor management | Register, update, deactivate and maintain doctor availability. | US-02, US-12, US-14 |
| E4 - Appointment scheduling | Schedule, list, update, cancel and complete appointments. | US-04, US-06, US-09, US-10, US-20 |
| E5 - Medical records and export | Record visits, read history and export authorised data. | US-05, US-11, US-17 |
| E6 - Platform and pipeline | Containers, CI/CD, tests, audit/operations visibility, monitoring and recovery. | US-18 and technical enablers T-E6.1 to T-E6.8 |

## Story dependencies

`Blocked by` describes a prerequisite that must be complete before a story can be pulled into a sprint. A dash means there is no prerequisite.

| Story | Blocked by |
| :---- | :--------- |
| US-01 - Sign in and receive role-appropriate access | - |
| US-02 - Register a doctor | US-01 |
| US-03 - Register a patient | US-01 |
| US-04 - Schedule an appointment without double-booking | US-01, US-02, US-03 |
| US-05 - Record diagnosis and visit notes | US-04 |
| US-06 - List and filter appointments | US-04 |
| US-07 - Search for and view a patient | US-03 |
| US-08 - Edit patient information | US-03 |
| US-09 - Update or cancel an appointment | US-04 |
| US-10 - Mark own appointment as completed | US-04 |
| US-11 - Read previous medical records | US-05 |
| US-12 - Update doctor details and availability | US-02 |
| US-13 - Deactivate a patient and preserve history | US-03 |
| US-14 - Deactivate a doctor and preserve history | US-02 |
| US-15 - Create a staff account and assign a role | US-01 |
| US-16 - Deactivate an account or reset its password | US-15 |
| US-17 - Export authorised data to CSV | US-01, US-11 |
| US-18 - Review security audit events | US-01 and the audit events emitted by the relevant actions |
| US-19 - View a patient's appointment statuses | US-03, US-04 |
| US-20 - Filter own appointments by status | US-04, US-06 |

## Task conventions and difficulty scale

Use task IDs in the form `T-XX.n`, where `XX` is the user story number; platform tasks use `T-E6.n`. A task should identify a specific code, test, documentation or delivery change, have one owner, and be small enough for one work session. If it cannot be completed in one sitting, split it before creating an issue. Use branch names such as `feature/US-03-patient-registration` and link the pull request to the relevant story and task.

The **task difficulty scale** below is the 1-to-10 scale supplied for this project: **1 = Easy; 10 = Difficult**. It is a quick difficulty rating for individual tasks, not an estimate in hours and not the story-point scale in [07-Estimation.md](07-Estimation.md). The patient-registration story is rated 4/10 overall; its five supplied task ratings are preserved below.

Tests are labelled by kind. **Formal / automated** tasks encode business rules or user flows as tests run in CI. **General / manual** tasks check wording, empty states, keyboard use or layout that a person can judge directly. Every story has at least one automated test task.

When a feature changes an action covered by NFR-06, its implementation emits an audit event and its automated tests verify the event's actor, action and timestamp while confirming that passwords, tokens and unnecessary clinical details are excluded.

## Detailed decomposition: highest-priority stories

### US-01 - Sign in and receive role-appropriate access

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-01.1 | Define credential, password-hash and session configuration for the API. | Code | 5 | - | Protected configuration is kept outside source control. |
| T-01.2 | Implement credential verification and the login endpoint. | Code | 5 | T-01.1 | Valid credentials create an authenticated session; invalid credentials are rejected. |
| T-01.3 | Build the login form and user-facing validation states. | Code | 3 | T-01.2 | The frontend can sign in and report invalid input clearly. |
| T-01.4 | Add deny-by-default authorization checks for protected API routes. | Code | 6 | T-01.2 | Role permissions are enforced on the server, not only hidden in the UI. |
| T-01.5 | Test valid/invalid login, unauthenticated access and role-denied API requests. | Test - formal / automated | 5 | T-01.2, T-01.4 | API tests run in CI and cover each supported role. |
| T-01.6 | Test login and role-restricted navigation in the browser. | Test - formal / automated | 4 | T-01.3, T-01.4 | Playwright verifies permitted and denied workflows. |
| T-01.7 | Check keyboard operation, visible errors and sign-in wording. | Test - general / manual | 2 | T-01.3 | A short checklist is completed and recorded. |

### US-02 - Register a doctor

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-02.1 | Add the doctor data model, required-field validation and database migration. | Code | 3 | - | Doctor name and specialisation persist in PostgreSQL. |
| T-02.2 | Implement the administrator-only doctor creation endpoint. | Code | 4 | T-02.1, T-01.4 | API rejects invalid data and non-administrator requests. |
| T-02.3 | Build the doctor registration form and doctor list view. | Code | 3 | T-02.2 | A newly saved doctor is visible to authorised staff. |
| T-02.4 | Test required fields, persistence and role restrictions at the API. | Test - formal / automated | 3 | T-02.2 | Automated API tests run in CI. |
| T-02.5 | Test the administrator create flow and denied receptionist flow. | Test - formal / automated | 3 | T-02.3, T-02.4 | Playwright covers both allowed and denied access. |
| T-02.6 | Check doctor form validation and empty-list presentation. | Test - general / manual | 2 | T-02.3 | Manual checklist completed. |

### US-03 - Register a patient

This is the patient-management task sequence provided for the project, made testable and linked to the CI delivery workflow.

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-03.1 | Design and implement the patient registration form for name, date of birth and contact details. | Code | 4 | - | **A1** from the supplied outline; required fields and validation states are visible. |
| T-03.2 | Define the patient database table/model and migration. | Code | 3 | T-03.1 | **A2**; patient data persists without replacing or deleting linked history. |
| T-03.3 | Create the API endpoint and service to add patients, emitting a safe audit event. | Code | 5 | T-03.2, T-E6.4 | **A3**; the API validates input and returns the saved patient. |
| T-03.4 | Test valid/invalid registration, persistence, audit-event content and the browser registration flow. | Test - formal / automated | 3 | T-03.3, T-03.1 | **A4**; API and Playwright tests pass in CI. |
| T-03.5 | Open the pull request, pass CI, merge to `main` and verify the published application image. | Delivery | 3 | T-03.4 | **A5**; the main-branch workflow completes and the image is available in GHCR. |

### US-04 - Schedule an appointment without double-booking

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-04.1 | Add the appointment model and migration with patient, doctor, date/time, reason and status. | Code | 4 | T-02.1, T-03.2 | Relationships and status values are constrained in the database. |
| T-04.2 | Implement the appointment service, an atomic doctor-slot conflict check and a safe audit event. | Code | 6 | T-04.1, T-E6.4 | Concurrent requests cannot reserve the same doctor's slot. |
| T-04.3 | Build the booking form with patient/doctor selection and clear validation. | Code | 4 | T-04.2 | The receptionist can submit a valid booking. |
| T-04.4 | Test conflict, non-conflict, boundary-time and concurrent booking cases. | Test - formal / automated | 5 | T-04.2 | Unit/API tests prove the rule and run in CI. |
| T-04.5 | Test a successful booking followed by a clashing booking in the browser. | Test - formal / automated | 4 | T-04.3, T-04.4 | Playwright verifies the refusal and explanation. |
| T-04.6 | Check booking validation wording and empty states. | Test - general / manual | 2 | T-04.3 | Manual checklist completed. |

### US-05 - Record diagnosis and visit notes

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-05.1 | Add the medical-record model and migration for visit date, diagnosis and notes. | Code | 4 | T-04.1 | Record links to the patient, doctor and visit. |
| T-05.2 | Implement doctor-only create/read services, API routes and safe audit events. | Code | 6 | T-05.1, T-01.4, T-E6.4 | A doctor can access only records for patients they treat. |
| T-05.3 | Build the visit-note form for the treating doctor. | Code | 3 | T-05.2 | Saved notes appear in the patient's clinical history. |
| T-05.4 | Test record persistence, relationship checks and forbidden roles. | Test - formal / automated | 5 | T-05.2 | API tests verify both allowed and denied access. |
| T-05.5 | Test recording a visit and attempting access from an unauthorised role. | Test - formal / automated | 4 | T-05.3, T-05.4 | Playwright verifies the clinical workflow and denial. |
| T-05.6 | Review clinical form labels, errors and keyboard flow. | Test - general / manual | 2 | T-05.3 | Manual checklist completed without exposing real patient data. |

### US-06 - List and filter appointments

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-06.1 | Implement appointment list queries with date/doctor filters and role scope. | Code | 4 | T-04.1, T-01.4 | Receptionist/admin see all; doctors see only their own appointments. |
| T-06.2 | Build the appointment list and filter controls. | Code | 3 | T-06.1 | Filters show matching results and a useful empty state. |
| T-06.3 | Test filter results and role-specific API visibility. | Test - formal / automated | 4 | T-06.1 | Automated API tests cover all roles and filter combinations. |
| T-06.4 | Test appointment listing and filtering in the browser. | Test - formal / automated | 3 | T-06.2, T-06.3 | Playwright verifies the core calendar workflow. |
| T-06.5 | Check list readability, keyboard focus and empty-state wording. | Test - general / manual | 2 | T-06.2 | Manual checklist completed. |

### US-07 - Search for and view a patient

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-07.1 | Implement patient name search, role-scoped administrative details and safe access-denial events. | Code | 3 | T-03.2, T-01.4, T-E6.4 | Search does not return clinical notes to receptionists. |
| T-07.2 | Build the patient search results and administrative detail view. | Code | 3 | T-07.1 | Staff can identify the intended patient. |
| T-07.3 | Test matching, no-match, role access and exclusion of clinical fields. | Test - formal / automated | 4 | T-07.1 | API tests run in CI. |
| T-07.4 | Test patient search and selection in the browser. | Test - formal / automated | 3 | T-07.2, T-07.3 | Playwright verifies search-to-detail flow. |
| T-07.5 | Check no-results wording, keyboard use and small-screen layout. | Test - general / manual | 2 | T-07.2 | Manual checklist completed. |

### US-08 - Edit patient information

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-08.1 | Implement the patient update endpoint with field validation and a safe audit event. | Code | 3 | T-03.2, T-01.4, T-E6.4 | Invalid updates are rejected and related history is retained. |
| T-08.2 | Build the patient edit form with clear save/error states. | Code | 3 | T-08.1 | An authorised receptionist can update administrative details. |
| T-08.3 | Test valid/invalid updates and preservation of linked records. | Test - formal / automated | 4 | T-08.1 | API tests run in CI. |
| T-08.4 | Test editing and saving patient details in the browser. | Test - formal / automated | 3 | T-08.2, T-08.3 | Playwright verifies the edit workflow. |
| T-08.5 | Check field errors, keyboard flow and responsive layout. | Test - general / manual | 2 | T-08.2 | Manual checklist completed. |

## Seed tasks and automated test tasks for the remaining stories

The next table reserves implementation and automated-test work for every remaining story. Refine these seed tasks into one-sitting issues before their sprint; keep the listed dependencies and preserve at least one automated test task per story.

| Story | Implementation task | Kind / difficulty | Blocked by | Automated test task | Test kind / difficulty; blocked by |
| :---- | :------------------ | :---------------- | :--------- | :------------------ | :------------------------------- |
| US-09 | T-09.1 Implement rescheduling, cancellation and conflict validation with audit events. | Code / 4 | US-04, T-E6.4 | T-09.2 Test reschedule, conflict refusal, audit event and cancelled status. | Formal / automated / 3; T-09.1 |
| US-10 | T-10.1 Add the doctor-only completed-status transition and audit event. | Code / 2 | US-04, T-E6.4 | T-10.2 Test own appointment completion, audit event and denial for another doctor's appointment. | Formal / automated / 2; T-10.1 |
| US-11 | T-11.1 Implement doctor-scoped medical-history queries and access-denial events. | Code / 4 | US-05, T-E6.4 | T-11.2 Test authorised history access, audit events and forbidden receptionist/admin access. | Formal / automated / 4; T-11.1 |
| US-12 | T-12.1 Implement administrator update of doctor specialisation and availability with an audit event. | Code / 3 | US-02, T-E6.4 | T-12.2 Test valid/invalid changes, audit event and preserved appointment links. | Formal / automated / 3; T-12.1 |
| US-13 | T-13.1 Add patient deactivation without deleting linked records and emit an audit event. | Code / 3 | US-03, T-E6.4 | T-13.2 Test inactive state, audit event and historical appointment availability. | Formal / automated / 3; T-13.1 |
| US-14 | T-14.1 Add doctor deactivation and exclude inactive doctors from new bookings. | Code / 3 | US-02, T-E6.4 | T-14.2 Test booking exclusion, audit event and preservation of historical links. | Formal / automated / 3; T-14.1 |
| US-15 | T-15.1 Implement account creation, supported-role assignment and an audit event. | Code / 4 | US-01, T-E6.4 | T-15.2 Test account creation, role assignment, audit event and role denial. | Formal / automated / 4; T-15.1 |
| US-16 | T-16.1 Implement account deactivation and a non-enumerating password-reset flow with audit events. | Code / 5 | US-15, T-E6.4 | T-16.2 Test disabled login, reset response, audit event and password handling. | Formal / automated / 5; T-16.1 |
| US-17 | T-17.1 Implement CSV export limited to the doctor's authorised patients and emit an audit event. | Code / 4 | US-11, T-E6.4 | T-17.2 Test CSV contents, role restriction and audit event. | Formal / automated / 4; T-17.1 |
| US-18 | T-18.1 Implement an administrator audit-event view without clinical content. | Code / 4 | US-01, T-E6.4 | T-18.2 Test actor/action/timestamp fields and sensitive-data exclusion. | Formal / automated / 4; T-18.1 |
| US-19 | T-19.1 Add the receptionist view of a patient's upcoming and past appointment statuses. | Code / 3 | US-03, US-04 | T-19.2 Test date/status results and absence of clinical notes. | Formal / automated / 3; T-19.1 |
| US-20 | T-20.1 Add a status filter scoped to the logged-in doctor's own appointments. | Code / 3 | US-04, US-06 | T-20.2 Test each status and denial of other doctors' appointments. | Formal / automated / 3; T-20.1 |

## E6 - Platform and pipeline tasks

These technical enablers support the application and are estimated alongside user stories in [07-Estimation.md](07-Estimation.md).

| Task ID | Task / repository change | Kind | Difficulty (1-10) | Blocked by | Verification / result |
| :------ | :----------------------- | :--- | :---------------: | :--------- | :-------------------- |
| T-E6.1 | Add frontend/API Dockerfiles and a Docker Compose development environment. | Code | 4 | - | A clean checkout starts the services using documented commands. |
| T-E6.2 | Configure GitHub Actions on pull requests for checks, tests and a container build. | Code / automated | 5 | T-E6.1 | Failed checks block the merge; results are visible on the pull request. |
| T-E6.3 | Build and publish the main-branch container image to GHCR with version tags. | Code / delivery | 4 | T-E6.2 | The published image can be traced to its source commit and release tag. |
| T-E6.4 | Add a shared audit-event model/writer; instrument login, doctor registration and role-denial actions, and provide the writer for later sensitive-action tasks. | Code | 5 | T-01.4 | Events include actor, action and timestamp while excluding credentials, tokens and clinical payloads. |
| T-E6.5 | Add service health checks and structured operational logs that exclude secrets and unnecessary patient details. | Code | 4 | T-E6.1 | Health state and request diagnostics are available without sensitive data. |
| T-E6.6 | Create a dashboard for service health, latency, error rates and container restarts. | Code / operations | 5 | T-E6.5 | The dashboard displays the agreed operational signals. |
| T-E6.7 | Configure encrypted database backups and document and verify a restore. | Code / operations | 4 | T-E6.1, T-03.2 | A backup can be restored using the documented procedure. |
| T-E6.8 | Add CI quality gates for role security, performance and accessibility checks. | Code / automated | 5 | T-E6.2, T-01.5, T-04.4 | The pipeline reports failures for the agreed NFR checks. |
