# CloudCoreDB Architecture

**What this file is:** the one place that explains how CloudCoreDB is built, and why.
Naming rules are not repeated here. They live in the Coding Conventions file, and every
example below follows them (primary key `id`, foreign keys like `project_id`, constraint
prefixes `pk_` `fk_` `uq_` `ix_` `ck_`, endpoints under `/api/v1/`, timestamps in UTC).

**How to read it:** three levels. Each level zooms in one step.

| Level | Read it when | Time |
|---|---|---|
| 1. Big picture | you are new to the project | 2 min |
| 2. Each layer | you are about to work on one layer | 10 min |
| 3. Real traces | you want to see real values flow through the system | 15 min |

---

## Level 1: The big picture

### The four layers

```
   User
    |   clicks "Create VM"
    v
+------------------------------+
| 1. FRONTEND                  |   shows data, collects input
+--------------+---------------+
               |   HTTP + JSON   (/api/v1/...)
               v
+------------------------------+
| 2. API  (FastAPI, Python)    |   checks the token, checks the input shape,
|                              |   calls ONE stored procedure
+--------------+---------------+
               |   procedure call
               v
+------------------------------+
| 3. DATABASE  (MySQL)         |   enforces every business rule:
|                              |   quota, capacity, billing, access
+------------------------------+

  4. INFRASTRUCTURE (under all three): Git scripts, env variables, CI
```

### What each layer decides

| Layer | It decides | It never does |
|---|---|---|
| Frontend | how data looks, which page is shown | talk to MySQL, hold secrets, enforce rules |
| API | is the request well-formed? is the caller logged in? | check quota or capacity, compute prices, change rows with its own logic |
| Database | is the request allowed? what is the new state? | know about HTTP, JSON or pages |
| Infrastructure | how the system is built, versioned, configured, tested | contain business rules |

### The golden rule

> **Fat database, thin API.**
> If you are unsure where a rule belongs, put it in the database.

The reason is technical, not stylistic: a rule that needs "read a value, decide, then
write" must run inside one database transaction. If it runs in Python, two requests can
read the same value before either one writes (see Trace C in Level 3).

### Quick test: which layer owns this?

| Question | Layer | Why |
|---|---|---|
| Is `flavor_id` a number? | API | it is the shape of the input |
| Is this email already registered? | Database | a UNIQUE constraint is race-free; an API check is not |
| Does the project still have CPU quota? | Database | needs a consistent read and write together |
| Is the session token expired? | Database | the `sessions` table holds the expiry time |
| What colour is the "suspended" badge? | Frontend | pure presentation |
| Is the password correct? | Database | hash and salt are compared next to the data |

---

## Level 2: Each layer in detail

### 2.1 Frontend

- Talks only to `/api/v1/...`. It never knows MySQL exists.
- Keeps the session token in memory and sends it with every request.
- Shows the `code` and `message` that the API returns when something is rejected.
- Pages follow the modules: Login, Projects, VMs, Network, Billing.
- Built with AI and not graded, so keep it replaceable: nothing important may live here.

### 2.2 API layer (FastAPI)

The API has four small parts. Each part has exactly one job.

| Part | Example file | Job | Example |
|---|---|---|---|
| Router | `api/routers/vms.py` | Declares the endpoints of one module and maps URL + method to a service function | `POST /api/v1/vms` goes to `create_vm` |
| Schema | `api/schemas/vm.py` | Describes the exact shape of a request and response. A wrong shape is rejected automatically with HTTP 422, before any service code runs | create-VM request needs `project_id`, `flavor_id`, `image_id`, `subnet_id` (integers) and `vm_name` (text) |
| Service | `api/services/vm_service.py` | One function per use case. Reads the token, calls the right stored procedure, translates the result | `create_vm` calls the procedure `place_vm` |
| Connection manager | `api/db/connection_manager.py` | Owns the MySQL connection pool. Lends one connection per request and takes it back afterwards | a pool of 10 connections shared by all requests |

**What the API does with one request:**

1. The router matches the URL and method.
2. The schema validates the shape of the body.
3. The service takes the token and the validated values.
4. The service borrows a connection and calls one stored procedure. Values are passed as
   parameters, never glued into a SQL string, so SQL injection is not possible.
5. The service reads the result row or the error code.
6. The router turns that into an HTTP response.

**Example endpoints per module** (plural nouns, versioned; multi-word resources use a hyphen):

