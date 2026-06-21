Bug 1- SQL injection in task search
File: /Users/swatigaur/live-assessment/qa-taskboard-assessment/src/app/api/projects/[id]/tasks/route.ts, line 27-34
Category: Security
Severity: Critical

The q search paraameter from the URL is concatenated directly into a raw sql string and executed with queryRawUnsafe with no parameterisation. An attacker whoc is a project member can inject a tautology to bypass the project filter and read tasks from every project. A more destructive payload could exfilterate the users table or drip data entirely.


Bug 2: - PATCH/api/tasks//[id]/route.ts, line no. 16-37
Categroy: Security
Severitu: Critical 

The PATCH handler check authentication but skips both the project membership check and role check before writing to the database. The sibiling DELETE handler in the same file correctly check both-- PATCH simply omitted them. Any authenticated user including viewers and users from unrelated projects can overwite any tasks.
 
Bug 3: Password hash in project detail response
File: src/app/api/project/[id]/route.ts 
category: Security
Severity: high

The project detail endpoint uses iunclude owner true and include user true in the prisma query which selects every column from the users table including passwordHash. Every project member who opens a project page recevies the bcrypt hashes of all the other members in the json response visible in browser devtools.

Bug 4: Viewer sees edit and delete controls in task modal

File: src/compomtes/TaskDetails.tsx 
Severity: high
Category: Security

When a viewer clicks on any task  card, the app opens a full edit modal and editable fields and save and delete buttons, but it should show a ready-only view since viewer have no write permission. The component receives no role prop and has no permission awareness- a viewer who clicks save or delete receives a generic error after the fact instead of never seeing those control at all.


