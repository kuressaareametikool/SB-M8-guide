# 08 — Complex Algorithms and Components

**Outcome ÕV8** — *loob suurema keerukusastmega rakendusi, kasutades ka matemaatiliselt ja loogiliselt keerukamaid algoritme ja rakenduse osiseid*
Prerequisites: guides 01–07

---

## Why this matters

This is the outcome that separates "followed a CRUD tutorial" from "can build software". Pick **one
genuinely hard component** and own it end to end: design, implementation, complexity analysis, tests,
documentation.

---

## Candidate components — pick one

| Component | The hard part | Algorithmic content |
|---|---|---|
| **Study-path validator** | Prerequisites must be satisfiable | Directed graph, topological sort, cycle detection |
| Timetable generator | Courses to rooms and slots with no clashes | Constraint satisfaction, backtracking, greedy + local search |
| Exam scheduler | Minimise students with back-to-back exams | Graph colouring |
| Grade prediction | "What do I need on the final to pass?" | Inverse of the weighted formula, interval arithmetic |
| Plagiarism similarity | Compare submitted texts | Levenshtein distance, shingling, Jaccard similarity |
| Elective recommendation | Suggest courses from history | Cosine similarity, k-nearest neighbours |
| Room-capacity optimiser | Fit groups into rooms | Bin packing, greedy with a proven bound |

**The study-path validator is the best default:** the domain is real, the graph is small enough to draw
by hand, and cycle detection has a clean correct answer you can verify yourself.

---

## Worked example: the study-path validator

### The problem

Courses have prerequisites. Given the full set, produce an order in which a student can take them — or
report why no order exists.

```
Programming Basics ──> OOP ──> Spring Boot
                        │
                        └───> Databases ──> Spring Boot
```

Two things can go wrong: a **cycle** (A needs B, B needs A) and a **dangling prerequisite** (A needs a
course that does not exist).

### Kahn's algorithm

```java
/**
 * Produces a valid study order for the given prerequisite graph.
 *
 * <p>Uses Kahn's algorithm: repeatedly take a course whose prerequisites are all already taken.
 * Runs in O(V + E) time and O(V) additional space, where V is the number of courses and E the
 * number of prerequisite edges.
 *
 * @param prerequisites course id → the ids it directly depends on; must contain every referenced id
 * @return an order in which every course appears after all of its prerequisites
 * @throws PrerequisiteCycleException if the graph contains a cycle
 * @throws UnknownCourseException     if a prerequisite refers to a course not in the map
 */
public StudyPlan order(Map<CourseId, Set<CourseId>> prerequisites) {
    validateReferences(prerequisites);

    Map<CourseId, Integer> inDegree = new HashMap<>();
    Map<CourseId, List<CourseId>> dependents = new HashMap<>();

    for (var entry : prerequisites.entrySet()) {
        inDegree.putIfAbsent(entry.getKey(), 0);
        for (CourseId prerequisite : entry.getValue()) {
            inDegree.merge(entry.getKey(), 1, Integer::sum);
            dependents.computeIfAbsent(prerequisite, k -> new ArrayList<>()).add(entry.getKey());
        }
    }

    Deque<CourseId> ready = inDegree.entrySet().stream()
            .filter(e -> e.getValue() == 0)
            .map(Map.Entry::getKey)
            .collect(toCollection(ArrayDeque::new));

    List<CourseId> ordered = new ArrayList<>(inDegree.size());

    while (!ready.isEmpty()) {
        CourseId next = ready.poll();
        ordered.add(next);
        for (CourseId dependent : dependents.getOrDefault(next, List.of())) {
            if (inDegree.merge(dependent, -1, Integer::sum) == 0) {
                ready.add(dependent);
            }
        }
    }

    if (ordered.size() != inDegree.size()) {
        throw new PrerequisiteCycleException(findCycle(prerequisites));
    }
    return new StudyPlan(ordered);
}
```

### Reporting the cycle, not just detecting it

"No valid order exists" is a useless error message. Find the actual cycle with a DFS and colour marking:

