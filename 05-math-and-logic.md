# 05 — Math and Logic in Applications

**Outcome ÕV2** — *kasutab rakenduste koostamisel matemaatika- ja loogikafunktsioone*
Prerequisites: guides 01–04

---

## Why this matters

The trap is treating this outcome as "call `Math.round()` somewhere". The real content is:

1. Choosing correct numeric types
2. Controlling rounding **deliberately**
3. Expressing business rules as testable boolean logic instead of nested `if`s

Everything in this guide is pure functions with no database and no HTTP, which makes it the easiest
material in the course to unit test — that is why it sits immediately before guide 06.

---

## Numbers: pick the right type

| Need | Type | Why |
|---|---|---|
| Money, grades, percentages | `BigDecimal` | Exact decimal arithmetic, explicit rounding |
| Counts, ids | `long` / `int` | Fast, exact |
| Scientific / statistical values | `double` | Fast, approximate — never for money or grades |

### The demonstration

```java
System.out.println(0.1 + 0.2);                  // 0.30000000000000004
System.out.println(0.1 + 0.2 == 0.3);           // false
System.out.println(new BigDecimal("0.1").add(new BigDecimal("0.2")));   // 0.3
```

Binary floating point cannot represent 0.1 exactly, the same way decimal cannot represent ⅓ exactly.

### Two `BigDecimal` traps

```java
new BigDecimal(0.1)        // ❌ 0.1000000000000000055511151231257827…  (double constructor)
new BigDecimal("0.1")      // ✅ exactly 0.1                            (String constructor)

new BigDecimal("2.0").equals(new BigDecimal("2.00"))            // false — scale differs!
new BigDecimal("2.0").compareTo(new BigDecimal("2.00")) == 0    // true  — use this
```

`equals` compares value **and scale**; `compareTo` compares value. In tests, AssertJ's
`isEqualByComparingTo` is the right assertion.

---

## Rounding is a decision, not a default

```java
/**
 * Weighted average, rounded to one decimal, HALF_UP as the grading regulation requires.
 */
public BigDecimal weightedAverage(List<Result> results) {
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
```

Every `divide` on `BigDecimal` **requires** a scale and a `RoundingMode`, otherwise a non-terminating
result throws `ArithmeticException`. The decision is forced on you deliberately.

| Mode | 2.5 → | 3.5 → | Used for |
|---|---|---|---|
| `HALF_UP` | 3 | 4 | Grades, most human-facing rounding |
| `HALF_EVEN` | 2 | 4 | Statistics, banking — avoids upward bias |
| `FLOOR` | 2 | 3 | Never round a grade this way without saying so |

Rounding **once**, at the end, is the rule. Rounding intermediate values accumulates error — keep
working scale high (4 decimals) and round only the returned value.

---

## Logic: rules as objects, not nested ifs

```java
// Before
if (enrolment.getFinalGrade() != null) {
    if (enrolment.getFinalGrade().compareTo(PASS_MARK) >= 0) {
        if (enrolment.getAttendanceRatio() >= 0.75) {
            if (!enrolment.hasOutstandingFees()) {
                issueCertificate(enrolment);
            }
        }
    }
}

// After
Predicate<Enrolment> graded    = e -> e.getFinalGrade() != null;
Predicate<Enrolment> passed    = e -> e.getFinalGrade().compareTo(PASS_MARK) >= 0;
Predicate<Enrolment> attended  = e -> e.getAttendanceRatio() >= 0.75;
Predicate<Enrolment> paid      = e -> !e.hasOutstandingFees();

Predicate<Enrolment> eligibleForCertificate =
        graded.and(passed).and(attended).and(paid);

if (eligibleForCertificate.test(enrolment)) {
    issueCertificate(enrolment);
}
```

Each predicate has a name, can be tested alone, and can be reused in a filter:

```java
List<Enrolment> eligible = enrolments.stream().filter(eligibleForCertificate).toList();
```

### Boolean logic worth teaching explicitly

- **Short-circuit:** `&&` stops at the first false, `&` does not. `graded && passed` is safe;
  `graded & passed` throws an NPE when the grade is null. This is why order matters.
- **De Morgan's laws:** `!(a && b)` ≡ `!a || !b`, `!(a || b)` ≡ `!a && !b`. Use them to simplify a
  negated condition instead of wrapping it.
- **Truth tables → decision tables.** Turning an argument about requirements into a table you can test
  row by row is the practical payoff of the theory.

| Graded | Passed | Attended | Paid | Certificate |
|---|---|---|---|---|
| no | – | – | – | no |
| yes | no | – | – | no |
| yes | yes | no | – | no |
| yes | yes | yes | no | no |
| yes | yes | yes | yes | **yes** |

---

## Declarative validation

Bean Validation moves the simple rules out of your code:

```java
public record RecordResultRequest(
        @NotNull @DecimalMin("0.0") @DecimalMax("100.0") BigDecimal points,
        @NotNull @DecimalMin("0.0") @DecimalMax("1.0")   BigDecimal weight,
        @NotNull @PastOrPresent LocalDate submittedAt) {}
```

Cross-field rules need a custom constraint:

```java
@Target(TYPE) @Retention(RUNTIME)
@Constraint(validatedBy = ResitAfterOriginalValidator.class)
public @interface ResitAfterOriginal {
    String message() default "Resit date must be after the original submission";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

class ResitAfterOriginalValidator implements ConstraintValidator<ResitAfterOriginal, ResitRequest> {
    @Override
    public boolean isValid(ResitRequest r, ConstraintValidatorContext ctx) {
        if (r.originalDate() == null || r.resitDate() == null) return true;  // @NotNull's job
        return r.resitDate().isAfter(r.originalDate());
    }
}
```

