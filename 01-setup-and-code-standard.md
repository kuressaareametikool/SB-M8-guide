# 01 — Setup and Code Standard

**Outcome ÕV7** — *kasutab korrektselt kokkulepitud koodistandardit*
Prerequisites: guide 00

---

## Why this comes first

The key word in the outcome is *agreed*. A standard is a group decision recorded in the repository and
then enforced by a machine. Arguing about brace placement in a code review wastes everyone's time; a
formatter settles it in milliseconds.

It also comes first for a practical reason: retrofitting a standard at the end produces one giant
reformatting commit that proves nothing. The git history is the evidence for this outcome.

---

## Before you start

Guide 00 covers getting a project generated and running. This guide assumes you have done that: a
Spring Boot skeleton that starts, connects to PostgreSQL, and applies its first migration.

What follows is what turns that skeleton into a project a team can work in.

---

## 1. Boot 4 starter names

Boot 4 split the codebase into smaller, focused jars, so tutorials written for Boot 3 give artifact
names that no longer resolve:

| Boot 3 | Boot 4 |
|---|---|
| `spring-boot-starter-web` | `spring-boot-starter-webmvc` |
| Flyway auto-configuration inside `spring-boot-autoconfigure` | `org.springframework.boot:spring-boot-flyway` (a separate module) |
| `@WebMvcTest` from `spring-boot-starter-test` | `org.springframework.boot:spring-boot-webmvc-test` |
| `@DataJpaTest` from `spring-boot-starter-test` | `org.springframework.boot:spring-boot-data-jpa-test` |
| `@AutoConfigureTestDatabase` from `spring-boot-starter-test` | `org.springframework.boot:spring-boot-jdbc-test` |

`spring-boot-starter-test` still exists and is still what you depend on for JUnit, AssertJ and Mockito —
but in Boot 4 it no longer drags the **test slices** in with it. Miss those modules and the tests do not
compile; miss `spring-boot-flyway` and the application starts, runs no migrations at all, and fails with
`Schema validation: missing table` because `ddl-auto=validate` is looking at an empty database.

The packages moved with the modules, which is the other thing no Boot 3 tutorial will tell you:

| Boot 3 import | Boot 4 import |
|---|---|
| `org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest` | `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest` |
| `org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest` | `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` |
| `org.springframework.boot.test.autoconfigure.orm.jpa.AutoConfigureTestDatabase` | `org.springframework.boot.jdbc.test.autoconfigure.AutoConfigureTestDatabase` |

Add manually to `pom.xml`:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-flyway</artifactId>
</dependency>
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-webmvc-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-data-jpa-test</artifactId>
  <scope>test</scope>
</dependency>
```

There is no H2 dependency, deliberately: tests run against real PostgreSQL through Testcontainers
(guide 11).

Most Spring Boot material online targets Boot 3 or earlier. If a snippet uses
`spring-boot-starter-web`, `@MockBean`, or `javax.*` imports, it is out of date for this project.
Checking which version a snippet targets is a habit worth forming now.

---

## 2. Package layout

Group by **feature**, not by technical layer. This scales better and is what teams actually do.

```
ee.tak24.kursusepunkt
├── course
│   ├── Course.java                 // entity
│   ├── CourseRepository.java       // Spring Data interface
│   ├── CourseService.java          // business logic
│   ├── CourseController.java       // web layer
│   └── dto/
│       ├── CreateCourseRequest.java
│       └── CourseResponse.java
├── student
├── enrolment
├── grading                         // the complex component (guide 08)
└── common                          // exceptions, config, shared utilities
```

Layer-first packaging (`controller/`, `service/`, `model/`) puts every unrelated class in the same
folder and makes package-private visibility useless.

---

## 3. The agreed standard

Create `CODESTYLE.md`. Keep it short. It must state:

```markdown
# Code standard — TAK-24 KursusePunkt
Agreed: 2026-09-08. Changes require a group decision.

## Formatting
- Google Java Format, enforced by Spotless. `make mvn ARGS=spotless:apply` before every commit.
- Line length 100. UTF-8. LF line endings. Final newline.

## Language
- All identifiers, comments and documentation in English.

## Naming
| Thing | Convention | Example |
|---|---|---|
| Class | UpperCamelCase, noun | `EnrolmentService` |
| Method | lowerCamelCase, verb | `calculateFinalGrade` |
| Boolean method | reads as a question | `isEligible`, `hasPassed` |
| Constant | UPPER_SNAKE_CASE | `MAX_ENROLMENTS` |
| Package | lowercase, singular | `...kursusepunkt.enrolment` |
| Test | behaviour in snake_case | `rejects_zero_total_weight` |
| DB table / column | snake_case | `enrolment`, `final_grade` |
| Branch | `feature/<issue>-slug` | `feature/42-grade-report` |

