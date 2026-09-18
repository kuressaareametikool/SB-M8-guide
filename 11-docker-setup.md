# 11 — Containerised Setup

**Supports ÕV7 (reproducible build), ÕV4 (real database in tests), ÕV9 (documented setup)**
Prerequisites: guide 01

---

## Why containerise

Three reasons:

1. **"It works on my machine" stops being an argument.** The image is the machine.
2. **Setup takes minutes.** No local JDK, no PostgreSQL install, no version drift between laptops.
   Upgrading Java is a one-line change to a base image, not an afternoon.
3. **Tests get a real database.** Testcontainers means your `@DataJpaTest` runs against PostgreSQL 16,
   not H2 pretending to be it — and H2 does lie, particularly about types, constraints and SQL dialect.

Nothing is installed on your machine: no JDK, no Maven, no PostgreSQL. These files are already in your
repository — they came from the template — and this guide explains what each one does, so you can
change them rather than treat them as magic.

---

## The files in your repository

| File | Purpose |
|---|---|
| `Dockerfile` | Multi-stage build → small, non-root runtime image |
| `.dockerignore` | Keeps build context small and secrets out of images |
| `compose.yaml` | Full stack: app + database (+ optional pgAdmin) |
| `compose.dev.yaml` | Development: database + application, both containerised |
| `.env.example` | Documented configuration; copied to the git-ignored `.env` |
| `.devcontainer/devcontainer.json` | One-click identical environment in VS Code |
| `.github/workflows/build.yml` | CI: tests, image build, smoke test |
| `Makefile` | Shorthand for the commands you run daily |

---

## Two modes, and when to use each

Both run entirely in containers. The difference is *how* the application is built and started.

### Development — `make dev-docker`

```bash
make dev-docker     # compose --profile docker-dev up
```

A Maven container with your source bind-mounted. DevTools restarts the application when compiled
classes change, so a code edit takes effect in seconds with no image rebuild. The dependency cache
lives in a named volume, so only the first run is slow. Debug port 5005 is open for your IDE.

This is daily work.

### Verification — `make up`

```bash
make up             # docker compose up --build -d
make logs
```

Builds the real production image from the `Dockerfile` and runs that. Slower (a full rebuild each time),
but it is the artifact you actually ship.

Use it before pushing and when handing over to the UI module. It catches what development mode cannot:
a missing dependency in the runtime image, a wrong entrypoint, a file that only existed on your disk.

> **Use both.** Development mode alone means finding out late that the image does not start.
> Verification mode alone makes every edit a two-minute wait.

---

## Reading the Dockerfile

### Multi-stage: why three stages

```dockerfile
FROM eclipse-temurin:26-jdk AS deps      # ~450 MB, has Maven, compilers, everything
...
FROM eclipse-temurin:26-jre AS runtime   # ~200 MB, can only run
```

The build tools never reach the final image. That is smaller, faster to pull, and has a smaller attack
surface — a compiler in a production container is a gift to an attacker.

### Layer caching: order matters

```dockerfile
COPY mvnw pom.xml ./
RUN ./mvnw -B dependency:go-offline       # cached unless pom.xml changes
COPY src/ src/
RUN ./mvnw -B clean package -DskipTests   # re-runs on any source change
```

Copy the `pom.xml` alone first. Dependency download is the slow part, and it only needs to happen when
dependencies change. Copying `src/` first would re-download every jar on every edit — a common and
invisible mistake that costs minutes on every build.

### Layered jars

```dockerfile
RUN java -Djarmode=tools -jar target/*.jar extract --layers --launcher --destination extracted
```

A Spring Boot fat jar is one 60 MB file: change one line of code and the whole thing is a new layer in
the registry. Extracting into layers separates dependencies (rarely change, ~55 MB) from your classes
(change constantly, ~200 KB), so a redeploy pushes and pulls kilobytes.

> `-Djarmode=tools ... extract` is the Boot 3.3+/4.x form. Older tutorials use
> `-Djarmode=layertools -jar app.jar extract`, which is deprecated.

### Non-root

```dockerfile
RUN groupadd --system --gid 1001 spring && useradd --system --uid 1001 --gid spring ...
USER spring
```

Containers run as root by default. If the application is compromised, root in the container is a much
better starting position for an attacker than an unprivileged user. This is two lines; there is no
excuse.

### Heap sizing

```dockerfile
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError"
```

Without `MaxRAMPercentage`, a JVM in a memory-limited container can size its heap from the *host's*
memory and get OOM-killed by the kernel with no Java stack trace — one of the most confusing failures a
beginner can meet. `ExitOnOutOfMemoryError` makes the container die and restart rather than limp along
in a degraded state.

---

## Reading compose.yaml

### Health checks and start order

```yaml
depends_on:
  db:
    condition: service_healthy
```

`depends_on` alone only waits for the container to *start*, not for PostgreSQL to accept connections.
Without the health condition, Flyway runs against a database that is not listening yet, the application
exits, and the student concludes Docker is broken. The `pg_isready` health check is what makes the
dependency real.

### Named volumes

```yaml
volumes:
  db-data:/var/lib/postgresql/data
```

Without this, `docker compose down` deletes the database. With it, data survives restarts and
`docker compose down -v` is the deliberate way to wipe it — which is exactly what you want when testing
that migrations run from empty.

### Profiles

