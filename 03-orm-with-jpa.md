# 03 — ORM with JPA and Hibernate

**Outcome ÕV4** — *kasutab parimate praktikate kohaselt ORM vahendeid*
Prerequisites: guides 01–02

---

## Why this matters

JPA/Hibernate 7 is stricter than Eloquent: there is a persistence context, entities have lifecycle
states, and laziness is explicit. Eloquent's "just call it and it loads" habit produces N+1 queries here.

The centrepiece of this guide is not annotations — it is **seeing the queries your code actually runs**.

---

## Entity basics

```java
@Entity
@Table(name = "enrolment",
       uniqueConstraints = @UniqueConstraint(columnNames = {"student_id", "course_id"}))
public class Enrolment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "course_id")
    private Course course;

    @OneToMany(mappedBy = "enrolment", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Result> results = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private EnrolmentStatus status = EnrolmentStatus.PENDING;

    @Version
    private long version;                 // optimistic locking

    protected Enrolment() { }             // required by JPA, not for your code

    public Enrolment(Student student, Course course) {
        this.student = student;
        this.course = course;
    }

    public void addResult(Result result) {   // keeps both sides in sync
        results.add(result);
        result.setEnrolment(this);
    }
}
```

### Rules stated as absolutes

| Rule | Why |
|---|---|
| Always `fetch = LAZY` on `@ManyToOne` | The JPA default is EAGER and it is wrong; it drags half the database into memory |
| `@Enumerated(EnumType.STRING)`, never ORDINAL | Reordering an enum must not silently rewrite your data |
| Owning side is the one with the foreign key | The other side uses `mappedBy`; keep both in sync with helper methods |
| Never `CascadeType.ALL` on `@ManyToOne` | Deleting an enrolment must not delete the student |
| `equals`/`hashCode` on a business key or assigned UUID | Generated ids are null before insert, which breaks `HashSet` membership |
| No `@Data` from Lombok on entities | It generates `equals`, `hashCode` and `toString` over every field, including lazy relations |

---

## The N+1 problem

Turn on SQL logging first, so the problem is visible rather than theoretical:

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate.format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE     # shows bound parameters
```

`open-in-view` is `true` by default and **hides** lazy-loading mistakes by keeping the session open
during response rendering. Turning it off makes wrong code fail loudly in development instead of
quietly in production.

### Seeing it happen

```java
// 1 query for the enrolments…
List<Enrolment> enrolments = repository.findByCourseId(courseId);

// …then 1 query per enrolment when the name is touched. 200 students = 201 queries.
enrolments.forEach(e -> System.out.println(e.getStudent().getName()));
```

### Fixes, in order of preference

```java
// 1. Fetch join — one query, related data included
@Query("select e from Enrolment e join fetch e.student where e.course.id = :courseId")
List<Enrolment> findByCourseWithStudents(@Param("courseId") Long courseId);

// 2. Entity graph — declarative, reusable
@EntityGraph(attributePaths = {"student", "results"})
List<Enrolment> findByCourseId(Long courseId);

// 3. Projection — fetch only the columns the screen shows
public interface EnrolmentRow {
    String getStudentName();
    BigDecimal getFinalGrade();
}

@Query("""
       select s.name as studentName, e.finalGrade as finalGrade
       from Enrolment e join e.student s
       where e.course.id = :courseId
       """)
List<EnrolmentRow> findRowsByCourse(@Param("courseId") Long courseId);
```

Projections are underrated: a list endpoint rarely needs entities at all.

> **Careful:** `join fetch` on a `@OneToMany` plus pagination makes Hibernate paginate **in memory**
> (it warns about this). For paginated collections, use two queries — ids first, then the data.

---

## Transactions

```java
@Service
@Transactional                                 // write methods
public class EnrolmentService {

    @Transactional(readOnly = true)            // read methods
    public EnrolmentResponse findById(Long id) { ... }
}
```

- `@Transactional` on services, **never** on controllers.
- Reads get `readOnly = true` — Hibernate skips dirty checking, which is both faster and safer.
- **The proxy caveat:** calling a `@Transactional` method from another method *of the same class*
  bypasses the proxy entirely, so nothing is transactional.

```java
@Service
public class BadService {
    public void outer() {
        inner();          // ❌ no transaction — the proxy is not involved
    }

    @Transactional
    public void inner() { ... }
}
```

---

## Migrations

`ddl-auto` must be `validate` in every environment, with schema changes written by hand in Flyway.

```
src/main/resources/db/migration/
  V1__create_course_and_student.sql
  V2__create_enrolment.sql
  V3__add_grading_key_to_course.sql
```

```sql
-- V2__create_enrolment.sql
CREATE TABLE enrolment (
    id          BIGSERIAL PRIMARY KEY,
    student_id  BIGINT      NOT NULL REFERENCES student(id),
    course_id   BIGINT      NOT NULL REFERENCES course(id),
    status      VARCHAR(20) NOT NULL,
    final_grade NUMERIC(4,1),
    version     BIGINT      NOT NULL DEFAULT 0,
    CONSTRAINT uq_enrolment_student_course UNIQUE (student_id, course_id)
);

