---
name: spring-boot-testing
description: Comprehensive guidance for writing unit and integration tests for Spring Boot applications. Use this skill whenever the user asks to write tests for Spring Boot components, generate test classes, mock Spring beans, set up test slices (@WebMvcTest, @DataJpaTest, @JsonTest), use Testcontainers for integration tests, or asks about Spring Boot testing best practices, JUnit 5, Mockito, or AssertJ in the context of Spring Boot. Also trigger when the user mentions covering service/business logic with tests, testing JPA repositories, testing REST controllers, or wants to add tests to an existing Spring Boot project — even if they don't explicitly say "Spring Boot" (look for Maven/Gradle projects with spring-boot-starter dependencies). Do NOT trigger for non-Spring Java testing (plain JUnit without Spring), frontend testing, or end-to-end testing with tools like Selenium or Cypress unless Spring Boot is the backend under test.
license: Proprietary. LICENSE.txt has complete terms
---

# Spring Boot Testing

A skill for creating comprehensive tests for Spring Boot applications across all layers.

## Repository structure

```
spring-boot-testing/
├── SKILL.md
├── LICENSE.txt
└── references/
    ├── best-practices.md        — Naming conventions, AAA pattern, assertions, test structure
    ├── service-testing.md       — Unit tests for @Service / @Component with Mockito
    ├── repository-testing.md    — @DataJpaTest for JPA repositories
    ├── controller-testing.md    — @WebMvcTest for REST controllers with MockMvc
    └── integration-testing.md   — @SpringBootTest with Testcontainers
```

## When to use which reference

| You need to test...                      | Read this reference                        |
|------------------------------------------|--------------------------------------------|
| @Service / @Component business logic     | `references/service-testing.md`            |
| JPA repositories / data access layer     | `references/repository-testing.md`         |
| REST controllers / web layer             | `references/controller-testing.md`         |
| Full application integration             | `references/integration-testing.md`        |
| Conventions, structure, assertions       | `references/best-practices.md` (always)    |

## General workflow

1. Read `references/best-practices.md` first to understand conventions (naming, AAA, AssertJ)
2. For each layer the user needs tested, read the corresponding reference
3. Generate the test class following the patterns in the reference
4. Verify: check imports, annotations, mocking setup, assertion coverage, edge cases

## Key principles

- Always use `@Mock` + `@InjectMocks` + `MockitoExtension` for pure unit tests (no Spring context needed)
- Use test slice annotations (`@WebMvcTest`, `@DataJpaTest`, `@JsonTest`) over full `@SpringBootTest` when testing a specific layer — they load only relevant beans and are significantly faster
- Follow Arrange-Act-Assert (AAA): set up test data and mocks → invoke the method → verify the result
- Use descriptive BDD-style test method names: `should<expected>_when<condition>` or `<method>_should<expected>_when<condition>`
- Prefer AssertJ's fluent API over JUnit's built-in assertions — it provides better readability and more expressive failure messages
- Generate one test class per production class, placed in the same package under `src/test/java`

## Task-specific usage guide

### If the user says "write tests for my project"

1. Ask which layer(s) they want to test (services, repositories, controllers, or integration)
2. Read the relevant reference files
3. Examine the source code of the classes under test to understand dependencies and behavior
4. Generate the test classes
5. If the project lacks testing dependencies, suggest adding the appropriate starters to the build file

### If the user provides specific source code

1. Read `references/best-practices.md`
2. Read the specific layer reference for the type of class provided
3. Generate the test class matching the provided code exactly (same package, same method signatures, same dependency injection pattern)
4. Cover: happy path, edge cases (null, empty, not found), and error/exception paths

### If the user asks about a testing concept

Provide a concise explanation with a code example. Link to the relevant reference section.
