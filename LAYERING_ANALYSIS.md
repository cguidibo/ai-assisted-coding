# Backend Layering Analysis: Student Enrollment System

## Summary
The code is structured as a procedural script with tightly coupled database, service, and application concerns. Database queries contain business logic, service functions orchestrate data fetching and transformation, and export logic mixes data concerns with file I/O. This creates fragile boundaries that break as requirements evolve.

---

## Detailed Findings

| **Area / Function** | **Problem Type** | **Why This Is a Problem** | **Impact on Future Changes** |
|---|---|---|---|
| **`get_student_enrollments()`** | Service logic in database layer | Filters results by `status = STATUS_ENROLLED`. This is a business rule—"what counts as enrolled"—not a data retrieval detail. A dashboard query might need different filtering; an audit log might need all records. | Adding a new enrollment status (e.g., "pending_approval") forces modifying this query. Reporting tools can't reuse it. Tests must mock the full query even when testing business logic. |
| **`get_course_by_key(enrollment_key)`** | Input normalization in wrong layer | Calls `.strip().upper()` on the enrollment key. This is input validation and canonicalization—service concern. Database layer should trust clean input or have generic parameterized queries. | Changing how enrollment keys are validated (e.g., case-sensitive, stricter format) requires modifying the query function. Can't test input rules separately from database queries. |
| **`enroll_with_key(user_id, email, enrollment_key)`** | Mixed database + service + validation | Does four things: (1) validates email format with `"@" not in email`, (2) fetches course by key, (3) inserts/updates enrollment, (4) fetches result. Combines input validation, business decision (course exists?), data mutation, and result retrieval. | Hard to test enrollment logic without database. Hard to change enrollment validation without touching data mutation. If business needs richer validation (e.g., check if student already paid), this function balloons. Can't reuse course-lookup logic in other business flows. |
| **`soft_unenroll_student(user_id, course_id)`** | Hidden business rule in database layer | Returns `bool` based on rowcount. Silently succeeds or fails. The concept of "soft" unenroll (status change vs. delete) is a business decision baked into the database layer. | If you add a new unenroll reason (e.g., "unenroll_with_refund" vs. "unenroll_no_refund"), the database layer must know about it. If you later need to track who unenrolled and why, you've already committed to a schema that only has "unenrolled" status. |
| **`seed_sample_data()`** | Test/example data coupled to database layer | Uses module-level constants `AVAILABLE_COURSE_KEYS` and `SAMPLE_ENROLLMENTS`. These are test fixtures, not database concerns. Seeding is mixed with database initialization. | Running a real database setup and a test database setup requires duplicating logic or extracting constants awkwardly. Can't have different seed data for different test scenarios without modifying the database module. Hard to separate "create schema" from "populate test data." |
| **Module-level constants** (`CURRENT_STUDENT`, `AVAILABLE_COURSE_KEYS`, `SAMPLE_ENROLLMENTS`, `STATUS_*`) | Configuration and test data in application layer | These are mixed across concerns: CURRENT_STUDENT is session state, AVAILABLE_COURSE_KEYS is seed data, STATUS_* are database enums. No clear owner. | Changing the current student requires code changes. Adding a new course requires code changes. Adding a new status requires code changes. Can't parameterize behavior. Tests and production use the same constants. |
| **`get_student_summary(user_id)`** | Service aggregation logic on top of query | Calls `get_student_enrollment_history()` and counts records by status. This is business reporting logic (i.e., "summarize this student's journey"). Implemented as a query + loop in Python instead of at the service or reporting layer. | To add a new summary metric (e.g., "time since first enrollment"), you modify this function and risk breaking the count logic. Reporting queries are scattered across functions. Can't easily build a reporting interface that needs multiple summary types. |
| **`export_database_snapshot(path)`** | Application concern in database layer | Takes data, structures it into a specific JSON format, and writes to a file. This is application/export logic, not database responsibility. The function signature couples it to a specific file path and data structure. | Changing snapshot format requires changing the database layer. Exporting to CSV or XML requires new functions in the database layer. Can't have multiple export formats. The snapshot format becomes a dependency of the database module rather than an application concern. |
| **`rows_to_dicts(rows)` utility** | Data transformation utility in wrong place | Converts SQLite Row objects to dicts. This is a data-handling utility that lives in the database module but is service-level concern (how we represent data to business logic). | If you want to represent data differently for different consumers (e.g., dicts for API, dataclass for internal logic), you have to duplicate this or pollute the database layer. Testing this utility requires importing the database module. |
| **Return type inconsistency** | Unclear contract between layers | Some functions return `list[dict]`, some return `Optional[dict]`, some return `bool`. No consistency in how errors or absence of data is signaled. `None` is used to mean "not found" and "validation failed" interchangeably. | Calling code can't distinguish "course doesn't exist" from "enrollment key was invalid." Adding new error cases (e.g., "student banned from course") requires changing return types or adding side channels. API contract is unclear. |
| **`get_available_course_keys()`, `get_student_enrollments()`, `get_student_enrollment_history()`, `get_all_enrollment_records()`** | Query functions have overlapping concerns | These four functions all query the courses and enrollments tables with different filters/joins. They're tightly coupled to the schema but with slightly different projections. There's no single source of truth for "what is a course?" or "what is an enrollment record?" | Adding a new column to courses (e.g., max_students) means updating multiple query functions. Changing the JOIN logic requires touching all of them. A new reporting view needs a new query function that probably duplicates 80% of an existing one. Testing business logic against different query results is impossible without mocking. |
| **`main()` function** | Orchestration logic defines implicit API | The `main()` function shows the intended usage pattern: create tables → seed data → fetch enrollments → enroll in course → fetch updated enrollments → get summary → export snapshot. This sequence is the only documented business flow. | If you want to unenroll a student and export a snapshot, you have to reverse-engineer how to sequence the functions. Adding a new student workflow (e.g., "show my transcript") requires guessing which functions to call. Integration tests are impossible without this orchestration code. |
| **Validation scattered across functions** | No consistent input validation layer | `enroll_with_key()` validates email; `get_course_by_key()` normalizes input; `get_student_course_record()` checks user_id and course_id; `get_student_enrollments()` checks user_id. Each function has its own guard logic. | Changing enrollment key format rules requires finding all the places that reference STATUS_* or normalize keys. New validation rules (e.g., "enrollment keys are now role-specific") must be added to multiple functions. Validation logic is tested implicitly through business function tests, not directly. |
| **Database queries + business filters combined** | Service layer filtering hidden in query results | Calls like `get_student_enrollments()` already filter by enrolled status. If you build a dashboard that needs "show me everything," you must call `get_student_enrollment_history()` instead. The query layer is making business decisions about what data to expose. | Creating a new query for an admin view ("all enrollments for all students") requires a new function. You can't compose queries. Business rules (like "active vs. unenrolled") are baked into query function names, not parameterizable. |