## Rules
- Constructor injection only. No `@Autowired` on fields.
- No commented-out code in main. Git remembers.
- No `System.out.println` outside tests.
- Public domain API carries Javadoc.

## Commits
Conventional Commits: `feat|fix|test|docs|refactor|chore(scope): summary`

## Merging
No merge without one classmate's review and a green CI build.
```

Have the class **vote** on the two or three genuinely discretionary items. Ownership matters more than
the specific choice.

---

## 4. Automate it

```xml
<plugin>
  <groupId>com.diffplug.spotless</groupId>
  <artifactId>spotless-maven-plugin</artifactId>
  <version>2.44.5</version>
  <configuration>
    <java>
      <googleJavaFormat>
        <version>1.27.0</version>
      </googleJavaFormat>
      <removeUnusedImports/>
      <trimTrailingWhitespace/>
      <endWithNewline/>
    </java>
  </configuration>
  <executions>
    <execution>
      <goals><goal>check</goal></goals>
      <phase>validate</phase>
    </execution>
  </executions>
</plugin>
```

Both versions matter on Java 26: Spotless 2.43.0 bundles a google-java-format built against an older
`javac`, and the build dies with
`NoSuchMethodError: com.sun.tools.javac.util.Log$DeferredDiagnosticHandler.getDiagnostics()`.

- `make mvn ARGS=spotless:apply` — fixes
- `make mvn ARGS=spotless:check` — verifies
- bound to `validate` — a badly formatted commit cannot pass the build

Pre-commit hook (`.git/hooks/pre-commit`, then `chmod +x`):

```bash
#!/bin/sh
# Runs in a container, so the hook works without a local Maven.
docker run --rm -v "$PWD":/w -v kursusepunkt-dev_maven-repo:/root/.m2 -w /w \
  maven:3.9-eclipse-temurin-26 mvn -q spotless:apply || exit 1
git add -u
```

---

## 5. CI

`.github/workflows/build.yml`:

```yaml
name: build
on: [push, pull_request]
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '26'
          distribution: 'temurin'
          cache: 'maven'
      - run: mvn -B verify
```

One green badge in the README is worth a page of promises.

---

## Exercises

### E1.1 — Repository (basic)
Push your project to your own Git repository, with the skeleton as its first commit.

### E1.2 — Standard (basic)
Write `CODESTYLE.md`, agree it with the group, record the date.

### E1.3 — Enforcement (intermediate)
Add Spotless bound to `validate`. Prove it works: commit a deliberately misformatted file, show the
failing build, then fix it. Both commits stay in history.

### E1.4 — CI (intermediate)
Add the workflow. Your repository must show one red run and the commit that turned it green.

### E1.5 — Review (intermediate)
Open a pull request for a small change. A classmate reviews it with at least three substantive,
**non-formatting** comments.

### E1.6 — Hook (advanced)
Add a pre-commit hook and document in `CODESTYLE.md` how teammates install it. Explain in two sentences
why a hook is not a substitute for CI.

---

## Solutions

<details>
<summary>E1.3 — showing the enforcement works</summary>

```bash
# break formatting on purpose
sed -i 's/public class CourseService {/public class CourseService{/' \
  src/main/java/ee/tak24/kursusepunkt/course/CourseService.java

git commit -am "chore: deliberately misformatted file to test spotless"
make test
# → [ERROR] Failed to execute goal ...spotless... The following files had format violations

make mvn ARGS=spotless:apply
git commit -am "style: apply spotless formatting"
make test   # BUILD SUCCESS
```

The point for the write-up: the failure came from the build, not from a human reading the diff.
</details>

<details>
<summary>E1.6 — why a hook is not enough</summary>

A hook lives in `.git/hooks`, which is **not** version controlled and not shared by cloning. Anyone can
skip it with `git commit --no-verify`. It is a convenience that gives fast local feedback; CI is the
gate, because it runs on the server where nobody can bypass it.

A common improvement: keep hooks in a committed `hooks/` directory and set
`git config core.hooksPath hooks` in the setup instructions.
</details>

---

## Checklist — ÕV7 achieved

- [ ] `CODESTYLE.md` exists, is specific, and carries an agreement date
- [ ] `make test` passes on a clean clone with no manual formatting step
- [ ] CI runs on every push and blocks merging when red
- [ ] Commit messages follow one convention across the whole history
- [ ] No commented-out dead code, no stray `System.out.println`
- [ ] At least one pull request reviewed by a classmate with substantive comments

**Exceeds:** pre-commit hook documented, Checkstyle rules beyond formatting, and review comments that
changed a design rather than a space.

