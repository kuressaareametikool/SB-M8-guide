# 10 — Capstone Brief and Assessment

**All outcomes** · Everything in guides 01–11 is practice for this.

---

## The brief: KursusePunkt

### Context

A vocational school tracks courses, students, enrolments and grades in spreadsheets. Grades are
calculated differently per course, prerequisites are checked by eye, and nobody trusts the numbers.
Build the replacement. **You build the backend; another module team builds the interface on top of your
API.**

### Required functionality

1. Teachers create courses, each with a grading rule chosen from several strategies and a set of
   weighted assignments.
2. Students are enrolled in courses; the same student cannot be enrolled twice in the same course.
3. Teachers record results per assignment; the final grade is **calculated, never typed in**.
4. Late submissions receive an automatic penalty according to a documented rule.
5. A course report returns every enrolled student, their final grade and the class distribution.
6. Prerequisites between courses are validated; an impossible configuration is refused with a clear
   explanation of *why*.
7. One role-restricted area: students see only their own grades.

### Constraints

- Spring Boot 4.1, Java 26, PostgreSQL, Flyway
- REST API only, no server-rendered views
- No business logic in controllers; no entities in JSON responses
- No `double` in any grade calculation
- Schema changes only via migrations, `ddl-auto=validate`
- `make test` passes on a clean clone with formatting and coverage gates enabled
- The whole project runs with Docker alone; no local JDK, Maven or PostgreSQL required

### Deliverables

| Deliverable | Covers | Guide |
|---|---|---|
| Git repository with full history | ÕV7 | 01 |
| Running API + committed `openapi.json` | ÕV3 | 02 |
| Correct persistence layer with migrations | ÕV4 | 03 |
| `PATTERNS.md` + strategy implementation | ÕV1 | 04 |
| Grade calculation with `BigDecimal` | ÕV2 | 05 |
| Test suite: unit, slice and mock tests | ÕV5, ÕV6 | 06, 07 |
| One complex component + `docs/algorithm.md` | ÕV8 | 08 |
| README, Javadoc, diagrams, three ADRs | ÕV9 | 09 |
| `docs/ai-usage.md` | — | — |
| 15-minute defence with live demo | all | — |

### The defence

Ten minutes of demo and walkthrough, five minutes of questions. Expect:

- "Show me where the grading strategy is selected."
- "What happens if two teachers save the same enrolment at once?"
- "Why is this a `BigDecimal`?"
- "What SQL does this line run?"

The defence is where partial outcomes are confirmed or fail.

---

## Assessment rubric

Each outcome is assessed separately as **not yet / achieved / exceeds**. **All nine must reach
*achieved* to pass**; *exceeds* raises the grade but never compensates for a missing outcome.

| Outcome | Not yet | Achieved | Exceeds |
|---|---|---|---|
| **ÕV1** Patterns | Names patterns from memory without finding them in code | Identifies and correctly uses DI, repository and strategy; explains each in own words | Implements a pattern not taught, justifies it, and names an anti-pattern they refactored away |
| **ÕV2** Math & logic | Uses `double` for grades; rounding is accidental | Correct `BigDecimal` use, explicit rounding, boundary cases handled | Own statistical function plus a documented decision table with full branch coverage |
| **ÕV3** MVC | Logic in controllers; entities serialised to JSON | Clean Controller–Service–Repository separation, DTOs at the boundary, correct status codes | Layering enforced by ArchUnit, central problem-detail handling, contract agreed with the UI module first |
| **ÕV4** ORM | `ddl-auto=update`; N+1 present and unnoticed | Correct relations and fetch types, Flyway migrations, N+1 found and fixed with evidence | Projections, entity graphs, correct paginated fetching, measured before/after query counts |
| **ÕV5** Unit tests | Few tests, or tests that need a database to pass | 15+ meaningful unit tests, AAA structure, boundary cases, coverage gate on domain packages | Parameterised boundary tables, a `@DataJpaTest`, and a written judgement on what was left untested |
| **ÕV6** Mocks | Mocks copied from a tutorial and unexplained | Correct `@Mock`/`@InjectMocks`/`@MockitoBean` use; asserts interaction *and* absence of interaction | `ArgumentCaptor`, injected `Clock`, explicit reasoning about mock vs fake |
| **ÕV7** Code standard | Inconsistent formatting; no agreed standard | `CODESTYLE.md` agreed, Spotless enforced in the build, consistent commit convention | CI gate, pre-commit hook, substantive peer reviews in pull requests |
| **ÕV8** Complexity | CRUD only | One non-trivial algorithm implemented and tested, plus two application-complexity features | Complexity analysis matched by measurements, a properly argued rejected alternative, concurrency or security handled deliberately |
| **ÕV9** Documentation | Thin README; Estonian mixed into code | Complete README, Javadoc on domain API, diagrams, correct English | ADRs, committed OpenAPI, and a passed fresh-eyes test with the reviewer's notes submitted |

### Weighting, if a single mark is required

| Component | Share |
|---|---|
| Working application (ÕV1–ÕV4, ÕV8) | 55% |
| Tests and code standard (ÕV5–ÕV7) | 25% |
| Documentation and defence (ÕV9) | 20% |

The pass condition is **all nine outcomes at *achieved***.

---

## Order of work

| Step | Guide | Due at the end |
|---|---|---|
| 1 | 01 Setup & standard | Skeleton pushed, CI green, `CODESTYLE.md` |
| 2 | 02 REST & MVC | CRUD for one resource, contract sketch |
| 3 | 03 ORM | Schema and migrations complete |
| 4 | 04 Patterns | Strategy implementation, `PATTERNS.md` |
| 5 | 05 Math & logic | Grade calculation |
| 6 | 06 Unit testing | 15 unit tests, coverage gate |
| 7 | 07 Mocks | Mock tests, one `@WebMvcTest` |
| 8 | 08 Complex component | Algorithm + `docs/algorithm.md` |
| 9 | 09 Documentation | Full submission |

Guide 11 (containerised setup) belongs with step 1, and is revisited for Testcontainers at step 6.

---

## If you are short of time

Cut the number of **resources**, not the depth. One fully modelled resource with clean layering, real
tests and a documented contract is worth more than four shallow ones.

Do not cut guide 08 or the testing guides — those are the hardest outcomes to add afterwards.
