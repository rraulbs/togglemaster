# ToggleMaster Monolithic Application Analysis

This analysis is based on the application source code (`app.py`, `Dockerfile`, `docker-compose.yaml`, `entrypoint.sh` and `requirements.txt`).

---

## 1. How the application works

### 1.1 Technologies used

| Layer | Technology | Role |
|---|---|---|
| Language | Python 3.9 (`python:3.9-slim` image; 3.9.25 on EC2) | Runtime |
| Web framework | Flask 2.2.2 | HTTP routes and JSON |
| Application server | Gunicorn 20.1.0 | Production WSGI server (port 5000) |
| Database driver | psycopg2-binary 2.9.5 | PostgreSQL connection |
| Database | PostgreSQL 16 (container in compose; RDS on AWS) | Persistence |
| Packaging | Docker + Docker Compose | Local execution |

### 1.2 Code structure

- `app.py` holds the entire application: configuration, database connection, table creation, routes and validations.
- `entrypoint.sh` validates the environment variables, waits for the database to be ready using `pg_isready`, runs `flask init-db` and starts Gunicorn.
- `Dockerfile` packages the application and installs `postgresql-client`, which is required by `pg_isready`.
- `docker-compose.yaml` brings up the application and a local PostgreSQL with a healthcheck.

### 1.3 Data model

There is a single table, `flags`:

| Column | Type | Notes |
|---|---|---|
| `id` | SERIAL, PK | |
| `name` | VARCHAR(100) | UNIQUE, NOT NULL |
| `is_enabled` | BOOLEAN | NOT NULL, default `false` |
| `created_at` | TIMESTAMPTZ | default `CURRENT_TIMESTAMP` |

### 1.4 Endpoints and basic flow

| Method | Route | Behavior | Responses |
|---|---|---|---|
| GET | `/health` | Returns `{"status":"ok"}` | 200 |
| POST | `/flags` | Creates a flag (`name` required, `is_enabled` optional) | 201, 400, 409 (duplicate), 500 |
| GET | `/flags` | Lists all flags, ordered by name | 200, 500 |
| GET | `/flags/<name>` | Retrieves a single flag | 200, 404, 500 |
| PUT | `/flags/<name>` | Updates `is_enabled` (must be a boolean) | 200, 400, 404, 500 |

**Request flow (example: `POST /flags`):**

1. The client sends `POST /flags` with `{"name": "new-feature", "is_enabled": true}`.
2. Gunicorn receives the request and passes it to Flask.
3. The `create_flag()` route validates the JSON (`name` is required).
4. The `get_db_connection()` function opens a **new connection** to PostgreSQL, using the environment variables.
5. The route runs the `INSERT` with a parameterized query (`%s`), which protects against SQL injection.
6. The `commit` persists the write. If the name already exists, an `IntegrityError` is raised and the response is 409.
7. The `finally` block closes the cursor and the connection.
8. The JSON response is returned to the client (201).

**Configuration:** `DB_HOST`, `DB_NAME`, `DB_USER` and `DB_PASSWORD` are read from the environment.

### 1.5 Code observations

1. **There is no DELETE endpoint.** The project statement calls for full CRUD (create, read, update and delete), but the API only implements C, R and U. This is a known limitation of the MVP.
2. **`/health` does not check the database.** It returns "ok" even if PostgreSQL is down, so it is a *liveness check*, not a *readiness check*.
3. **There is no connection pool.** Each request opens and closes a connection. It works for an MVP, but it is costly under load.
4. **500 errors return `str(e)` to the client** (the `details` field), which may leak internal details.
5. **There is no authentication.** Any client that can reach the API can create or modify flags.
6. **The README messages are in English** (`created successfully`), but the code responds in Portuguese (`criada com sucesso`), as seen in the API tests on AWS. This is only a documentation mismatch.

---

## 2. Why the application is considered a monolith

A monolith is an application in which **all the logic lives in a single codebase and is deployed as a single unit**.

### 2.1 Evidence in the project

1. **Single codebase with no layer separation.** In `app.py`, the same function handles HTTP routing, validation, business logic and data access (the SQL is written inside the route itself). There are no separate modules, services or repositories.
2. **Single deployment unit.** There is only one artifact (a Docker image, or a Gunicorn process on EC2). To change anything, such as a validation or a query, the entire application must be redeployed.
3. **Scaling only as a whole.** To scale, the whole application must be replicated. It is not possible to scale only reads or only writes.
4. **Single stack and shared database.** The entire domain uses the same technology (Python/Flask) and the same PostgreSQL schema.
5. **Single lifecycle.** A failure that brings down the process takes down all features at the same time.

### 2.2 The role of the database

The database is a component separate from the application: in Docker Compose it is the `db` container, and on AWS it is RDS. This is standard practice and **does not make the application any less of a monolith**, because the term refers to the application, not to the infrastructure around it. The key point is that all the application logic lives in a single service:
 
> The application is a monolith because all of its logic (API, validation, business rules and data access) lives in a single codebase and is deployed as a single unit. The database is an external component, but the application as a whole cannot be evolved, scaled or deployed in parts.
 
In a microservices architecture, ToggleMaster could be split into, for example, a flag management service, an authentication service and a flag evaluation service, each with its own deployment and, often, its own database.

---

## 3. Advantages and disadvantages for an MVP

### 3.1 Advantages

| Advantage | How it shows up in ToggleMaster |
|---|---|
| **Simplicity** | A single file and a flow that is easy to understand and debug |
| **Development speed** | A CRUD API comes up quickly, with no contracts between services |
| **Simple deployment** | One artifact and one process; on EC2, just copy the code, install the dependencies and run |
| **Less coordination** | A small team works on a single codebase |
| **Lower initial cost** | A small EC2 instance and a small RDS instance are enough (about US$28 per month in the Phase 1 estimate) |
| **Data consistency** | A single database, with no distributed transactions |
| **Less operational overhead** | No service discovery, messaging or distributed tracing |
| **Fast idea validation** | This is the point of an MVP: get it live and collect feedback |

### 3.2 Disadvantages

| Disadvantage | Impact on ToggleMaster |
|---|---|
| **Scaling only as a whole** | If flag reads are far more frequent than writes, there is no way to scale reads alone |
| **Coupling** | Business logic, HTTP and SQL mixed together make testing and changes harder |
| **Risky deployments** | A small change requires redeploying everything |
| **Single point of failure** | One EC2 instance, one process (Gunicorn in the foreground) and one Single-AZ RDS instance: if any of them goes down, the API is unavailable |
| **Hard to evolve into microservices** | Before splitting, layers and responsibilities must be separated |
| **Single stack** | The whole system is tied to Python/Flask |
| **Growing complexity** | With more features, `app.py` tends to become hard to maintain |
| **Limited team scalability** | Multiple teams working on the same codebase cause conflicts and dependencies |

### 3.3 Conclusion

For an MVP, the monolith is the right choice: it reduces time to first deployment, complexity and cost, which is what matters when validating an idea. The disadvantages (independent scaling, coupling, single point of failure) only become relevant as the product grows in users and features. That is why the natural path is to start monolithic and split into services once there is a proven need, which is aligned with the project's proposal to gradually turn it into a distributed system over the coming phases.