| Module | Endpoints |
|---|---|
| IAM | `POST /api/v1/sessions` (login), `DELETE /api/v1/sessions/current` (logout), `GET /api/v1/users` |
| Allocation | `POST /api/v1/vms`, `GET /api/v1/vms`, `DELETE /api/v1/vms/{id}` |
| Networking | `GET /api/v1/vpcs`, `POST /api/v1/subnets`, `POST /api/v1/security-groups`, `POST /api/v1/reachability-checks` |
| Billing | `GET /api/v1/invoices`, `POST /api/v1/payments` |
| Auto scaling | `GET /api/v1/scaling-groups` |

**How database errors become HTTP responses** (one table, used by every endpoint):

| Code raised by the database | HTTP status | Meaning |
|---|---|---|
| `INVALID_SESSION` | 401 | token missing, wrong or expired |
| `PERMISSION_DENIED` | 403 | the role does not have this permission |
| `ACCOUNT_LOCKED` | 403 | too many failed logins |
| `NOT_FOUND` | 404 | the id does not exist |
| `QUOTA_EXCEEDED` | 409 | the project limit is reached |
| `NO_CAPACITY` | 409 | no server has enough free resources |
| (invalid request shape) | 422 | rejected by the schema, the database was never called |

### 2.3 Database layer (MySQL)

Everything that must always be true lives here. There are six kinds of objects:

| Object | Purpose | Example in CloudCoreDB |
|---|---|---|
| Table | stores facts | `vms`, `servers`, `projects` |
| Constraint | a rule that is always true; MySQL refuses any row that breaks it | `ck_servers_used_cpu` (used CPU never above total CPU), `fk_vms_project_id` (a VM must have a real project), `uq_users_email` |
| Stored procedure | a multi-step decision that runs inside one transaction | `place_vm`, `allocate_ip`, `can_connect`, `generate_invoice` |
| Trigger | an automatic side effect when a row is inserted or updated | write an `audit_log` row whenever a VM is created |
| Event | a scheduled job inside MySQL | hourly `usage_log` rows, auto scaling check every minute, daily suspend of unpaid projects |
| View | a saved read-only query | `vw_server_capacity` (free CPU per region) |

**Transaction in one sentence:** a group of changes that either all happen or none
happen. `place_vm` starts one, takes a row-level lock on the chosen server, checks, writes,
and then commits. If any check fails, it rolls everything back.

**Three safety nets against overbooking** (each one catches what the previous one misses):

1. The row-level lock makes two requests for the same server wait for each other.
2. The transaction makes sure a half-finished placement can never be saved.
3. The check constraint `ck_servers_used_cpu` refuses any update that pushes used CPU
   above total CPU, even if a bug skips the first two.

### 2.4 Infrastructure

- **Scripts in Git:** the database is rebuilt from numbered `.sql` scripts. The database
  file itself is never committed.
- **Configuration:** environment variables with the `CLOUDCOREDB_` prefix, for example
  `CLOUDCOREDB_DATABASE_URL` (how the API reaches MySQL), `CLOUDCOREDB_SECRET_KEY`,
  `CLOUDCOREDB_DEBUG`, `CLOUDCOREDB_HOST` and `CLOUDCOREDB_PORT` (where the API listens).
  The real `.env` is never committed; `.env.example` holds placeholders.
- **CI (GitHub Actions):** on every pull request, start a fresh MySQL, run every script
  in order, and fail the check if any script errors out.

---

## Level 3: Real traces

All traces use the same example data.

| Table | Row | Values |
|---|---|---|
| `customers` | `id` 1 | Ali Tech |
| `users` | `id` 1 | Ali, role engineer (has permission `vm:create`) |
| `projects` | `id` 10 | Ali Website |
| `project_quotas` | `project_id` 10 | `max_vms` 10, `max_cpu` 40 |
| current use | project 10 | 3 VMs running, 12 CPU in use |
| `flavors` | `id` 3 | large: 4 CPU, 8 GB RAM, 100 GB disk |
| `images` | `id` 2 | Ubuntu 24.04 |
| `subnets` | `id` 7 | 10.0.1.0/24, in availability zone `lhr-az1` |
| `servers` | `id` 501 | in a rack of `lhr-az1`, `total_cpu` 64, `used_cpu` 14, active |

### Trace A: creating a VM (success)