```java
private List<CourseId> findCycle(Map<CourseId, Set<CourseId>> graph) {
    Set<CourseId> visiting = new LinkedHashSet<>();     // current DFS path, in order
    Set<CourseId> visited  = new HashSet<>();

    for (CourseId start : graph.keySet()) {
        List<CourseId> cycle = dfs(start, graph, visiting, visited);
        if (cycle != null) return cycle;
    }
    return List.of();
}

private List<CourseId> dfs(CourseId node, Map<CourseId, Set<CourseId>> graph,
                           Set<CourseId> visiting, Set<CourseId> visited) {
    if (visiting.contains(node)) {                      // found it — extract the loop
        List<CourseId> path = new ArrayList<>(visiting);
        return path.subList(path.indexOf(node), path.size());
    }
    if (!visited.add(node)) return null;

    visiting.add(node);
    for (CourseId prerequisite : graph.getOrDefault(node, Set.of())) {
        List<CourseId> cycle = dfs(prerequisite, graph, visiting, visited);
        if (cycle != null) return cycle;
    }
    visiting.remove(node);
    return null;
}
```

The API can now answer: *"Databases → Spring Boot → Databases"*, which a user can act on.

> **Recursion depth.** DFS recurses once per node on the path. For a course catalogue that is fine; for
> a graph with 100 000 nodes it is a `StackOverflowError`, and the iterative version with an explicit
> stack is the answer. Knowing when the limit matters is part of the outcome.

---

## Complexity, stated and measured

Every student writes a short complexity note: time, space, big-O, and what happens as n grows.

**Then measure it.** A JMH benchmark is overkill, but timing the algorithm over inputs of 10, 100,
1 000 and 10 000 and putting the numbers in a table turns theory into evidence:

| n (courses) | edges | time (ms) | time / n |
|---|---|---|---|
| 10 | 15 | 0.04 | 0.004 |
| 100 | 180 | 0.31 | 0.003 |
| 1 000 | 1 900 | 2.9 | 0.003 |
| 10 000 | 19 500 | 31 | 0.003 |

Linear: time per element stays flat. A student who claims O(n log n) but whose measurements are clearly
quadratic has learned something valuable — that is a *good* outcome for this exercise, not a failure.

```java
// Crude but honest measurement; note the warm-up.
for (int n : List.of(10, 100, 1_000, 10_000)) {
    Map<CourseId, Set<CourseId>> graph = randomDag(n, 2);
    for (int i = 0; i < 50; i++) validator.order(graph);        // warm up the JIT
    long start = System.nanoTime();
    for (int i = 0; i < 100; i++) validator.order(graph);
    System.out.printf("n=%d  %.2f ms%n", n, (System.nanoTime() - start) / 100 / 1e6);
}
```

---

## Application complexity, not just algorithmic

The outcome says *rakenduse osiseid* — application parts — as well as algorithms. Add **at least two**:

### Caching

```java
@Cacheable(value = "studyPlans", key = "#curriculumId")
public StudyPlan planFor(Long curriculumId) { ... }

@CacheEvict(value = "studyPlans", key = "#curriculumId")
public void invalidate(Long curriculumId) { ... }
```

Document the eviction rule and the staleness risk. "It's cached forever and nobody knows why the change
didn't appear" is the failure mode.

### Async and scheduled work

```java
@Async
public CompletableFuture<Report> generate(Long courseId) { ... }

@Scheduled(cron = "0 0 3 * * *")           // 03:00 nightly
public void recalculateAllGrades() { ... }
```

Think about overlap: what happens if two runs collide? (`@SchedulerLock`, a database flag, or making the
job idempotent.)

### Other options

- **Pagination and sorting** everywhere a list can grow unbounded
- **Streaming export** — CSV or PDF written to the response, not built in memory
- **Bulk import** — CSV upload with per-row validation, all-or-nothing transaction, readable error report
- **Security** — Spring Security with roles (student, teacher, admin) and method-level authorisation
- **Concurrency safety** — optimistic locking with `@Version` (guide 03)

