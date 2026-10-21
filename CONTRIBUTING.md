# Contributing

Thanks for contributing to the Hospital Management project. This guide describes the expected workflow for changes.

## Before you start

- Check the README and relevant planning documents in `docs/` to understand the requirements and role-based access rules.
- For a bug, feature, or task, open or find a GitHub Issue first. Describe the problem or goal and link related work.
- Ask in the issue before making a change that alters a requirement or access-control decision.

## Make a change

1. Create a branch from the latest `main`. Use a short, descriptive name, such as `feat/patient-search` or `fix/appointment-conflict`.
2. Keep the change focused on the issue. Follow the existing code and documentation style.
3. Add or update tests for changed behaviour, especially permissions and sensitive patient data. Run the relevant checks available in the project and record their results. If you cannot run them, explain why and include any manual checks performed.
4. Update the README or planning documents when the change affects setup, requirements, or project decisions.

Do not commit credentials, secrets, or real patient data. Use synthetic data in examples and tests.

## Commits and pull requests

- Use clear commit subjects; Conventional Commit style is encouraged, for example `feat(appointments): reject clashing bookings`.
- Push your branch and open a pull request against `main`. Do not push changes directly to `main`.
- Fill in the pull request template: link the issue, summarize the change, and describe how it was tested.
- Address review feedback and make sure the required checks pass before merging.

## Reporting issues

Use the matching GitHub Issue template for bugs, user stories, epics, or tasks. Include enough detail to reproduce a bug, while avoiding secrets and personal or medical information.
