# Lab 3: Project management with GitHub Projects

Use the results of **Lab 2** as the starting point for this lab.

In Lab 2, you decomposed the requirements into Epics, User Stories and Tasks, estimated them with Story Points and planned the sprints inside your `README.md`. In this lab, you will move that plan from a static document into a **GitHub Project** connected to your repository, so that the work can be tracked, prioritized, linked to code and visualized during development.

The results of this lab will be used in the following sessions when creating the CI/CD workflow, automated tests and other integration/delivery/monitoring activities. Every issue you create now will later be linked to a branch, a pull request and a pipeline run.

> **Prerequisites:** the public GitHub repository from Lab 1 and the completed task decomposition (Epics, User Stories, Tasks, Story Points, sprint outline) from Lab 2. If you did not finish Lab 2, do it before proceeding.

All examples in this document use a made-up project, **LendIt** (a small web app for lending items between friends). Use them only as a pattern. Your own project must use your own backlog.

---

## Task 1: Import the task decomposition into a GitHub Project

### 1.1 Create the project

1. Open your profile (or organization) and go to **Projects → New project**.
2. Choose the **Table** template (you will add a Board view in Task 4) and name the project after your repository, e.g. `LendIt – Product Backlog`.
3. Link the project to your repository: open the repository, go to **Projects → Link a project** and select the project you just created.
4. Add your team members under the project's **Settings → Manage access**.

### 1.2 Mark the hierarchy: Epic, User Story, Task

Every item from your decomposition becomes a **GitHub Issue** in your repository. It must be clearly labelled as one of three levels.

- **Personal repositories:** create three labels (**Issues → Labels → New label**): `epic`, `user-story`, `task`. Apply exactly one to every issue.
- **Organization repositories:** you can use **Issue types** (Settings → Issue types) instead of labels. Create `Epic`, `User Story` and `Task`, and set the type on each issue. Labels are still fine if you prefer them.

Also name every issue with its identifier from Lab 2, so it stays traceable:

| Level | Title example | Label |
|-------|---------------|-------|
| Epic | `E1 – User management` | `epic` |
| User Story | `US-1.1 register-user` | `user-story` |
| Task | `T-1.1.2 Implement POST /register endpoint` | `task` |

### 1.3 Reproduce the hierarchy with sub-issues

Use GitHub's native **sub-issues** so the hierarchy is visible, not only written in titles:

1. Create the Epic issue.
2. Open it and use **Create sub-issue** (or **Add existing issue**) to add its User Stories.
3. Open each User Story and add its Tasks the same way.

The result is a tree: `Epic → User Story → Task`. In the project table, show the **Parent issue** and **Sub-issues progress** columns (use **+** at the right of the table header) so you can see how much of each Story is done.

### 1.4 Definition of Done and testing tasks

Every **User Story** issue must contain, in its description, a clear **Definition of Done (DoD)** and acceptance criteria, and must have **at least one testing task** among its sub-issues.

You can enforce this by adding an issue template to your repository. Create `.github/ISSUE_TEMPLATE/user-story.md`:

```markdown
---
name: User Story
about: A user-facing piece of functionality
labels: user-story
---

## Story
As a <role>, I want <goal>, so that <benefit>.

## Acceptance criteria
- [ ] Given ... when ... then ...
- [ ] ...

## Definition of Done
- [ ] All acceptance criteria are met
- [ ] Code is merged to `main` through a reviewed pull request
- [ ] Automated tests for this story are written and passing
- [ ] Manual test case (if any) is executed and documented
- [ ] Documentation / README updated where relevant

## Dependencies
Blocked by: #<issue number> (set it in the Relationships panel as well)

## Story Points
<value from the Lab 2 scale>
```

And a lighter template for tasks, `.github/ISSUE_TEMPLATE/task.md`:

```markdown
---
name: Task
about: An implementable unit of work within a User Story
labels: task
---

## Description
...

## Parent user story
#<issue number>

## Done when
- [ ] ...
```

Testing tasks are normal `task` issues. Keep them identifiable in the title, for example:

- `T-1.1.4 [TEST][auto] Unit tests for registration validation`
- `T-1.1.5 [TEST][auto] Integration test for POST /register`
- `T-1.1.6 [TEST][manual] Register via UI, duplicate e-mail`

### 1.5 Add the issues to the project

Issues must be in the project to appear on its views:

- **Single items:** in the project table, click **+ Add item** at the bottom and search for the issue.
- **Automatically:** open the project's **⋯ → Workflows → Auto-add to project**, select your repository and use a filter such as `is:issue is:open`. New issues from the repository are then added automatically.

> **Optional (CLI):** if you have many issues, you can script the creation with the GitHub CLI.
>
> ```bash
> gh auth refresh -s project
> gh issue create --title "US-1.1 register-user" --label user-story --body-file us-1-1.md
> gh project item-add <PROJECT_NUMBER> --owner "@me" --url <ISSUE_URL>
> ```

