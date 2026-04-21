# Productivity API — Notes Backend

A secure RESTful Flask API that powers a personal notes productivity tool.
Users can register, log in, and manage their own private notes. All resource
routes are protected: you cannot read, create, edit, or delete another user's
notes.

---

## Tech Stack

- **Flask 2.2.2** — web framework
- **Flask-SQLAlchemy 3.0.3** — ORM (SQLite by default)
- **Flask-Migrate 4.0.0** — database migrations via Alembic
- **Flask-Bcrypt 1.0.1** — password hashing
- **Faker 15.3.2** — seed data generation
- **uv** — fast Python package manager

---

## Database Schema

```
Table users {
  id integer [primary key]
  username varchar(80) [unique, not null]
  password varchar(200) [not null]
}

Table notes {
  id integer [primary key]
  title varchar(120) [not null]
  content text [not null]
  user_id integer [not null, ref: > users.id]
}
```

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/pjperfect/productivity-api.git
cd productivity-api
```

### 2. Install dependencies with `uv`

```bash
# Install uv if you don't have it
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create a virtual environment and sync all dependencies
uv venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
uv sync
```

### 3. Set environment variables (optional)

```bash
export SECRET_KEY="your-strong-secret-key"
export DATABASE_URL="sqlite:///productivity.db"
```

### 4. Apply database migrations

```bash
export FLASK_APP=app.py
flask db upgrade
```

> `flask db init` and `flask db migrate` have already been run and the
> `migrations/` folder is committed to the repo. You only need `flask db upgrade`
> to apply the schema to your local database.

### 5. Seed the database

```bash
python seed.py
```

This creates **5 demo users** (each with 8 notes). All demo users share the
password `password123`. The script prints three sample usernames you can use
to log in immediately.

---

## Running the Server

```bash
flask run
```

The API runs at `http://127.0.0.1:5000` by default.

---

## API Endpoints

### Authentication

| Method   | Path             | Description                | Auth required |
| -------- | ---------------- | -------------------------- | ------------- |
| `POST`   | `/signup`        | Register a new user        | No            |
| `POST`   | `/login`         | Log in and start a session | No            |
| `DELETE` | `/logout`        | End the current session    | No            |
| `GET`    | `/check_session` | Return the logged-in user  | No            |

#### POST `/signup`

```json
// Request body
{ "username": "alice", "password": "secret123" }

// 201 response
{ "id": 1, "username": "alice" }
```

#### POST `/login`

```json
// Request body
{ "username": "alice", "password": "secret123" }

// 200 response
{ "id": 1, "username": "alice" }
```

#### GET `/check_session`

```json
// 200 response (logged in)
{ "id": 1, "username": "alice" }

// 401 response (not logged in)
{ "error": "not authenticated" }
```

---

### Notes (all routes require an active session)

| Method   | Path          | Description                            |
| -------- | ------------- | -------------------------------------- |
| `GET`    | `/notes`      | Paginated list of the user's own notes |
| `POST`   | `/notes`      | Create a new note                      |
| `PATCH`  | `/notes/<id>` | Update a note (owner only)             |
| `DELETE` | `/notes/<id>` | Delete a note (owner only)             |

#### GET `/notes`

Query parameters:

| Param      | Default | Max  | Description           |
| ---------- | ------- | ---- | --------------------- |
| `page`     | `1`     | —    | Page number (1-based) |
| `per_page` | `10`    | `50` | Results per page      |

```json
// 200 response
{
  "notes": [
    { "id": 3, "title": "Meeting prep", "content": "...", "user_id": 1 }
  ],
  "page": 1,
  "per_page": 10,
  "total": 42,
  "pages": 5
}
```

#### POST `/notes`

```json
// Request body
{ "title": "My note", "content": "Body text here" }

// 201 response
{ "id": 7, "title": "My note", "content": "Body text here", "user_id": 1 }
```

#### PATCH `/notes/<id>`

```json
// Request body (any subset of fields)
{ "title": "Updated title" }

// 200 response
{ "id": 7, "title": "Updated title", "content": "Body text here", "user_id": 1 }
```

#### DELETE `/notes/<id>`

```
// 204 No Content on success
// 403 Forbidden if you don't own the note
// 404 Not Found if the note doesn't exist
```

---

## Project Structure

```
productivity-api/
├── app.py              # App factory and entry-point
├── config.py           # Flask configuration classes
├── models.py           # SQLAlchemy models (User, Note)
├── seed.py             # Database seed script
├── pyproject.toml      # uv dependencies
├── README.md
├── migrations/         # Flask-Migrate generated migrations
│   └── README          # Migration usage instructions
└── routes/
    ├── __init__.py     # Marks routes/ as a Python package
    ├── auth.py         # /signup  /login  /logout  /check_session
    └── notes.py        # /notes  CRUD + pagination
```

---

## Authentication Flow

This API uses **server-side sessions** (Flask's signed cookie). After a
successful `/login` or `/signup` the server stores `user_id` in the session.
Every protected route reads `session["user_id"]` to identify the caller.

The [sessions frontend template](https://github.com/learn-co-curriculum/flask-c10-summative-lab-sessions-and-jwt-clients)
in the lab repo is designed to work with this flow out of the box.
