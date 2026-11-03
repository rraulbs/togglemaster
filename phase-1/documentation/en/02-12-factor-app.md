# 12-Factor App Applied to ToggleMaster

This analysis evaluates the ToggleMaster monolithic application (`app.py`, `Dockerfile`, `docker-compose.yaml`, `entrypoint.sh` and `requirements.txt`) against the 12 factors of the [12-Factor App](https://12factor.net/) methodology. For each factor, we present the evidence in the code, the status, and the suggested improvement for a production environment.

In Phase 1, the application is deployed on AWS **as is**: the code (`app.py`) was not changed, and the only change in the `toggle-master-monolith/` directory was the local database image in `docker-compose.yaml` (`postgres:13` to `postgres:16`), to keep parity with RDS. The improvements described here are recommendations for a more robust production environment and are not part of this phase's scope.

The analysis was updated after deployment, so the evidence reflects what was actually provisioned (details in `03-aws-architecture.md`).

**Legend:** ✅ meets | ⚠️ partially meets | ❌ does not meet

---

## 1. Overview

| # | Factor | Status | Summary |
|---|---|---|---|
| I | Codebase | ✅ | A single Git-versioned codebase, with local and AWS deploys |
| II | Dependencies | ✅ | Declared and pinned in `requirements.txt`; there is an implicit system dependency |
| III | Config | ⚠️ | Environment variables and, on AWS, secrets in Secrets Manager; inconsistencies remain (`DB_PORT`, SSL, validation) |
| IV | Backing services | ✅ | The database is an attached resource, swappable through environment variables alone |
| V | Build, release, run | ❌ | Manual deployment on EC2; no versioned artifact or pipeline |
| VI | Processes | ✅ | Stateless application; all state lives in PostgreSQL |
| VII | Port binding | ✅ | Self-contained: Gunicorn exposes port 5000 |
| VIII | Concurrency | ⚠️ | Scales through processes, but has no connection pool and defaults to a single worker |
| IX | Disposability | ⚠️ | Starts fast and uses `exec`, but has no process manager on EC2 |
| X | Dev/prod parity | ⚠️ | Same database engine, database version and Python, but local runs on Docker and EC2 on a venv |
| XI | Logs | ⚠️ | Writes to stdout/stderr, but exceptions are not logged |
| XII | Admin processes | ✅ | `flask init-db` runs as a one-off command, using the same code |

**Scorecard:** 6 factors met, 5 partially met and 1 not met.

---

## 2. Detailed analysis by factor

### I. Codebase — ✅ Meets

**Principle:** one codebase tracked in version control, with many deploys.

**Evidence:** the project is a single Git repository. The same code runs locally (Docker Compose) and on EC2.

**Note:** since it is a monolith, there is a single codebase for a single application, which is exactly what the factor asks for. Splitting it into several services would require one repository (or module) per service.

---

### II. Dependencies — ✅ Meets (with caveats)

**Principle:** explicitly declare and isolate dependencies.

**Evidence:**
- `requirements.txt` with pinned versions: Flask 2.2.2, Werkzeug 2.3.8, psycopg2-binary 2.9.5 and gunicorn 20.1.0.
- Isolation through a container (Dockerfile) or a virtual environment (`venv`) on EC2. During deployment, `pip install -r requirements.txt` in the `venv` with Python 3.9.25 (Amazon Linux 2023) completed without errors.

**Caveats:**
1. `entrypoint.sh` depends on `pg_isready`, which is part of the `postgresql-client` package. With Docker it is installed by the Dockerfile. On EC2, it is only needed if `entrypoint.sh` is used; if Gunicorn is started directly (as was done), the dependency does not apply. In both cases, **it is not declared outside the Dockerfile**.
2. `app.py` imports `click` directly, but it is not in `requirements.txt` (it only arrives as a transitive dependency of Flask). The import is also unused.
3. There is no *lockfile* with transitive dependencies, and the pinned versions are old (Flask 2.2.2, Python 3.9).

**Improvements:**
- Document the installation of `postgresql-client` if `entrypoint.sh` is used on EC2.
- Remove the unused `import click`.
- Generate a complete version file (`pip freeze` or `pip-tools`) and plan the upgrade of Python and Flask.

---

### III. Config — ⚠️ Partially meets

**Principle:** store config in the environment, separate from code.

**Evidence (positive points):**
- `app.py` reads `DB_HOST`, `DB_NAME`, `DB_USER` and `DB_PASSWORD` with `os.getenv`. No credentials are in the code.
- The same image works in any environment, changing only the variables.
- On AWS, `DB_USER` and `DB_PASSWORD` come from the secret managed by RDS in **AWS Secrets Manager**, read by the EC2 instance through an IAM Role (`LabInstanceProfile`) and exported as environment variables before starting Gunicorn. The password is not typed, does not stay in the shell history or in a file, and `app.py` did not need to change.

**Problems found:**
1. **`DB_PORT` inconsistency:** `entrypoint.sh` requires this variable and uses it in `pg_isready`, but `app.py` does not read it. `psycopg2` ends up using the default port 5432. It works with RDS PostgreSQL, but the configuration does not match what it declares.
2. **Incomplete validation:** `entrypoint.sh` validates `DB_HOST`, `DB_PORT` and `DB_NAME`, but not `DB_USER` and `DB_PASSWORD`. In addition, with `set -u` enabled, a missing variable aborts the script with a generic error before the friendly message is displayed.
3. **Manual variable handling:** `docker-compose.yaml` versions `user`/`password`, which is acceptable for local development but must not be copied to production. On EC2, reading the secret is done manually in each session: the variables only exist in the terminal where they were exported (the procedure must be repeated and Gunicorn restarted after reconnecting), and the application reads the password only once, at startup. Since the RDS secret is rotated automatically (every 7 days by default), a rotation requires restarting the application.
4. **SSL without certificate validation:** the connection does not set `sslmode`, and the `psycopg2` default (`prefer`) negotiates SSL when the server offers it. RDS PostgreSQL 15 or later requires SSL by default, so traffic is encrypted, but the server certificate is not verified.
5. **Hardcoded application port:** `5000` is written in the Dockerfile, in the entrypoint and in `app.py`.

**Improvements:**
- Read `DB_PORT` in `app.py` and pass it to `psycopg2.connect`.
- Validate all required variables with clear messages:
  ```bash
  : "${DB_HOST:?DB_HOST is not set}"
  : "${DB_PORT:?DB_PORT is not set}"
  : "${DB_NAME:?DB_NAME is not set}"
  : "${DB_USER:?DB_USER is not set}"
  : "${DB_PASSWORD:?DB_PASSWORD is not set}"
  ```
- Automate reading the secret (for example, in the `systemd` service or by the application itself) and handle rotation without a manual restart. **Secrets Manager** is already used in Phase 1; the evolution is to stop depending on a manual step.
- Use `sslmode=verify-full` in the RDS connection, with the AWS root certificate.
- Make the port configurable (`PORT`).

---

### IV. Backing services — ✅ Meets

**Principle:** treat backing services (databases, queues, caches) as attached resources.

**Evidence:** the database is accessed exclusively through environment variables. In Docker Compose, `DB_HOST=db` points to the PostgreSQL container. On AWS, it was enough to adjust the connection variables (`DB_HOST` with the RDS endpoint, plus `DB_NAME`, `DB_USER` and `DB_PASSWORD`, according to the database created), **without changing a single line of code**, which was confirmed during deployment. This is the project's best example of adherence.

**Improvement:** none required. As a good practice, the same major PostgreSQL version was kept on RDS and in the compose file (16). Version 13, originally used in the compose file, left RDS standard support on 02/28/2026.

---

### V. Build, release, run — ❌ Does not meet

**Principle:** strictly separate the build, release and run stages.

**Evidence:**
- **Build:** there is a Dockerfile, but on AWS the code was copied to EC2 (`scp`) and the dependencies were installed with `pip install` directly on the instance, mixing build and run in a single manual operation.
- **Release:** there is no artifact versioning (tags, registry image). It is not possible to point to "version X is live" or to do a precise rollback.
- **Run:** database initialization (`flask init-db`) happens at container startup, which mixes a release step (schema migration) with execution. On EC2, it was run manually, once.
- There is no CI/CD pipeline.

**Improvements:**
- Version with Git tags and build Docker images with a version tag, published to a registry (Amazon ECR).
- Automate build and deployment with a CI/CD pipeline.
- Separate the schema migration from application startup (release step).
- For Phase 1, document the manual procedure (done in `03-aws-architecture.md`) and record the deployed commit.

---

### VI. Processes — ✅ Meets

**Principle:** execute the app as stateless processes, sharing nothing.

**Evidence:** the application keeps no session, in-memory cache or local files. Each request opens its own connection and all persistent state lives in PostgreSQL. This allows the application to be restarted or replicated without data loss.

**Note:** the absence of authentication also means there is no user session. When authentication is added, it should use tokens or external storage, never local memory.

---

### VII. Port binding — ✅ Meets

**Principle:** export services via port binding, without relying on an external web server.

**Evidence:** the application is self-contained. Gunicorn binds to `0.0.0.0:5000` and serves HTTP directly, with no Apache or Nginx required. On AWS, the API responded at `http://<public-ip>:5000` with only port 5000 open in the Security Group.

**Improvement:** read the port from a variable (`--bind 0.0.0.0:${PORT:-5000}`). In production, it is common to place a reverse proxy or Load Balancer in front for TLS and ports 80/443.

---

### VIII. Concurrency — ⚠️ Partially meets

**Principle:** scale out via the process model.

**Evidence (positive):** since it is stateless, the application can be replicated across several processes or instances.

**Problems:**
1. Gunicorn is started without `--workers`, so it runs with **a single worker** by default.
2. There is no connection *pool*: each request opens and closes a connection. When scaling workers or instances, the number of connections to RDS grows quickly and may exhaust the database limit.

**Improvements:**
- Set `--workers` according to the number of vCPUs (for example, `2 x CPUs + 1`).
- Adopt a connection pool (`psycopg2.pool`) or an intermediary such as RDS Proxy.
- Plan future horizontal scaling with a Load Balancer and Auto Scaling.

---

### IX. Disposability — ⚠️ Partially meets

**Principle:** processes that start fast and shut down gracefully.

**Evidence (positive):**
- Startup is fast and `entrypoint.sh` uses `exec gunicorn`, so Gunicorn receives system signals (SIGTERM) and shuts down gracefully.
- Compose uses `depends_on` with a *healthcheck* to wait for the database.

**Problems:**
1. The database wait loop (`until pg_isready`) **has no retry limit**: if the database never comes up, the container is stuck forever.
2. On EC2, Gunicorn runs in the foreground; if the SSH session closes, the application goes down, and there is no automatic restart on failure. In practice, this requires restarting the application (and re-exporting the variables) in each new session.
3. The Dockerfile `CMD` is ignored, because `entrypoint.sh` hardcodes the Gunicorn command.

**Improvements:**
- Add a retry limit and *timeout* to the database wait.
- Run on EC2 as a `systemd` service (`Restart=always`) or in a container with a restart policy.
- Make the entrypoint respect `CMD` (`exec "$@"`).

---

### X. Dev/prod parity — ⚠️ Partially meets

**Principle:** keep development, staging and production as similar as possible.

**Evidence (positive):**
- The local environment uses PostgreSQL 16, the same engine and major version as RDS (16.15), avoiding the trap of testing on one database and running production on another.
- Python also matches: 3.9 in the container (`python:3.9-slim`) and 3.9.25 on EC2 (Amazon Linux 2023).

**Differences found:**
1. Locally the application runs on **Docker** (Debian); on EC2, it runs directly with a **venv** (Amazon Linux 2023). Operating system, system packages and way of running differ.
2. The compose file mounts `.:/app` as a volume (a development convenience) and uses fixed credentials.
3. The local database is a container; the production one is a managed service, with different parameters and SSL.

**Improvements:**
- Run the same Docker container on EC2, which brings the two environments closer.
- Keep the same major PostgreSQL version in the compose file and on RDS when upgrading either one.
- Avoid mounting code as a volume outside the development environment.

---

### XI. Logs — ⚠️ Partially meets

**Principle:** treat logs as event streams, sent to stdout/stderr.

**Evidence (positive):**
- The application does not write log files; everything goes to stdout/stderr.
- The Dockerfile sets `PYTHONUNBUFFERED=1`, so output appears without delay.

**Problems:**
1. **Exceptions are not logged.** The `except` blocks return the error in the HTTP response, but write nothing to the log. If a route fails, there is no trace on the server.
2. Gunicorn does not log requests by default (the *access log* is disabled).
3. The application uses `print()` instead of the `logging` module, with no levels or structured format.
4. The error returned to the client includes `str(e)`, which may leak internal details.

**Improvements:**
- Use `logging` with levels (INFO, ERROR) and, preferably, JSON format.
- Log the exception on the server and return a generic message to the client.
- Enable the access log: `--access-logfile - --error-logfile -`.
- On AWS, send logs to **CloudWatch Logs** (CloudWatch Agent on EC2 or the `awslogs` driver on Docker), with a defined retention period.

---

### XII. Admin processes — ✅ Meets

**Principle:** run admin tasks as one-off processes, using the same code and config.

**Evidence:** table creation is a command of the application itself (`@app.cli.command("init-db")`), run with `flask init-db`. It uses the same code, the same environment variables and can be run manually at any time.

**Notes:**
- Being a one-off command, it can be run manually at any time. In a direct deployment on EC2 (without `entrypoint.sh`), it is enough to run `flask init-db` once to create the table, which was exactly the procedure used in the deployment and the behavior the factor expects.
- Quality caveat: `entrypoint.sh` calls `flask init-db || true`, and the `init_db()` function catches exceptions and only prints them. As a result, **table creation failures may go unnoticed** and the application starts even without the schema. This is an error-handling matter (see also factor XI), not a matter of the factor's principle.
- For schema evolution, a migration tool (Alembic, Flyway) will be needed, since `CREATE TABLE IF NOT EXISTS` does not alter existing tables.

**Improvements:**
- Make initialization fail visibly (non-zero exit code).
- Adopt a schema migration tool.

---

## 3. Production recommendations

The improvements below are suggestions for evolving the application. **They are not part of the Phase 1 scope**, which keeps the application as is.

1. **Logs (XI):** log exceptions with `logging`, enable the Gunicorn access log and do not return `str(e)` to the client.
2. **Config (III):** read `DB_PORT` in the code, validate all required variables and use `sslmode=verify-full` in the RDS connection.
3. **Secrets (III):** Secrets Manager is already used in Phase 1, with a manual read on EC2. The evolution is to automate the read (for example, in the `systemd` service) and handle secret rotation without a manual restart.
4. **Execution (IX):** run the application as a `systemd` service or in a container with automatic restart, and make `/health` check the database.
5. **Delivery (V, X):** version Docker images in a registry and automate deployment with CI/CD, running the same container in development and production.
6. **Scalability (VIII):** set `--workers` and adopt a connection pool or RDS Proxy.
7. **Schema (XII):** adopt a database migration tool.

---

## 4. Conclusion

ToggleMaster starts out with a solid foundation in several factors: **a single codebase in Git, declared dependencies, the database treated as an attached resource, stateless processes, port binding and admin tasks as one-off commands**. This explains why the application was moved from the local environment to AWS by changing only environment variables, which the deployment confirmed.

The gaps are concentrated in **production operations**: the build/release/run cycle is manual (factor V), logs do not record failures (XI), secrets management improved with Secrets Manager but still depends on a manual step in each session and on a restart after rotation (III), and there is little process and connection resilience (VIII and IX). These are expected problems in an MVP, and most have simple, low-cost fixes. For this phase, the application is deployed as is; the recommendations in section 3, such as a delivery pipeline, automated secret reading and observability, align with the project's planned evolution into a distributed and automated system in the coming phases.
