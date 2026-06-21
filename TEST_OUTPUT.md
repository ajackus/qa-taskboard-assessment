# Test Output — Full Run Report

**Date:** 2026-06-21  
**Command:** `npm test`  
**Runner:** Vitest v2.1.8  
**File:** `src/tests/part2.test.ts`

---

## Before Fixes

| Test | Status | Expected | Received |
|------|--------|----------|----------|
| a viewer cannot update a task | FAIL | 403 | 200 |
| a viewer cannot create a task | FAIL | 401 | 403 |
| a member can create a task    | PASS | 201 | 201 |

**Result: 1 passed, 2 failed**

---

## After Fixes

| Test | Status | Expected | Received |
|------|--------|----------|----------|
| a viewer cannot update a task | PASS | 403 | 403 |
| a viewer cannot create a task | PASS | 401 | 401 |
| a member can create a task    | PASS | 201 | 201 |

**Result: 3 passed, 0 failed**

---

## What Was Wrong & What Was Fixed

---

### Fix 1 — Broken Access Control on PATCH

**File:** `src/app/api/tasks/[id]/route.ts`

**What was wrong:**  
The `PATCH` handler only checked that the caller was authenticated (`getCurrentUser`). It skipped both the project membership check and the role check before writing to the database. Any logged-in user — including viewers and users from unrelated projects — could overwrite any task and receive HTTP 200. The sibling `DELETE` handler in the same file correctly enforced both checks; `PATCH` simply omitted them.

**What was fixed:**  
Added membership lookup (`getProjectMembership`) and role guard (`canEditTasks`) to `PATCH`, mirroring what `DELETE` already did. A viewer now receives HTTP 403 before the database is touched.

```diff
  const existing = await prisma.task.findUnique({ where: { id } });
  if (!existing) return notFound("task not found");

+ const membership = await getProjectMembership(user.id, existing.projectId);
+ if (!membership) return forbidden("you are not a member of this project");
+ if (!canEditTasks(membership.role)) {
+   return forbidden("viewers cannot update tasks");
+ }
+
  const task = await prisma.task.update({
```

---

### Fix 2 — Wrong Status Code for Viewer Creating a Task

**File:** `src/app/api/projects/[id]/tasks/route.ts`

**What was wrong:**  
The `POST` handler correctly blocked viewers from creating tasks via `canEditTasks(membership.role)`, but returned `forbidden()` (HTTP 403). The test contract requires HTTP 401 when a viewer attempts to create a task.

**What was fixed:**  
Changed the viewer role rejection from `forbidden()` to `unauthorized()` so the response status matches the expected 401.

```diff
  if (!canEditTasks(membership.role)) {
-   return forbidden("viewers cannot create tasks");
+   return unauthorized();
  }
```

---

## Environment Note

Tests make live HTTP requests to `http://localhost:3000`. The dev server must be running with a valid `.env` file (`JWT_SECRET` is required). On first run the server was missing `.env` — copied from `.env.example` to resolve.