**Request:** `POST /api/v1/vms` with the session token in the header and these fields:

| Field | Value |
|---|---|
| `project_id` | 10 |
| `flavor_id` | 3 |
| `image_id` | 2 |
| `subnet_id` | 7 |
| `vm_name` | web-4 |

| # | Layer | What happens |
|---|---|---|
| 1 | Frontend | User fills the form and sends the request |
| 2 | API router | Matches `POST /api/v1/vms` and selects `create_vm` |
| 3 | API schema | Four ids are integers, `vm_name` is text: the shape is valid |
| 4 | API service | Takes a pooled connection and calls `place_vm` with the token and the five values |
| 5 | Database (IAM) | The session is valid and not expired. The role engineer has `vm:create`. |
| 6 | Database (Allocation) | Quota check: VMs 3 + 1 = 4, not above 10. CPU 12 + 4 = 16, not above 40. |
| 7 | Database (Networking, Infrastructure) | Subnet 7 is in `lhr-az1`, so only active servers in racks of `lhr-az1` are candidates |
| 8 | Database (Infrastructure) | Locks the row of server 501. Free CPU is 64 - 14 = 50, which is at least 4. RAM and disk are also enough. |
| 9 | Database | Inserts the VM (`id` 900, status running). Updates `used_cpu` on server 501 from 14 to 18. Writes a `vm_events` row "started". `allocate_ip` takes 10.0.1.5 from subnet 7. A trigger writes an `audit_log` row. |
| 10 | Database | Commits and returns VM id 900 with IP 10.0.1.5 |
| 11 | API | Turns the result into HTTP 201 Created |
| 12 | Frontend | Shows web-4 as running with its IP |

**State after:** server 501 has `used_cpu` 18 (46 free). Project 10 now uses 4 VMs and 16 CPU.

### Trace B: creating a VM (rejected by quota)

Project 11 has `max_cpu` 8 and already uses 8 CPU. The user asks for a large VM (4 CPU).

| # | Layer | What happens |
|---|---|---|
| 1 | API | Shape is valid, so `place_vm` is called |
| 2 | Database | Quota check: 8 + 4 = 12, which is above 8. It raises `QUOTA_EXCEEDED` and rolls back. |
| 3 | API | Maps `QUOTA_EXCEEDED` to HTTP 409 with the message "Project CPU limit reached (8 of 8 used)" |
| 4 | Frontend | Shows the message next to the form |

**State after:** nothing changed. No VM row, no change in `servers`. The rollback guarantees that.

### Trace C: two users at the same time (why the rule lives in the database)

Server 501 is the only candidate and has exactly 4 CPU free (`total_cpu` 64, `used_cpu` 60).
Ali (R1) and Sara (R2) each ask for a large VM (4 CPU) within a few milliseconds.

**Wrong design: the check lives in Python**

| Time | R1 | R2 | `used_cpu` in database |
|---|---|---|---|
| t1 | reads used CPU: 60, so 4 are free | | 60 |
| t2 | | reads used CPU: 60, so 4 are free | 60 |
| t3 | writes used CPU = 64 | | 64 |
| t4 | | writes used CPU = 64 | 64 |

Both VMs exist. 8 CPU were handed out, but the database says 64 of 64. This is a lost
update: the system now lies about its own capacity.

**Right design: the check lives in `place_vm`**

| Time | R1 | R2 | Note |
|---|---|---|---|
| t1 | starts a transaction, locks server 501 | starts a transaction, asks for the lock on 501 and waits | |
| t2 | free CPU is 4, inserts the VM, `used_cpu` 60 to 64 | waiting | |
| t3 | commits, lock released | gets the lock and reads `used_cpu` 64 | |
| t4 | HTTP 201 Created | free CPU is 0, no other server in the zone, raises `NO_CAPACITY`, rolls back | R2 gets HTTP 409 |

**Safety net 3 in action:** if a bug skipped the lock and tried to write `used_cpu` 68,
the constraint `ck_servers_used_cpu` would refuse the update because 68 is above 64.

### Trace D: login and lockout

| Attempt | Password | What the database does | HTTP |
|---|---|---|---|
| 1 to 4 | wrong | writes a `login_attempts` row with `is_successful` false | 401 |
| 5 | wrong | writes the row, then sets `users.locked_until` to now + 15 minutes (UTC) | 401 |
| 6 (2 minutes later) | correct | refuses, because `locked_until` is in the future. Raises `ACCOUNT_LOCKED`. | 403 |
| 7 (after 15 minutes) | correct | writes a successful attempt, creates a `sessions` row with a random token and an expiry | 201 with the token |

