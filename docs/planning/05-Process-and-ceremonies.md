## 5. Process and ceremonies

### Working model and sprint cadence

The project uses four consecutive two-week sprints. Two weeks gives enough time to complete a small end-to-end increment while leaving regular opportunities to inspect progress and adjust the plan during the semester. One student performs the Product Owner, Developer and Scrum Master responsibilities as separate working roles.

| Sprint | Goal | Planned scope and increment |
| :----- | :--- | :-------------------------- |
| Sprint 1 - Walking skeleton | Establish a usable path from login to persisted data. | US-01 and US-02; create the application structure, database connection and doctor registration flow; establish branch and pull-request conventions; configure initial CI and build the container. |
| Sprint 2 - Core records | Complete patient and doctor administration and publish the image. | US-03, US-07, US-08, US-12 to US-15; preserve linked history when records are deactivated; publish the container image to GHCR from the main-branch pipeline and add the shared audit-event writer. |
| Sprint 3 - Appointments and clinical records | Connect scheduling to the visit workflow. | US-04 to US-06 and US-09 to US-11; demonstrate appointment conflict prevention, appointment status changes and medical history access; add service health/logging and establish the backup/restore procedure. |
| Sprint 4 - Harden and observe | Verify the complete product and prepare a reliable demonstration. | US-16 to US-20, end-to-end and security checks, performance checks against the stated NFR targets, browser and accessibility checks, HTTPS and secure configuration review, backup/restore verification, monitoring dashboard and final documentation. |

The sprint backlog is chosen from the ordered product backlog based on dependencies and the sprint goal. Work is broken into tasks on the project board, and each task carries a Definition of Done. Unfinished work returns to the product backlog for re-estimation and re-ordering; it is not silently carried into the next sprint.

### Scrum ceremonies

| Ceremony | When and duration | What happens | Output |
| :-------- | :---------------- | :----------- | :----- |
| Sprint Planning | First day of each sprint; 45 minutes. | Review the ordered backlog, confirm available time and dependencies, select stories that fit the sprint, and define a focused sprint goal. | A sprint goal and sprint backlog with tasks and acceptance criteria. |
| Sprint Review | Last day of each sprint; about 30 minutes. | Run the working increment and demonstrate completed stories to a classmate or other stakeholder. Collect feedback against the acceptance criteria. | Demonstration notes and any feedback that changes backlog ordering or story detail. |
| Retrospective | Immediately after the review; 20 minutes. | Reflect on what helped, what slowed work down and one change to try in the next sprint. | One concrete improvement action, recorded in the repository with an owner and target sprint. |

### Solo coordination

Because the project is currently managed by one student, it does not claim to hold a daily team Scrum. After each work session, add a dated entry to a repository work log with **Done**, **Next** and **Blockers**. This gives a short record of progress and makes impediments visible. If the project becomes a team effort, hold a brief daily Scrum to coordinate work and surface blockers.

### Product backlog refinement

Refinement is continuous, with roughly 10% of sprint capacity reserved to keep the top of the backlog ready for planning. Review and update stories when any of these triggers occurs:

1. A new or changed stakeholder requirement is not represented in the backlog.
2. A story is too large to complete within one sprint and needs to be split.
3. A story is missing acceptance criteria or cannot be tested clearly.
4. A story spills over; re-estimate and re-order it rather than carrying it forward without review.
5. Implementation reveals a technical constraint, risk or dependency that changes the estimate or solution scope.
6. Sprint Review feedback changes the value or expected behavior of a story.

Refinement produces smaller, testable stories with named roles, clear benefits, acceptance criteria, requirement references and updated MoSCoW priority. The top of the backlog should always be ready to pull into the next sprint.

### Engineering workflow and delivery

- Create a short-lived branch for each story or fix, using `feature/US-XX-short-name` or `fix/US-XX-short-name`.
- Open a pull request into `main` and link it to the backlog item. Direct pushes to `main` are not part of the workflow.
- Run CI on every pull request. The checks include code validation, unit and API integration tests, affected Playwright end-to-end tests, and a container build.
- Merge only after the acceptance criteria are reviewed and required checks pass. A successful change on `main` builds and publishes the application image to GHCR.
- Use semantic version tags in the form `vMAJOR.MINOR.PATCH` for release images, and retain the source commit identifier for traceability.
- Monitor service health, request latency, error rates and container restarts through health checks, structured logs and a dashboard. Logs must not contain passwords, tokens or unnecessary patient information.

This workflow applies the Definition of Done in [04-Product-backlog.md](04-Product-backlog.md) to every backlog item.
