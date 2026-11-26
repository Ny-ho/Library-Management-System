# Library Management System — Project Blueprint

**Project type:** Backend-focused, production-minded application  
**Primary goal:** Build a reliable library management backend while learning real backend engineering practices.  
**Initial approach:** Modular monolith, API-first development.  
**Frontend:** Deferred until the backend is functional.

---

# 1. Project Vision

Build a Library Management System (LMS) that allows a library to manage its books, physical book copies, members, borrowing, returns, reservations, and overdue fines.

This should not be merely a CRUD application.

The project should demonstrate the ability to:

- Design a relational database.
- Build REST APIs using FastAPI.
- Implement authentication and authorization.
- Manage database transactions correctly.
- Handle concurrent requests safely.
- Write meaningful automated tests.
- Build a maintainable application architecture.
- Deploy an application using Docker.
- Make engineering decisions based on actual requirements.

## Main design principle

Correctness before complexity.

We will not introduce Redis, Celery, microservices, Kafka, or other infrastructure simply to make the project look sophisticated. Each technology must solve an actual problem.

---

# 2. Technology Stack

| Component | Technology | Purpose |
|---|---|---|
| Language | Python | Application development |
| API framework | FastAPI | REST API |
| ORM | SQLModel / SQLAlchemy | Database interaction |
| Database | PostgreSQL | Persistent relational data |
| Migrations | Alembic | Database schema versioning |
| Admin interface | SQLAdmin | Administrative dashboard |
| Authentication | JWT | API authentication |
| Password hashing | Argon2 | Secure password storage |
| Containerization | Docker | Reproducible environment |
| Container orchestration | Docker Compose | Run application dependencies |
| Testing | Pytest | Automated testing |
| HTTP testing | HTTPX | API integration tests |
| Cache | Redis | Deferred |
| Background jobs | Celery | Deferred |
| CI | GitHub Actions | Automated checks |

### Important decisions

- PostgreSQL is the authoritative database.
- SQLModel is built on SQLAlchemy, so we can use SQLAlchemy capabilities when necessary.
- Alembic manages database changes. We will not rely on creating tables automatically in production.
- SQLAdmin provides administrative interfaces, but business rules remain in the application services.
- Redis and Celery are optional future additions, not initial dependencies.

---

# 3. Functional Requirements

## 3.1 Users and roles

The system initially supports three roles:

**Member**
- Register and log in.
- Browse books.
- View available copies.
- Borrow books.
- Return books.
- Reserve unavailable books.
- View borrowing history.
- View outstanding fines.

**Librarian**
- Manage books and physical copies.
- Register or manage members.
- Issue and receive books.
- Manage reservations.
- Review overdue books.

**Administrator**
- Manage users and roles.
- Access administrative operations.
- Manage system-wide settings.
- Review audit information.

A user should not automatically gain access to an operation simply because they are authenticated. Authorization must also check their role and, where relevant, ownership of the resource.

## 3.2 Book management

- Create books.
- Update book information.
- Delete or archive books.
- Search by title, author, ISBN, and category.
- Browse books with pagination.
- Manage multiple authors.
- Manage categories.

## 3.3 Physical copy management

A book title and a physical copy are different things.

For example:

The library owns five physical copies of *Clean Code*.

There should be:
- One Book record.
- Five BookCopy records.

Each copy has its own identifier and borrowing status.

This distinction is essential because the system must know exactly which physical copy is borrowed, available, lost, or under maintenance.

## 3.4 Borrowing and returning

- Borrow an available copy.
- Assign a borrowing date.
- Calculate the due date.
- Return a borrowed copy.
- Track borrowing history.
- Reject borrowing when the member has reached their borrowing limit.
- Reject borrowing when the member has outstanding restrictions.

## 3.5 Reservations

When all copies of a book are unavailable:

- A member can reserve the book.
- Reservations are placed in a queue.
- When a copy becomes available, the earliest eligible reservation gets priority.
- The member has a limited period to collect the book.
- Expired reservations are released.

## 3.6 Fines

- Calculate overdue fines.
- Track outstanding fines.
- Record fine payment status.
- Prevent borrowing if policy requires it.

Initially, payment is recorded manually. Payment gateway integration is outside the MVP.

---

# 4. Non-Functional Requirements

These describe how well the system should behave.

**Reliability**
- Borrowing must not assign the same copy to two members.
- Invalid state transitions must be rejected.
- Failed operations must not leave partial database changes.