CREATE INDEX idx_enrolment_course ON enrolment(course_id);
```

`ddl-auto=update` is the single most common beginner mistake. It works right up until it silently drops
nothing, adds nothing, and leaves the running database diverged from the code with no record of how it
got there. Migrations are the professional answer, and `validate` is the seatbelt that catches a
mismatch at startup.

---

## Optimistic locking

Two users editing the same enrolment at once is a real scenario, not a hypothetical.

```java
@Version
private long version;
```

Hibernate adds `WHERE version = ?` to every update and increments it. The second writer gets
`OptimisticLockingFailureException` — which your `@RestControllerAdvice` turns into a 409 with a useful
message rather than a silent overwrite.

---

## Exercises

### E3.1 — Model (basic)
Model at least one `@OneToMany`, one `@ManyToOne` and one `@ManyToMany` — or a join entity with extra
columns. Explain in a comment which you chose and why.

### E3.2 — Migrations (basic)
Write the full Flyway history. The application must start against an **empty** database with
`ddl-auto=validate` and no manual SQL.

### E3.3 — N+1 (intermediate) ⭐ *the core exercise*
Deliberately create an N+1 situation. Capture the query log as proof. Fix it with a fetch join. Capture
the log again. Both logs go in `docs/performance.md` with the query counts stated.

### E3.4 — Query styles (intermediate)
Add one derived query, one `@Query` with JPQL, and one projection interface. Comment on why each was the
right tool in that place.

### E3.5 — Pagination (intermediate)
Paginate the largest list endpoint with `Pageable`. Then try adding `join fetch` on a collection to the
same query and record what Hibernate warns.

### E3.6 — Locking (advanced)
Add `@Version` and write a test that simulates two concurrent updates, asserting the second fails.

---

## Solutions

<details>
<summary>E3.3 — evidence that satisfies the exercise</summary>

```markdown
## Before — GET /api/courses/1/enrolments
201 queries for 200 enrolments:

select e1_0.id, e1_0.course_id, e1_0.student_id ... from enrolment e1_0 where e1_0.course_id=?
select s1_0.id, s1_0.email, s1_0.name from student s1_0 where s1_0.id=?
select s1_0.id, s1_0.email, s1_0.name from student s1_0 where s1_0.id=?
... (198 more)

Page load: 340 ms

## After — @EntityGraph(attributePaths = "student")
1 query:

select e1_0.id, ..., s1_0.id, s1_0.name, s1_0.email
from enrolment e1_0 join student s1_0 on s1_0.id=e1_0.student_id
where e1_0.course_id=?

Page load: 12 ms
```

The query counts are the evidence. State them explicitly.
</details>

<details>
<summary>E3.5 — what Hibernate warns, and the fix</summary>

```
HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

Hibernate cannot paginate correctly in SQL when a `join fetch` on a collection multiplies rows, so it
fetches **everything** and paginates in memory. On a large table this is a production incident.

The standard fix is two queries:

```java
@Query("select e.id from Enrolment e where e.course.id = :courseId")
Page<Long> findIdsByCourse(@Param("courseId") Long courseId, Pageable pageable);

@Query("select distinct e from Enrolment e join fetch e.results where e.id in :ids")
List<Enrolment> findWithResultsByIds(@Param("ids") List<Long> ids);
```

Page the ids (no collection join, so SQL pagination works), then fetch the page's data in one query.
</details>

<details>
<summary>E3.6 — a concurrency test</summary>

`@DataJpaTest` replaces your datasource with an embedded one and does **not** auto-configure Flyway, so
on this stack it needs the three extra annotations below — otherwise the slice starts against an empty
schema and `ddl-auto=validate` fails. Guide 11 explains each one.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@Import(ContainerConfig.class)
@ImportAutoConfiguration(FlywayAutoConfiguration.class)
class EnrolmentLockingTest {

    @Autowired private EnrolmentRepository repository;
    @Autowired private TestEntityManager em;

    @Test
    void second_writer_loses_when_both_edit_the_same_enrolment() {
        Enrolment saved = repository.save(anEnrolment());
        em.flush();
        em.clear();

        Enrolment first  = repository.findById(saved.getId()).orElseThrow();
        Enrolment second = repository.findById(saved.getId()).orElseThrow();
        em.detach(second);                       // second reader holds the old version

        first.setStatus(CONFIRMED);
        repository.saveAndFlush(first);          // version 0 → 1

        second.setStatus(CANCELLED);
        assertThatThrownBy(() -> repository.saveAndFlush(second))
                .isInstanceOf(ObjectOptimisticLockingFailureException.class);
    }
}
```

Discussion point: optimistic locking does not prevent the conflict, it **detects** it. The API's job is
then to tell the user something useful — which is why this maps to a 409, not a 500.
</details>

---

## Checklist — ÕV4 achieved

- [ ] No EAGER associations without a written justification
- [ ] `open-in-view=false` and the application still works
- [ ] Query count for the main list endpoint is constant, not proportional to row count
- [ ] Schema reproducible from migrations alone, `ddl-auto=validate`
- [ ] Unique constraints and indexes exist in the migration, not only in annotations
- [ ] Transactions on services with `readOnly` on reads

**Exceeds:** projections, entity graphs, pagination done correctly with collections, and `@Version`
with a test proving the conflict is detected.