### Expected result of Task 1

- All Epics, User Stories and Tasks from Lab 2 exist as issues in the project.
- Each issue is labelled (or typed) as `epic`, `user-story` or `task`.
- The hierarchy is represented with sub-issues.
- Each User Story has a Definition of Done and at least one testing task.

---

## Task 2: Dependencies, priority and Story Points

Now add the planning data to the project. All of it should be visible in the project **table view**, so you can sort, filter and group by it.

### 2.1 Add custom fields

In the table view, click **+** at the right of the header and choose **New field**:

| Field | Type | Values |
|-------|------|--------|
| `Story Points` | Number | Same scale as in Lab 2 (e.g. 1, 2, 3, 5, 8, 13) |
| `Priority` | Single select | `P0 – Critical`, `P1 – High`, `P2 – Medium`, `P3 – Low` |
| `Sprint` | Iteration | Duration equal to the sprint length defined in Lab 2 |

> If the template you selected already contains `Priority` or `Size`/`Estimate` fields, you can reuse them, but Story Points must be a **number** field so that sums can be calculated.

Fill in the values for every User Story (Story Points as estimated in Lab 2). Assign each story to its sprint using the `Sprint` field, following the sprint outline from Lab 2.

> Story Points and Priority are different things. **Story Points** describe the size and complexity of the work; **Priority** describes how soon it is needed. A small story can be critical, and a large one can be low priority.

### 2.2 Define dependencies

Use the native **issue dependencies** (do not only write them in the description):

1. Open the issue that cannot start before another one is finished.
2. In the right sidebar, click **Relationships**.
3. Select **Mark as blocked by** and choose the issue(s) it depends on. (You can also do it from the other side with **Mark as blocking**.)

Blocked issues are shown with a **Blocked** icon on the project views, so bottlenecks are easy to spot. Use dependencies at both levels you defined in Lab 2: between User Stories (`US-1.2 login-user` is blocked by `US-1.1 register-user`) and between Tasks (`T-1.1.2 endpoint` is blocked by `T-1.1.1 migration`).

Useful search filters in the project and in the repository Issues tab:

```text
is:blocked
is:blocking
blocked-by:12
blocking:12
```

CLI alternative:

```bash
gh issue edit 14 --add-blocked-by 12
gh issue view 14 --json blockedBy,blocking
```

### 2.3 Create views for planning

Create at least these two table views (use **+ New view** next to the existing view tabs, and remember to **save** the view changes):

- **Backlog by priority:** sorted by `Priority`, filtered with `label:user-story`, with the fields `Story Points`, `Priority`, `Sprint` visible.
- **Sprint plan:** grouped by `Sprint`, with the **sum** of `Story Points` shown per group (group-by header → field sum). Check that the total of each sprint is consistent with the plan from Lab 2.

### Expected result of Task 2

- Every User Story has Story Points, Priority and a Sprint.
- Dependencies between stories and between tasks are defined with the Relationships panel.
- The two planning views exist and show the correct data.

---

## Task 3: Linking issues to branches

From the development phase onward, **every implementation task is done on its own branch that is linked to its issue**. This link is what lets GitHub show the branch and pull request on the issue, close the issue automatically on merge, and lets your pipeline identify what a change belongs to.

### 3.1 Branch naming convention

Agree on a convention within the team and write it into `README.md` (or `CONTRIBUTING.md`). A suggestion:

```text
<issue-number>-<story-or-task-id>-<short-description>

12-t-1-1-2-register-endpoint
15-us-1-2-login-user
```

Rules:

- one branch per Task (or per small User Story), always created from an up-to-date `main`;
- the issue number is the first element of the name, so it can be parsed by automation later;
- never commit directly to `main`.

### 3.2 Create the branch from the issue (GitHub UI)

1. Open the issue (a Task, or a User Story without sub-tasks) from the project board or table.
2. In the right sidebar, find the **Development** section and click **Create a branch**.
3. In the dialog, adjust the **branch name** to follow your convention, choose the **source branch** (normally `main`) and select **Create branch** (the repository destination should be your repository).
4. The branch is created **on the remote repository** and appears in the issue's Development section, so the issue and the branch are now linked.

### 3.3 Get the branch locally (git)

The branch now exists only on GitHub. Fetch it and start working on it locally:

```bash
git fetch origin
git switch 12-t-1-1-2-register-endpoint
```

`git switch` sees that the branch exists on `origin` and creates a local branch that tracks it. The older equivalent is:

```bash
git checkout -b 12-t-1-1-2-register-endpoint origin/12-t-1-1-2-register-endpoint
```

Then work as usual:

```bash
git add .
git commit -m "Add POST /register endpoint (#12)"
git push
```

