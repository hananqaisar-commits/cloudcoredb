# CloudCoreDB — Coding Conventions

All contributors and AI coding agents must follow these conventions when creating or modifying project files.

## 1. File & Directory Naming

- Use `snake_case` for source files and directory names.
- Use descriptive names that clearly represent their purpose.
- Keep standard root-level documentation names uppercase.

**Examples:**
- `database.py`
- `user_service.py`
- `connection_manager.py`
- `database/`
- `README.md`

## 2. Python Naming

Follow Python PEP 8 naming conventions.

| Element | Convention | Examples |
|---|---|---|
| Files | `snake_case` | `user_service.py`, `database.py` |
| Classes | `PascalCase` | `UserService`, `DatabaseManager` |
| Functions | `snake_case` | `create_user()`, `get_database()` |
| Variables | `snake_case` | `user_id`, `database_name` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_CONNECTIONS`, `DEFAULT_PORT` |

- Use 4 spaces for indentation.
- Use type hints where appropriate.
- Avoid wildcard imports and unused imports.
- Use `is_`, `has_`, or `can_` for boolean names.

## 3. Database Naming

- Table names must be lowercase `snake_case` and plural.
- Column names must be lowercase `snake_case` and singular where applicable.
- Every table should use `id` as its primary key.
- Never use reserved SQL keywords as identifiers.

**Examples:**

| Object | Convention | Examples |
|---|---|---|
| Tables | Plural | `users`, `databases`, `connections`, `sessions` |
| Primary keys | `id` | `id` |
| Foreign keys | `<table>_id` | `user_id`, `database_id`, `connection_id` |
| Columns | `snake_case` | `first_name`, `email`, `is_active` |

## 4. Foreign Keys, Indexes & Constraints

Use consistent prefixes for database constraint and index names.

| Object | Convention | Examples |
|---|---|---|
| Primary key | `pk_` | `pk_users` |
| Foreign key | `fk_` | `fk_orders_user_id` |
| Unique | `uq_` | `uq_users_email` |
| Index | `ix_` | `ix_users_email` |
| Check | `ck_` | `ck_products_price` |

**Important:** `fk_` is for the constraint name, not the column name.

Correct: `user_id`

Incorrect: `fk_user_id`

## 5. Date & Timestamp Naming

- Use `_at` for date-time columns.
- Use `_date` for date-only columns.
- Use `_time` for time-only columns.
- Store timestamps in UTC.

**Examples:**
- `created_at`
- `updated_at`
- `deleted_at`
- `expires_date`
- `birth_date`
- `start_time`
- `end_time`

Use `created_at` and `updated_at` consistently wherever applicable.

## 6. Boolean & Status Naming

- Boolean columns must have clear true/false meanings.
- Prefer `is_`, `has_`, and `can_` prefixes.
- Use `status` for values representing multiple states.

**Examples:**
- `is_active`
- `is_verified`
- `has_access`
- `can_connect`
- `status`

## 7. SQL Conventions

- Write SQL keywords in UPPERCASE.
- Use lowercase `snake_case` for identifiers.
- Use meaningful table aliases.
- Prefer explicit column names over `SELECT *`.
- Always use parameterized queries for application input.

**Examples:**
- `SELECT`
- `INSERT INTO`
- `UPDATE`
- `DELETE FROM`
- `INNER JOIN`

## 8. API Naming

- Use lowercase plural nouns for resource endpoints.
- Use consistent API versioning.
- Use HTTP methods according to their intended purpose.
- Keep request and response field naming consistent.

**Examples:**
- `/api/v1/users`
- `/api/v1/databases`
- `/api/v1/connections`
- `/api/v1/sessions`
- `/api/v1/transactions`

## 9. Environment & Security

- Environment variable names must use `UPPER_SNAKE_CASE`.
- Use the `CLOUDCOREDB_` prefix for project-specific variables.
- Never hardcode passwords, API keys, tokens, or credentials.
- Never commit a real `.env` file.
- Keep `.env` in `.gitignore`.
- Maintain `.env.example` with safe placeholder values.
- Never expose secrets in frontend code, API responses, logs, or documentation.

**Examples:**
- `CLOUDCOREDB_DATABASE_URL`
- `CLOUDCOREDB_SECRET_KEY`
- `CLOUDCOREDB_DEBUG`
- `CLOUDCOREDB_HOST`
- `CLOUDCOREDB_PORT`

## 10. Git Conventions

Use lowercase `kebab-case` for branch names.

**Branch examples:**
- `feature/user-authentication`
- `feature/database-connections`
- `fix/connection-timeout`
- `refactor/query-engine`
- `docs/api-documentation`

Keep commits descriptive and focused. Never overwrite existing uncommitted work or force-push shared branches without authorization.

---

**Consistency Rule:** Always follow existing project conventions. Do not introduce conflicting naming styles or rename existing components unnecessarily.
