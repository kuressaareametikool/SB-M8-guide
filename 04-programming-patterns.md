# 04 — Programming Patterns

**Outcome ÕV1** — *tunneb enamlevinud programmeerimismustreid*
Prerequisites: guides 01–03

---

## Why this matters

Laravel hides its patterns behind facades; Spring makes them explicit and named. The outcome is met when
you can point at a class and say *"that is a strategy, injected as a singleton by a container that does
dependency inversion"* — not when you can list patterns from a textbook.

---

## Patterns the framework hands you

| Pattern | Where it appears | What it buys you |
|---|---|---|
| Dependency Injection / IoC | Constructor injection into `@Service`, `@RestController` | Classes declare what they need; the container wires it. Makes mocking possible (guide 07) |
| Singleton | Default bean scope — one instance per context | Cheap, but beans must be **stateless** |
| Repository | `interface CourseRepository extends JpaRepository<Course, Long>` | Persistence hidden behind a collection-like interface |
| Proxy | `@Transactional`, `@Cacheable` — Spring wraps your bean | Explains why self-invocation of `@Transactional` does nothing (guide 03) |
| Template Method | `JdbcClient`, `RestClient` | Boilerplate fixed, the varying step passed in |
| Observer | `ApplicationEventPublisher` + `@EventListener` | Decouples "grade saved" from "send notification" |
| Builder | `MockMvcRequestBuilders`, `RestClient.builder()` | Readable construction of objects with many options |
| Factory | `@Bean` methods in a `@Configuration` class | Centralised, conditional object creation |
| Adapter | Your own interface wrapping a third-party client | Lets you swap the vendor and mock your own type |

### Dependency injection, concretely

```java
@Service
public class GradingService {
    private final EnrolmentRepository repository;    // final = cannot be forgotten
    private final Clock clock;

    GradingService(EnrolmentRepository repository, Clock clock) {
        this.repository = repository;
        this.clock = clock;
    }
}
```

Constructor injection over field injection because: the class is honest about its dependencies, the
fields can be `final`, and you can construct it in a unit test with `new` and two mocks — no Spring
context required. Field injection (`@Autowired` on a field) makes all three impossible.

---

## The pattern you write yourself: Strategy

Grading rules differ per course — some weight assignments, some take the best N results, some are
pass/fail. This is the single best teaching example in the project, because guides 04, 05, 06, 07 and 08
all land on it.

```java
public interface GradingStrategy {
    String key();
    BigDecimal calculate(List<Result> results);
}
```

```java
@Component
class WeightedAverageStrategy implements GradingStrategy {

    @Override public String key() { return "WEIGHTED"; }

    @Override
    public BigDecimal calculate(List<Result> results) {
        BigDecimal weightSum = results.stream()
                .map(Result::weight)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        if (weightSum.compareTo(BigDecimal.ZERO) == 0) {
            throw new IllegalArgumentException("Total weight must be greater than zero");
        }

        BigDecimal weighted = results.stream()
                .map(r -> r.points().multiply(r.weight()))
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        return weighted.divide(weightSum, 1, RoundingMode.HALF_UP);
    }
}

@Component
class BestOfNStrategy implements GradingStrategy {

    @Override public String key() { return "BEST_OF_5"; }

    @Override
    public BigDecimal calculate(List<Result> results) {
        return results.stream()
                .map(Result::points)
                .sorted(Comparator.reverseOrder())
                .limit(5)
                .reduce(BigDecimal.ZERO, BigDecimal::add)
                .divide(BigDecimal.valueOf(Math.min(5, Math.max(1, results.size()))), 1, HALF_UP);
    }
}
```

```java
@Service
public class GradingService {

    private final Map<String, GradingStrategy> strategies;

    // Spring injects EVERY GradingStrategy bean; we index them by key.
    GradingService(List<GradingStrategy> found) {
        this.strategies = found.stream()
                .collect(toMap(GradingStrategy::key, identity()));
    }

    public BigDecimal grade(Course course, List<Result> results) {
        GradingStrategy strategy = strategies.get(course.getGradingKey());
        if (strategy == null) {
            throw new UnknownGradingStrategyException(course.getGradingKey());
        }
        return strategy.calculate(results);
    }
}
```

Note what this removes: **no `switch` over course types**, and adding a third rule means adding one class
and changing nothing else. That is the open/closed principle made concrete, and injecting a `List<T>` of
every implementation is a Spring idiom worth knowing on its own.

---

## Observer: events instead of direct calls

```java
public record GradeFinalised(Long enrolmentId, BigDecimal grade, Instant at) {}

@Service
public class GradingService {
    private final ApplicationEventPublisher events;

    public void finalise(Enrolment enrolment) {
        enrolment.setFinalGrade(grade(enrolment.getCourse(), enrolment.getResults()));
        events.publishEvent(new GradeFinalised(enrolment.getId(), enrolment.getFinalGrade(), now()));
    }
}

@Component
class GradeAuditListener {
    @EventListener
    void on(GradeFinalised event) {
        auditRepository.save(AuditEntry.of(event));
    }
}
```

Why this beats calling `auditService.record(...)` directly: `GradingService` stops depending on auditing
at all, a second listener (notification, statistics) can be added without touching it, and the unit test
for grading no longer needs an audit mock.

The counter-argument: events make the flow harder to follow in an
IDE, and `@EventListener` runs in the same transaction unless you use `@TransactionalEventListener`. Use
events where the coupling actually hurts, not everywhere.

