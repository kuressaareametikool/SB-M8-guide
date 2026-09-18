# 06 — Unit Testing

**Outcome ÕV5** — *mõistab ühiktestide olemust ning nende kasutamisvõimalusi*
Prerequisites: guides 01–05

---

## Why this matters

The outcome says *mõistab* — **understands**. Judgement (what deserves a test, and which kind) counts as
much as writing assertions. 200 tests that all need a database do not meet it.

---

## What a unit test is, and is not

A unit test:

- exercises **one unit of behaviour** in isolation
- runs in **milliseconds**
- needs **no** database, network, file system or real clock
- fails for **exactly one reason**

If it starts a Spring context, it is an integration test — useful, but a different tool with a different
cost.

### The pyramid for this project

| Level | Tool | Share | Speed |
|---|---|---|---|
| Unit | JUnit 5 + AssertJ, plain `new` | ~70% | milliseconds |
| Slice | `@WebMvcTest`, `@DataJpaTest` | ~20% | ~1 s each |
| End-to-end | `@SpringBootTest` + Testcontainers | ~10% | seconds |

The shape matters because of feedback speed. A suite you run after every save changes how you work; a
suite you run before lunch does not.

---

## Anatomy: Arrange–Act–Assert

```java
class WeightedAverageStrategyTest {

    private final WeightedAverageStrategy strategy = new WeightedAverageStrategy();

    @Test
    void weights_results_and_rounds_half_up() {
        // Arrange
        List<Result> results = List.of(
                result("80.0", "0.6"),
                result("55.0", "0.4"));

        // Act
        BigDecimal grade = strategy.calculate(results);

        // Assert
        assertThat(grade).isEqualByComparingTo("70.0");
    }

    @Test
    void rejects_zero_total_weight() {
        List<Result> results = List.of(result("80.0", "0.0"));

        assertThatThrownBy(() -> strategy.calculate(results))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("weight");
    }

    private static Result result(String points, String weight) {
        return new Result(new BigDecimal(points), new BigDecimal(weight));
    }
}
```

Three habits visible here: `isEqualByComparingTo` rather than `isEqualTo` (the scale trap from guide
05), a **test data factory method** so the arrange step stays readable, and a test name that states a
behaviour rather than a method name.

---

## JUnit 5 features worth teaching

### Parameterised tests

```java
@ParameterizedTest(name = "{0} days late → {1}%")
@CsvSource({"0, 0", "1, 2", "9, 18", "10, 20", "30, 20"})
void late_penalty_is_capped(long daysLate, int expected) {
    assertThat(penalty.forDaysLate(daysLate)).isEqualByComparingTo(valueOf(expected));
}
```

One test, many rows — perfect for decision tables.

### Nested grouping

```java
class GradingServiceTest {

    @Nested
    class WhenCourseUsesWeightedGrading {
        @Test void averages_by_weight() { ... }
        @Test void rejects_zero_weight() { ... }
    }

    @Nested
    class WhenCourseUsesBestOfN {
        @Test void takes_the_highest_five() { ... }
        @Test void handles_fewer_than_five_results() { ... }
    }
}
```

### Other essentials

| Feature | Use |
|---|---|
| `@DisplayName` | Readable output; can be a full English sentence |
| `@BeforeEach` | Shared arrange steps — avoid `@BeforeAll` with mutable state |
| `assertAll(...)` | When several properties of one result matter and you want all failures at once |
| `@Disabled("reason")` | Never without a reason, never for more than a sprint |

---

## AssertJ over JUnit assertions

```java
assertEquals(expected, actual);                          // which one is which?
assertThat(actual).isEqualTo(expected);                  // reads left to right

assertThat(enrolments)
        .hasSize(3)
        .extracting(Enrolment::getStatus)
        .containsExactly(CONFIRMED, CONFIRMED, PENDING);

assertThat(response.errors())
        .containsEntry("name", "must not be blank");
```

Failure messages are the real argument: AssertJ tells you what the collection contained, JUnit tells you
that two objects were not equal.

---

## Naming and independence

Name tests after **behaviour**:

```java
void rejects_enrolment_after_deadline()          // ✅
void testEnrol2()                                // ❌
void shouldWork()                                // ❌
```

Each test must pass **alone, in any order, repeatedly**. A test that depends on data left by another is
the most expensive kind of technical debt a student can create, because it fails months later in CI for
reasons nobody can reproduce.

Quick check: run one test on its own. Then run the class in reverse order (`junit.jupiter.testmethod.order.default`).

---

## Slice tests, briefly

`@DataJpaTest` is the right tool for proving a query works — mocking the repository there would defeat
the purpose:

