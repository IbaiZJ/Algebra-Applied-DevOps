# 9. Technology stack

The project uses the following stack to build, test and deploy the hospital management application. It separates the user interface, API and database while keeping development and deployment consistent.

## Frontend

- **Next.js and React** — We use Next.js to build the web application and React to create reusable interface components. This combination supports the patient, doctor and appointment workflows.
- **Tailwind CSS** — We use Tailwind CSS to style the interface with reusable utility classes and maintain a consistent design.

## Backend

- **Python** — We use Python for the backend because its clear syntax and broad ecosystem support maintainable application development.
- **FastAPI** — We use FastAPI to build the REST API. It provides request validation and automatically generated API documentation, and connects the frontend to the application logic and data.

## Database

- **PostgreSQL** — We use PostgreSQL as the relational database for patients, doctors, appointments and medical records. Its constraints and transactions help keep related data consistent and prevent conflicting appointments.

## Containers

- **Docker** — We use Docker to package the application and its dependencies into consistent environments for development and deployment.
- **Docker Compose** — We use Docker Compose to run the frontend, API and database together during local development.

## Testing

- **Playwright** — We use Playwright for end-to-end browser tests. These tests verify important user workflows across the application, including authentication, role-based access and appointment management.

## CI/CD

- **GitHub Actions** — We use GitHub Actions to automate the CI/CD pipeline. The pipeline runs the relevant checks and tests, including Playwright tests, and supports repeatable application deployments.

## Security and operations

Since the application handles sensitive medical information, we design it with authentication, role-based authorization, secure network connections, protected configuration and secrets, database backups and an audit trail. We also use application logging and service monitoring to help diagnose errors and track operational health.