**Security**
- Passwords must never be stored as plaintext.
- Protected endpoints require authentication.
- Role restrictions must be enforced on the backend.
- Secrets must be stored in environment variables.
- Input must be validated.

**Maintainability**
- Separate HTTP routing from business logic.
- Avoid putting all application code in `main.py`.
- Keep database models and API schemas conceptually separate.
- Use clear module boundaries.

**Performance**
- Use pagination for large collections.
- Add indexes for frequently queried fields.
- Avoid unnecessary database queries.
- Introduce caching only when measurements justify it.

**Testability**
- Important business rules must have automated tests.
- Database behavior must be tested against PostgreSQL.
- Concurrency-sensitive operations need dedicated tests.

---

# 5. High-Level Architecture

We will use a **modular monolith**.

That means one application and one deployment unit, but internally separated into modules.

```text
                  Client / API Consumer
                           |
                           v
                     FastAPI App
                           |
             Authentication / Authorization
                           |
                           v
                     API Routers
                           |
                           v
                      Services
                           |
                           v
                 Repositories / ORM
                           |
                           v
                     PostgreSQL
```

SQLAdmin connects to the database through SQLAlchemy and provides a separate administrative interface.

```text
                 SQLAdmin Dashboard
                         |
                         v
                  Admin Model Views
                         |
                         v
                     PostgreSQL
```

Important: SQLAdmin is an administrative interface, not a replacement for application services. Operations that affect business invariants should use safe custom actions or service methods rather than unrestricted database editing.

## Responsibilities

**Routers**
- Receive HTTP requests.
- Validate request data.
- Call services.
- Return HTTP responses.

**Services**
- Implement business rules.
- Coordinate database operations.
- Manage transactions.
- Enforce state transitions.

**Repositories**
- Encapsulate reusable database queries when useful.
- Avoid duplicating query logic across services.

**Models**
- Represent database entities and relationships.

**Schemas**
- Define request and response structures.

We will not create unnecessary abstraction layers. A repository is useful when it simplifies the application, not merely because every tutorial has one.

---

# 6. Database Design

## Main entities

### User
Represents a library member, librarian, or administrator.

Important fields:
- id
- full_name
- email
- hashed_password
- role
- is_active
- created_at

### Book
Represents a book title, not a physical copy.

Important fields:
- id
- title
- isbn
- description
- publisher
- publication_year
- created_at

### Author
Represents an author.

Important fields:
- id
- name

### BookAuthor
Many-to-many association between Book and Author.

Fields:
- book_id
- author_id

### Category
Represents a classification such as programming, history, or science.

Fields:
- id
- name

### BookCopy
Represents one physical copy of a book.

Fields:
- id
- book_id
- copy_code
- status
- created_at

Possible statuses:
- available
- borrowed
- maintenance
- lost

### Borrowing
Represents one borrowing transaction.

Fields:
- id
- user_id
- book_copy_id
- borrowed_at
- due_at
- returned_at
- status

Possible statuses:
- active
- returned
- lost

### Reservation
Represents a member's request for a book.

Fields:
- id
- user_id
- book_id
- created_at
- expires_at
- status

Possible statuses:
- queued
- ready
- fulfilled
- cancelled
- expired

### Fine
Represents a fine associated with a borrowing.

Fields:
- id
- borrowing_id
- amount
- status
- created_at
- paid_at

Possible statuses:
- unpaid
- paid
- waived

### AuditLog (later phase)
Records important administrative and business actions.

Fields:
- id
- actor_user_id
- action
- entity_type
- entity_id
- created_at

## Relationships

```text
User 1 -------- N Borrowing

Book 1 -------- N BookCopy

Book N -------- N Author
       through BookAuthor

Book N -------- N Category
       through BookCategory

BookCopy 1 ---- N Borrowing

Book 1 -------- N Reservation

User 1 -------- N Reservation

Borrowing 1 --- 0..1 Fine
```

A borrowing belongs to one user and one physical copy.

A book can have many physical copies.

A reservation targets a book title rather than a specific copy. This allows the reservation queue to work regardless of which copy becomes available.

## Database integrity rules

Examples of important constraints:

- Email must be unique.
- ISBN should be unique when supplied.
- Copy code must be unique.
- A copy cannot have multiple active borrowings.
- A borrowing must reference an existing user and copy.
- A fine must reference a valid borrowing.
- Reservation status must be valid.
- Foreign keys must protect relationships.

PostgreSQL constraints and indexes should enforce critical rules wherever possible. Application validation alone is not enough.