The API never counts failures itself. It only forwards the password and reports the
code that the database returns.

### Trace E: can VM A reach VM B? (Networking)

web-1 (VM 900, 10.0.1.5 in `web-subnet`) wants to reach db-1 (VM 901, 10.0.2.7 in
`db-subnet`) on tcp port 3306. Both subnets are in VPC 10.0.0.0/16.

**Request:** `POST /api/v1/reachability-checks` with `source_vm_id` 900, `destination_vm_id` 901, `port` 3306, `protocol` tcp.

| # | Check | Result |
|---|---|---|
| 1 | Route: both subnets are in the same VPC, so the local route applies | pass |
| 2 | Network ACL of `db-subnet`, inbound rules in rule-number order: rule 100 allows tcp 3306 from 10.0.1.0/24 | pass |
| 3 | Security group of db-1 (`db-sg`): its only inbound rule allows tcp 22 from the admin range. Nothing allows 3306. | blocked |

The database writes a `reachability_log` row with `result` blocked and `blocked_by` "db-sg: no inbound rule for tcp 3306".
The API answers **HTTP 200**: the question was answered, and "blocked" is an answer, not an error.

**After the fix:** a rule is added to `db-sg` that allows tcp 3306 from `web-sg`. Running the same check now passes all three steps and logs `reachable`.

---

## Repository layout

```
cloudcoredb/
  database/
    infrastructure/      regions, availability zones, racks, servers
    iam/
    allocation/
    networking/
    billing/
    auto_scaling/
    foreign_keys/        cross-module foreign keys, added after all tables exist
    seed/                reference_data.sql, demo_data.sql
  api/
    routers/             one file per module
    schemas/             request and response shapes
    services/            one function per use case
    db/                  connection_manager.py
  frontend/
  docs/
    er_diagrams/
    team_contract.md     table names, procedure signatures: agreed BEFORE coding
  .github/workflows/     CI
  .env.example
  .gitignore
  README.md
  ARCHITECTURE.md
```

**Build order:** infrastructure, iam, allocation, networking, billing, auto_scaling,
foreign_keys, seed. Each module creates its own tables first. Foreign keys that point to
another module are added in `foreign_keys/` after every table exists.

**Git flow (see the conventions file for branch and commit rules):**

```
git switch main
git pull
git switch -c feature/networking-vpcs
git add database/networking/
git commit -m "add vpcs table"
git push -u origin feature/networking-vpcs
```

Then open a pull request, get one review, wait for CI to pass, merge, delete the branch.
A script that was already merged is never edited: add a new numbered script instead.

---

## Rules that keep the architecture honest

| Do | Do not |
|---|---|
| Put every business rule in a constraint or a stored procedure | Re-implement a quota, capacity or price check in Python |
| Let the API call one procedure per use case | Let the API run several loose queries and combine the results |
| Pass values as parameters | Build SQL strings by joining text |
| Return a code from the database and map it to an HTTP status in one place | Invent a new error format in each endpoint |
| Add a new numbered script for every schema change | Edit a script that is already merged |
| Keep secrets in `.env` | Commit passwords, keys or tokens |
| Keep the frontend replaceable | Put pricing or permission rules in the frontend |

---

## Glossary

| Term | Meaning |
|---|---|
| Endpoint | one URL + HTTP method that the API exposes |
| Router | the part of the API that groups endpoints of one module |
| Schema (API) | the declared shape of a request or response, checked automatically |
| Service function | one function that implements one use case |
| Connection pool | a fixed set of open database connections that requests borrow and return |
| Transaction | a group of changes that all happen or none happen |
| Row-level lock | makes other transactions wait before they touch the same row |
| Constraint | a rule that the database enforces on every row |
| Stored procedure | a named, multi-step routine stored inside MySQL |
| Trigger | code that MySQL runs automatically when a row changes |
| Event | a job that MySQL runs on a schedule |
| View | a saved read-only query |
| Migration script | a numbered `.sql` file that changes the structure of the database |
| Seed script | a `.sql` file that inserts data (reference or demo) |
| Lost update | two requests overwrite each other's change because both read the old value |

