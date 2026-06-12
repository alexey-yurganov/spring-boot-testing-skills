# Integration Testing with Spring Boot

Use `@SpringBootTest` for integration tests that validate the entire application context. Use Testcontainers to run real databases in Docker containers rather than relying on in-memory databases.

## Required dependencies

```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
testImplementation 'org.testcontainers:testcontainers:1.20.0'
testImplementation 'org.testcontainers:postgresql:1.20.0'       // or mysql, mssql, etc.
testImplementation 'org.testcontainers:junit-jupiter:1.20.0'
```

For Maven, add a `<testcontainers-bom>` or individual `testcontainers-*` dependencies.

## Basic structure with Testcontainers

```java
import static org.assertj.core.api.Assertions.*;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.context.Testcontainers;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserServiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository userRepository;

    // tests go here
}
```

- `@SpringBootTest(webEnvironment = RANDOM_PORT)` starts the full app on a random port
- `@Testcontainers` enables Testcontainers lifecycle management (container starts before tests, stops after)
- `@Container` marks the container for managed lifecycle
- `@DynamicPropertySource` overrides Spring properties at runtime before context loads
- `TestRestTemplate` behaves like `RestTemplate` but auto-resolves the random port

## Testing full application flow

```java
@Test
void shouldCreateAndRetrieveUser() {
    // Create a user via the REST API
    Map<String, String> request = Map.of("name", "John", "email", "john@example.com");

    ResponseEntity<User> createResponse = restTemplate.postForEntity(
        "/api/users", request, User.class);

    assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    assertThat(createResponse.getBody()).isNotNull();
    Long userId = createResponse.getBody().getId();

    // Retrieve via the REST API
    ResponseEntity<User> getResponse = restTemplate.getForEntity(
        "/api/users/" + userId, User.class);

    assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
    assertThat(getResponse.getBody().getEmail()).isEqualTo("john@example.com");

    // Verify data persisted in the database
    assertThat(userRepository.findById(userId)).isPresent();
}
```

## Testing validation end-to-end

```java
@Test
void shouldRejectInvalidInput() {
    Map<String, String> invalidRequest = Map.of("name", "", "email", "not-an-email");

    ResponseEntity<Map> response = restTemplate.postForEntity(
        "/api/users", invalidRequest, Map.class);

    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
    assertThat(response.getBody()).containsKey("errors");
}
```

## Testing 404 handling end-to-end

```java
@Test
void shouldReturn404_whenUserNotFound() {
    ResponseEntity<Map> response = restTemplate.getForEntity(
        "/api/users/99999", Map.class);

    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

## Using TestRestTemplate with headers

```java
@Test
void shouldIncludeCustomHeader() {
    HttpHeaders headers = new HttpHeaders();
    headers.set("Authorization", "Bearer test-token");
    HttpEntity<Void> entity = new HttpEntity<>(headers);

    ResponseEntity<User> response = restTemplate.exchange(
        "/api/users/1", HttpMethod.GET, entity, User.class);

    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
}
```

## Database assertion directly

```java
@Test
void shouldPersistUserCorrectly() {
    Map<String, String> request = Map.of("name", "Jane", "email", "jane@example.com");

    ResponseEntity<User> response = restTemplate.postForEntity(
        "/api/users", request, User.class);

    assertThat(response.getStatusCode()).isCreated();

    // Direct DB assertion
    Optional<User> saved = userRepository.findByEmail("jane@example.com");
    assertThat(saved).isPresent();
    assertThat(saved.get().getName()).isEqualTo("Jane");
}
```

## Testing with multiple containers

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
class FullIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }
    // ...
}
```

## Using `@Sql` for test data setup

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
@Sql(scripts = "/data/seed-users.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
class UserSearchIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldSearchUsers() {
        ResponseEntity<User[]> response = restTemplate.getForEntity(
            "/api/users/search?query=john", User[].class);

        assertThat(response.getBody()).hasSize(2);
    }
}
```

## When to use integration tests vs slice tests

| Scenario                                  | Use                          |
|-------------------------------------------|------------------------------|
| Testing a single service with mocks       | `@ExtendWith(MockitoExtension.class)` |
| Testing a controller in isolation         | `@WebMvcTest`                |
| Testing a repository with real DB queries | `@DataJpaTest`               |
| Testing across layers (full flow)         | `@SpringBootTest` + Testcontainers |
| Testing external integrations (API calls)   | `@SpringBootTest` + WireMock |

## Common pitfalls

- **Container not starting**: ensure Docker is running and the image name/tag is correct
- **Port conflicts**: use `WebEnvironment.RANDOM_PORT` rather than a fixed port
- **Slow startup**: Testcontainers containers take several seconds to start. Share containers across test classes using `@Testcontainers(parallel = true)` and `static` fields
- **Not cleaning test data**: each `@SpringBootTest` test runs in a transactional boundary by default if `@Transactional` is present. If you remove `@Transactional`, clean up data manually or use `@Sql` scripts
- **Hardcoded connection strings**: always use `@DynamicPropertySource` rather than hardcoding datasource properties in `application.properties`
