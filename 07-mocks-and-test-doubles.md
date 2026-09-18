# 07 — Mocks and Test Doubles

**Outcome ÕV6** — *kasutab testides mock-klasse*
Prerequisites: guides 01–06

---

## Why this matters

Mocks exist so a unit test can stay a unit test. If `GradingService` needs a repository, a notification
sender and a clock, you do not want a database, an SMTP server and a real calendar in the test.

This outcome is where dependency injection (guide 04) pays off: a class that takes its collaborators
through the constructor can be tested with `new` and two fakes. A class that reaches out for them
cannot.

---

## The vocabulary

The words are part of the outcome; they are not interchangeable.

| Double | What it does | Example |
|---|---|---|
| **Dummy** | Passed but never used | A placeholder argument to satisfy a signature |
| **Stub** | Returns canned answers | `when(repo.findById(1L)).thenReturn(...)` |
| **Mock** | Stub **plus** you assert on the interaction | `verify(notifications).send(any())` |
| **Spy** | Real object with some methods overridden | `spy(new PriceCalculator())` |
| **Fake** | Working lightweight implementation | In-memory repository, H2 database |

The distinction that matters in practice: a **stub** answers questions, a **mock** records what you
asked. If your test never calls `verify`, you are using stubs, and saying "mock" is sloppy rather than
wrong.

---

## Mockito in a plain unit test

```java
@ExtendWith(MockitoExtension.class)
class EnrolmentServiceTest {

    @Mock  private EnrolmentRepository repository;
    @Mock  private NotificationSender notifications;
    @InjectMocks private EnrolmentService service;

    @Test
    void notifies_student_when_enrolment_is_confirmed() {
        Enrolment enrolment = anEnrolment();
        when(repository.findById(1L)).thenReturn(Optional.of(enrolment));
        when(repository.save(any(Enrolment.class))).thenAnswer(i -> i.getArgument(0));

        service.confirm(1L);

        ArgumentCaptor<Notification> sent = ArgumentCaptor.forClass(Notification.class);
        verify(notifications).send(sent.capture());

        assertThat(sent.getValue().recipient()).isEqualTo(enrolment.getStudent().getEmail());
        assertThat(enrolment.getStatus()).isEqualTo(CONFIRMED);
    }

    @Test
    void does_not_notify_when_the_course_is_full() {
        when(repository.findById(2L)).thenReturn(Optional.of(fullCourseEnrolment()));

        assertThatThrownBy(() -> service.confirm(2L))
                .isInstanceOf(CourseFullException.class);

        verifyNoInteractions(notifications);
    }
}
```

`verifyNoInteractions` is the underused half of mocking: **proving something did not happen** is often
the more valuable assertion. "No email is sent when enrolment fails" is a real requirement, and it is
invisible to an assertion on the return value.

### The Mockito API you need

```java
when(mock.method()).thenReturn(value);              // stub
when(mock.method()).thenThrow(new Exception());     // stub a failure
when(mock.save(any())).thenAnswer(i -> i.getArgument(0));   // echo the argument back

verify(mock).method();                              // called exactly once
verify(mock, times(2)).method();
verify(mock, never()).method();
verifyNoInteractions(mock);
verifyNoMoreInteractions(mock);                     // use sparingly — brittle

ArgumentCaptor<T> captor = ArgumentCaptor.forClass(T.class);
verify(mock).method(captor.capture());
assertThat(captor.getValue())...
```

**Matchers are all-or-nothing:** if one argument uses a matcher, they all must. `verify(mock).send("a",
any())` fails at runtime; use `verify(mock).send(eq("a"), any())`.

---

## `@MockitoBean` for slice tests

In a `@WebMvcTest` the controller is real and everything below it is mocked, so you test routing,
validation and serialisation without a database:

```java
@WebMvcTest(CourseController.class)
class CourseControllerTest {

    @Autowired    private MockMvcTester mvc;          // fluent, AssertJ-based
    @MockitoBean  private CourseService courseService;

    @Test
    void rejects_a_course_with_a_blank_name() {
        assertThat(mvc.post().uri("/api/courses")
                        .contentType(APPLICATION_JSON)
                        .content("""
                                {"name": "", "gradingKey": "WEIGHTED", "credits": 5}
                                """))
                .hasStatus(HttpStatus.BAD_REQUEST)
                .bodyJson().extractingPath("$.errors.name").isNotNull();

        verifyNoInteractions(courseService);
    }

    @Test
    void returns_404_as_a_problem_detail() {
        when(courseService.findById(9999L)).thenThrow(new CourseNotFoundException(9999L));

        assertThat(mvc.get().uri("/api/courses/9999"))
                .hasStatus(HttpStatus.NOT_FOUND)
                .bodyJson().extractingPath("$.title").isEqualTo("Course not found");
    }
}
```

> **Version note.** `@MockBean` was deprecated in Boot 3.4 and **removed in Boot 4** — use
> `@MockitoBean`, and `@MockitoSpyBean` in place of `@SpyBean`. Any tutorial still using the old names
> predates your stack.

`MockMvcTester` is the modern entry point; the older `MockMvc` + `perform(...)` style still works and
more examples of it exist online, but do not mix the two in one class.

---

## Making time and randomness testable

The most practically useful trick in this outcome. Never call `LocalDate.now()` inside domain code:

```java
// ❌ untestable
public boolean isLate(Assignment assignment) {
    return LocalDate.now().isAfter(assignment.getDueDate());
}

// ✅ testable
@Bean
Clock clock() { return Clock.systemDefaultZone(); }

@Service
public class SubmissionService {
    private final Clock clock;

    SubmissionService(Clock clock) { this.clock = clock; }

    public boolean isLate(Assignment assignment) {
        return LocalDate.now(clock).isAfter(assignment.getDueDate());
    }
}
```