---

# 7. Important Business Workflows

## 7.1 Borrowing a book

```text
Member requests a book copy
           |
           v
Authenticate member
           |
           v
Check borrowing permissions
           |
           v
Check borrowing limit and restrictions
           |
           v
Start database transaction
           |
           v
Lock selected BookCopy row
           |
           v
Recheck copy availability
           |
       Available?
        /     \
      No       Yes
      |         |
    Reject    Create Borrowing
                |
                v
         Mark copy borrowed
                |
                v
          Commit transaction
                |
                v
          Return response
```

### Why the transaction and row lock?

Imagine two members attempt to borrow the same copy at almost exactly the same time.

Without concurrency protection, both requests might read the copy as available.

Both could then create a borrowing record.

We must prevent that.

PostgreSQL row-level locking, combined with database constraints and transaction handling, will ensure that only one operation succeeds.

This is one of the most important engineering problems in the project.

## 7.2 Returning a book

```text
Member or librarian returns copy
             |
             v
Find active borrowing
             |
             v
Start transaction
             |
             v
Mark borrowing returned
             |
             v
Calculate overdue fine
             |
             v
Check reservation queue
             |
       Reservation exists?
          /       \
        Yes        No
         |          |
    Mark copy      Mark copy
       ready       available
         |          |
         v          v
      Commit transaction
```

The actual reservation allocation must be implemented carefully so two return operations cannot allocate the same copy to multiple members.

## 7.3 Reserving a book

```text
Member requests reservation
             |
             v
Check member eligibility
             |
             v
Check existing reservation
             |
             v
Create queue entry
             |
             v
Return queue information
```

When a copy becomes available, the system selects the earliest eligible queued reservation.

The allocation and state changes must happen transactionally.

---

# 8. API Design

All endpoints will initially use the `/api/v1` prefix.

## Authentication

| Method | Endpoint | Purpose |
|---|---|---|
| POST | /auth/register | Register member |
| POST | /auth/login | Login |
| POST | /auth/refresh | Refresh access token |
| POST | /auth/logout | Logout / revoke refresh token |

## Users

| Method | Endpoint | Purpose |
|---|---|---|
| GET | /users/me | Current user |
| GET | /users | List users (authorized) |
| GET | /users/{id} | User details |
| PATCH | /users/{id} | Update user |
| DELETE | /users/{id} | Deactivate or remove user |

## Books

| Method | Endpoint | Purpose |
|---|---|---|
| POST | /books | Create book |
| GET | /books | List and search books |
| GET | /books/{id} | Book details |
| PATCH | /books/{id} | Update book |
| DELETE | /books/{id} | Archive book |

## Book copies

| Method | Endpoint | Purpose |
|---|---|---|
| POST | /books/{id}/copies | Add physical copy |
| GET | /books/{id}/copies | List copies |
| PATCH | /copies/{id} | Update copy details |
| GET | /copies/{id} | Copy details |

## Borrowings

| Method | Endpoint | Purpose |
|---|---|---|
| POST | /borrowings | Borrow a copy |
| GET | /borrowings/me | Current member's borrowings |
| GET | /borrowings | List borrowings (authorized) |
| GET | /borrowings/{id} | Borrowing details |
| POST | /borrowings/{id}/return | Return a copy |
| POST | /borrowings/{id}/renew | Renew borrowing |

## Reservations

| Method | Endpoint | Purpose |
|---|---|---|
| POST | /reservations | Reserve a book |
| GET | /reservations/me | My reservations |
| GET | /reservations | List reservations (authorized) |
| POST | /reservations/{id}/cancel | Cancel reservation |

## Fines

| Method | Endpoint | Purpose |
|---|---|---|
| GET | /fines/me | My fines |
| GET | /fines | List fines (authorized) |
| GET | /fines/{id} | Fine details |

These are proposed endpoints. We will implement them as the corresponding features are developed, rather than creating empty routes in advance.

---

# 9. Authentication and Authorization

## Authentication

Initial flow:

1. Member registers with an email and password.
2. Password is hashed using Argon2.
3. Member logs in.
4. Backend verifies credentials.
5. Backend issues a short-lived JWT access token.
6. Protected endpoints verify the token.

Refresh tokens and rotation can be introduced when we implement the authentication module.

JWT payload should contain only necessary information, such as:
- subject/user ID
- expiration
- issued-at time
- token identifier where required

Never store plaintext passwords or sensitive information in JWT payloads.

## Authorization

Authentication answers:

