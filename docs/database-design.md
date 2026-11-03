
# JulleyOnline - Database Design

## Project: Real-Time Collaborative Kanban Board

### Overview

This application supports multiple organizations,
multiple tenants (branches), and multiple users.

Each organization may contain several tenants.
Each tenant manages its own boards, columns,
tasks, and activity history.

The system must prevent unauthorized users
from accessing another tenant's private data.

### Technology

Database: PostgreSQL
ORM: Prisma
Backend: Node.js and Express.js
Real-Time Communication: Socket.IO

### Database Tables

1. organizations
2. tenants
3. users
4. memberships
5. boards
6. columns
7. tasks
8. activity_logs

---

## 1. Organizations Table

Purpose:
Stores parent company information.

Table Name: organizations

| Column | Data Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| name | VARCHAR(255) | Company name |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update |

Example Organizations:
- ABC Technologies
- XYZ Solutions

Relationship:
One organization can have multiple tenants.

organizations (1) -> (Many) tenants

Rules:
- Every organization has a unique ID.
- Organization names cannot be empty.
- Organization operations require authorization.

---

## 2. Tenants Table

Purpose:
Stores branches or workspaces belonging
to organizations.

Table Name: tenants

| Column | Data Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| organization_id | UUID | Parent organization FK |
| name | VARCHAR(255) | Branch name |
| created_at | TIMESTAMP | Creation time |

Example Tenants:

ABC Technologies:
- Hyderabad
- Bangalore
- Chennai

XYZ Solutions:
- Mumbai
- Pune

Relationship:
organizations (1) -> (Many) tenants

Foreign Key:
organization_id references organizations.id

Rules:
- Every tenant belongs to one organization.
- Tenant names should be unique within an organization.
- Tenant data must remain isolated.
- Users require authorization to access a tenant.

---

## 3. Users Table

Purpose:
Stores registered user information.

Table Name: users

| Column | Data Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| name | VARCHAR(255) | Employee name |
| email | VARCHAR(255) | Unique login email |
| password_hash | VARCHAR(255) | Hashed password |
| created_at | TIMESTAMP | Registration time |

Example Users:
- Rahul
- Priya
- Arjun
- Sneha

Relationship:
users (1) -> (Many) memberships

Security Rules:
- Every user must have a unique email.
- Passwords must never be stored as plain text.
- Passwords will be securely hashed.
- Users can belong to multiple authorized tenants.
- User permissions are determined by memberships.

---

## 4. Memberships Table

Purpose:
Connects users to organizations and tenants
and defines role-based permissions.

Table Name: memberships

| Column | Data Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| user_id | UUID | User Foreign Key |
| organization_id | UUID | Organization FK |
| tenant_id | UUID | Tenant FK, nullable |
| role | VARCHAR(30) | User role |
| created_at | TIMESTAMP | Membership time |

Supported Roles:
- ORG_ADMIN
- TENANT_ADMIN
- MEMBER

Example Memberships:

Rahul:
Organization: ABC Technologies
Role: ORG_ADMIN
Tenant: NULL (organization-wide)

Priya:
Organization: ABC Technologies
Tenant: Hyderabad
Role: TENANT_ADMIN

Arjun:
Organization: ABC Technologies
Tenant: Hyderabad
Role: MEMBER

Sneha:
Organization: ABC Technologies
Tenant: Bangalore
Role: MEMBER

Relationships:
users (1) -> (Many) memberships
organizations (1) -> (Many) memberships
tenants (1) -> (Many) memberships

Security Rules:
- Org Admin memberships use NULL tenant_id.
- Tenant Admin and Member require tenant_id.
- Tenant must belong to the selected organization.
- Duplicate memberships for the same scope are blocked.
- Users cannot assign themselves elevated roles.
- Backend must validate every protected action.

Database Constraints:
- Foreign keys for users and organizations.
- Foreign key for tenants.
- Enforce matching tenant and organization.
- Check valid role and tenant scope combinations.
- Unique organization-level memberships per user.
- Unique tenant-level memberships per user.