---

## Anti-patterns to name out loud

| Anti-pattern | Why it hurts |
|---|---|
| **Fat controller** — business logic in the controller | Cannot be unit tested without HTTP |
| **Pass-through service** — a service that only forwards to the repository | Adds a layer and no value; either give it a job or drop it |
| **Field injection** (`@Autowired` on fields) | Hides dependencies, blocks constructor-based testing |
| **God service** — 900 lines in `CourseService` | Split by responsibility |
| **Stateful singleton** — a mutable field on a `@Service` | One instance shared by every request; a race condition waiting to happen |
| **`switch` on a type code** | The thing strategy exists to remove |

---

## Exercises

### E4.1 — Strategy (basic)
Implement `GradingStrategy` with at least three concrete strategies, one of them non-trivial (for
example: "best 5 of 7, but the final exam must be passed").

### E4.2 — Open/closed (basic)
Add a fourth strategy. Prove with `git diff` that no existing file changed.

### E4.3 — Events (intermediate)
Publish a domain event when a grade is finalised, consume it with `@EventListener`, write an audit row.
Explain in one paragraph why this beats calling the audit service directly — and name one drawback.

### E4.4 — Pattern inventory (intermediate)
Write `PATTERNS.md` listing five patterns you found in your own code or in Spring itself, with a file
and line reference for each and two sentences on the problem it solves.

### E4.5 — Refactor (intermediate)
Find a `switch` or `if`/`else` chain over a type code in your project and replace it with a strategy or
a polymorphic method. Keep both versions in history.

### E4.6 — Stateful bean (advanced)
Deliberately add a mutable field to a `@Service`, write a test that hits it from multiple threads, and
demonstrate the corruption. Then fix it and explain why the singleton scope caused it.

---

## Solutions

<details>
<summary>E4.1 — a non-trivial strategy</summary>

```java
@Component
class BestOfNWithMandatoryExamStrategy implements GradingStrategy {

    private static final BigDecimal PASS_MARK = new BigDecimal("50.0");

    @Override public String key() { return "BEST_5_OF_7_EXAM_REQUIRED"; }

    @Override
    public BigDecimal calculate(List<Result> results) {
        Result exam = results.stream()
                .filter(Result::isFinalExam)
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("No final exam result recorded"));

        if (exam.points().compareTo(PASS_MARK) < 0) {
            return BigDecimal.ZERO;          // failing the exam fails the course
        }

        List<BigDecimal> best = results.stream()
                .filter(r -> !r.isFinalExam())
                .map(Result::points)
                .sorted(Comparator.reverseOrder())
                .limit(5)
                .toList();

        if (best.isEmpty()) {
            return exam.points().setScale(1, HALF_UP);
        }

        BigDecimal coursework = best.stream()
                .reduce(BigDecimal.ZERO, BigDecimal::add)
                .divide(BigDecimal.valueOf(best.size()), 4, HALF_UP);

        return coursework.multiply(new BigDecimal("0.6"))
                .add(exam.points().multiply(new BigDecimal("0.4")))
                .setScale(1, HALF_UP);
    }
}
```

Every branch here is a test case in guide 06: no exam, failed exam, fewer than five results, exactly
five, more than seven.
</details>

<details>
<summary>E4.3 — the drawback</summary>

Two acceptable answers:

1. **Traceability.** "Find usages" on `AuditEntry` no longer shows who triggers it. The flow lives in
   annotations rather than in call sites, which is harder for a newcomer to follow.
2. **Transaction semantics.** A plain `@EventListener` runs synchronously inside the publisher's
   transaction, so a failing listener rolls back the grade. If that is not what you want, you need
   `@TransactionalEventListener(phase = AFTER_COMMIT)` — and then the audit row can be lost if the
   listener fails after the commit. Decoupling moved the problem, it did not delete it.
</details>

<details>
<summary>E4.6 — the stateful singleton</summary>

```java
@Service
public class BadCounterService {
    private int gradesCalculated = 0;                  // ❌ shared mutable state

    public int record() { return ++gradesCalculated; } // not atomic
}
```

```java
@Test
void counter_loses_increments_under_concurrency() throws Exception {
    BadCounterService service = new BadCounterService();
    ExecutorService pool = Executors.newFixedThreadPool(8);
    CountDownLatch done = new CountDownLatch(1000);

    for (int i = 0; i < 1000; i++) {
        pool.submit(() -> { service.record(); done.countDown(); });
    }
    done.await(5, SECONDS);

    assertThat(service.currentCount()).isLessThan(1000);   // increments were lost
}
```

`++` is read-modify-write, three operations, none atomic. One bean instance serves every request, so
eight threads interleave inside it.

The fix is to hold no state (`@Service` beans should be stateless), or — if a counter genuinely belongs
here — use `AtomicInteger`, or better, a metrics counter from Micrometer.
</details>

---

## Checklist — ÕV1 achieved

- [ ] Strategy selected at runtime from data, not from an `if`/`switch` chain
- [ ] All beans use constructor injection; no `@Autowired` fields anywhere
- [ ] `PATTERNS.md` maps five patterns to real file locations
- [ ] You can answer, unprompted: "what breaks if I make this bean stateful?"
- [ ] At least one named anti-pattern was found and refactored away

**Exceeds:** a pattern used that was not taught here, justified in writing, plus a demonstrated
concurrency problem and its fix.

