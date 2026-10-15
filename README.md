# RaceDay — Part 2: RESTful API

**Module:** PROG6212 – Programming 2B
**Student Number:** ST10184019
**GitHub:** [TONII11](https://github.com/TONII11)
**Part:** 2 of 3 — RESTful API Development (100 Marks)

RaceDay is an event management platform for the South African road running, walking and cycling
community. Part 2 is the ASP.NET Core Web API that powers it: it connects to SQL Server through
Entity Framework Core (Code-First), enforces role-based access with server-side sessions, is fully
documented in Swagger, and is covered by automated unit tests that run in GitHub Actions.

The Part 1 planning documents (ERD, endpoint plan, SQL script) are preserved in [`/docs`](docs/).
Part 1's original README is in [`docs/Part1_README.md`](docs/Part1_README.md).

## Roles

- **Organiser** – creates, edits and deletes their own events; defines categories; views all enrolments
  for their events; captures finish times and positions.
- **Participant** – registers, browses events, enrols in an event under a chosen category, views their
  own enrolments and their own results.

Role is chosen at registration (`role`: `"Organiser"` or `"Participant"`), stored through the `Roles`
lookup table, and enforced on every protected endpoint. Organisers can only act on events they own.

## Tech stack

ASP.NET Core 8 Web API · EF Core 8 (Code-First, SQL Server) · Swashbuckle (Swagger) · xUnit +
`WebApplicationFactory` · GitHub Actions

## Project layout

```
RaceDay.sln
RaceDay.Api/
  Controllers/   Auth, Users, Events, Categories, Enrolments, Results
  Models/        Entity classes (Role, User, RaceEvent, Category, Enrolment, RaceResult)
  Data/          RaceDayDbContext (schema matches Part 1) + DemoDataSeeder (Development only)
  Dtos/          Request/response models with validation attributes
  Filters/       SessionAuthorizeAttribute – session + role enforcement
  Program.cs     Services, session, Swagger, startup database creation
RaceDay.Tests/   xUnit tests (in-memory database, real controllers and session pipeline)
docs/            Part 1: ERD, endpoint plan, SQL script
.github/workflows/  part1-docs-validation.yml, part2-build-test.yml
```

## Setup and run

**Prerequisites:** .NET 8 SDK, and SQL Server (LocalDB, which ships with Visual Studio, or SQL Server
Express/Developer).

1. Clone the repository and open `RaceDay.sln` in Visual Studio 2022 (or use the CLI).
2. Check the connection string in `RaceDay.Api/appsettings.json` (`ConnectionStrings:RaceDayDB`).
   The default uses `(localdb)\MSSQLLocalDB` and a database called `RaceDayDB`.
3. Run the API:
   ```bash
   dotnet run --project RaceDay.Api --launch-profile http
   ```
   or press **F5** in Visual Studio with `RaceDay.Api` as the startup project.
4. Swagger UI opens at **http://localhost:5080/swagger** (the root URL redirects there).

On start-up the API creates the database and tables if they do not exist. The two roles are seeded by
EF Core. If you have already run the Part 1 SQL script, the API uses that database as-is (the schema is
identical).

### Demo data (Development only)

When running in the Development environment the API also loads the same sample data as the Part 1 script
(2 organisers, 2 participants, 3 events, 5 categories, 3 enrolments, 1 result). If you ran the Part 1
script yourself, its placeholder password hashes are replaced so you can log in. All demo accounts use
the password **`Password123!`**:

| Role | Email |
|---|---|
| Organiser | `naledi.khumalo@raceday.co.za` |
| Organiser | `pieter.vdm@raceday.co.za` |
| Participant | `thandeka.mokoena@gmail.com` |
| Participant | `ryan.botha@gmail.com` |

### Using Swagger

1. Expand **POST /api/auth/login**, click *Try it out*, and log in.
2. The server-side session cookie is stored by the browser, so every later request from Swagger UI is
   authenticated automatically. Use **POST /api/auth/logout** to end the session.
3. Protected endpoints return **401** when not logged in and **403** when the role is wrong.

### Optional: EF Core migrations

The app calls `EnsureCreated()` when no migrations exist and `Migrate()` once they do. To generate a
migration:

```bash
dotnet tool install --global dotnet-ef
dotnet ef migrations add InitialCreate --project RaceDay.Api
```

## Authentication design

- `POST /api/auth/register` validates the details, checks the role and email, hashes the password with
  the ASP.NET Core Identity `PasswordHasher` (PBKDF2 with a per-user salt) and stores only the hash.
- `POST /api/auth/login` verifies the hash, then stores the **user ID and role** in the server-side
  session (`.RaceDay.Session` cookie, HttpOnly, 30-minute idle timeout).
- `[SessionAuthorize]` / `[SessionAuthorize(Role = "Organiser")]` read the session on each request:
  no session → **401**, wrong role → **403**. Ownership checks (an Organiser may only touch their own
  events) are done inside the endpoints and also return **403**.

## Endpoints

All routes match the Part 1 plan (`docs/API_Endpoint_Plan.md`).

| Method | Route | Role | Success | Failure codes |
|---|---|---|---|---|
| POST | `/api/auth/register` | Public | 201 | 400, 409 |
| POST | `/api/auth/login` | Public | 200 | 401 |
| POST | `/api/auth/logout` | Any | 200 | 401 |
| GET | `/api/users/me` | Any | 200 | 401 |
| PUT | `/api/users/me` | Any | 200 | 400, 401 |
| GET | `/api/events` (`?date=&type=&location=&upcoming=`) | Public | 200 | 400 |
| GET | `/api/events/{id}` (includes categories) | Public | 200 | 404 |
| POST | `/api/events` | Organiser | 201 | 400, 401, 403 |
| PUT | `/api/events/{id}` | Organiser (owner) | 200 | 400, 401, 403, 404 |
| DELETE | `/api/events/{id}` | Organiser (owner) | 204 | 401, 403, 404, 409 |
| GET | `/api/events/{eventId}/categories` | Public | 200 | 404 |
| POST | `/api/events/{eventId}/categories` | Organiser (owner) | 201 | 400, 401, 403, 404 |
| DELETE | `/api/categories/{id}` | Organiser (owner) | 204 | 401, 403, 404, 409 |
| POST | `/api/events/{eventId}/enrolments` | Participant | 201 | 400, 401, 403, 404 |
| GET | `/api/users/me/enrolments` | Participant | 200 | 401, 403 |
| GET | `/api/events/{eventId}/enrolments` | Organiser (owner) | 200 | 401, 403, 404 |
| POST | `/api/enrolments/{enrolmentId}/results` | Organiser (owner) | 201 | 400, 401, 403, 404, 409 |
| GET | `/api/users/me/results` | Participant | 200 | 401, 403 |
| GET | `/api/events/{eventId}/results` | Organiser (owner) | 200 | 401, 403, 404 |

### Differences from the Part 1 plan (explained)

The plan's routes, roles and request bodies are implemented unchanged. These small additions were made
while building:

1. **`409 Conflict` on `DELETE /api/events/{id}` and `DELETE /api/categories/{id}`** when enrolments
   exist. The Part 1 schema uses `NO ACTION` foreign keys from `Enrolments`, so deleting would fail with a
   database error; the API returns a clear message instead.
2. **Extra `400 Bad Request` cases** on enrolment and results: enrolling in an event that has already
   taken place, choosing a category from a different event, capturing a result before the event has
   happened, or a finish position greater than the total finishers (mirrors `CK_Results_Position`).
3. **Enrolment status:** a new enrolment is saved as `Confirmed` straight away (the plan has no
   confirmation step). The `Pending` and `Cancelled` values from the schema are still valid.
4. **Optional `upcoming=true`** filter on `GET /api/events`, to support the Part 3 home page.
5. **Result responses** also include `finishTime` as `hh:mm:ss` next to `finishTimeSeconds`.

## Database

The EF Core model is configured in `RaceDayDbContext` with the exact table names, column names, data
types, primary keys, foreign keys (`ON DELETE` behaviour as in the Part 1 script), unique constraints,
check constraints and defaults from the Part 1 ERD and SQL script. The script's `UNIQUE` constraints are
created by EF Core as unique indexes (functionally identical).

## Unit tests

`RaceDay.Tests` uses xUnit and `WebApplicationFactory`, which starts the real API (controllers, session
middleware and role filter) in memory with an isolated EF Core in-memory database per test class.

| Area | What is verified |
|---|---|
| Registration and login | Both roles register; duplicate email (409); bad role / short password / bad email / missing fields (400); password is hashed in the database; correct and wrong credentials; session needed for `/users/me`; logout clears the session; profile update |
| Event management | Organiser create / update / delete; Participant and anonymous are rejected (403 / 401); another Organiser cannot edit or delete (403); validation (400); public list, filter and detail |
| Categories | Organiser add / delete; Participant and non-owner rejected; invalid age range; 404 and 409 cases |
| Role enforcement | Organiser calling Participant-only endpoints and the reverse return 403; anonymous calls return 401 |
| Enrolments | Participant enrols and the participant/event/category link is stored; "my enrolments"; duplicate enrolment, wrong category, past event, unknown event/category; Organiser views enrolments for own event only |
| Results | Organiser captures a result; Participant sees only their own; duplicates (409); invalid values; result before the event (400); ownership and role checks |

Run them locally:

```bash
dotnet test RaceDay.sln
```

## CI/CD

`.github/workflows/part2-build-test.yml` checks the repository structure, then restores, builds and runs
all unit tests on every push. `part1-docs-validation.yml` from Part 1 is kept.

**CI/CD screenshot:** *add a screenshot of the green Part 2 workflow run here once it has passed on GitHub.*

## Video

Unlisted YouTube walkthrough: *add link here*

## AI usage disclosure

An AI assistant (Claude) was used to generate the initial version of the Part 2 code and tests. *Replace
this with an accurate description of what you reviewed, changed and wrote yourself before submitting.*
