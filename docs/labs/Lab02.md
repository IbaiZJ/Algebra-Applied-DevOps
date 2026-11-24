# Lab 2: Task decomposition and estimation

Use the results of **Lab 1** as the starting point for this lab.

In [Lab 1](
https://vle.algebra.hr/pluginfile.php/58485/mod_resource/content/1/Lab01.md), you analyzed the requirements and created the public GitHub repository.


In this lab, you will take selected items from your Product Backlog and decompose them into smaller, implementable tasks that can be assigned to team members and tracked during development.

The results of this lab will be used in the following sessions when creating the project's CI/CD workflow, automated tests, and other integration/delivery/monitoring activities**.

To start, use the project scenario from Moodle and your completed `README.md` from Lab 1 (if you didn't make the requirement analysis, do it now before proceeding further). and perform task decomposition in the same file (for now, in the exercise you will define them on the Github Projects).

Exercise material should help you define the expected project structure.

## Task 1

Using the analyzed requirements from the previous exercises, perform the task decomposition, in respect to the following hierarchy:

```text
Epic
 └── User Story
      ├── Task
      ├── Task
      ├── Task
      └── Task
```

Each user story should have a clearly defined short name/identifier. Later on user stories/tasks will be connected to specific branches. **Indicate which user stories and which tasks have dependencies on other user stories and tasks respectively.**

Each User Story also needs to have the appropriate Tasks related to testing. See if it would make sense to require specific formal (code) tests or general test cases. Also, you should define if the tests should be automated or executed manually.

## Task 2
Introduce **Story Points** to estimate the relative size and complexity of your User Stories. Of course, story Points should not represent a direct number of hours.

They should provide a relative estimate that considers factors such as:

- amount of work
- technical complexity
- uncertainty
- dependencies
- testing effort

Use a consistent scale such as, for example, Fibonacci:
```
1   2   3   5   8   13 ...
```

    in which each numerical value has a clearly communicated complexity meaning.

## Task 3
Define the duration of the sprint and then give the rough outline of projected sprint backlogs for the planned duration of the project.

## Task 4
Create Activity UML diagram for the main functionality of the application and version it as a `.drawio` file in the `docs` folder of your project repository.

## Task 5
Define which tools (programming lanugages, frameworks, database, other services) will be used for project development.

## Task 4
Create GitHub project associated with the repository that you created in the previous lab. See how you would create the corresponding tasks using Projects UI, as that will be our next exericse. 