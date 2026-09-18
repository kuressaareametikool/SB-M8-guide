# 02 — REST and MVC Architecture

**Outcome ÕV3** — *realiseerib rakenduse MVC arhitektuuriga rakendusena*
Prerequisites: guide 01

---

## Why this matters

The UI is built in a separate module, so this project is **backend only**: a REST API with no templates.
That does not remove the outcome — it sharpens it. The controller and the model still live here; the
view is rendered by a different team, over HTTP, in JSON. What you must prove is the **separation**, and
a consumer you cannot see is a stricter test of it than a template you control yourself.

Modern Spring applications are **layered MVC**: the textbook "Model" splits into entity, service and
DTO, and the "View" becomes the JSON representation your API returns.

---

## The layers

| Layer | Annotation | Its one job | Rule |
|---|---|---|---|
| Controller | `@RestController` | Translate HTTP to a method call and a response | No business logic, no repository calls |
| Service | `@Service` | Business rules, transaction boundary | Knows nothing about HTTP |
| Repository | Spring Data interface | Persistence | Returns entities, not responses |
| Entity | `@Entity` | Domain state and invariants | Never serialised to JSON |
| Request DTO | `record` | Validated input for one use case | One per use case |
| Response DTO | `record` | The "view" — exactly what the client needs | Shaped by the consumer, not by the schema |

Dependencies point one way only:

```
Controller ──> Service ──> Repository ──> Database
```

If a repository imports anything from the controller package, the architecture has broken.

---

## A controller that behaves

```java
@RestController
@RequestMapping("/api/courses")
public class CourseController {

    private final CourseService courseService;

    CourseController(CourseService courseService) {
        this.courseService = courseService;
    }

    @GetMapping
    public Page<CourseResponse> list(Pageable pageable) {
        return courseService.findAll(pageable);
    }

    @GetMapping("/{id}")
    public CourseResponse get(@PathVariable Long id) {
        return courseService.findById(id);          // throws CourseNotFoundException
    }

    @PostMapping
    public ResponseEntity<CourseResponse> create(@Valid @RequestBody CreateCourseRequest request) {
        CourseResponse created = courseService.create(request);
        return ResponseEntity
                .created(URI.create("/api/courses/" + created.id()))
                .body(created);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        courseService.delete(id);
    }
}
```

Three things to notice: **constructor injection**, methods that are three lines because everything hard
lives in the service, and no `try`/`catch` anywhere — errors are handled centrally.

### The DTOs

```java
public record CreateCourseRequest(
        @NotBlank @Size(max = 120) String name,
        @NotBlank String gradingKey,
        @NotNull @Positive Integer credits) {}

public record CourseResponse(
        Long id,
        String name,
        String gradingKey,
        int credits,
        int enrolledCount) {}
```

`enrolledCount` is a good example of why a response DTO is not a mirror of the entity: the client needs
a number the entity does not store as a field.

### The service

```java
@Service
@Transactional
public class CourseService {

    private final CourseRepository repository;

    CourseService(CourseRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public CourseResponse findById(Long id) {
        return repository.findById(id)
                .map(CourseMapper::toResponse)
                .orElseThrow(() -> new CourseNotFoundException(id));
    }

    public CourseResponse create(CreateCourseRequest request) {
        if (repository.existsByNameIgnoreCase(request.name())) {
            throw new DuplicateCourseException(request.name());
        }
        Course course = new Course(request.name(), request.gradingKey(), request.credits());
        return CourseMapper.toResponse(repository.save(course));
    }
}
```

---

## REST conventions to agree on

| Situation | Status | Body |
|---|---|---|
| Resource fetched | 200 | the resource |
| Resource created | 201 + `Location` header | the created resource |
| Deleted | 204 | empty |
| Validation failed | 400 | problem detail listing the fields |
| Not found | 404 | problem detail |
| Conflict (already enrolled) | 409 | problem detail |
| Not authorised | 401 / 403 | problem detail |

Nouns in paths, plural, no verbs:

```
POST /api/courses/5/enrolments     ✅
POST /api/createEnrolment          ❌
GET  /api/courses?page=0&size=20   ✅
GET  /api/getAllCourses            ❌
```

Versioning (`/api/v1/...` or none) is a decision — record it in an ADR (guide 09).

---

## Why entities must not be serialised

Returning a JPA entity as JSON means:

1. **Lazy-loading exceptions** outside the transaction
2. **Accidental exposure** of fields like `passwordHash`
3. **Infinite recursion** on bidirectional relations
4. **A client contract that silently changes** whenever the schema does

The fourth is the serious one here, because the UI module depends on your JSON being stable. Map to a
response record in the service.

```java
final class CourseMapper {
    private CourseMapper() {}

    static CourseResponse toResponse(Course course) {
        return new CourseResponse(
                course.getId(),
                course.getName(),
                course.getGradingKey(),
                course.getCredits(),
                course.getEnrolments().size());
    }
}
```

---

## Errors, centrally

Spring Framework 7 has RFC 9457 problem details built in. Use them rather than inventing an error shape.