---

## 5. Boards Table

Purpose:
Stores project boards within each tenant.

Table Name: boards

| Column | Data Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| organization_id | UUID | Organization FK |
| tenant_id | UUID | Tenant FK |
| name | VARCHAR(255) | Board name |
| description | TEXT | Board description |
| created_by | UUID | Creator user ID |
| created_at | TIMESTAMP | Creation time |

Example Boards:

Hyderabad:
- Website Development
- Mobile App Development

Bangalore:
- CRM Development
- AI Automation

Mumbai:
- Customer Support

Relationships:
organizations (1) -> (Many) boards
tenants (1) -> (Many) boards
boards (1) -> (Many) columns

Security Rules:
- Every board belongs to exactly one tenant.
- Board organization must match tenant organization.
- Only authorized users can access boards.
- Cross-tenant board access is forbidden.
- Create, update and delete follow RBAC rules.

Database Constraints:
- Primary key on id.
- Foreign keys for related records.
- Validate organization and tenant consistency.
- Index tenant_id for faster queries.

---

## 6. Columns Table

Purpose:
Stores Kanban status columns inside boards.

Table Name: columns

| Column | Data Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| board_id | UUID | Board Foreign Key |
| tenant_id | UUID | Tenant Foreign Key |
| name | VARCHAR(100) | Column name |
| position | INTEGER | Column display order |

Default Columns:
1. To Do
2. In Progress
3. Testing
4. Done

Relationships:
boards (1) -> (Many) columns
tenants (1) -> (Many) columns
columns (1) -> (Many) tasks

Security Rules:
- Every column belongs to one board.
- Column tenant must match board tenant.
- Users must have board access.
- Cross-tenant column operations are blocked.

Ordering Rules:
- Position determines column display order.
- Position must not be negative.
- Reordering must preserve a valid sequence.

Database Constraints:
- Primary key on id.
- Foreign keys for board_id and tenant_id.
- Enforce board and tenant consistency.
- Index board_id.

---

## 7. Tasks Table

Purpose:
Stores individual work items on Kanban boards.

Table Name: tasks

| Column | Data Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| board_id | UUID | Parent board FK |
| column_id | UUID | Current column FK |
| tenant_id | UUID | Tenant Foreign Key |
| title | VARCHAR(255) | Task title |
| description | TEXT | Task description |
| assigned_to | UUID | Assigned user, nullable |
| position | INTEGER | Order within column |
| version | INTEGER | Concurrency version |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last modification |

Example Tasks:
- Design Login Page
- Develop Registration API
- Test Authentication
- Deploy Dashboard
- Fix API Validation

Relationships:
boards (1) -> (Many) tasks
columns (1) -> (Many) tasks
tenants (1) -> (Many) tasks
users (1) -> (Many) assigned tasks

Security Rules:
- Every task belongs to one tenant.
- Task board and column must belong to that tenant.
- A column must belong to the task's board.
- Task assignees require authorized tenant membership.
- Task operations require authentication and RBAC.
- Cross-tenant task access is forbidden.

Task Ordering:
- position determines order within a column.
- Drag-and-drop changes column_id and/or position.
- Moving tasks may reorder affected tasks.
- Reordering must use database transactions.
- Concurrent reordering must preserve consistent order.

Concurrency Handling:
- Every task starts with version = 1.
- Successful updates increase the version.
- Requests include the expected task version.
- Stale updates return HTTP 409 Conflict.
- A failed update must not overwrite newer changes.

Example:

Rahul opens Task A, version 1.
Priya opens Task A, version 1.

Rahul saves a change:
Task version becomes 2.

Priya attempts to save version 1:
Backend detects a conflict and rejects the update.

Additional Rules:
- Validate nonempty task titles.
- Reject invalid destination columns.
- Prevent operations on deleted or unauthorized tasks.
- Use transactions to maintain consistency.
- A board-level ordering revision or serialization
  strategy can handle concurrent moves of different tasks.

