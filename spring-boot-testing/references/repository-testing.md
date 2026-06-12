# Testing Spring Data JPA Repositories

Use `@DataJpaTest` to test JPA repositories. It configures an in-memory database, scans `@Entity` classes, and enables Spring Data JPA repositories — without loading the full application context.

## Basic structure

```java
import static org.assertj.core.api.Assertions.*;

@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;

    // tests go here
}
```

- `@DataJpaTest` loads only JPA-related beans (entity manager, data source, repositories)
- `TestEntityManager` provides helper methods to persist and flush test data directly
- Each test method runs in a transaction that is rolled back after the test

## Using TestEntityManager vs repository for setup

```java
// Using TestEntityManager (preferred for setup — bypasses custom logic)
User user = new User("john@example.com", "John");
entityManager.persistAndFlush(user);

// Using repository directly (also works, but may invoke custom logic)
userRepository.save(new User("john@example.com", "John"));
```

`persistAndFlush()` immediately writes to the database and makes the data visible to queries. Use it when test setup must be visible to the query under test.

## Testing custom derived queries

```java
@Test
void shouldFindUsersByEmailDomain() {
    entityManager.persistAndFlush(new User("user1@example.com", "User 1"));
    entityManager.persistAndFlush(new User("user2@example.com", "User 2"));
    entityManager.persistAndFlush(new User("other@gmail.com", "Other"));

    List<User> result = userRepository.findByEmailEndingWith("@example.com");

    assertThat(result).hasSize(2);
    assertThat(result).extracting(User::getEmail)
        .containsExactlyInAnyOrder("user1@example.com", "user2@example.com");
}
```

## Testing @Query annotations

```java
@Test
void shouldFindActiveUsersAfterDate() {
    User recent = new User("recent@example.com", "Recent");
    recent.setCreatedAt(LocalDate.now().minusDays(5));
    recent.setActive(true);

    User old = new User("old@example.com", "Old");
    old.setCreatedAt(LocalDate.now().minusDays(40));
    old.setActive(true);

    User inactive = new User("inactive@example.com", "Inactive");
    inactive.setCreatedAt(LocalDate.now().minusDays(5));
    inactive.setActive(false);

    entityManager.persistAndFlush(recent);
    entityManager.persistAndFlush(old);
    entityManager.persistAndFlush(inactive);

    List<User> result = userRepository.findActiveUsersSince(LocalDate.now().minusDays(30));

    assertThat(result).hasSize(1);
    assertThat(result.get(0).getEmail()).isEqualTo("recent@example.com");
}
```

## Testing derived query uniqueness and existence

```java
@Test
void shouldReturnTrue_whenEmailExists() {
    entityManager.persistAndFlush(new User("existing@example.com", "Existing"));

    boolean exists = userRepository.existsByEmail("existing@example.com");

    assertThat(exists).isTrue();
}

@Test
void shouldReturnFalse_whenEmailDoesNotExist() {
    boolean exists = userRepository.existsByEmail("nonexistent@example.com");

    assertThat(exists).isFalse();
}
```

## Testing pagination and sorting

```java
@Test
void shouldReturnPaginatedResults() {
    for (int i = 1; i <= 10; i++) {
        entityManager.persistAndFlush(new User("user" + i + "@example.com", "User " + i));
    }

    Pageable pageable = PageRequest.of(0, 3, Sort.by("email").ascending());
    Page<User> page = userRepository.findAll(pageable);

    assertThat(page.getContent()).hasSize(3);
    assertThat(page.getTotalElements()).isEqualTo(10);
    assertThat(page.getTotalPages()).isEqualTo(4);
    assertThat(page.isFirst()).isTrue();
}
```

## Testing with @Embedded fields and @Embeddable

```java
@Test
void shouldQueryByEmbeddedField() {
    Address address = new Address("123 Main St", "Springfield", "IL", "62701");
    User user = new User("user@example.com", "User");
    user.setAddress(address);
    entityManager.persistAndFlush(user);

    List<User> result = userRepository.findByAddressCity("Springfield");

    assertThat(result).hasSize(1);
}
```

## Custom configuration with embedded database

By default, `@DataJpaTest` uses an in-memory database (H2). If you need to disable replacing the data source:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest {
    // uses your configured datasource (e.g., testcontainers or a real DB)
}
```

## Testing with @Enumerated fields

```java
@Test
void shouldFindUsersByStatus() {
    entityManager.persistAndFlush(new User("active@example.com", "Active", UserStatus.ACTIVE));
    entityManager.persistAndFlush(new User("inactive@example.com", "Inactive", UserStatus.INACTIVE));

    List<User> result = userRepository.findByStatus(UserStatus.ACTIVE);

    assertThat(result).hasSize(1);
    assertThat(result.get(0).getEmail()).isEqualTo("active@example.com");
}
```

## Common pitfalls

- **Transaction boundaries**: `@DataJpaTest` tests are transactional by default. If testing lazy loading, add `@Transactional(propagation = Propagation.NOT_SUPPORTED)` or call `entityManager.clear()` before the assertion
- **Not flushing**: derived queries and `@Query` annotations read from the database. Use `persistAndFlush()` or `flush()` after setup to ensure data is visible
- **Mixing repository test with service logic**: `@DataJpaTest` should only test data access. Business logic belongs in service unit tests
- **Missing `@ActiveProfiles`**: if your entities use database-specific features, add `@ActiveProfiles("test")` to load the right configuration
- **Asserting entity equality**: entities loaded in different persistence contexts may not be `equals()`. Assert on field values rather than object identity