---

## Key Layering Violations

### 1. **Database Layer Leaking Business Rules**
- Service decisions (enrollment status, key format, what "enrolled" means) are embedded in query functions.
- The database layer shouldn't know about business concepts like "enrolled vs. unenrolled"—it should store data and run queries.

### 2. **Service Logic Scattered Without a Cohesive Service Layer**
- Enrollment logic lives in `enroll_with_key()`, summary logic in `get_student_summary()`, export logic in `export_database_snapshot()`.
- No single place where you can understand the full business workflow.
- Validation, orchestration, and data transformation are fragmented.

### 3. **Configuration and Test Data Mixed Into Code**
- `CURRENT_STUDENT`, `AVAILABLE_COURSE_KEYS`, and `SAMPLE_ENROLLMENTS` are hardcoded module-level constants.
- Seeding logic is coupled to these constants in `seed_sample_data()`.
- Can't easily swap test vs. production data or parameterize setup.

### 4. **Export/Application Logic in the Database Module**
- `export_database_snapshot()` is application-level I/O that reads from the database and writes to a file.
- It creates a specific JSON structure, which is a business/application concern.
- Couples the database module to export formats and file paths.

### 5. **Implicit Dependencies and Ordering**
- Functions depend on each other without clear contracts: `enroll_with_key()` calls `get_course_by_key()` and then `get_student_course_record()`.
- The `main()` function shows the "correct" way to call things, but this isn't documented in the API.
- New developers must reverse-engineer the intended flow.

---

## Why This Matters for Scalability, Maintainability, and Testability

| **Concern** | **Current Problem** | **Example Impact** |
|---|---|---|
| **Scalability** | Database and business logic are intertwined. Can't switch database layers or add caching without touching business code. | Adding Redis caching for course lookups requires modifying `get_course_by_key()`. Can't mock the database for load testing business logic. |
| **Maintainability** | A single business change (e.g., "enrollment keys are now case-sensitive") requires hunting through multiple functions and testing them all together. | To add a new enrollment rule (e.g., "no concurrent enrollments in conflicting courses"), you must modify both validation and business logic, possibly across multiple functions. Hard to verify all cases are covered. |
| **Testability** | Business logic can't be tested without hitting the database. Validation rules are tested implicitly. Query behavior is tightly coupled to business decisions. | Unit testing "can this student enroll?" requires setting up SQLite, seeding data, and running the full `enroll_with_key()` pipeline. Can't test validation rules in isolation. Hard to test edge cases (e.g., duplicate enrollment attempts) without database setup. |
| **Extensibility** | Adding new features (e.g., "waitlists," "concurrent enrollments," "enrollment tiers") requires modifying existing functions and likely the database layer. | To add waitlist support, you'd modify the schema, update seed data, change enroll logic, and likely add new status values—all tightly coupled changes. Can't implement waitlist as a separate service layer. |

---

## Current State: Procedural Script, Not a Backend

The code reads like a **procedural script with utilities**:
- Functions are task-oriented ("do this operation").
- Data flows through functions without clear ownership.
- Business rules are embedded in implementation details.
- No abstraction between the application intent ("enroll this student") and database operations.
- Test data is mixed with setup code, making it hard to distinguish "what should happen" from "what does happen."

This structure works for a single script or prototype but breaks when:
- You need to swap out the database (e.g., PostgreSQL instead of SQLite).
- You need to add validation or business rules without touching data queries.
- You need to test business logic without database overhead.
- You need to support multiple clients (CLI, API, Streamlit UI) with different data needs.