"Who are you?"

Authorization answers:

"Are you allowed to do this?"

Example:

A member can view their own borrowing history but cannot view another member's private borrowing history.

A librarian can manage borrowing operations but should not automatically be allowed to change administrator accounts.

Authorization must be enforced in the backend, not just hidden in the frontend.

---

# 10. SQLAdmin Design

SQLAdmin will be part of the project from the beginning of the administrative layer.

Initial admin views:
- UserAdmin
- BookAdmin
- AuthorAdmin
- CategoryAdmin
- BookCopyAdmin
- BorrowingAdmin
- ReservationAdmin
- FineAdmin

The admin panel should support:
- Listing records.
- Searching and filtering.
- Viewing relationships.
- Editing permitted fields.
- Creating records where safe.
- Restricting access based on administrator permissions.

Important restrictions:

- Password hashes must not be exposed for editing or viewing.
- Critical borrowing state changes should use safe operations.
- Admin access must be authenticated and protected.
- Deleting a book with borrowing history should not casually destroy related records.
- SQLAdmin must not become a way to bypass business rules.

We'll implement and test the admin authentication separately.

---

# 11. Infrastructure and Project Structure

## Proposed folder structure

```text
library-management-system/
│
├── app/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   │
│   ├── auth/
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── schemas.py
│   │   ├── security.py
│   │   └── dependencies.py
│   │
│   ├── users/
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── router.py
│   │   ├── service.py
│   │   └── repository.py
│   │
│   ├── books/
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── router.py
│   │   ├── service.py
│   │   └── repository.py
│   │
│   ├── borrowings/
│   ├── reservations/
│   ├── fines/
│   │
│   └── admin/
│       ├── setup.py
│       ├── authentication.py
│       └── views.py
│
├── alembic/
├── tests/
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_books.py
│   ├── test_borrowings.py
│   └── test_reservations.py
│
├── .env
├── .env.example
├── .gitignore
├── Dockerfile
├── compose.yaml
├── requirements.txt
├── alembic.ini
└── README.md
```

This is a proposed structure, not something we must create all at once.

## Docker services

Initial Compose setup:

```text
Docker Compose
     |
     |---- PostgreSQL
     |
     |---- FastAPI application
```

During development, we may run FastAPI locally inside a Python virtual environment while PostgreSQL runs in Docker. This gives VS Code access to installed packages and autocomplete.

Later, we can containerize the application itself.

---

# 12. Testing Strategy

Testing is not something we add at the end.

We will write tests as we implement features.

## Unit tests

Test individual business rules:
- Password verification.
- Fine calculation.
- Borrowing eligibility.
- Reservation eligibility.

## Integration tests

Test interactions with PostgreSQL:
- Creating a book.
- Creating physical copies.
- Borrowing a copy.
- Returning a copy.
- Creating reservations.
- Database constraints.

## API tests

Test:
- Correct HTTP status codes.
- Request validation.
- Authentication.
- Authorization.
- Error responses.
- Pagination.

## Concurrency tests

Particularly important:

Simulate two members attempting to borrow the same copy simultaneously.

Expected result:

- One borrowing succeeds.
- The other is rejected.
- Only one active borrowing exists.
- The copy is not assigned twice.

Also test simultaneous reservation allocation.

Concurrency correctness must be demonstrated with tests, not merely claimed in the README.

---

# 13. Development Roadmap

We will build this incrementally.

## Phase 1 — Project foundation

- Create repository.
- Create Python virtual environment.
- Install dependencies.
- Configure environment variables.
- Set up PostgreSQL using Docker Compose.
- Connect FastAPI to PostgreSQL.
- Understand database sessions and connection management.

**Deliverable:** A running FastAPI application connected to PostgreSQL.

## Phase 2 — Database foundation

- Understand SQLModel models.
- Create initial User, Book, Author, Category, and BookCopy models.
- Define relationships.
- Configure Alembic.
- Create and apply migrations.
- Learn constraints, indexes, and foreign keys.

**Deliverable:** A properly structured database.

## Phase 3 — SQLAdmin

- Install SQLAdmin.
- Connect it to the application.
- Create model views.
- Implement admin authentication.
- Configure filters, search, and permissions.

**Deliverable:** A protected administrative dashboard.

## Phase 4 — Authentication

- Registration.
- Password hashing.
- Login.
- JWT access tokens.
- Authentication dependencies.
- Role-based authorization.

**Deliverable:** Secure user authentication.

## Phase 5 — Book management

