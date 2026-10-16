# RaceDay - Part 1: System Planning and Database

**Module:** PROG6212/w - Programming 2B
**Part:** 1 of 3 (System Planning and Database) - 100 Marks

## System Description

RaceDay is a full-stack event management platform built for the South African road running, walking, and cycling community. It replaces the paper-based registration and spreadsheet-driven workflows that many local events (park runs, charity walks, community races) still rely on.

The platform lets **Event Organisers** create and manage events and categories, and capture participant results, while **Participants** can browse upcoming events, enter events by selecting a category, and track their personal race history.

This Part 1 submission covers the planning phase only: no application code is written yet. The goal is to design the data model and API surface before any implementation begins, so that Part 2 (the REST API) and Part 3 (the MVC front end) can be built directly against an approved plan.

## Roles

- **Organiser** - creates, edits, and deletes events; manages event categories; captures participant results; views all enrolments for their own events.
- **Participant** - registers an account, browses events, enrols in an event under a chosen category, views their own enrolments, and tracks their personal results.

Role-based access will be enforced at the API level in Part 2 and reflected in the MVC interface in Part 3.

## Contents of /docs

| File | Description |
|---|---|
| `RaceDay_ERD.png` | Entity Relationship Diagram covering all 6 entities (Users, EventTypes, Events, Categories, Enrolments, Results), with primary keys, foreign keys, and cardinality. |
| `RaceDay_API_Endpoint_Plan.md` | Full endpoint plan covering Authentication, User Profile, Events, Categories, Event Enrolments, and Results. |
| `RaceDay_Database_Script.sql` | SQL Server script that creates the full schema (matching the ERD exactly) and seeds it with realistic sample data - 2 Organisers, 2 Participants, 3 Events, categories per event, and sample enrolments/results. |

## Design Notes

- **EventTypes** was added as its own lookup table (rather than a free-text column on Events) to normalise the Run/Walk/Cycle classification and keep the schema clean for filtering.
- **Enrolments** has a `UNIQUE (ParticipantId, EventId)` constraint so a Participant cannot enrol in the same event twice.
- **Results** has a one-to-one relationship with **Enrolments** (`UNIQUE` foreign key), since a result only exists once a Participant has enrolled and the event has concluded.
- Passwords are stored as hashes (`PasswordHash` column); actual hashing logic will be implemented with the authentication code in Part 2.

## How the SQL Script was Tested

The script was written to run cleanly on a fresh SQL Server instance in SSMS: it creates the `RaceDayDB` database if it doesn't exist, drops any existing tables in dependency order (so it is safely re-runnable during testing), recreates all six tables with their constraints, and then seeds sample data. Verification `SELECT` statements are included at the bottom (commented out) and were run manually to confirm the seed data loaded correctly.

## Next Steps

- **Part 2** will implement the REST API in ASP.NET Core against this plan, connect it to the database via EF Core, add authentication/session management and role enforcement, and cover the endpoints with unit tests.

- # RaceDay – Part 2 RESTful API

**Module:** PROG6212/w – Programming 2B  
**System:** RaceDay event management platform  
**Technology:** ASP.NET Core 8 Web API, Entity Framework Core 8, SQL Server, session authentication, Swagger, xUnit and GitHub Actions.

## Part 2 rubric coverage

| Requirement | Implementation |
|---|---|
| ASP.NET Core Web API | `src/RaceDay.Api` |
| EF Core Code-First | `RaceDayDbContext` + entity configuration |
| SQL Server | `UseSqlServer` connection string |
| Register/Login | `/api/auth/register`, `/api/auth/login` |
| Session authentication | `.RaceDay.Session`, `UserId` and `Role` session keys |
| Password hashing | ASP.NET Core `PasswordHasher` |
| Two roles | Organiser and Participant, enforced server-side |
| Events | Full create/read/update/delete with ownership |
| Categories | Create/read/update/delete with ownership |
| Enrolments | Participant entry and organiser management |
| Results | Organiser capture and participant history |
| Swagger | `/swagger` |
| Unit tests | `tests/RaceDay.Api.Tests` |
| CI/CD | `.github/workflows/part2-ci.yml` |

## 1. Prerequisites

- Visual Studio 2022 with the ASP.NET and web development workload.
- .NET 8 SDK.
- SQL Server / LocalDB and SSMS.
- Git and a GitHub account.

## 2. Database

Part 1 already defines `RaceDayDB` and its six entities. Run the approved Part 1 SQL script in SSMS first.

The API's EF Core model uses the same tables, columns, keys, foreign keys, constraints and relationships. Do not create a second database with different names.

### EF Core Code-First verification

From the solution directory:

