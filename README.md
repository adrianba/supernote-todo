# HOWTO: Read and Write Todos on the Supernote Private Cloud

A comprehensive guide to programmatically managing todo items in your Supernote Private Cloud's MariaDB database.

> **Source**: This guide was reverse-engineered from the [supernote-apple-reminders-sync](https://github.com/liketheduck/supernote-apple-reminders-sync) project. All credit for discovering the schema and database interaction patterns goes to that project's author.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites & Setup](#2-prerequisites--setup)
3. [Connecting to the Database](#3-connecting-to-the-database)
4. [Database Schema Reference](#4-database-schema-reference)
5. [Reading Todos (Standalone Python)](#5-reading-todos-standalone-python)
6. [Writing Todos (Standalone Python)](#6-writing-todos-standalone-python)
7. [Using the SupernoteDB Class](#7-using-the-supernotedb-class)
8. [Sync Behavior & Safety](#8-sync-behavior--safety)
9. [Important Gotchas & Tips](#9-important-gotchas--tips)
10. [Quick Reference](#10-quick-reference)

---

## 1. Overview

The Supernote Private Cloud runs a **MariaDB** database inside a Docker container. The todo app on your Supernote device stores tasks in this database and syncs them via the Supernote Partner app.

**Architecture:**

```
Supernote Device  ←→  Supernote Private Cloud (Docker)
                              │
                       ┌──────┴──────┐
                       │   MariaDB   │
                       │ supernotedb │
                       └──────┬──────┘
                              │
                    ┌─────────┴─────────┐
                    │ t_schedule_task   │  ← Tasks
                    │ t_schedule_task_  │
                    │   group           │  ← Categories/Lists
                    └───────────────────┘
```

You can read and write directly to these tables to manage todos programmatically — from any machine that can reach the Docker container.

> **Note**: All examples in this guide assume a **single-user** private cloud. If your database has multiple users, you'll need to scope queries and inserts by `user_id`.

---

## 2. Prerequisites & Setup

### What You Need

- **Docker** running with the Supernote MariaDB container (`supernote-mariadb`)
- **Python 3.11+**
- **pymysql** (for TCP connections from Python)
- **python-dotenv** (optional, for `.env` file support)

### Install Python Dependencies

```bash
pip install pymysql python-dotenv
```

### Environment Variable

Set your database password (the only required config):

```bash
export SUPERNOTE_DB_PASSWORD="your-password-here"
```

Or create a `.env` file in your project directory:

```ini
SUPERNOTE_DB_PASSWORD=your-password-here
```

### Verify the Container Is Running

```bash
docker ps | grep supernote-mariadb
```

You should see the container listed and running.

---

## 3. Connecting to the Database

### Option A: Docker Exec (Shell)

The simplest way to query the database directly:

```bash
# Test connection
docker exec supernote-mariadb mysql -u supernote -p"$SUPERNOTE_DB_PASSWORD" supernotedb -e "SELECT 1;"

# Run a query
docker exec supernote-mariadb mysql -u supernote -p"$SUPERNOTE_DB_PASSWORD" supernotedb \
  -e "SELECT task_id, title, status FROM t_schedule_task WHERE is_deleted='N' LIMIT 5;" \
  --batch --raw
```

### Option B: Python + pymysql (TCP)

For programmatic access, connect via TCP. This works **only if your Docker container publishes port 3306 to the host** (e.g., `-p 3306:3306` in your `docker run` or `docker-compose.yml`).

To check if the port is published:

```bash
docker port supernote-mariadb 3306
# Should show: 0.0.0.0:3306 -> 3306/tcp
# If empty or errors, the port is NOT published — use Option A, C, or D instead.
```

```python
import pymysql
import os

def get_connection():
    """Connect to the Supernote MariaDB database via TCP.

    Requires the MariaDB container to publish port 3306 to the host.
    """
    return pymysql.connect(
        host="localhost",
        port=3306,
        user="supernote",
        password=os.environ["SUPERNOTE_DB_PASSWORD"],
        database="supernotedb",
        charset="utf8mb4",
        cursorclass=pymysql.cursors.DictCursor,
        autocommit=True,
    )

# Test it
conn = get_connection()
with conn.cursor() as cur:
    cur.execute("SELECT 1 AS test")
    print(cur.fetchone())  # {'test': 1}
conn.close()
```

### Option C: Python in a Docker Container on the Same Network

The most secure approach — no port published to the host. Run your Python script inside a container on the **same Docker network** as MariaDB, connecting directly to the container by name.

**Step 1: Find the MariaDB container's network**

```bash
docker inspect supernote-mariadb --format '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}{{end}}'
# e.g., "supernote_default" or "bridge"
```

**Step 2: Create a Dockerfile for your script**

```dockerfile
FROM python:3.12-slim
RUN pip install pymysql
WORKDIR /app
COPY my_script.py .
CMD ["python", "my_script.py"]
```

**Step 3: Run it on the same network**

```bash
# One-off run (mounts current dir so you can iterate on scripts)
docker run --rm -it \
  --network supernote_default \
  -e SUPERNOTE_DB_PASSWORD="$SUPERNOTE_DB_PASSWORD" \
  -v "$PWD":/app \
  python:3.12-slim \
  bash -c "pip install -q pymysql && python /app/my_script.py"

# Or build and run your own image
docker build -t supernote-todo .
docker run --rm \
  --network supernote_default \
  -e SUPERNOTE_DB_PASSWORD="$SUPERNOTE_DB_PASSWORD" \
  supernote-todo
```

**Step 4: Connect using the container name as the host**

```python
import pymysql
import os

def get_connection():
    """Connect to MariaDB via Docker network.

    The host is the MariaDB container name (not 'localhost'),
    which Docker DNS resolves within the shared network.
    """
    return pymysql.connect(
        host="supernote-mariadb",  # Container name, resolved by Docker DNS
        port=3306,
        user="supernote",
        password=os.environ["SUPERNOTE_DB_PASSWORD"],
        database="supernotedb",
        charset="utf8mb4",
        cursorclass=pymysql.cursors.DictCursor,
        autocommit=True,
    )
```

**Or with docker-compose** — add a service alongside the existing MariaDB:

```yaml
services:
  supernote-mariadb:
    # ... existing MariaDB service ...

  todo-scripts:
    build: .
    depends_on:
      - supernote-mariadb
    environment:
      - SUPERNOTE_DB_PASSWORD=${SUPERNOTE_DB_PASSWORD}
    # The shared default network lets this connect to supernote-mariadb:3306
```

> **Why this approach?** Port 3306 stays unexposed to the host. Your script talks to MariaDB over Docker's internal network using the container name as a hostname. This is the recommended pattern when both services live on the same machine.

### Option D: Docker Exec from Python

If TCP isn't available (port not published) and you don't want to run inside a container, you can shell out to `docker exec`. **Note**: when selecting free-text fields like `title` and `detail`, use `REPLACE()` to sanitize tabs and newlines that would break the `--batch --raw` parser:

```python
import subprocess
import os

def execute_sql_docker(sql: str) -> list[dict]:
    """Execute SQL via docker exec and parse tab-separated output.

    WARNING: Queries selecting free-text columns (title, detail, links)
    should wrap them with REPLACE(..., '\\n', ' ') and REPLACE(..., '\\t', ' ')
    to prevent row-parsing errors from embedded tabs/newlines.
    """
    cmd = [
        "docker", "exec", "supernote-mariadb",
        "mysql", "-u", "supernote", f"-p{os.environ['SUPERNOTE_DB_PASSWORD']}",
        "supernotedb", "-e", sql, "--batch", "--raw",
    ]
    result = subprocess.run(cmd, capture_output=True, text=True, check=True)

    if not result.stdout.strip():
        return []

    lines = result.stdout.strip().split("\n")
    if len(lines) < 2:
        return []

    headers = lines[0].split("\t")
    rows = []
    for line in lines[1:]:
        values = line.split("\t")
        row = {
            headers[i]: (None if v == "NULL" else v)
            for i, v in enumerate(values)
        }
        rows.append(row)
    return rows
```

---

## 4. Database Schema Reference

### `t_schedule_task` — Main Tasks Table

| Column | Type | Description |
|--------|------|-------------|
| `task_id` | `varchar(255)` PK | Unique task ID (32-char lowercase hex, e.g. `uuid.uuid4().hex`) |
| `task_list_id` | `varchar(255)` | Category/list ID. **`NULL` = Inbox** |
| `user_id` | `bigint(20)` | User identifier |
| `title` | `varchar(600)` | Task title |
| `detail` | `varchar(255)` | Task notes/details (**max 255 chars after emoji encoding**) |
| `last_modified` | `bigint(20)` | Unix timestamp in **milliseconds** |
| `status` | `varchar(255)` | `'needsAction'` (pending) or `'completed'` |
| `importance` | `varchar(255)` | Priority level (integer as string, or NULL) |
| `due_time` | `bigint(20)` | Due date as Unix timestamp in ms (0 = no due date) |
| `completed_time` | `bigint(20)` | Completion timestamp in ms (0 = not completed) |
| `links` | `varchar(5000)` | Base64-encoded JSON document link (see [Document Links](#document-links)) |
| `is_deleted` | `char(2)` | `'N'` = active, `'Y'` = soft-deleted |
| `is_reminder_on` | `char(2)` | `'N'` or `'Y'` |
| `recurrence` | `varchar(255)` | Recurrence pattern (empty string = none) |
| `sort` | `int(11)` | **Required (non-NULL).** Position of the task within its list (`task_list_id`). Device assigns `0, 1, 2, …`. ⚠️ See [Sort Fields & Device Visibility](#sort-fields--device-visibility). |
| `sort_completed` | `int(11)` | **Required (non-NULL).** Completed-view order. `0` for active tasks. |
| `planer_sort` | `int(11)` | **Required (non-NULL).** Planner-view order. `0` by default. |
| `all_sort` | `int(11)` | "All" view order. Leave **`NULL`** (device leaves it unset). |
| `all_sort_completed` | `int(11)` | "All" completed order. Leave **`NULL`**. |
| `sort_time` | `bigint(20)` | Sort tie-break timestamp (ms). Set to `last_modified`. |
| `planer_sort_time` | `bigint(20)` | Planner timestamp (ms). `0` unless the task has a due date, in which case the device sets it to the planner/due time. |
| `all_sort_time` | `bigint(20)` | "All" sort timestamp. Leave **`NULL`** (device leaves it unset). |

### `t_schedule_task_group` — Categories/Lists

| Column | Type | Description |
|--------|------|-------------|
| `task_list_id` | `varchar(255)` PK | Unique list ID (32-char lowercase hex) |
| `user_id` | `bigint(20)` | User identifier |
| `title` | `varchar(255)` | Category name (e.g., "Work", "Personal") |
| `last_modified` | `bigint(20)` | Last modified timestamp (ms) |
| `is_deleted` | `char(2)` | `'N'` or `'Y'` |
| `create_time` | `bigint(20)` | Creation timestamp (ms) |

### `u_user` — Users Table

Identifies the user that tasks belong to. The `user_id` column links to the `user_id` field in `t_schedule_task` and `t_schedule_task_group`.

| Column | Type | Null | Key | Default | Description |
|--------|------|------|-----|---------|-------------|
| `user_id` | `bigint(20) unsigned` | NO | PRI | `0` | Unique user identifier (matches `user_id` in tasks/categories) |
| `user_name` | `varchar(100)` | NO | | | Display name |
| `email` | `varchar(50)` | YES | UNI | `NULL` | Email address (unique) |
| `sex` | `char(2)` | NO | | `1` | Gender code |
| `birthday` | `varchar(24)` | YES | | | Birthday |
| `personal_sign` | `varchar(150)` | YES | | `NULL` | Personal signature / bio |
| `hobby` | `varchar(20)` | YES | | `NULL` | Hobby |
| `education` | `varchar(24)` | YES | | `NULL` | Education |
| `job` | `varchar(32)` | YES | | `NULL` | Job |
| `avatars_url` | `varchar(100)` | YES | | `NULL` | Avatar image URL |
| `address` | `varchar(255)` | YES | | `NULL` | Address |
| `password` | `varchar(32)` | NO | | `NULL` | Password (hashed) |
| `create_time` | `datetime` | NO | | `NULL` | Account creation time |
| `update_time` | `datetime` | NO | | `NULL` | Last update time |
| `is_normal` | `char(2)` | NO | | `Y` | Account active flag (`Y`/`N`) |
| `file_server` | `char(2)` | YES | | | File server identifier |
| `avatars_position` | `varchar(30)` | YES | | `NULL` | Avatar crop/position metadata |
| `account_status` | `char(2)` | YES | | `N` | Account status |

### Key Concepts

- **Inbox**: Tasks with `task_list_id = NULL` — there is no explicit Inbox category row.
- **Soft deletes**: Always filter `WHERE is_deleted = 'N'`. Never hard-delete unless you know what you're doing.
- **Timestamps**: All timestamps are **Unix epoch in milliseconds** (multiply Python's `time.time()` by 1000).
- **Status**: Only two values — `'needsAction'` (open) and `'completed'` (done).
- **Task IDs**: 32-character lowercase hex strings. Generate with `uuid.uuid4().hex`.
- **Sort fields are required**: Tasks inserted with `NULL` integer sort columns may **not show up on the device**. See below.

### Sort Fields & Device Visibility

> ⚠️ **A task will not appear in the device To-Do app unless its integer sort
> columns are populated.** This was confirmed by comparing tasks created on the
> device (visible) against the schema's defaults.

The device maintains separate ordering for each To-Do view (the list view, the
Planner view, and the "All" view). Each has an integer order column and a
matching timestamp column. From live device-created tasks, the working
convention is:

| Column | Set to | Notes |
|--------|--------|-------|
| `sort` | **next position in the list** (`0, 1, 2, …`) | Per `task_list_id`. Must be non-NULL. This is the field most responsible for visibility. |
| `sort_completed` | `0` | Non-NULL. |
| `planer_sort` | `0` | Non-NULL. |
| `all_sort` | `NULL` | Device leaves this unset. |
| `all_sort_completed` | `NULL` | Device leaves this unset. |
| `sort_time` | `last_modified` (now, in ms) | Tie-break timestamp. |
| `planer_sort_time` | `0` (or the planner/due time for a dated task) | Only the device's dated tasks carried a non-zero value here. |
| `all_sort_time` | `NULL` | Device leaves this unset. |

The simplest safe default is to set `sort` to the next index within the list
(`SELECT COALESCE(MAX(sort), -1) + 1 FROM t_schedule_task WHERE task_list_id <=> %s AND is_deleted = 'N'`),
`sort_completed`/`planer_sort`/`planer_sort_time` to `0`, `sort_time` to the
current time in ms, and leave `all_sort`, `all_sort_completed`, and
`all_sort_time` as `NULL`. (Using a constant `0` for `sort` also works but all
tasks in a list will share an order.)

> 💡 This requirement was first surfaced by the r/Supernote_dev community
> (thread `1towv4r`) and confirmed here against live database rows.

---

## 5. Reading Todos (Standalone Python)

All examples below use a `get_connection()` function from [Section 3](#3-connecting-to-the-database) — either Option B (TCP with published port) or Option C (container on the same Docker network). Both return a pymysql connection; the only difference is the `host` value.

### List All Active Tasks

```python
def list_active_tasks(conn):
    """Get all active (non-deleted, non-completed) tasks."""
    sql = """
    SELECT
        t.task_id,
        t.title,
        t.detail,
        t.status,
        t.due_time,
        t.last_modified,
        t.links,
        COALESCE(g.title, 'Inbox') AS category
    FROM t_schedule_task t
    LEFT JOIN t_schedule_task_group g
        ON t.task_list_id = g.task_list_id
    WHERE t.is_deleted = 'N'
      AND t.status = 'needsAction'
    ORDER BY t.last_modified DESC;
    """
    with conn.cursor() as cur:
        cur.execute(sql)
        return cur.fetchall()

# Usage
conn = get_connection()
for task in list_active_tasks(conn):
    title = decode_emoji(task["title"])  # See emoji section below
    due = ms_to_datetime(task["due_time"])
    print(f"[{task['category']}] {title} (due: {due})")
conn.close()
```

### List All Tasks (Including Completed)

```python
def list_all_tasks(conn, include_completed=True):
    """Get all non-deleted tasks."""
    status_filter = "" if include_completed else "AND t.status = 'needsAction'"
    sql = f"""
    SELECT
        t.task_id, t.title, t.detail, t.status,
        t.due_time, t.completed_time, t.last_modified, t.links,
        COALESCE(g.title, 'Inbox') AS category
    FROM t_schedule_task t
    LEFT JOIN t_schedule_task_group g ON t.task_list_id = g.task_list_id
    WHERE t.is_deleted = 'N' {status_filter}
    ORDER BY t.last_modified DESC;
    """
    with conn.cursor() as cur:
        cur.execute(sql)
        return cur.fetchall()
```

### List Tasks by Category

```python
def list_tasks_by_category(conn, category_name: str):
    """Get tasks in a specific category."""
    if category_name.lower() == "inbox":
        cat_filter = "t.task_list_id IS NULL"
        params = ()
    else:
        with conn.cursor() as cur:
            cur.execute(
                "SELECT task_list_id FROM t_schedule_task_group "
                "WHERE title = %s AND is_deleted = 'N'",
                (category_name,),
            )
            row = cur.fetchone()
            if not row:
                return []
            cat_filter = "t.task_list_id = %s"
            params = (row["task_list_id"],)

    sql = f"""
    SELECT t.task_id, t.title, t.status, t.due_time,
           COALESCE(g.title, 'Inbox') AS category
    FROM t_schedule_task t
    LEFT JOIN t_schedule_task_group g ON t.task_list_id = g.task_list_id
    WHERE t.is_deleted = 'N' AND {cat_filter}
    ORDER BY t.last_modified DESC;
    """
    with conn.cursor() as cur:
        cur.execute(sql, params)
        return cur.fetchall()
```

### Get a Specific Task

```python
def get_task(conn, task_id: str):
    """Get a single task by its ID."""
    sql = """
    SELECT t.*, COALESCE(g.title, 'Inbox') AS category
    FROM t_schedule_task t
    LEFT JOIN t_schedule_task_group g ON t.task_list_id = g.task_list_id
    WHERE t.task_id = %s AND t.is_deleted = 'N';
    """
    with conn.cursor() as cur:
        cur.execute(sql, (task_id,))
        return cur.fetchone()
```

### List All Categories

```python
def list_categories(conn):
    """Get all active categories (plus implicit Inbox)."""
    sql = """
    SELECT task_list_id, title
    FROM t_schedule_task_group
    WHERE is_deleted = 'N'
    ORDER BY title;
    """
    with conn.cursor() as cur:
        cur.execute(sql)
        categories = cur.fetchall()

    # Inbox is implicit (task_list_id = NULL), not stored as a row
    return [{"task_list_id": None, "title": "Inbox"}] + list(categories)
```

### Decode Document Links

Tasks can link to pages in Supernote notebooks. The link is stored as Base64-encoded JSON:

```python
import base64
import json

def decode_document_link(links_value: str) -> dict | None:
    """Decode the Base64 JSON document link from the `links` column."""
    if not links_value:
        return None
    try:
        decoded = base64.b64decode(links_value).decode("utf-8")
        return json.loads(decoded)
    except Exception:
        return None

# Example output:
# {
#     "appName": "note",
#     "fileId": "F20250108110301551948uvnbySoVU5dZ",
#     "filePath": "/storage/emulated/0/Note/Work/Meeting Notes.note",
#     "page": 1,
#     "pageId": "P202501081103015749022LimUrUrjRGv"
# }
```

### Timestamp Helpers

```python
from datetime import datetime

def ms_to_datetime(ms) -> datetime | None:
    """Convert millisecond Unix timestamp to datetime (or None if 0/NULL)."""
    if not ms or int(ms) <= 0:
        return None
    return datetime.fromtimestamp(int(ms) / 1000)

def datetime_to_ms(dt: datetime) -> int:
    """Convert datetime to millisecond Unix timestamp."""
    return int(dt.timestamp() * 1000)

def now_ms() -> int:
    """Current time as millisecond Unix timestamp."""
    return int(datetime.now().timestamp() * 1000)
```

### Emoji Decoding

Supernote's MariaDB uses `utf8` (3-byte), which cannot store emoji (4-byte characters). The sync project encodes emoji as `[U+XXXX]`. You may encounter this in titles and notes:

```python
import re

def decode_emoji(text: str) -> str:
    """Decode [U+XXXX] sequences back to actual emoji characters."""
    if not text or "[U+" not in text:
        return text or ""

    def replace_match(m):
        try:
            return chr(int(m.group(1), 16))
        except (ValueError, OverflowError):
            return m.group(0)

    return re.sub(r"\[U\+([0-9A-Fa-f]+)\]", replace_match, text)

# Example:
# decode_emoji("Buy groceries [U+1F6D2]")  → "Buy groceries 🛒"
```

---

## 6. Writing Todos (Standalone Python)

### Create a New Task

```python
import uuid

def create_task(
    conn,
    title: str,
    detail: str = "",
    category_name: str = "Inbox",
    due_date: datetime | None = None,
    status: str = "needsAction",
):
    """Create a new todo task.

    Args:
        title: Task title.
        detail: Task notes (max 255 chars after emoji encoding).
        category_name: Category name (e.g., "Inbox", "Work").
        due_date: Optional due date as a datetime.
        status: 'needsAction' (default) or 'completed'.

    Returns:
        The generated task_id.
    """
    task_id = uuid.uuid4().hex
    ts = now_ms()

    # Look up category ID (None for Inbox)
    category_id = None
    if category_name.lower() != "inbox":
        category_id = get_category_id(conn, category_name)
        if category_id is None:
            category_id = create_category(conn, category_name)

    # Encode emoji for Supernote's 3-byte utf8
    encoded_title = encode_emoji(title)
    encoded_detail = encode_emoji(detail)[:255]  # Truncate AFTER encoding

    # Get user_id from existing data
    user_id = get_user_id(conn)

    due_time = datetime_to_ms(due_date) if due_date else 0
    completed_time = datetime_to_ms(datetime.now()) if status == "completed" else 0

    # Sort fields MUST be populated or the task won't appear on the device.
    # `sort` is the task's position within its list; use the next free index.
    sort_index = get_next_sort_index(conn, category_id)
    # Planner timestamp is 0 unless the task is dated (has a due date).
    planer_sort_time = due_time if due_time else 0

    sql = """
    INSERT INTO t_schedule_task (
        task_id, task_list_id, user_id, title, detail,
        last_modified, is_reminder_on, status, importance,
        due_time, completed_time, links, is_deleted,
        sort, sort_completed, planer_sort, all_sort,
        all_sort_completed, sort_time, planer_sort_time, all_sort_time
    ) VALUES (
        %s, %s, %s, %s, %s,
        %s, 'N', %s, NULL,
        %s, %s, NULL, 'N',
        %s, 0, 0, NULL, NULL, %s, %s, NULL
    );
    """
    with conn.cursor() as cur:
        cur.execute(sql, (
            task_id, category_id, user_id, encoded_title, encoded_detail,
            ts, status,
            due_time, completed_time, sort_index, ts, planer_sort_time,
        ))
    return task_id


def get_next_sort_index(conn, category_id: str | None) -> int:
    """Return the next `sort` position for a task within its list.

    Tasks are ordered per-list by the integer `sort` column. New tasks go to
    the end. `category_id` is None for the Inbox (task_list_id IS NULL).
    """
    with conn.cursor() as cur:
        cur.execute(
            "SELECT COALESCE(MAX(sort), -1) + 1 AS next_sort "
            "FROM t_schedule_task "
            "WHERE task_list_id <=> %s AND is_deleted = 'N'",
            (category_id,),
        )
        row = cur.fetchone()
        return int(row["next_sort"]) if row and row["next_sort"] is not None else 0


def get_user_id(conn) -> int:
    """Get the user ID from the database (assumes single user)."""
    with conn.cursor() as cur:
        cur.execute("SELECT DISTINCT user_id FROM t_schedule_task LIMIT 1")
        row = cur.fetchone()
        if row:
            return int(row["user_id"])
        cur.execute("SELECT id FROM u_user LIMIT 1")
        row = cur.fetchone()
        return int(row["id"]) if row else 1


def get_category_id(conn, name: str) -> str | None:
    """Look up a category ID by name."""
    with conn.cursor() as cur:
        cur.execute(
            "SELECT task_list_id FROM t_schedule_task_group "
            "WHERE LOWER(title) = LOWER(%s) AND is_deleted = 'N'",
            (name,),
        )
        row = cur.fetchone()
        return row["task_list_id"] if row else None
```

### Update an Existing Task

```python
_UNSET = object()  # Sentinel to distinguish "not provided" from None

def update_task(
    conn,
    task_id: str,
    title: str | None = None,
    detail: str | None = None,
    status: str | None = None,
    due_date=_UNSET,
    category_name: str | None = None,
):
    """Update fields on an existing task.

    - Omitted arguments are left unchanged.
    - Pass due_date=None to *clear* the due date.
    - Pass due_date=datetime(...) to *set* a due date.

    IMPORTANT: This preserves the existing `links` (document link) field.
    """
    # Build SET clauses dynamically
    sets = []
    params = []

    if title is not None:
        sets.append("title = %s")
        params.append(encode_emoji(title))

    if detail is not None:
        sets.append("detail = %s")
        params.append(encode_emoji(detail)[:255])

    if status is not None:
        sets.append("status = %s")
        params.append(status)
        if status == "completed":
            sets.append("completed_time = %s")
            params.append(now_ms())
        else:
            sets.append("completed_time = %s")
            params.append(0)

    if due_date is not _UNSET:
        sets.append("due_time = %s")
        params.append(datetime_to_ms(due_date) if due_date else 0)

    if category_name is not None:
        if category_name.lower() == "inbox":
            sets.append("task_list_id = %s")
            params.append(None)
        else:
            cat_id = get_category_id(conn, category_name)
            if cat_id is None:
                cat_id = create_category(conn, category_name)
            sets.append("task_list_id = %s")
            params.append(cat_id)

    if not sets:
        return  # Nothing to update

    sets.append("last_modified = %s")
    params.append(now_ms())
    params.append(task_id)

    sql = f"UPDATE t_schedule_task SET {', '.join(sets)} WHERE task_id = %s;"
    with conn.cursor() as cur:
        cur.execute(sql, params)
```

### Complete / Uncomplete a Task

```python
def complete_task(conn, task_id: str):
    """Mark a task as completed."""
    update_task(conn, task_id, status="completed")

def uncomplete_task(conn, task_id: str):
    """Mark a task as active again."""
    update_task(conn, task_id, status="needsAction")
```

> **Note:** On device-created active tasks, `sort_completed` is `0`. We don't
> have a confirmed sample of how the device sets `sort_completed` when a task is
> completed (it likely becomes a position index in the completed view). Leaving
> it at `0` works for read/write round-trips; if completed-task ordering looks
> off on-device, set `sort_completed` to the next index among completed tasks.

### Soft-Delete a Task

```python
def delete_task(conn, task_id: str):
    """Soft-delete a task (set is_deleted = 'Y')."""
    sql = """
    UPDATE t_schedule_task
    SET is_deleted = 'Y', last_modified = %s
    WHERE task_id = %s;
    """
    with conn.cursor() as cur:
        cur.execute(sql, (now_ms(), task_id))
```

### Create a Category

```python
def create_category(conn, name: str) -> str:
    """Create a new category/list. Returns the new task_list_id."""
    cat_id = uuid.uuid4().hex
    ts = now_ms()
    user_id = get_user_id(conn)

    sql = """
    INSERT INTO t_schedule_task_group (
        task_list_id, user_id, title, last_modified, is_deleted, create_time
    ) VALUES (%s, %s, %s, %s, 'N', %s);
    """
    with conn.cursor() as cur:
        cur.execute(sql, (cat_id, user_id, name, ts, ts))
    return cat_id
```

### Emoji Encoding (for Writes)

```python
def encode_emoji(text: str) -> str:
    """Encode emoji (4-byte chars) to [U+XXXX] for Supernote's 3-byte utf8."""
    if not text:
        return text or ""
    result = []
    for char in text:
        if ord(char) > 0xFFFF:
            result.append(f"[U+{ord(char):X}]")
        else:
            result.append(char)
    return "".join(result)

# Example:
# encode_emoji("Buy groceries 🛒")  → "Buy groceries [U+1F6D2]"
```

### Encode a Document Link

If you want to attach a notebook page link to a task:

```python
import base64
import json

def encode_document_link(
    file_id: str,
    file_path: str,
    page: int,
    page_id: str,
    app_name: str = "note",
) -> str:
    """Encode a document link as Base64 JSON for the `links` column."""
    link = {
        "appName": app_name,
        "fileId": file_id,
        "filePath": file_path,
        "page": page,
        "pageId": page_id,
    }
    return base64.b64encode(json.dumps(link).encode("utf-8")).decode("utf-8")

# Example: create a task with a document link
link_b64 = encode_document_link(
    file_id="F20250108110301551948uvnbySoVU5dZ",
    file_path="/storage/emulated/0/Note/Work/Meeting Notes.note",
    page=3,
    page_id="P202501081103015749022LimUrUrjRGv",
)

# Use in an INSERT:
# cur.execute(
#     "INSERT INTO t_schedule_task (..., links, ...) VALUES (..., %s, ...)",
#     (..., link_b64, ...),
# )
#
# Or UPDATE an existing task:
# cur.execute(
#     "UPDATE t_schedule_task SET links = %s, last_modified = %s WHERE task_id = %s",
#     (link_b64, now_ms(), task_id),
# )
```

---

## 7. Using the SupernoteDB Class

The [supernote-apple-reminders-sync](https://github.com/liketheduck/supernote-apple-reminders-sync) project provides a ready-made `SupernoteDB` class that wraps all the above operations.

### Setup

```bash
# Clone the project
git clone https://github.com/liketheduck/supernote-apple-reminders-sync.git
cd supernote-apple-reminders-sync

# Install dependencies
pip install -r requirements.txt

# Set your password
export SUPERNOTE_DB_PASSWORD="your-password-here"

# For TCP mode (default is docker):
export SUPERNOTE_DB_MODE=tcp
```

### Basic Usage

```python
from src.supernote_db import SupernoteDB
from src.models import UnifiedTask, DocumentLink
from datetime import datetime

# Initialize (reads config from environment variables)
db = SupernoteDB()

# Test connection
assert db.test_connection(), "Cannot connect to database"

# List all categories
categories = db.list_categories()
print(categories)  # {'abc123...': 'Work', 'def456...': 'Personal'}

# List all active tasks
tasks = db.list_tasks(include_completed=False)
for task in tasks:
    print(f"[{task.category}] {task.title} — {task.status}")

# List tasks in a specific category
work_tasks = db.list_tasks(category="Work")

# Get a specific task
task = db.get_task("your-task-id-here")
if task:
    print(task.title, task.document_link)
```

### Creating Tasks

```python
from src.models import UnifiedTask
from datetime import datetime

# Simple task
task = UnifiedTask(
    title="Review quarterly report",
    category="Work",
)
task_id = db.create_task(task)
print(f"Created task: {task_id}")

# Task with details and due date
task = UnifiedTask(
    title="Submit expense report",
    notes="Include receipts from conference",
    category="Work",
    due_date=datetime(2026, 5, 1),
)
db.create_task(task)

# Completed task
task = UnifiedTask(
    title="Old finished item",
    completed=True,
    completion_date=datetime.now(),
    status="completed",
)
db.create_task(task)
```

### Updating Tasks

```python
# Fetch, modify, save
task = db.get_task("some-task-id")
task.title = "Updated title"
task.notes = "Added some notes"
db.update_task(task)

# Complete a task
task = db.get_task("some-task-id")
task.completed = True
task.status = "completed"
task.completion_date = datetime.now()
db.update_task(task)
```

### Deleting Tasks

```python
# Soft delete (recommended — sets is_deleted = 'Y')
db.delete_task("some-task-id")

# Hard delete (actually removes the row)
db.delete_task("some-task-id", soft=False)
```

### Managing Categories

```python
# Create a new category
cat_id = db.create_category("Shopping")

# List all with IDs
cats = db.list_categories_with_ids()
# [{'id': 'abc123...', 'name': 'Work'}, ...]

# Rename
db.rename_category(cat_id, "Groceries")

# Delete (soft)
db.delete_category(cat_id)
```

### Working with Document Links

```python
from src.models import DocumentLink

# Decode from a task
task = db.get_task("some-task-id")
if task.document_link:
    link = task.document_link
    print(f"Linked to: {link.file_path}, page {link.page}")
    print(f"Readable: {link.to_readable_string()}")
    # → 📎 Meeting Notes.note (page 3)

# Create a task with a document link
link = DocumentLink(
    app_name="note",
    file_id="F20250108110301551948uvnbySoVU5dZ",
    file_path="/storage/emulated/0/Note/Work/Meeting Notes.note",
    page=3,
    page_id="P202501081103015749022LimUrUrjRGv",
)
task = UnifiedTask(title="Follow up on meeting", document_link=link)
db.create_task(task)
```

---

## 8. Sync Behavior & Safety

When you write directly to the MariaDB database, the Supernote device will pick up your changes on its next sync. But what happens if both you and the device modify the same task? This section covers what we know and how to stay safe.

### How the Supernote Device Syncs

The Supernote Private Cloud architecture looks like this:

```
Supernote Device  ←──WebSocket (port 18072)──→  Private Cloud Docker
                                                       │
                                                  ┌────┴────┐
                                                  │ MariaDB │
                                                  └────┬────┘
                                                       │
                                                  Your Scripts
```

The device syncs to the MariaDB via the Partner app / Private Cloud using an **undocumented WebSocket protocol** on port 18072. There is no public API documentation for this sync mechanism. What we can infer from the database schema and the [supernote-apple-reminders-sync](https://github.com/liketheduck/supernote-apple-reminders-sync) project:

- **MariaDB is the shared source of truth.** Both the device and your scripts read/write the same tables.
- **`last_modified` is the key timestamp.** The device almost certainly uses this to determine what's changed. It's a Unix timestamp in milliseconds.
- **Soft deletes are used.** The device sets `is_deleted = 'Y'` rather than removing rows, suggesting it expects rows to persist.
- **No separate sync state table exists** for the device↔DB relationship — unlike the liketheduck project which maintains its own `sync_state.db` for tracking Apple↔Supernote pairings.

### What Happens When You Write to the Database

**Reads are always safe.** SELECT queries have no side effects.

**Writes (INSERT/UPDATE) will be picked up by the device** on its next sync, as long as you set `last_modified` to a current timestamp. The device appears to treat the DB as authoritative — if a row is newer than what the device has locally, it updates.

**The unknown**: there is no public documentation on how the device handles conflicts when both sides modify the same task between syncs. Based on the `last_modified` column pattern, the most likely behavior is **last-write-wins** — whichever side wrote most recently (highest `last_modified`) takes precedence.

### The liketheduck Sync Engine's Approach

The [supernote-apple-reminders-sync](https://github.com/liketheduck/supernote-apple-reminders-sync) project implements its own conflict resolution for syncing between Apple Reminders and the MariaDB. This is instructive for understanding one approach to building a sync layer:

| Scenario | Behavior |
|----------|----------|
| Only one side changed | Propagate change to the other side |
| Both sides changed | Compare `last_modified` — **most recent wins** |
| Both changed within 60 seconds | Configurable preference (default: Apple wins) |
| Task deleted on one side | Delete on the other side too |
| New task on one side | Create on the other side |

It detects changes using **content hashing** (SHA-256 of title + notes + status + priority + category) compared against a stored hash from the last sync. This prevents sync loops where a write on side A triggers a "change" detected on the next sync.

### Safety Practices for Writing to the Database

Follow these rules to minimize risk when writing directly to the MariaDB:

#### 1. Always set `last_modified` to now

```python
last_modified = int(datetime.now().timestamp() * 1000)
```

This ensures the device sees your changes as newer. If you forget this, the device may overwrite your changes with its older local copy.

#### 2. Use soft deletes, never hard deletes

```python
# ✅ Soft delete
UPDATE t_schedule_task SET is_deleted = 'Y', last_modified = {now_ms} WHERE task_id = '...';

# ❌ Hard delete — may confuse device sync
DELETE FROM t_schedule_task WHERE task_id = '...';
```

#### 3. Generate proper UUIDs for new tasks

Use `uuid.uuid4().hex` (32-char lowercase hex). This prevents collisions with device-generated IDs.

#### 4. Don't modify tasks while actively editing on the device

If you're editing a task on the Supernote while your script also modifies it, the outcome is unpredictable. The safest pattern:

- Write to the DB when the device is idle or not connected
- Or create new tasks (no conflict possible) rather than modifying existing ones

#### 5. Preserve the `links` field on updates

If a task has a document link (Base64 in the `links` column), read it first and re-set it when updating. Clearing this field breaks the link between the task and a notebook page.

```python
# Read existing task first
existing = get_task(conn, task_id)
existing_links = existing["links"]  # Preserve this!

# Update with links preserved
cur.execute("""
    UPDATE t_schedule_task
    SET title = %s, last_modified = %s, links = %s
    WHERE task_id = %s
""", (new_title, now_ms(), existing_links, task_id))
```

#### 6. Consider a sync state table for bidirectional sync

If you're building a sync integration (not just one-way writes), follow the liketheduck project's pattern: maintain a separate SQLite database that tracks content hashes and timestamps per task. This lets you detect what actually changed since the last sync rather than doing a full diff every time.

### What We Don't Know

These aspects of the Supernote device sync are undocumented:

- **Exact conflict resolution policy** — is it truly last-write-wins by `last_modified`, or is there device-side logic we can't see?
- **Sync frequency** — how often does the device push/pull from the DB? Is it real-time, periodic, or on-demand?
- **WebSocket protocol** — port 18072 carries the sync traffic, but the protocol is proprietary and undocumented
- **Concurrent write handling** — does MariaDB row-level locking protect against race conditions, or could partial updates occur?
- **Field-level vs. row-level sync** — does the device sync entire rows or individual changed fields?
- **`all_sort*` columns** — `all_sort`, `all_sort_completed`, and `all_sort_time` are `NULL` on device-created tasks, so their exact role in the "All" view is unconfirmed. Leaving them `NULL` works (see [Sort Fields & Device Visibility](#sort-fields--device-visibility)).

For most use cases (creating tasks, marking complete, reading), these unknowns don't matter. They only become relevant if you're doing high-frequency writes or modifying the same tasks the user is actively editing on the device.

---

## 9. Important Gotchas & Tips

### Tasks Don't Appear on the Device

If a task you inserted via SQL never shows up in the device To-Do app, the most
likely cause is **NULL integer sort columns**. The device requires `sort`,
`sort_completed`, and `planer_sort` to be non-NULL. Populate them as described in
[Sort Fields & Device Visibility](#sort-fields--device-visibility):

```sql
-- ✅ Visible: integer sort columns populated
sort = <next index in list>, sort_completed = 0, planer_sort = 0,
sort_time = <now ms>, planer_sort_time = 0,
all_sort = NULL, all_sort_completed = NULL, all_sort_time = NULL

-- ❌ Hidden: integer sort columns left NULL
sort = NULL, sort_completed = NULL, planer_sort = NULL
```

### Timestamps Are Milliseconds

Every timestamp in the database is **Unix epoch in milliseconds**, not seconds. Always multiply by 1000 when writing, divide by 1000 when reading.

```python
# ✅ Correct
int(datetime.now().timestamp() * 1000)

# ❌ Wrong — this is seconds
int(datetime.now().timestamp())
```

### Always Filter Soft Deletes

Every query should include `WHERE is_deleted = 'N'`. Deleted tasks are kept in the database with `is_deleted = 'Y'`.

### Inbox Has No Category Row

The "Inbox" is not a row in `t_schedule_task_group`. Tasks in the Inbox have `task_list_id = NULL`. To query Inbox tasks:

```sql
WHERE task_list_id IS NULL
```

### Preserve Document Links on Update

When updating a task, **do not overwrite the `links` field** unless you intend to change the document link. The `SupernoteDB.update_task()` method handles this automatically by reading the existing link first.

### Detail Column Is 255 Characters

The `detail` column is `varchar(255)`. Emoji encoding expands characters (e.g., one emoji → ~10 chars like `[U+1F6D2]`), so **truncate after encoding**:

```python
encoded = encode_emoji(detail)[:255]
```

### Emoji Encoding

Supernote's MariaDB uses `utf8` (3-byte max), which cannot store characters above U+FFFF (most emoji). The project works around this by encoding emoji as `[U+XXXX]` text. If you write emoji directly, MariaDB will either truncate or error. Always use `encode_emoji()` when writing and `decode_emoji()` when reading.

### Task IDs

Task IDs are 32-character lowercase hex strings. Generate them with:

```python
import uuid
task_id = uuid.uuid4().hex  # e.g., "a1b2c3d4e5f6..."
```

### SQL Injection Prevention

When using the standalone examples (pymysql), always use **parameterized queries** (`%s` placeholders). The `SupernoteDB` class uses string escaping + ID validation because it also supports Docker exec mode.

---

## 10. Quick Reference

### Common SQL Queries

```sql
-- All active tasks with categories
SELECT t.task_id, t.title, t.status,
       COALESCE(g.title, 'Inbox') AS category
FROM t_schedule_task t
LEFT JOIN t_schedule_task_group g ON t.task_list_id = g.task_list_id
WHERE t.is_deleted = 'N' AND t.status = 'needsAction';

-- All categories
SELECT task_list_id, title FROM t_schedule_task_group WHERE is_deleted = 'N';

-- Tasks due today (replace XXXXXX with today's start/end in ms)
SELECT task_id, title, due_time FROM t_schedule_task
WHERE is_deleted = 'N' AND status = 'needsAction'
  AND due_time BETWEEN {start_of_day_ms} AND {end_of_day_ms};

-- Count tasks per category
SELECT COALESCE(g.title, 'Inbox') AS category, COUNT(*) AS count
FROM t_schedule_task t
LEFT JOIN t_schedule_task_group g ON t.task_list_id = g.task_list_id
WHERE t.is_deleted = 'N' AND t.status = 'needsAction'
GROUP BY g.title;

-- Recently modified tasks
SELECT task_id, title, last_modified
FROM t_schedule_task
WHERE is_deleted = 'N'
ORDER BY last_modified DESC LIMIT 10;
```

### Python Snippet: Full Read/Write Example

```python
#!/usr/bin/env python3
"""Minimal example: list, create, complete, and delete a Supernote todo."""

import os
import uuid
import pymysql
from datetime import datetime

def connect():
    return pymysql.connect(
        host="localhost", port=3306,
        user="supernote", password=os.environ["SUPERNOTE_DB_PASSWORD"],
        database="supernotedb", charset="utf8mb4",
        cursorclass=pymysql.cursors.DictCursor, autocommit=True,
    )

def now_ms():
    return int(datetime.now().timestamp() * 1000)

conn = connect()
cur = conn.cursor()

# 1. List active tasks
cur.execute("""
    SELECT task_id, title, status FROM t_schedule_task
    WHERE is_deleted='N' AND status='needsAction'
    ORDER BY last_modified DESC LIMIT 5
""")
print("Active tasks:", cur.fetchall())

# 2. Create a task
task_id = uuid.uuid4().hex
cur.execute("SELECT DISTINCT user_id FROM t_schedule_task LIMIT 1")
row = cur.fetchone()
if row:
    user_id = row["user_id"]
else:
    cur.execute("SELECT id FROM u_user LIMIT 1")
    row = cur.fetchone()
    user_id = row["id"] if row else 1
ts = now_ms()

# Inbox task (task_list_id = NULL): use the next sort index for the Inbox.
cur.execute(
    "SELECT COALESCE(MAX(sort), -1) + 1 AS next_sort FROM t_schedule_task "
    "WHERE task_list_id IS NULL AND is_deleted = 'N'"
)
sort_index = cur.fetchone()["next_sort"]

# Populate the integer sort columns — tasks with NULL sort won't show on-device.
cur.execute("""
    INSERT INTO t_schedule_task
    (task_id, task_list_id, user_id, title, detail, last_modified,
     is_reminder_on, status, due_time, completed_time, is_deleted,
     sort, sort_completed, planer_sort, all_sort, all_sort_completed,
     sort_time, planer_sort_time, all_sort_time)
    VALUES (%s, NULL, %s, %s, %s, %s, 'N', 'needsAction', 0, 0, 'N',
            %s, 0, 0, NULL, NULL, %s, 0, NULL)
""", (task_id, user_id, "Test task from Python", "", ts, sort_index, ts))
print(f"Created task: {task_id}")

# 3. Complete it
cur.execute("""
    UPDATE t_schedule_task SET status='completed', completed_time=%s, last_modified=%s
    WHERE task_id=%s
""", (ts, ts, task_id))
print("Completed task")

# 4. Soft-delete it
cur.execute("""
    UPDATE t_schedule_task SET is_deleted='Y', last_modified=%s WHERE task_id=%s
""", (now_ms(), task_id))
print("Deleted task")

conn.close()
```

### Environment Variables Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `SUPERNOTE_DB_PASSWORD` | **(required)** | MariaDB password |
| `SUPERNOTE_DB_MODE` | `docker` | `docker` or `tcp` |
| `SUPERNOTE_DB_HOST` | `localhost` | MariaDB host (TCP mode) |
| `SUPERNOTE_DB_PORT` | `3306` | MariaDB port (TCP mode) |
| `SUPERNOTE_DOCKER_CONTAINER` | `supernote-mariadb` | Docker container name |
| `SUPERNOTE_DB_NAME` | `supernotedb` | Database name |
| `SUPERNOTE_DB_USER` | `supernote` | Database user |