Database Indexes:
- tenant_id
- board_id
- column_id
- (tenant_id, board_id)
- (column_id, position)

---

## 8. Activity Logs Table

Purpose:
Records important changes made by users.

Table Name: activity_logs

| Column | Data Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| tenant_id | UUID | Tenant Foreign Key |
| user_id | UUID | Actor user FK |
| entity_type | VARCHAR(50) | Task, Board, Column, etc. |
| entity_id | UUID | Affected record ID |
| action | VARCHAR(50) | Type of action |
| details | JSONB | Structured change details |
| created_at | TIMESTAMP | Action timestamp |

Example Activity Logs:
- Rahul created Design Login Page.
- Priya updated the task description.
- Arjun moved a task to Testing.
- Rahul deleted an old task.

Supported Actions:
- CREATED
- UPDATED
- MOVED
- DELETED
- ASSIGNED

Relationships:
tenants (1) -> (Many) activity_logs
users (1) -> (Many) activity_logs

Security Rules:
- Every tenant activity log requires tenant_id.
- Only authorized users can view tenant logs.
- Cross-tenant history must never be exposed.
- Logs cannot be freely modified by normal users.
- Do not store passwords, tokens, or secrets.

Implementation Rules:
- Write activity logs with the business transaction.
- Record the actor, action, time and changed fields.
- Preserve logs according to the retention policy.
- Use JSONB for appropriate before/after values.
- Deleted entities may still have activity history.

Database Indexes:
- tenant_id
- user_id
- (tenant_id, created_at)

---

## 9. Complete Database Relationships

Main hierarchy:

Organization
    |
    +-- Tenants
          |
          +-- Boards
          |     |
          |     +-- Columns
          |     |     |
          |     |     +-- Tasks
          |     |
          |     +-- Tasks
          |
          +-- Activity Logs

Users
    |
    +-- Memberships
    |     |
    |     +-- Organizations
    |     +-- Tenants
    |
    +-- Assigned Tasks
    |
    +-- Activity Logs

Important:
Users are shared identities, but their access to
business records is determined by memberships.

---

## 10. Multi-Tenant Data Isolation

Tenant isolation is a mandatory requirement.

Example:

Organization: ABC Technologies

Hyderabad tenant:
- Website Development board
- Rahul
- Priya

Bangalore tenant:
- CRM Development board
- Sneha

Hyderabad-only users must not access Bangalore
private business data.

Isolation Strategy:

1. Authenticate every user.
2. Identify authorized organization memberships.
3. Verify access to the selected tenant.
4. Check role permissions for every operation.
5. Scope database queries using tenant_id.
6. Validate tenant consistency of related records.
7. Use database constraints where appropriate.
8. Reject unauthorized API requests.
9. Restrict WebSocket rooms by tenant.
10. Test all cross-tenant and cross-org access attempts.

Tenant IDs supplied by clients are not trusted.

The backend determines whether a user has access.

PostgreSQL Row-Level Security may be added as
an additional defense-in-depth measure.

---

## 11. Authentication and Authorization

Authentication:
Verifies the user's identity.

Authorization:
Determines what the user can access or change.

Authentication Plan:
- Secure password hashing.
- Login using email and password.
- JWT verification.
- Protected API endpoints.
- Secure token handling.
- Logout support.

Role-Based Access Control:

ORG_ADMIN:
- Manage organization settings.
- Manage organization tenants.
- Assign organization and tenant roles.
- Perform organization-wide permitted actions.

TENANT_ADMIN:
- Manage assigned tenant settings.
- Manage authorized tenant members.
- Create and manage tenant boards.
- Manage columns and tasks.

MEMBER:
- View authorized boards.
- Create tasks when permitted.
- Update permitted tasks.
- Move permitted tasks.
- View authorized activity history.

Every permission must be checked on the backend.

---

## 12. Real-Time Collaboration