```java
@DataJpaTest
class EnrolmentRepositoryTest {

    @Autowired private EnrolmentRepository repository;
    @Autowired private TestEntityManager em;

    @Test
    void finds_enrolments_with_students_in_one_query() {
        Course course = em.persist(new Course("Java", "WEIGHTED", 5));
        em.persist(new Enrolment(em.persist(new Student("Mari")), course));
        em.persist(new Enrolment(em.persist(new Student("Jaan")), course));
        em.flush();
        em.clear();

        List<Enrolment> found = repository.findByCourseWithStudents(course.getId());

        assertThat(found).hasSize(2)
                .extracting(e -> e.getStudent().getName())
                .containsExactlyInAnyOrder("Mari", "Jaan");
    }
}
```

`em.clear()` matters: without it the entities are already in the persistence context and the test passes
even when the query is wrong.

---

## Coverage, honestly

JaCoCo gives a number; the number is a hint, not a goal.

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.12</version>
  <executions>
    <execution><goals><goal>prepare-agent</goal></goals></execution>
    <execution>
      <id>check</id>
      <phase>verify</phase>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>PACKAGE</element>
            <includes><include>ee.tak24.kursusepunkt.grading.*</include></includes>
            <limits>
              <limit>
                <counter>BRANCH</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Note the choices: **branch** coverage, not line, and only on the **domain** packages. 90% line coverage
of getters proves nothing; 80% branch coverage of the grading algorithm is strong evidence.

---

## Exercises

### E6.1 — Domain tests (basic)
At least 15 unit tests over your domain logic, including every grading strategy and the penalty
function. No Spring context in any of them.

### E6.2 — Boundary table (basic)
One `@ParameterizedTest` covering a boundary table with at least five rows.

### E6.3 — Query test (intermediate)
One `@DataJpaTest` proving a custom query returns what you expect, with `em.clear()` before the assertion.

### E6.4 — Coverage gate (intermediate)
Add JaCoCo with a branch threshold on the domain packages and make `make test` fail below it.

### E6.5 — Judgement (intermediate)
In `docs/testing.md`, name one thing you deliberately did **not** unit test, and justify it.

### E6.6 — Bug hunt (advanced)
Take the class supplied with this exercise, which contains three bugs. Find them by writing tests only —
no debugger, no line-by-line reading. Commit the failing tests first, then the fixes.

---

## Solutions

<details>
<summary>E6.1 — the test list for one strategy</summary>

For `BestOfNWithMandatoryExamStrategy`, the behaviours worth a test each:

```java
weights_coursework_sixty_percent_and_exam_forty()
returns_zero_when_the_final_exam_is_failed()
throws_when_no_final_exam_result_exists()
takes_only_the_five_highest_coursework_results()
handles_fewer_than_five_coursework_results()
handles_exactly_one_result_which_is_the_exam()
rounds_to_one_decimal_half_up()
```

Seven tests for one class. Notice each name states an observable behaviour — you could hand this list to
another developer as a specification, which is the actual test of good test naming.
</details>

<details>
<summary>E6.5 — an answer that earns the mark</summary>

> **Not unit tested: `CourseMapper`.**
>
> The mapper is a field-for-field translation from entity to record with no branching and no
> calculation. A unit test for it would assert that `toResponse(course).name()` equals
> `course.getName()`, which restates the implementation rather than checking a behaviour: any change
> that breaks the mapping breaks the test in exactly the same way, so it catches nothing a compiler
> would not.
>
> The mapping *is* covered indirectly by the `@WebMvcTest` for `CourseController`, which asserts on the
> JSON body. If the mapper ever gains a conditional — say, hiding grades from unauthorised viewers —
> that logic gets its own test immediately.

The reasoning is what is being graded, not the conclusion. "I ran out of time" is not the same answer.
</details>

<details>
<summary>E6.6 — the technique</summary>

Work from the **contract**, not the code:

1. Read the Javadoc or method signature. What should it do for a typical input?
2. Write that test. Run it. Green means the happy path is fine.
3. Now attack the boundaries — empty input, single element, zero, negative, the maximum, `null`,
   duplicates, unsorted input.
4. Every red test is either a bug or a misunderstanding of the contract. Both are worth finding.

Classic seeded bugs: an off-by-one in a loop bound (`<` vs `<=`), integer division where decimal was
meant (`5/2 == 2`), and a boundary comparison that should be `>=` but is `>`. All three survive a
happy-path test and die instantly to a boundary test.
</details>

---

## Checklist — ÕV5 achieved

- [ ] 15+ meaningful unit tests over domain logic
- [ ] Unit layer runs in under ten seconds
- [ ] No test depends on execution order or another test's data
- [ ] Test names state behaviours
- [ ] Boundary and error cases covered, not just happy paths
- [ ] Coverage gate on domain packages, branch counter
- [ ] You can explain the difference between a unit and an integration test using your own code

**Exceeds:** parameterised boundary tables, a `@DataJpaTest` that would catch a real query regression,
and a written judgement about what was deliberately left untested.