---

## Exercises

### E8.1 — Implement (core)
Implement your chosen component behind a clean interface, with unit tests covering the happy path and at
least three edge cases (empty input, single element, cycle or degenerate case).

### E8.2 — Document (core)
Write `docs/algorithm.md`: the problem, the approach chosen, **one alternative you rejected and why**,
the complexity, and the measured timings.

### E8.3 — Measure (intermediate)
Produce the timing table over at least four input sizes and state whether it matches your complexity
claim.

### E8.4 — Application parts (intermediate)
Add two of the application-complexity items above, with a written note on the risk each introduces.

### E8.5 — Useful errors (intermediate)
Make the failure case actionable: report *which* cycle, *which* room is over capacity, *which* rows of
the import failed.

### E8.6 — Defend (core)
Demonstrate the component live in five minutes, including a deliberately broken input.

---

## Solutions

<details>
<summary>E8.1 — the edge cases that matter for a graph algorithm</summary>

```java
@Test void empty_catalogue_produces_an_empty_plan() { ... }
@Test void single_course_with_no_prerequisites_is_its_own_plan() { ... }
@Test void linear_chain_is_returned_in_dependency_order() { ... }
@Test void diamond_dependency_places_the_shared_prerequisite_first() { ... }
@Test void two_disconnected_subgraphs_both_appear_in_the_plan() { ... }
@Test void direct_cycle_is_reported_with_both_courses() { ... }
@Test void indirect_cycle_of_three_is_reported_in_order() { ... }
@Test void self_dependency_is_reported_as_a_cycle() { ... }
@Test void prerequisite_referring_to_an_unknown_course_is_rejected() { ... }
```

The diamond is the case that catches a naive implementation emitting a course before all of its
prerequisites.
</details>

<details>
<summary>E8.2 — the "rejected alternative" section, done well</summary>

> **Rejected: depth-first search with post-order output.**
>
> DFS also produces a topological order and needs no in-degree bookkeeping, so it is slightly less code.
> I rejected it for two reasons.
>
> First, recursion depth: our prerequisite chains are shallow today, but the algorithm is also used for
> the full national curriculum import, where a pathological chain could exceed the default stack. Kahn's
> algorithm is iterative by construction.
>
> Second, error reporting: Kahn's leaves every course that is part of (or downstream of) a cycle with a
> non-zero in-degree, which gives me the *set* of affected courses for free. I still run a DFS to
> extract the exact loop for the message, but only on failure — the common path stays iterative.
>
> The cost is a slightly larger implementation and one extra map.

Name a real trade-off, not a strawman.
</details>

<details>
<summary>E8.4 — the risk note for caching</summary>

> `studyPlans` is cached in memory with no TTL, keyed by curriculum id, and evicted explicitly whenever
> a prerequisite edge is added or removed.
>
> **Risk:** the eviction is only triggered through `CurriculumService`. If prerequisites are ever changed
> by a direct SQL migration or by a second application instance, this instance serves a stale plan
> indefinitely, and the symptom (a student sees an order that contradicts the catalogue) looks like an
> algorithm bug rather than a cache bug.
>
> **Mitigation taken:** a 30-minute TTL as a backstop, and the cache name appears in the actuator
> endpoint so it can be cleared without a restart. **Not taken:** a distributed cache — out of scope for
> a single-instance deployment, but it is the first thing to revisit if we scale out.

---

Caching without an eviction story is the most common way to "improve performance" and introduce a bug
that only appears under real use.
</details>

---

## Checklist — ÕV8 achieved

- [ ] The algorithm is your own work and you can walk through it line by line
- [ ] Edge cases handled explicitly rather than crashing
- [ ] The complexity claim matches the measurements
- [ ] Failure cases produce actionable messages, not "invalid input"
- [ ] The component is isolated well enough to unit test without the rest of the application
- [ ] At least two application-complexity features, each with a written risk note

**Exceeds:** a rejected alternative argued properly, plus concurrency or security handled deliberately
rather than added as a checkbox.