Technology:
Socket.IO

Tenant-specific channels:

tenant:<tenant_id>

Example:
tenant:hyderabad-id
tenant:bangalore-id

Rules:
- Authenticate WebSocket connections.
- Verify tenant membership before joining rooms.
- Never allow unauthorized room joining.
- Broadcast task changes to authorized rooms.
- Revalidate authorization when necessary.
- Resynchronize after connection recovery.

Real-Time Events:
- task:created
- task:updated
- task:moved
- task:deleted
- presence:update
- board:updated

Example Workflow:

1. Rahul moves a task.
2. React sends an API request.
3. Express verifies permissions.
4. Backend checks task version.
5. PostgreSQL commits the update.
6. An activity log is recorded.
7. Socket.IO broadcasts the committed change.
8. Other authorized users update their screens.

---

## 13. Optimistic Updates and Undo/Redo

Optimistic UI:
- Show a user action immediately.
- Send the operation to the backend.
- Confirm the server result.
- Restore or reconcile state on failure.

Undo/Redo:
- Track reversible user actions.
- Submit inverse operations through the backend.
- Recheck permissions and current versions.
- Reject reversals that conflict with newer edits.
- Keep server data authoritative.

Undo/redo must not bypass tenant isolation.

---

## 14. Network Recovery

When a connection is lost:

1. Detect the disconnection.
2. Display a reconnecting state.
3. Reconnect securely.
4. Reauthenticate the socket.
5. Rejoin authorized tenant rooms.
6. Fetch the latest board state.
7. Reconcile any pending local operations.
8. Resume real-time updates.

The client must not assume every event was received.

---

## 15. Database Indexes

Recommended indexes:

tenants:
- organization_id

memberships:
- user_id
- organization_id
- tenant_id

boards:
- tenant_id

columns:
- board_id
- tenant_id

tasks:
- tenant_id
- board_id
- column_id
- (column_id, position)

activity_logs:
- tenant_id
- (tenant_id, created_at)

Indexes improve query performance but must be
evaluated against actual query patterns.

---

## 16. Database Transactions

Transactions are used for operations that must
succeed or fail together.

Example: Moving a Task

1. Validate tenant permissions.
2. Check the expected task version.
3. Validate destination column.
4. Update task and affected positions.
5. Increment the task version.
6. Insert an activity log.
7. Commit the transaction.
8. Broadcast the committed result.

If an operation fails, roll back the transaction.

---

## 17. Database Security

Security Requirements:
- Never store plain-text passwords.
- Never commit database credentials.
- Store connection settings in backend/.env.
- Keep .env excluded from Git.
- Use a restricted database user for application queries.
- Use a separate migration-capable role.
- Enforce foreign-key relationships.
- Validate all incoming inputs.
- Prevent SQL injection through parameterized queries.
- Enforce permissions on the backend.
- Consider PostgreSQL Row-Level Security.

Database User:
julleyonline_app

Database Name:
collaborative_kanban

Database Platform:
PostgreSQL 18

---

## 18. Implementation Plan

This file documents the intended design.
The database tables have not yet been created.

Next Implementation Steps:

1. Set up Prisma in the Express backend.
2. Configure the PostgreSQL connection.
3. Define Prisma models for all eight tables.
4. Add keys, relationships and indexes.
5. Implement additional database constraints.
6. Generate and review the migration.
7. Run the database migration.
8. Verify the tables in pgAdmin.
9. Insert realistic sample data.
10. Test tenant isolation and data integrity.

---

## Conclusion

The JulleyOnline database supports:

- Multi-organization architecture.
- Multi-tenant workspaces.
- Secure user memberships and roles.
- Kanban boards and task ordering.
- Task versioning for concurrency.
- Activity tracking.
- Tenant-level data isolation.
- Real-time collaboration.
- Network recovery and undo/redo.

The actual implementation will be verified using
PostgreSQL, Prisma migrations, backend APIs,
automated tests and real browser demonstrations.