```yaml
pgadmin:
  profiles: ["tools"]
```

Optional services do not start by default. `docker compose --profile tools up` opts in.

---

## Testcontainers: a real database in tests

This is the payoff for guide 06 and the reason H2 can be retired.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-testcontainers</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>postgresql</artifactId>
  <scope>test</scope>
</dependency>
```

```java
@TestConfiguration(proxyBeanMethods = false)
public class ContainerConfig {

    @Bean
    @ServiceConnection            // wires spring.datasource.* automatically
    PostgreSQLContainer<?> postgres() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }
}
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)   // do NOT swap in an embedded database
@Import(ContainerConfig.class)
class EnrolmentRepositoryTest {

    @Autowired private EnrolmentRepository repository;

    @Test
    void unique_constraint_prevents_double_enrolment() {
        // This test only means something against real PostgreSQL:
        // H2 handles constraint violations and error codes differently.
    }
}
```

`@ServiceConnection` (Boot 3.1+) replaces the old `@DynamicPropertySource` boilerplate. Reuse one
container across the test suite rather than starting one per class — otherwise the suite takes minutes.

### Running the application against a container in development

```java
// src/test/java/.../TestKursusePunktApplication.java
public class TestKursusePunktApplication {
    public static void main(String[] args) {
        SpringApplication.from(KursusePunktApplication::main)
                .with(ContainerConfig.class)
                .run(args);
    }
}
```

Run this class from the IDE and you get the application plus a throwaway PostgreSQL, with no compose
file at all. No compose file, no manual cleanup.

---

## Dev containers

`.devcontainer/devcontainer.json` gives every student the same JDK, Maven and extensions inside a
container. "Reopen in Container" in VS Code, or Codespaces in the browser for anyone whose laptop is not
cooperating.

The trade-off: the first build is slow on a poor connection, and you skip learning to set up a local
toolchain, which you will need eventually.

---

## Exercises

### E11.1 — Run it (basic)
Start the full stack with `make up`. Confirm the API answers and the health endpoint reports `UP`.

### E11.2 — Persistence (basic)
Create a course through the API. Run `make down`, then `make up`. Is it still there? Now run
`make clean` and `make up`. Explain both results in two sentences.

### E11.3 — Image size (intermediate)
Build the image and record its size with `docker images`. Then build a naive single-stage version using
the JDK image, and compare. Put both numbers in `docs/docker.md`.

### E11.4 — Cache behaviour (intermediate)
Change one line in a Java file and rebuild. Time it. Then change `pom.xml` and rebuild. Time that.
Explain the difference in terms of layer caching.

### E11.5 — Testcontainers (intermediate)
Convert one `@DataJpaTest` from H2 to Testcontainers. Find one behaviour that differs between the two
databases and document it.

### E11.6 — Health check (advanced)
Break the database connection (stop the `db` container) and observe what the application's health
endpoint reports and what compose does about it. Then explain the difference between liveness and
readiness.

---

## Solutions

<details>
<summary>E11.2 — why the results differ</summary>

After `make down` and `make up`, **the course is still there**: `down` removes containers but the named
volume `db-data` survives, and PostgreSQL's data directory lives in that volume, not in the container's
writable layer.

After `make clean`, **it is gone**: `down -v` deletes the volume, so the next start initialises an empty
database and Flyway re-applies every migration from V1. This is the correct way to verify the claim in
guide 03 that "the application must start against an empty database with no manual SQL" — and if that
fails, the migration history is broken.
</details>

<details>
<summary>E11.3 — the numbers to expect</summary>

| Build | Approximate size |
|---|---|
| Single-stage on `26-jdk` with the fat jar | ~520 MB |
| Multi-stage on `26-jre`, layered | ~270 MB |

Roughly half, and the saved half is compilers and build tooling that the running application never uses.
The security argument is the stronger one: nothing in the runtime image can compile or download code.

To go further, try `jlink` or an `-alpine` JRE base: smaller still, but with musl libc quirks and fewer
prebuilt images available.
</details>

<details>
<summary>E11.6 — liveness vs readiness</summary>

With the database stopped, `/actuator/health` reports `DOWN` because the `db` health indicator fails.

The distinction:

- **Liveness** — "is this process broken beyond recovery?" If yes, restart it. A missing database does
  **not** qualify: restarting the application will not bring PostgreSQL back, and a restart loop makes
  recovery slower and the logs useless.
- **Readiness** — "can this instance serve traffic right now?" If no, stop routing requests to it, but
  leave it running so it can recover when the database returns.

Spring Boot exposes both separately:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
```

giving `/actuator/health/liveness` and `/actuator/health/readiness`. The database indicator belongs in
readiness, not liveness — which is exactly the mistake the compose health check in this repository would
make if it were wired to a restart policy.
</details>

---

## Checklist

- [ ] `docker compose up --build` produces a working API on a clean machine
- [ ] The runtime image is multi-stage, JRE-based, and runs as a non-root user
- [ ] `depends_on` uses a health condition, not bare ordering
- [ ] Database data survives `down` and is removed by `down -v`
- [ ] `.env` is git-ignored; `.env.example` is committed and documented
- [ ] No secret, password or key is baked into any image layer
- [ ] At least one test runs against real PostgreSQL via Testcontainers
- [ ] CI builds the image and smoke-tests it, not just the jar

