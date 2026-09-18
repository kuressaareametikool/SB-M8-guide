# 00 — Getting Started

**Start here.** By the end of this guide you have a Spring Boot project running against a PostgreSQL
database, and you can change a line of code and see the result.

Prerequisites: **Docker** and **Git**. Nothing else — no JDK, no Maven, no PostgreSQL.

Everything in this project runs in a container: the build, the application, the tests, the database,
even the command that generates the project in the first place. Your machine contributes an editor and
a Docker daemon.

Why this way: every machine behaves identically, a broken local JDK cannot stop you, and the version of
Java the project uses is a line in a file rather than a thing you installed once and forgot.

---

### 1. Check Docker works

```bash
docker run --rm hello-world
```

If that prints a greeting, you have everything you need. If it fails, fix this before going further —
nothing below will work otherwise.

### 2. Get the project

The template repository holds a working Spring Boot skeleton with the container setup, one worked
resource and its tests already in place.

1. Open **https://github.com/<your-org>/tak24-spring-template**
2. Click **Use this template → Create a new repository**
3. Name it, make it private or public as your course requires, and create it
4. Clone your new repository:

```bash
git clone https://github.com/<you>/kursusepunkt.git
cd kursusepunkt
```

You now own the history from the first commit. The template is not a dependency — nothing links back
to it, and you are free to change every file in it.

### 3. What you were given

```
kursusepunkt/
├── pom.xml                      Spring Boot 4.1, Java 26, Spotless, JaCoCo
├── Dockerfile                   multi-stage production image
├── compose.yaml                 full stack
├── compose.dev.yaml             development stack
├── Makefile                     make help lists everything
├── CODESTYLE.md                 fill in the agreement date (guide 01)
├── .devcontainer/               VS Code environment
├── .github/workflows/build.yml  CI
└── src/
    ├── main/java/…/course/      one resource through the whole stack
    ├── main/resources/
    │   ├── application.yaml
    │   └── db/migration/V1__create_course.sql
    └── test/java/…/             web slice test, Testcontainers test, ArchUnit rules
```

The `course` package is a worked example of everything guides 02 and 03 cover: a controller that only
translates HTTP, a service that owns the rules and the transaction, request and response records so no
entity is ever serialised, and a Flyway migration rather than `ddl-auto`.

Read it before you extend it. The rest of the domain — students, enrolments, assignments, results —
follows the same shape.

### 4. Run it

```bash
make dev-docker
```

or, without the Makefile:

```bash
docker compose -f compose.dev.yaml --profile docker-dev up
```

The first run downloads a Maven image and every dependency — expect several minutes. Later runs take
seconds, because the dependency cache lives in a named volume.

You should see Flyway apply `V1__create_course`, then:

```
Tomcat started on port 8080 (http)
Started KursusePunktApplication in 4.2 seconds
```

Check it:

```bash
curl http://localhost:8080/actuator/health
# {"status":"UP"}
```

### 5. Change something

Create a course through the API:

```bash
curl -i -X POST http://localhost:8080/api/courses \
  -H 'Content-Type: application/json' \
  -d '{"name":"Java Backend","gradingKey":"WEIGHTED","credits":5}'
# HTTP/1.1 201 Created
# Location: /api/courses/1

curl http://localhost:8080/api/courses
```

Then change something in `CourseController` and save. DevTools recompiles and restarts the application
inside the container within a few seconds — no rebuild, no `docker compose up` again.

> **If restarts do not happen**, your IDE is not writing compiled classes into the mounted folder.
> Either enable "build automatically", or run `make mvn ARGS=compile` in another terminal after each
> change.

### 6. Attach a debugger (optional)

The `app-dev` service exposes JVM debug port **5005**. In IntelliJ: *Run → Edit Configurations → Add →
Remote JVM Debug*, host `localhost`, port `5005`. Breakpoints in your source then work exactly as if the
application were running locally.

### Running Maven commands

You never type `mvn` directly. Every goal runs in a container against the shared dependency cache:

```bash
make test                         # verify: tests, formatting, coverage
make build                        # package the jar
make mvn ARGS="spotless:apply"    # any goal at all
make mvn ARGS="dependency:tree"
```

If you prefer the raw form, this is what `make mvn` expands to:

```bash
docker run --rm -v "$PWD":/w -v kursusepunkt-dev_maven-repo:/root/.m2 -w /w \
  maven:3.9-eclipse-temurin-26 mvn -B <goal>
```

---

## Editing comfortably

Your editor runs on the host and the code lives on the host, so normal editing works. Two ways to get
full IDE features without installing a JDK:

**VS Code dev container** — open the project and choose *Reopen in Container*. The JDK, Maven and Java
extensions all live inside `.devcontainer`, so IntelliSense, run and debug work with nothing installed
locally.

**IntelliJ with a Docker SDK** — *File → Project Structure → SDKs → Add → Download JDK* places a JDK
inside IntelliJ's own directory rather than on your PATH. That is enough for code completion; running
and debugging still go through the container and port 5005.

Without either, you get syntax highlighting but no completion. That is workable for a first week and
frustrating after it, so set one of them up early.

---

## Verifying the whole thing works

Run this before moving to guide 01. All four must pass:

```bash
# 1. The database is reachable
docker compose -f compose.dev.yaml exec db pg_isready -U app -d kursusepunkt

# 2. The application starts and is healthy
curl -fs http://localhost:8080/actuator/health

# 3. Migrations applied
docker compose -f compose.dev.yaml exec db \
  psql -U app -d kursusepunkt -c "SELECT version, description FROM flyway_schema_history;"

# 4. The production image builds
make image
docker images kursusepunkt:local
```

---

## Your first commit

The repository already has history — the template's initial commit is yours now. Set up your local
configuration and commit your first change:

```bash
cp .env.example .env        # .env is git-ignored; .env.example is not
# fill in the agreement date in CODESTYLE.md (guide 01)
git commit -am "chore: record code standard agreement date"
git push
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `port is already allocated` on 5432 | A PostgreSQL is already running on your machine | Stop it, or change `DB_PORT` in `.env` |
| `Connection refused` to the database | The app started before PostgreSQL was ready | The compose health check handles this; if you started services individually, start `db` first |
| `FlywayValidateException` on start | Entities and schema disagree | Write the missing migration; never switch to `ddl-auto=update` |
| App starts, then exits code 137 | Container ran out of memory | Raise Docker Desktop's memory limit (4 GB minimum) |
| Changes do not trigger a restart | Compiled classes are not reaching the mounted folder | Enable automatic build in the IDE, or run `make mvn ARGS=compile` |
| `permission denied` writing `target/` on Linux | Container writes as root, host user cannot overwrite | `sudo chown -R $USER:$USER target/`, or add `user: "${UID}:${GID}"` to `app-dev` |
| Very slow file access on macOS/Windows | Bind-mount performance | Add `:delegated` to the source mount, or enable VirtioFS in Docker Desktop |

---

## What is next

- **[Guide 01](01-setup-and-code-standard.md)** — project layout, code standard, CI
- **[Guide 11](11-docker-setup.md)** — how the Dockerfile and compose files actually work, and
  Testcontainers for real database tests
