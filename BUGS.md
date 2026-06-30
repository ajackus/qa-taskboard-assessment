# TaskBoard Assessment — Top 4 Bugs Report

This document reports the top 4 findings in the TaskBoard application codebase, prioritized by business impact.

---

### Bug 1: SQL Injection Vulnerability in Task Search API
* **File & Line Reference**: [src/app/api/projects/[id]/tasks/route.ts:27-34](file:///c:/Users/piyus/OneDrive/Documents/Anjali%20Yadav/qa-taskboard-assessment/src/app/api/projects/%5Bid%5D/tasks/route.ts#L27-L34)
* **Category**: Security
* **Severity**: Critical
* **Description**: The task search query param `q` is directly interpolated into a raw SQL query string and executed using `prisma.$queryRawUnsafe()`. This exposes the application to SQL Injection, allowing malicious users to extract sensitive data, manipulate other tables, or bypass permissions.
* **Curl Proof**:
  ```bash
  curl -X GET "http://localhost:3000/api/projects/cm1234/tasks?q='%20UNION%20SELECT%20id,owner_id,name,description,null,null,null,0,null,null%20FROM%20projects%20--" \
    -H "Authorization: Bearer <token>"
  ```

---

### Bug 2: Password Hash Leakage in Project Details API
* **File & Line Reference**: [src/app/api/projects/[id]/route.ts:25-40](file:///c:/Users/piyus/OneDrive/Documents/Anjali%20Yadav/qa-taskboard-assessment/src/app/api/projects/%5Bid%5D/route.ts#L25-L40)
* **Category**: Security
* **Severity**: High
* **Description**: The project detail API retrieves `owner: true` and `memberships: { include: { user: true } }` without specifying selected fields. This leaks the `passwordHash` of the project owner and all project members to any project viewer, enabling offline dictionary/brute-force attacks.
* **Curl Proof**:
  ```bash
  curl -X GET "http://localhost:3000/api/projects/<projectId>" \
    -H "Authorization: Bearer <token>"
  # Response includes:
  # "owner": { "id": "...", "email": "...", "passwordHash": "$2a$10$..." }
  ```

---

### Bug 3: Database-Level Duplicate Email Constraint Missing (Race Condition Risk)
* **File & Line Reference**: [prisma/schema.prisma:23-37](file:///c:/Users/piyus/OneDrive/Documents/Anjali%20Yadav/qa-taskboard-assessment/prisma/schema.prisma#L23-L37) and [src/app/api/auth/register/route.ts:17-20](file:///c:/Users/piyus/OneDrive/Documents/Anjali%20Yadav/qa-taskboard-assessment/src/app/api/auth/register/route.ts#L17-L20)
* **Category**: Data Integrity / Security
* **Severity**: High
* **Description**: The `User` model does not enforce a uniqueness constraint or index on the `email` field at the database level (`email String` instead of `email String @unique`). While the registration endpoint attempts a check before insertion, it does not prevent duplicate emails under concurrent registration requests (race condition), which corrupts login semantics.
* **Curl Proof**:
  ```bash
  # Attempt concurrent POST requests to register two users with the same email:
  curl -X POST http://localhost:3000/api/auth/register \
    -H "Content-Type: application/json" \
    -d '{"email":"duplicate@example.com","password":"password123","name":"First"}'
  curl -X POST http://localhost:3000/api/auth/register \
    -H "Content-Type: application/json" \
    -d '{"email":"duplicate@example.com","password":"password123","name":"Second"}'
  # Both accounts will be successfully registered in the database.
  ```

---

### Bug 4: Search API Response Field Mismatch (camelCase vs snake_case) and Missing Assignee
* **File & Line Reference**: [src/app/api/projects/[id]/tasks/route.ts:27-36](file:///c:/Users/piyus/OneDrive/Documents/Anjali%20Yadav/qa-taskboard-assessment/src/app/api/projects/%5Bid%5D/tasks/route.ts#L27-L36)
* **Category**: Architecture / Data Integrity
* **Severity**: Medium
* **Description**: The search query returns raw database records with database-mapped `snake_case` fields (such as `project_id`, `assignee_id`) rather than Prisma's standard `camelCase` properties. This mismatch breaks the client-side expectations and metadata update logic. In addition, the query lacks a join for the `assignee` model, causing all cards in search results to render as "unassigned" in the UI.
* **UI Finding Format**:
  > When searching for a task using the search bar, the app returns matching tasks but displays them with unassigned assignees and broken fields, but it should return them with proper assignee names and camelCase fields matching the board representation.
* **Curl Proof**:
  ```bash
  curl -X GET "http://localhost:3000/api/projects/<projectId>/tasks?q=Draft" \
    -H "Authorization: Bearer <token>"
  # Response fields contain: "project_id", "assignee_id" instead of "projectId", "assigneeId", and "assignee" is missing.
  ```
