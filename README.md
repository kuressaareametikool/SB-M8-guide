# Spring Boot Developer Study Guide (TAK-24)

Backend development with **Spring Boot 4.1, Java 26 and JPA/Hibernate 7**. Every learning outcome (ÕV)
is practised on one running project rather than on disconnected exercises.

The guides are numbered in **build order**, not in ÕV order — the application has to exist before it
can be tested, optimised or documented. Work through them in sequence, starting with
[guide 00](00-getting-started.md).

**Everything runs in Docker.** The build, the application, the tests and the database are all
containerised — you need Docker and Git, and no local JDK, Maven or PostgreSQL.

Start from the template repository, which has a working skeleton and the container setup already
committed:

**https://github.com/<your-org>/tak24-spring-template** → *Use this template*

```bash
git clone https://github.com/<you>/kursusepunkt.git
cd kursusepunkt && make dev-docker
```

[Guide 00](00-getting-started.md) walks through it.

| # | Guide | Outcome |
|---|---|---|
| 00 | [Getting started](00-getting-started.md) | — |
| 01 | [Setup and code standard](01-setup-and-code-standard.md) | ÕV7 |
| 02 | [REST and MVC architecture](02-rest-and-mvc-architecture.md) | ÕV3 |
| 03 | [ORM with JPA and Hibernate](03-orm-with-jpa.md) | ÕV4 |
| 04 | [Programming patterns](04-programming-patterns.md) | ÕV1 |
| 05 | [Math and logic in applications](05-math-and-logic.md) | ÕV2 |
| 06 | [Unit testing](06-unit-testing.md) | ÕV5 |
| 07 | [Mocks and test doubles](07-mocks-and-test-doubles.md) | ÕV6 |
| 08 | [Complex algorithms and components](08-complex-algorithms.md) | ÕV8 |
| 09 | [Documentation in English](09-documentation.md) | ÕV9 |
| 10 | [Capstone brief and rubric](10-capstone-and-rubric.md) | all |
| 11 | [Containerised setup](11-docker-setup.md) | supports ÕV4, ÕV7, ÕV9 |

## The learning outcomes

| Code | Outcome (ET) | Guide |
|---|---|---|
| ÕV1 | tunneb enamlevinud programmeerimismustreid | 04 |
| ÕV2 | kasutab rakenduste koostamisel matemaatika- ja loogikafunktsioone | 05 |
| ÕV3 | realiseerib rakenduse MVC arhitektuuriga rakendusena | 02 |
| ÕV4 | kasutab parimate praktikate kohaselt ORM vahendeid | 03 |
| ÕV5 | mõistab ühiktestide olemust ning nende kasutamisvõimalusi | 06 |
| ÕV6 | kasutab testides mock-klasse | 07 |
| ÕV7 | kasutab korrektselt kokkulepitud koodistandardit | 01 |
| ÕV8 | loob suurema keerukusastmega rakendusi | 08 |
| ÕV9 | dokumenteerib loodud rakendused inglise keeles | 09 |

## The running project: KursusePunkt

The backend of a course-and-grade management system — courses, students, enrolments, assignments,
weighted grade calculation and reporting endpoints — exposed as a REST API.

**The user interface is built by a separate module**, so this project is backend only: no server-rendered
views. That does not weaken ÕV3; a consumer you cannot see is a stricter test of layering than a
template you control yourself.

Any domain of similar complexity is acceptable as a substitute: a gym booking system, a library, a
bike-rental service.

## Stack

| Tool | Version | Notes |
|---|---|---|
| Java | 26 | JDK 25 is the current LTS; nothing here needs a 26-only feature |
| Spring Boot | 4.1.x | Spring Framework 7, modular starters |
| Hibernate | 7 | via Spring Data JPA |
| PostgreSQL | 16 | H2 in tests |
| Flyway | 12 | versioned migrations, `ddl-auto=validate` |
| JUnit 5 + Mockito 5 + AssertJ | current | `@MockitoBean`, `MockMvcTester` |
| springdoc-openapi | current | API documentation |
| Spotless + Google Java Format | current | enforced in the build |

> Most Spring Boot material online still targets Boot 3. `spring-boot-starter-web`, `@MockBean` and
> `javax.*` imports are the tells. Check which version a snippet was written for before using it.

## How each guide is structured

1. **Why it matters** — the point of the outcome
2. **Theory** — the concepts and the rules
3. **Code** — working examples from the running project
4. **Exercises** — tasks, marked basic / intermediate / advanced
5. **Solutions** — worked answers, collapsed so you can try first
6. **Checklist** — what "achieved" looks like for this outcome
