# Best Practices for Spring Boot Tests

Read this reference first before any other reference. It defines conventions that apply across all testing layers.

## Test method naming

Use BDD-style names that describe behaviour, not implementation. This makes test failures immediately informative.

```
should<expected_outcome>_when<condition>
```

**Examples:**
```java
@Test
void shouldReturnUser_whenValidId() { ... }

@Test
void shouldThrowNotFoundException_whenUserDoesNotExist() { ... }

@Test
void shouldReturnEmptyList_whenNoUsersMatchSearch() { ... }

@Test
void shouldUpdateEmail_whenNewEmailIsValid() { ... }
```

Alternative pattern (when the method name helps clarity):
```java
@Test
void findById_shouldReturnUser_whenValidId() { ... }
```

## AAA pattern (Arrange-Act-Assert)

Every test method follows three blocks separated by blank lines:

```java
@Test
void shouldReturnUser_whenValidId() {
    User mockUser = new User(1L, "john@example.com");
    when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));

    User result = userService.findById(1L);

    assertThat(result.getEmail()).isEqualTo("john@example.com");
}
```

- **Arrange**: create test data, configure mocks
- **Act**: invoke the method under test (single line)
- **Assert**: verify the result using AssertJ

## AssertJ over JUnit assertions

Prefer AssertJ's fluent assertions. They compose better and produce readable failure messages.

```java
// Prefer this:
assertThat(result).isNotNull();
assertThat(result.getEmail()).startsWith("john").endsWith(".com");
assertThat(results).hasSize(3).contains(user1, user2);

// Avoid this:
assertNotNull(result);
assertEquals("john@example.com", result.getEmail());
assertEquals(3, results.size());
```

Common AssertJ patterns:
```java
assertThat(optionalResult).isPresent();
assertThat(optionalResult).isEmpty();
assertThat(listResult).containsExactly(user1, user2);
assertThat(listResult).extracting(User::getEmail).contains("john@example.com");
assertThat(exception).hasMessage("User not found");
assertThatThrownBy(() -> service.findById(-1L))
    .isInstanceOf(ResourceNotFoundException.class)
    .hasMessageContaining("not found");
```

## Test class structure

Place test classes in the same package as the source class under `src/test/java`. Use the `Test` suffix.

| Source class         | Test class               | Location                                      |
|----------------------|--------------------------|-----------------------------------------------|
| `UserService`        | `UserServiceTest`        | `src/test/java/com/example/UserServiceTest.java` |
| `UserRepository`     | `UserRepositoryTest`     | `src/test/java/com/example/UserRepositoryTest.java` |
| `UserController`     | `UserControllerTest`     | `src/test/java/com/example/UserControllerTest.java` |

## What to test

Test behaviour, not implementation. For each public method, cover:

1. **Happy path** — the primary success scenario
2. **Edge cases** — null inputs, empty collections, boundary values
3. **Error paths** — exceptions thrown, error responses returned
4. **State changes** — side effects (database updates, external API calls) through verification

## Verification checklist

After generating a test class, verify:

- [ ] All needed imports are present (JUnit 5, Mockito, AssertJ, Spring test annotations)
- [ ] The test class uses the correct annotation(s) for its layer
- [ ] Mocks are created with `@Mock` (not `Mockito.mock()`) and injected with `@InjectMocks`
- [ ] Each test method follows AAA pattern
- [ ] Each test method has a descriptive BDD-style name
- [ ] Assertions use AssertJ (not JUnit `assertEquals`/`assertNotNull`)
- [ ] Both happy path and error/edge case tests exist
- [ ] Mock interactions are verified with `verify()` where relevant