```java
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(CourseNotFoundException.class)
    ProblemDetail notFound(CourseNotFoundException e) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
        problem.setTitle("Course not found");
        return problem;
    }

    @ExceptionHandler(DuplicateCourseException.class)
    ProblemDetail conflict(DuplicateCourseException e) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, e.getMessage());
        problem.setTitle("Course already exists");
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail invalid(MethodArgumentNotValidException e) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation failed");
        problem.setProperty("errors", e.getBindingResult().getFieldErrors().stream()
                .collect(toMap(FieldError::getField, FieldError::getDefaultMessage, (a, b) -> a)));
        return problem;
    }
}
```

A client receives:

```json
{
  "type": "about:blank",
  "title": "Validation failed",
  "status": 400,
  "errors": { "name": "must not be blank" }
}
```

One handler beats a `try`/`catch` in every controller method, and it guarantees the UI module gets one
consistent error format.

---

## Working with the UI module

The contract is a deliverable in its own right:

- **OpenAPI is the interface document.** Generate it from the code with springdoc and commit the
  generated `openapi.json`, so changes show up in diffs.
- **Agree the contract before implementing.** A short API sketch reviewed by the UI side in week 2
  prevents a painful week 8.
- **A breaking change is a conversation, not a commit.** Renaming a JSON field breaks clients; adding an
  optional one does not.
- **CORS** is configured deliberately and documented, never pasted as `allowedOrigins("*")`.

```java
@Configuration
class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:5173")   // the UI module in development
                .allowedMethods("GET", "POST", "PUT", "DELETE");
    }
}
```

---

## Exercises

### E2.1 — CRUD (basic)
Full CRUD for `Course` through the whole stack: request record, response record, mapper, service,
controller. Correct status codes for all five operations.

### E2.2 — Error handling (basic)
Add `@RestControllerAdvice` covering not-found, validation and conflict. Verify each with `curl` and
paste the responses into your README.

### E2.3 — Layering test (intermediate)
Prove no controller talks to a repository, with ArchUnit:

```java
@AnalyzeClasses(packages = "ee.tak24.kursusepunkt")
class ArchitectureTest {
    @ArchTest
    static final ArchRule controllers_do_not_touch_repositories =
            noClasses().that().resideInAPackage("..controller..")
                    .should().dependOnClassesThat().resideInAPackage("..repository..");
}
```

Adapt the packages to your feature-first layout. Use ArchUnit **1.5.0 or newer**: older releases cannot
read Java 26 class files, so they import zero classes and every rule fails with *"failed to check any
classes"* — which looks like a broken rule rather than a broken parser.

### E2.4 — Pagination (intermediate)
Return `Page<CourseResponse>` from the list endpoint and document the query parameters.

### E2.5 — Contract first (intermediate)
Write the API contract for `Enrolment` **before** writing the code. Afterwards, note in
`docs/adr/` what you had to change and why.

### E2.6 — Nested resources (advanced)
Design and implement `POST /api/courses/{id}/enrolments`. Justify in two sentences why it is nested
rather than a top-level `/api/enrolments` with a `courseId` field.

---

## Solutions

<details>
<summary>E2.3 — making the ArchUnit rule work with feature-first packages</summary>

Feature-first packaging means there is no `..controller..` package. Match on class names instead:

```java
@ArchTest
static final ArchRule controllers_do_not_touch_repositories =
        noClasses().that().haveSimpleNameEndingWith("Controller")
                .should().dependOnClassesThat().haveSimpleNameEndingWith("Repository");

@ArchTest
static final ArchRule entities_do_not_leak_to_controllers =
        noClasses().that().haveSimpleNameEndingWith("Controller")
                .should().dependOnClassesThat().areAnnotatedWith(Entity.class);
```

The second rule is the more valuable one — it mechanically enforces the "no entity in JSON" rule.
</details>

<details>
<summary>E2.6 — nested vs top-level</summary>

Nested is right when the child cannot exist without the parent and is always addressed through it. An
enrolment is meaningless without a course, and the client always has the course id in hand when
creating one, so `/api/courses/{id}/enrolments` reads as the action it is and makes the required
relationship impossible to omit.

A top-level `/api/enrolments` would be the better choice if enrolments were queried across courses as a
first-class list — which is why many APIs offer **both**: nested for creation, top-level with filters
for search.
</details>

<details>
<summary>E2.2 — verifying with curl</summary>

```bash
curl -i http://localhost:8080/api/courses/9999
# HTTP/1.1 404
# {"type":"about:blank","title":"Course not found","status":404,"detail":"Course 9999 not found"}

curl -i -X POST http://localhost:8080/api/courses \
  -H 'Content-Type: application/json' \
  -d '{"name":"","gradingKey":"WEIGHTED","credits":5}'
# HTTP/1.1 400
# {"title":"Validation failed","status":400,"errors":{"name":"must not be blank"}}
```

If you get a stack trace instead of a problem detail, your advice is not being picked up — check that
it is in a scanned package.
</details>

---

## Checklist — ÕV3 achieved

- [ ] No controller method longer than ~15 lines
- [ ] No entity appears in a JSON response
- [ ] Status codes and error bodies consistent across every endpoint
- [ ] Validation on every request DTO
- [ ] Pagination on every list that can grow unbounded
- [ ] You can explain what the "view" is in a headless application

**Exceeds:** layering enforced by an ArchUnit test, a documented request-flow diagram, and a contract
agreed with the UI module before implementation.

