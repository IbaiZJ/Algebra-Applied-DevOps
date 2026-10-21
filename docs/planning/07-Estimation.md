# 7. Estimation

## Story-point scale

Story points compare the relative effort of stories; they are not hours or days. Estimates consider the amount of work, technical complexity, uncertainty, dependencies and testing effort. The scale follows the values used in the lab:

| Points | Meaning | Calibration |
| :----: | :------ | :---------- |
| 1 | Trivial | Understood change, one or two files and a small automated test. |
| 2 | Small | Routine change to an established pattern, with limited risk. |
| 3 | Standard | Several small tasks and tests, without major unknowns. **US-03, patient registration, is the 3-point anchor.** |
| 5 | Substantial | Multiple layers or a business rule that needs careful testing. |
| 8 | Large or unfamiliar | High uncertainty, security-sensitive work or broad role enforcement. |
| 13 | Too large to plan | Split the story before it enters a sprint; do not treat 13 as a normal estimate. |

The 1-to-10 task-difficulty values in [06-Task-decomposition.md](06-Task-decomposition.md) are a separate scale supplied for individual tasks. They are not added to story points.

## Product-backlog estimates

| ID | User story | Points | Rationale |
| :-- | :---------- | :----: | :-------- |
| US-01 | Sign in and receive role-appropriate access | 8 | Authentication, secure sessions and deny-by-default permissions have high uncertainty and affect every protected route. |
| US-02 | Register a doctor | 2 | A small create flow for a simple entity, once the application and authorization pattern exist. |
| US-03 | Register a patient | 3 | Anchor: one form, persisted entity, validation and a focused test path. |
| US-04 | Schedule an appointment without double-booking | 5 | Related records plus an atomic business rule and conflict/boundary tests. |
| US-05 | Record diagnosis and visit notes | 3 | A standard create flow with a clinical-access check and persistence tests. |
| US-06 | List and filter appointments | 2 | Read-only filters, with role scoping and a manageable set of query tests. |
| US-07 | Search for and view a patient | 2 | A name query and a limited administrative detail view. |
| US-08 | Edit patient information | 2 | Routine update flow with validation and a linked-history regression check. |
| US-09 | Update or cancel an appointment | 2 | Extend the existing appointment workflow with status and reschedule rules. |
| US-10 | Mark own appointment as completed | 1 | A small status transition, provided the ownership check is reused. |
| US-11 | Read previous medical records | 3 | Clinical history query with strict doctor-to-patient authorization. |
| US-12 | Update doctor details and availability | 2 | Routine edit flow with validation of availability. |
| US-13 | Deactivate a patient and preserve history | 1 | A small state change with a regression test for linked records. |
| US-14 | Deactivate a doctor and preserve history | 1 | A small state change plus exclusion from future booking choices. |
| US-15 | Create a staff account and assign a role | 3 | Account creation depends on the authentication model and needs role-assignment tests. |
| US-16 | Deactivate an account or reset its password | 3 | Account-state changes and reset behavior require security-focused tests. |
| US-17 | Export authorised data to CSV | 3 | Output generation is routine; enforcing the patient's authorization scope adds risk. |
| US-18 | Review security audit events | 3 | A limited operations view with sensitive-field filtering and access tests. |
| US-19 | View a patient's appointment statuses | 1 | A scoped view assembled from existing appointment data. |
| US-20 | Filter own appointments by status | 1 | A small filter that reuses the existing doctor-scoped appointment list. |
| **Story subtotal** | **20 user stories** | **51** | **All stories are below 13 points.** |

## Platform and pipeline estimates

The platform work is tracked under epic E6. These points count toward the sprint forecast just like story points; the pipeline is part of the deliverable, not free work.

| Task ID | Enabler | Points | Rationale |
| :------ | :------ | :----: | :-------- |
| T-E6.1 | Containerize the frontend/API and provide Docker Compose development setup | 2 | A known stack, with some setup and configuration work. |
| T-E6.2 | Run automated checks, tests and a container build in CI on pull requests | 2 | A short workflow using the existing project commands. |
| T-E6.3 | Publish versioned main-branch images to GHCR | 2 | Extend the CI workflow with registry authentication and release tags. |
| T-E6.4 | Add a shared, privacy-safe audit-event model and writer | 3 | A reusable event structure plus initial instrumentation and tests for sensitive-field exclusion. |
| T-E6.5 | Add health checks and privacy-safe structured operational logging | 2 | Small operational additions once the container setup exists. |
| T-E6.6 | Build a dashboard for health, latency, errors and restarts | 3 | Several operational signals and a dashboard configuration. |
| T-E6.7 | Configure encrypted backups and verify the restore procedure | 2 | A bounded database-operations task with a documented recovery check. |
| T-E6.8 | Add role-security, performance and accessibility quality gates to CI | 3 | Integrate measurable NFR checks with the automated pipeline. |
| **Enabler subtotal** | **8 technical enablers** | **19** | |
| **Planned delivery total** | **Stories and enablers** | **70** | **51 story points + 19 enabler points.** |

## Capacity and re-estimation

There is no completed-sprint velocity yet. Sprint 1 starts with a provisional capacity of **14 points**. At each Sprint Review, count only the story and enabler points that meet the Definition of Done, then use that completed total as the next sprint's evidence-based capacity. The sprint totals in [08-Sprint-backlogs.md](08-Sprint-backlogs.md) are forecasts; if actual velocity is lower, return the lowest-priority unfinished work to the product backlog and re-order it rather than silently carrying it forward.

Re-estimate a story when refinement reveals a new dependency, an unfamiliar technical constraint or missing acceptance criteria. Split any story that reaches 13 points before it is selected for a sprint.