```powershell
dotnet tool install --global dotnet-ef
dotnet ef migrations add InitialCreate --project src/RaceDay.Api --startup-project src/RaceDay.Api
dotnet ef database update --project src/RaceDay.Api --startup-project src/RaceDay.Api
```

For a clean Code-First demonstration, you may create a fresh `RaceDayDB_CodeFirst` database by temporarily changing the connection string and running the commands above. Compare the generated schema to the Part 1 SQL schema.

## 3. Run the API

Open `RaceDay.sln` in Visual Studio.

Set `RaceDay.Api` as the startup project and run with HTTPS.

Swagger:
`https://localhost:7132/swagger`

## 4. Seeded users from Part 1

The Part 1 SQL seed contains placeholder password hashes. For Part 2 demonstrations, register fresh accounts through Swagger:

- Organiser: choose `Organiser`
- Participant: choose `Participant`

This demonstrates that passwords are hashed by the API rather than stored in plain text.

## 5. Authentication demonstration

1. `POST /api/auth/register`
2. Register an Organiser.
3. `POST /api/auth/login`
4. Confirm the response contains the user ID and role.
5. Call `POST /api/events` successfully as Organiser.
6. Register/login as Participant.
7. Attempt `POST /api/events`; it must return `403 Forbidden`.
8. Call `POST /api/events/{eventId}/enrolments` as Participant; it must return `201 Created`.
9. Call `POST /api/auth/logout`.
10. Call a protected endpoint again; it must reject the unauthenticated request.

The server never trusts a role sent by the client. Role checks are performed from the server-side session.

## 6. Swagger

Swagger exposes every implemented endpoint and documents:
- purpose
- request body
- route parameters
- response types/status codes
- session cookie security definition

Use Swagger as the primary Part 2 demonstration tool.

## 7. Unit tests

Run:

```powershell
dotnet test RaceDay.sln
```

The test suite covers:
- valid and invalid registration
- duplicate email
- password hashing
- valid and invalid login
- session creation
- unauthenticated profile access
- authenticated profile access
- organiser event creation
- participant rejection of organiser functions
- unauthorised event ownership
- event deletion
- participant enrolment
- invalid category/event combination
- organiser rejection of participant-only enrolment
- organiser result capture
- participant rejection of result capture
- duplicate result rejection
- category role enforcement

The tests use EF Core's in-memory provider so they are deterministic and do not require a live SQL Server in CI.

## 8. GitHub Actions

The workflow in `.github/workflows/part2-ci.yml`:
1. checks out the repository;
2. installs .NET 8;
3. restores dependencies;
4. builds the complete solution;
5. runs all xUnit tests.

A green workflow is required before submission. Take a screenshot of the green GitHub Actions run and place it in this README before final submission.

## 9. Recommended repository structure

```text
RaceDay/
├── .github/
│   └── workflows/
│       └── part2-ci.yml
├── docs/
│   ├── RaceDay_ERD.png
│   ├── RaceDay_API_Endpoint_Plan.md
│   └── RaceDay_Database_Script.sql
├── src/
│   └── RaceDay.Api/
│       ├── Controllers/
│       ├── Data/
│       ├── DTOs/
│       ├── Models/
│       ├── Services/
│       ├── Properties/
│       ├── Program.cs
│       ├── appsettings.json
│       └── RaceDay.Api.csproj
├── tests/
│   └── RaceDay.Api.Tests/
├── RaceDay.sln
└── README.md
```

Copy the Part 1 `/docs` files into this repository before pushing.

## 10. Part 2 video checklist

Your unlisted YouTube video should personally demonstrate:
- solution structure;
- EF Core model/database relationship;
- registration and password hashing;
- login and session creation;
- Organiser versus Participant access;
- event CRUD;
- categories;
- participant enrolment;
- organiser enrolment view/status;
- result capture and participant result history;
- Swagger;
- unit tests passing;
- GitHub Actions green build.

**Important academic-integrity note:** The POE explicitly requires students to disclose AI use and states that AI-generated code must not be submitted as the student's own work without analysis, synthesis and critical thinking. Review, understand, test and adapt every part of this project and add the required disclosure to your submission.

## Next Steps

- **Part 3** will build the MVC front end that consumes the Part 2 API, add Azure Blob Storage for event banners and profile pictures, and containerise the application with Docker.

## AI Disclosure

AI tools were used to assist with planning structure, drafting the SQL script, and formatting this documentation. All design decisions (entity structure, relationships, endpoint design) were reviewed and understood before submission.

## CI/CD Screenshot

*(Insert screenshot of the green GitHub Actions build here before submission.)*

## Video Link

*(Insert unlisted YouTube video link here before submission.)*