Referencing the issue number (`#12`) in commit messages makes the commit appear on the issue timeline.

### 3.4 Alternative: do it all from the terminal with GitHub CLI

```bash
gh issue develop 12 --name 12-t-1-1-2-register-endpoint --base main --checkout
```

This creates the branch on the remote, **links it to issue #12** and checks it out locally in a single command. To list the branches linked to an issue:

```bash
gh issue develop --list 12
```

> A branch created only with `git checkout -b` and pushed manually is **not** linked to the issue. Either create it through the issue (3.2 or 3.4), or link it afterwards by opening a pull request that references the issue.

### 3.5 Pull request and closing the issue

When the work is ready:

```bash
git push
```

Open a pull request into `main`. In the description, use a closing keyword with the issue number:

```text
Closes #12
```

When the pull request is merged, issue #12 is closed automatically and the item moves to **Done** in the project (see Task 4). The pull request is also shown in the issue's Development section and in the project's **Linked pull requests** field.

### 3.6 Why this matters for the pipeline

In the upcoming CI/CD lab, the issue number in the branch name and in the pull request will be used to identify the work item, trigger workflows only for relevant branches (e.g. `on: pull_request`) and connect build and test results back to the task. Keep the convention consistent from now on.

### Expected result of Task 3

- The branch naming convention is documented in the repository.
- At least one Task has a branch created from the issue, visible in its **Development** section.
- You can describe how the same branch is checked out locally and how it is linked to its pull request.

---

## Task 4: Kanban board view

Create a **Board** view of your project that visualizes the flow of work.

### 4.1 Define the workflow columns

The columns of a board are the options of a single select field, usually `Status`. Edit its options so that your board has **at least four columns** that reflect your team's real workflow. For example:

| Column | Meaning | Suggested column limit |
|--------|---------|------------------------|
| `Backlog` | Not yet planned for a sprint | none |
| `Ready` | Planned for the current sprint and no longer blocked | none |
| `In Progress` | A branch exists and someone is working on it | 3 |
| `In Review` | Pull request is open, waiting for review and tests | 3 |
| `Done` | Merged and Definition of Done fulfilled | none |

To edit the options, open the project's table view, click the `Status` column header → **Edit field**, or use the **+** on the right of the board.

### 4.2 Create the Board view

1. Click **+ New view** next to the view tabs and choose **Board**.
2. Set the **Column field** to `Status`.
3. Name the view `Kanban – Sprint`, and filter it to the current sprint, e.g. `sprint:@current`.
4. Show on the cards: `Story Points`, `Priority`, assignees and the labels.
5. Optionally group the cards into **swimlanes** by `Parent issue` or assignees.

### 4.3 Set column limits (WIP limits)

Kanban limits **work in progress** to avoid starting more than the team can finish.

1. In the board, open the menu (⋯) at the top of a column.
2. Select **Set column limit** and enter the number.
3. The current count and the limit are shown at the top of the column and are highlighted when exceeded.

Note that the limit is only a visual warning. GitHub does not stop anyone from adding cards beyond it, so the team must respect it. Display of **field sums** at the top of each column also shows the total Story Points per column.

### 4.4 Automate the status changes

In the project, go to **⋯ → Workflows** and enable:

- **Item added to project** → set `Status` to `Backlog`;
- **Item closed** → set `Status` to `Done`;
- **Pull request merged** → set `Status` to `Done`.

Other statuses (`Ready`, `In Progress`, `In Review`) are moved by hand by the team, by dragging cards between columns.

### Expected result of Task 4

- A saved Board view with at least four columns and column limits on the "work in progress" columns.
- Cards display Story Points and Priority.
- Automatic workflows move closed or merged items to `Done`.

---

## Final deliverables checklist

- [ ] GitHub Project linked to the repository
- [ ] All Epics, User Stories and Tasks created as issues, labelled/typed and connected through sub-issues
- [ ] Each User Story has a Definition of Done and at least one testing task
- [ ] Fields `Story Points`, `Priority` and `Sprint` filled in for all User Stories
- [ ] Dependencies between stories and tasks defined through **Relationships**
- [ ] Branch naming convention documented, at least one branch created from an issue
- [ ] Kanban board with at least four columns and column limits
- [ ] Link to the GitHub Project added to `README.md`

---

## Useful resources

- [GitHub Docs – Planning and tracking with Projects](https://docs.github.com/en/issues/planning-and-tracking-with-projects)
- [GitHub Docs – Creating issue dependencies](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/creating-issue-dependencies)
- [GitHub Docs – Customizing the board layout](https://docs.github.com/en/issues/planning-and-tracking-with-projects/customizing-views-in-your-project/customizing-the-board-layout)
- [GitHub CLI – `gh issue develop`](https://cli.github.com/manual/gh_issue_develop)