```java
@Test
void submission_three_days_after_the_deadline_is_late() {
    Clock fixed = Clock.fixed(Instant.parse("2026-03-04T10:00:00Z"), ZoneOffset.UTC);
    SubmissionService service = new SubmissionService(fixed);

    assertThat(service.isLate(assignmentDue(LocalDate.of(2026, 3, 1)))).isTrue();
}
```

"Submitted three days late" becomes a deterministic test instead of a flaky one that breaks at midnight
or in another time zone. The same argument applies to random number generators: inject the source.

---

## When not to mock

- **Don't mock value objects.** Build a real `BigDecimal`, a real `Result`. Mocking them adds setup and
  removes the real behaviour you depend on.
- **Don't mock types you don't own.** Wrap a third-party client in your own interface (adapter pattern)
  and mock that. Mocking a vendor's class encodes assumptions about code you cannot see.
- **Don't mock the repository in a `@DataJpaTest`.** The point there is the real query.
- **Beware over-mocking.** Six mocks and ten `when` lines means the test is asserting your wiring, not
  your behaviour — and that is a design smell in the class under test, not in the test.
- **Don't mock what you can fake cheaply.** An in-memory `Map`-backed implementation of your repository
  interface is often clearer than a pile of stubbing, and it survives refactoring better.

```java
class InMemoryEnrolmentRepository implements EnrolmentRepository {
    private final Map<Long, Enrolment> store = new HashMap<>();
    private final AtomicLong ids = new AtomicLong();

    @Override public Optional<Enrolment> findById(Long id) { return ofNullable(store.get(id)); }
    @Override public Enrolment save(Enrolment e) { ... }
}
```

---

## Exercises

### E7.1 — Service tests (basic)
Two service tests using `@Mock` and `@InjectMocks`: one asserting an interaction with `verify`, one
asserting an absence with `verifyNoInteractions`.

### E7.2 — Captor (basic)
One test using `ArgumentCaptor` to check **what** was passed to a collaborator, not just that it was
called.

### E7.3 — Web slice (intermediate)
One `@WebMvcTest` with `@MockitoBean`, covering a valid submission, an invalid one, and a not-found case.

### E7.4 — Clock (intermediate) ⭐
Refactor one class that calls `LocalDate.now()` to take an injected `Clock`, and write a test with
`Clock.fixed`. Register the bean so production still works.

### E7.5 — Fake vs mock (intermediate)
Write one test with a hand-written fake instead of Mockito. In `docs/testing.md`, describe where you
chose a fake over a mock and why.

### E7.6 — Over-mocking (advanced)
Find the test in your suite with the most stubbing. Either justify it in writing or refactor the class
under test so that fewer collaborators are needed — and say which you did.

---

## Solutions

<details>
<summary>E7.4 — wiring the Clock without breaking production</summary>

```java
@Configuration
class TimeConfig {
    @Bean
    @ConditionalOnMissingBean          // a test can supply its own
    Clock clock() {
        return Clock.systemDefaultZone();
    }
}
```

In an integration test:

```java
@TestConfiguration
static class FixedClockConfig {
    @Bean Clock clock() {
        return Clock.fixed(Instant.parse("2026-03-04T10:00:00Z"), ZoneOffset.UTC);
    }
}
```

In a unit test you do not need Spring at all — `new SubmissionService(Clock.fixed(...))`.

A subtlety worth raising: `Clock.systemDefaultZone()` uses the server's zone, which differs between a
student's laptop and CI. For anything stored or compared, prefer `Clock.systemUTC()` and convert to a
local zone only at the edge.
</details>

<details>
<summary>E7.5 — where a fake beats a mock</summary>

> I used an in-memory fake for `EnrolmentRepository` in `GradingServiceTest`.
>
> The service calls `findById`, `save`, and `findByCourseId` in one flow, and the saved object is read
> back later in the same test. With Mockito that needs three `when` blocks plus a `thenAnswer` that
> echoes the argument, and the stubbing has to be kept in sync with the production code every time the
> flow changes. The fake stores objects in a `HashMap`, so read-after-write simply works and the test
> reads as a scenario rather than as a list of stubs.
>
> I kept Mockito for `NotificationSender`, because there the point is exactly the interaction — I want
> to assert that `send` was called once with the right recipient, which a fake would make harder.

That contrast — fake for state, mock for interaction — is the answer being looked for.
</details>

<details>
<summary>E7.6 — what over-mocking is telling you</summary>

A test like this:

```java
@Mock CourseRepository courses;
@Mock StudentRepository students;
@Mock EnrolmentRepository enrolments;
@Mock NotificationSender notifications;
@Mock AuditService audit;
@Mock FeeService fees;
@InjectMocks EnrolmentService service;   // six collaborators
```

is not a bad test. It is a bad **class**: `EnrolmentService` is doing enrolment, notification, auditing
and billing. The refactor is to split it — or to publish a domain event (guide 04) so that auditing and
notification become listeners, which removes two mocks from this test and puts them in two small tests
of their own.

Mocks are a design pressure sensor. When they hurt, listen to them.
</details>

---

## Checklist — ÕV6 achieved

- [ ] Correct vocabulary used in discussion and in comments (stub vs mock vs fake)
- [ ] At least one test asserts an interaction, and one asserts an absence of interaction
- [ ] `ArgumentCaptor` used where the argument's content matters
- [ ] `@MockitoBean` (not `@MockBean`) in slice tests
- [ ] No production code made uglier just to enable mocking (no public fields, no setters added for tests)
- [ ] Time injected as a `Clock`, not read from the system inside domain code

**Exceeds:** a hand-written fake with a written justification, and evidence that over-mocking was
noticed and acted on.

