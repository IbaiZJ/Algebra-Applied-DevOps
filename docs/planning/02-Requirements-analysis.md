## 2. Requirements analysis

### 2.1. Functional requirements
#### Patients
- **FR-01** - A receptionist or administrator can register a new patient by entering basic information (first name, last name, date of birth, contact details).
- **FR-02** - Receptionists and administrators can search for patients by name and view their administrative details (name, date of birth and contact details). Clinical information is governed by FR-15 and FR-16.
- **FR-03** - A receptionist or administrator can edit a patient's administrative information (name, date of birth and contact details).
- **FR-04** - A receptionist or administrator can deactivate a patient record. A deactivated patient cannot be selected for a new appointment; deactivation does not delete the patient record, past appointments or medical records.

#### Doctors
- **FR-05** - An administrator can register a doctor with name and specialisation.
- **FR-06** - Receptionists, doctors and administrators can view the doctor list and each doctor's name, specialisation and availability.
- **FR-07** - An administrator can update a doctor's details, including specialisation and availability.
- **FR-08** - An administrator can deactivate a doctor. A deactivated doctor cannot be selected for new appointments; past appointments and medical records linked to that doctor are preserved.

#### Appointments
- **FR-09** - A receptionist or administrator can create an appointment for an active patient with an active doctor, specifying the date, time and reason.
- **FR-10** - A new appointment is assigned the `scheduled` status. An appointment can have one of three statuses: `scheduled`, `completed` or `cancelled`.
- **FR-11** - The system rejects an appointment if that doctor already has another `scheduled` appointment at the same date and time, and explains the conflict. A `cancelled` appointment does not block the time slot.
- **FR-12** - A receptionist or administrator can update or cancel a `scheduled` appointment. Only the doctor assigned to an appointment can mark it as `completed`.
- **FR-13** - Receptionists and administrators can list and filter all appointments by date and doctor. Doctors can list and filter only appointments assigned to them.

#### Medical records
- **FR-14** - The doctor assigned to a visit can create a medical record for that appointment, containing the visit date, diagnosis and notes. The record is linked to the patient, appointment and doctor.
- **FR-15** - Medical records remain available when a patient or doctor is deactivated and are not deleted through routine user actions. They are retained permanently unless applicable data-protection or retention requirements require otherwise (see NFR-07). Receptionists and administrators cannot read diagnoses or visit notes.
- **FR-16** - A doctor can read a patient's previous medical records only when the patient is linked to one of that doctor's `scheduled` or `completed` appointments.

#### Authentication
- **FR-17** - Users must log in with credentials before accessing any protected functionality.
- **FR-18** - The system supports exactly three staff roles: receptionist, doctor and administrator.
- **FR-19** - An administrator can create and deactivate staff accounts, assign each account one of the three supported roles, and initiate a password-reset process.

#### Access control and export
- **FR-20** - The system checks the user's role on every protected operation, including direct API requests, and denies operations the role is not permitted to perform.
- **FR-21** - A doctor can export authorised patient data as a CSV report, limited to patients linked to that doctor's `scheduled` or `completed` appointments. Receptionists and administrators cannot use this export.

### 2.2. Non-Functional requirements

#### Security and privacy
- **NFR-01** - All pages and API operations that access patient, appointment or medical-record data require an authenticated session. The system denies access by default and enforces the permissions defined in the roles and access control matrix on the server.
- **NFR-02** - The system protects medical records using role-based access control. Doctors can access only records for patients they treat; receptionists cannot access diagnoses or visit notes; administrators cannot read medical histories. These restrictions apply to search, detail views, exports and API endpoints.
- **NFR-03** - Credentials and secrets are never stored in source control or in plain text. Passwords are stored using a modern, salted password-hashing algorithm, and password reset flows do not reveal whether an account exists.
- **NFR-04** - Data transmitted between the browser and the application is protected with HTTPS in deployed environments. Database credentials and other secrets are provided through protected environment configuration.
- **NFR-05** - The application validates and safely handles user input, uses parameterized database operations, and protects authenticated actions against common web threats, including injection, cross-site scripting and cross-site request forgery where applicable.
- **NFR-06** - Access to sensitive records and security-relevant actions (including login failures, account changes, permission denials, record changes and exports) is recorded with the user, action and timestamp. Audit logs must not contain passwords, authentication tokens or unnecessary medical details and must be accessible only to authorised operators.
- **NFR-07** - Personal and health information is collected and retained only for the application's stated purposes and handled in accordance with applicable data-protection requirements, including the GDPR where applicable. Retention and deletion policies must preserve records that are required to remain available under the functional requirements.

#### Performance and capacity
- **NFR-08** - Under normal operating load of up to 50 concurrent users, 95% of patient, doctor and appointment list, search and detail requests complete within 2 seconds, excluding network time outside the application's control.
- **NFR-09** - Under the same load, 95% of standard create, update and cancellation operations complete within 3 seconds. Requests that exceed the limit return a clear error and do not leave partially saved data.
- **NFR-10** - The system supports at least 10,000 patient records, 1,000 user accounts and 100,000 appointments without requiring a change to the user-facing workflows.

#### Availability and reliability
- **NFR-11** - The deployed application targets 99.5% availability per calendar month, excluding announced maintenance. If a required service is unavailable, the application displays a clear error and does not report an unsuccessful operation as successful.
- **NFR-12** - Database transactions preserve data consistency. In particular, appointment creation must prevent conflicting bookings even when requests arrive concurrently, and failed operations must not create incomplete records.
- **NFR-13** - Automated database backups are created at least once every 24 hours, encrypted, and retained for at least 30 days. A documented restore procedure must be available to administrators.
- **NFR-14** - The service targets a recovery point objective (RPO) of 24 hours and a recovery time objective (RTO) of 8 hours after a major failure. Backup restoration is verified at least quarterly.

#### Usability and accessibility
- **NFR-15** - A first-time receptionist can complete the main patient registration and appointment scheduling workflows using the interface without consulting technical documentation. Forms provide clear labels, identify required fields and explain validation errors next to the relevant input.
- **NFR-16** - The web interface supports current stable versions of Chrome, Firefox, Edge and Safari on desktop and tablet screen sizes. Core workflows remain usable at a viewport width of 360 CSS pixels without horizontal scrolling.
- **NFR-17** - The interface follows WCAG 2.1 Level AA accessibility guidance for core workflows, including keyboard operation, visible focus, meaningful labels, sufficient text contrast and programmatically associated form errors.

#### Maintainability and operability
- **NFR-18** - The frontend, API and database can be started consistently in a local development environment using the documented Docker Compose setup. Configuration that differs by environment is kept outside application code.
- **NFR-19** - The application exposes health checks for its services and produces structured logs that include a timestamp, severity and request or correlation identifier. Logs and health checks must not expose credentials, tokens or unnecessary patient information.
- **NFR-20** - Changes to the application are integrated through the documented CI pipeline. The pipeline runs automated checks, including the end-to-end workflows identified by the project, and reports failures before a change is accepted into the main branch.
- **NFR-21** - The application provides clear, user-facing error messages for recoverable failures and records diagnostic details for operators. Error messages do not disclose stack traces, database details or other sensitive implementation information.