**Validation lives in two places, deliberately:** the request DTO rejects malformed input at the edge,
and the domain protects its own invariants regardless of who calls it. The second is not redundant — a
scheduled job or an import does not pass through a controller.

---

## Dates are maths too

`java.time` only — never `Date` or `Calendar`.

| Type | Use for |
|---|---|
| `LocalDate` | Calendar days: deadlines, birthdays |
| `LocalDateTime` | A wall-clock time with no zone — rarely what you want |
| `Instant` | Timestamps; store UTC |
| `Duration` / `Period` | Elapsed time / calendar amounts |

```java
/**
 * Late penalty: 2% per started day, capped at 20%.
 * Returns the percentage to subtract, never negative.
 */
public BigDecimal latePenaltyPercent(LocalDate due, LocalDate submitted) {
    long daysLate = ChronoUnit.DAYS.between(due, submitted);
    if (daysLate <= 0) {
        return BigDecimal.ZERO;              // on time or early
    }
    BigDecimal penalty = BigDecimal.valueOf(daysLate * 2L);
    return penalty.min(new BigDecimal("20"));
}
```

Never call `LocalDate.now()` inside domain code — inject a `Clock` instead. Guide 07 explains why, and
it is the single most useful testability trick in this course.

---

## Exercises

### E5.1 — Weighted grade (basic)
Implement the weighted calculation with `BigDecimal`, scale 1, `HALF_UP`, and a guard for zero total
weight.

### E5.2 — Penalty (basic)
Implement `latePenaltyPercent` and write a decision table covering: on time, one minute late, one day,
exactly the cap, beyond the cap, and submitted before the assignment opened.

### E5.3 — Predicates (intermediate)
Refactor one nested `if` block of at least three levels into composed predicates. Keep both versions in
history and explain the improvement in the commit message.

### E5.4 — Statistics (intermediate)
Add one statistic to the report endpoint — median, standard deviation, or grade distribution —
implemented by you rather than by a library.

### E5.5 — Cross-field validation (intermediate)
Write a custom constraint for a two-field rule in your domain.

### E5.6 — Rounding investigation (advanced)
Compute a course average two ways: rounding each result to one decimal first, then averaging; versus
averaging at scale 4 and rounding once. Find an input where the answers differ and write up why.

---

## Solutions

<details>
<summary>E5.2 — the decision table as a parameterised test</summary>

```java
@ParameterizedTest(name = "{0} days late → {1}%")
@CsvSource({
    "-3,  0",    // submitted early
    " 0,  0",    // on time
    " 1,  2",
    " 9, 18",
    "10, 20",    // exactly the cap
    "30, 20"     // beyond the cap
})
void late_penalty_is_two_percent_per_day_capped_at_twenty(long daysLate, int expected) {
    LocalDate due = LocalDate.of(2026, 3, 1);
    LocalDate submitted = due.plusDays(daysLate);

    assertThat(penalty.latePenaltyPercent(due, submitted))
            .isEqualByComparingTo(BigDecimal.valueOf(expected));
}
```

The "one minute late" case is the interesting one to discuss: with `LocalDate` it is **not** late at
all, because the same calendar day yields zero days between. If the regulation means "after 23:59", the
types are right; if it means "after the 14:00 deadline", you need `LocalDateTime` and a different
function. The exercise is really about noticing the ambiguity in the requirement.
</details>

<details>
<summary>E5.4 — median without a library</summary>

```java
/**
 * Median of the given grades.
 *
 * @param grades non-empty list; not modified by this method
 * @return the middle value, or the mean of the two middle values when the count is even
 * @throws IllegalArgumentException if the list is empty
 */
public BigDecimal median(List<BigDecimal> grades) {
    if (grades.isEmpty()) {
        throw new IllegalArgumentException("Cannot take the median of no grades");
    }
    List<BigDecimal> sorted = grades.stream().sorted().toList();
    int n = sorted.size();
    int mid = n / 2;

    return (n % 2 == 1)
            ? sorted.get(mid).setScale(1, HALF_UP)
            : sorted.get(mid - 1).add(sorted.get(mid))
                    .divide(BigDecimal.valueOf(2), 1, HALF_UP);
}
```

Note `grades.stream().sorted()` rather than `Collections.sort(grades)` — the Javadoc promises not to
modify the caller's list, and mutating an argument is a bug waiting to be reported as "the report
reorders my data".
</details>

<details>
<summary>E5.6 — where the two rounding orders diverge</summary>

Results: 80.25, 80.25, 80.24.

- **Round first:** 80.3 + 80.3 + 80.2 = 240.8 → average **80.27 → 80.3**
- **Round once:** (80.25 + 80.25 + 80.24) / 3 = 80.2466… → **80.2**

A tenth of a grade point, which is the difference between a 4 and a 5 on some scales. The rule —
*keep working scale high and round only the value you return* — exists because rounding is not
distributive over division.
</details>

---

## Checklist — ÕV2 achieved

- [ ] No `double` anywhere in grade or money paths
- [ ] Every `divide` states a scale and a rounding mode
- [ ] `BigDecimal` compared with `compareTo`, never `equals`
- [ ] Edge cases (zero, empty list, boundary values) handled, not assumed away
- [ ] At least one nested conditional refactored into named predicates
- [ ] `java.time` only; no `Date`, no `Calendar`

**Exceeds:** an own statistical function, a documented decision table with full branch coverage, and a
written justification for the rounding mode chosen.