- CRUD operations.
- Authors and categories.
- Physical copies.
- Search.
- Filtering.
- Pagination.

**Deliverable:** Complete book management API.

## Phase 6 — Borrowing system

- Borrowing eligibility.
- Borrow operations.
- Returning books.
- Due dates.
- Transaction handling.
- Row-level locking.
- Borrowing history.

**Deliverable:** A correct borrowing workflow.

## Phase 7 — Reservations

- Reservation creation.
- Queue ordering.
- Cancellation.
- Copy allocation.
- Reservation expiry.

**Deliverable:** Working reservation system.

## Phase 8 — Fines

- Overdue calculations.
- Fine records.
- Payment status.
- Borrowing restrictions.

**Deliverable:** Fine management.

## Phase 9 — Reliability and advanced backend engineering

- More concurrency tests.
- Better error handling.
- Idempotency where needed.
- Audit logging.
- Redis if justified.
- Background jobs with Celery if justified.
- Performance investigation.

**Deliverable:** A more robust backend.

## Phase 10 — Production preparation

- Dockerize the application.
- Configure deployment environment.
- GitHub Actions CI.
- Logging.
- Security review.
- API documentation.
- README.
- Deployment.

**Deliverable:** Portfolio-ready project.

---

# 14. Engineering Rules for This Project

1. Understand the problem before writing code.
2. Do not create every file and module in advance.
3. Implement one small feature at a time.
4. Explain important code and design decisions.
5. Run and test code before moving forward.
6. Do not copy code blindly.
7. Do not introduce technology without a reason.
8. Use database constraints for important invariants.
9. Handle errors explicitly.
10. Write tests for business rules.
11. Keep secrets out of Git.
12. Update documentation as the project evolves.

## How we will learn

For each implementation step:

1. Explain the problem.
2. Explain the design decision.
3. Identify the files involved.
4. Implement the smallest useful piece.
5. Run it.
6. Test it.
7. Explain what happened and why.
8. Move to the next piece.

The objective is not to finish by copying a tutorial. The objective is to be able to explain and defend the system we built.

---

# 15. Reusable Blueprint Template for Future Projects

This is the part that can be reused for another project.

When starting a new project, replace the domain-specific information and answer these sections:

## A. Project vision
- What problem are we solving?
- Who are the users?
- What is the main value of the application?

## B. Functional requirements
- What can each type of user do?
- What are the main workflows?
- What is included in the MVP?

## C. Non-functional requirements
- Security requirements.
- Reliability requirements.
- Performance requirements.
- Scalability requirements.
- Maintainability requirements.

## D. Technology decisions
- Language.
- Framework.
- Database.
- ORM.
- Authentication.
- Infrastructure.
- Testing.
- Deployment.

For every technology, explain why it is needed.

## E. System architecture
- Application boundaries.
- Modules.
- Component responsibilities.
- Communication between components.
- External dependencies.

## F. Database design
- Entities.
- Attributes.
- Relationships.
- Constraints.
- Indexes.
- Transactions.
- Data lifecycle.

## G. API design
- Resources.
- Endpoints.
- HTTP methods.
- Request and response schemas.
- Error responses.
- Authentication requirements.

## H. Critical workflows
- Main user journeys.
- State transitions.
- Failure cases.
- Concurrent operations.
- Transaction boundaries.

## I. Security
- Authentication.
- Authorization.
- Data validation.
- Secrets.
- Rate limiting where appropriate.
- Auditability.

## J. Testing
- Unit tests.
- Integration tests.
- API tests.
- Security tests.
- Concurrency tests.

## K. Infrastructure
- Local development.
- Containers.
- Environment configuration.
- CI/CD.
- Deployment.
- Monitoring.

## L. Implementation roadmap
- Foundation.
- Core data model.
- Authentication.
- Core business logic.
- Advanced features.
- Testing and hardening.
- Deployment.

## M. Engineering trade-offs
For every important design decision, record:
- The problem.
- Options considered.
- Chosen solution.
- Why it was selected.
- Its disadvantages.
- When we might reconsider it.

---

# Final Project Objective

At the end, we should have more than a functioning library application.

We should have a backend that demonstrates:

- Relational database design.
- REST API development.
- Authentication and authorization.
- SQLAdmin integration.
- Database migrations.
- Transaction management.
- Concurrency control.
- Automated testing.
- Containerization.
- Production-oriented engineering decisions.

**The project is complete when we can explain not only what the code does, but why the system is designed that way.**