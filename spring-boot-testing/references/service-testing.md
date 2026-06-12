# Testing Spring Boot Services

Use `@ExtendWith(MockitoExtension.class)` for service unit tests. This avoids loading the full Spring context, keeping tests fast and focused.

## Required dependencies

```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

`spring-boot-starter-test` includes JUnit 5, Mockito, AssertJ, and Hamcrest.

## Basic test class structure

```java
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;

    // tests go here
}
```

- `@Mock` creates a mock instance of the dependency
- `@InjectMocks` injects the mocks into the service under test (by constructor, setter, or field)
- `@ExtendWith(MockitoExtension.class)` initialises the mocks before each test

## Mock setup patterns

```java
// Return a value
when(userRepository.findById(1L)).thenReturn(Optional.of(user));

// Return a value only on first call, different on second
when(userRepository.findByEmail(anyString()))
    .thenReturn(Optional.of(user))
    .thenReturn(Optional.empty());

// Throw an exception
when(userRepository.save(any())).thenThrow(new DataIntegrityViolationException("Duplicate email"));

// Do nothing for void methods
doNothing().when(emailService).sendWelcomeEmail(anyString());

// Throw for void methods
doThrow(new MailSendException("SMTP down")).when(emailService).sendWelcomeEmail(anyString());
```

## Testing void methods and verifying interactions

Use `verify()` to confirm that side effects happened:

```java
@Test
void shouldDeleteUser_whenValidId() {
    when(userRepository.findById(1L)).thenReturn(Optional.of(user));

    userService.deleteUser(1L);

    verify(userRepository).delete(user);
}
```

Check exact invocation count or that something never happened:

```java
verify(userRepository, times(1)).delete(user);
verify(userRepository, never()).delete(any());
verify(emailService, atLeastOnce()).sendWelcomeEmail(anyString());
```

## Complete example

**Source service:**
```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;

    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));
    }

    public User createUser(String name, String email) {
        if (userRepository.findByEmail(email).isPresent()) {
            throw new DuplicateResourceException("Email already in use: " + email);
        }
        User user = new User(name, email);
        User saved = userRepository.save(user);
        emailService.sendWelcomeEmail(saved.getEmail());
        return saved;
    }

    public void deleteUser(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));
        userRepository.delete(user);
    }
}
```

**Test class:**
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUser_whenValidId() {
        User user = new User(1L, "john@example.com");
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        User result = userService.findById(1L);

        assertThat(result).isNotNull();
        assertThat(result.getEmail()).isEqualTo("john@example.com");
    }

    @Test
    void shouldThrowNotFound_whenIdDoesNotExist() {
        when(userRepository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.findById(99L))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("99");
    }

    @Test
    void shouldCreateUser_whenEmailIsUnique() {
        User userToSave = new User("john", "john@example.com");
        User savedUser = new User(1L, "john@example.com");
        when(userRepository.findByEmail("john@example.com")).thenReturn(Optional.empty());
        when(userRepository.save(any())).thenReturn(savedUser);

        User result = userService.createUser("john", "john@example.com");

        assertThat(result.getId()).isEqualTo(1L);
        verify(emailService).sendWelcomeEmail("john@example.com");
    }

    @Test
    void shouldThrowDuplicate_whenEmailExists() {
        when(userRepository.findByEmail("john@example.com"))
            .thenReturn(Optional.of(new User(1L, "john@example.com")));

        assertThatThrownBy(() -> userService.createUser("john", "john@example.com"))
            .isInstanceOf(DuplicateResourceException.class)
            .hasMessageContaining("already in use");

        verify(userRepository, never()).save(any());
        verify(emailService, never()).sendWelcomeEmail(anyString());
    }

    @Test
    void shouldDeleteUser_whenValidId() {
        User user = new User(1L, "john@example.com");
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        userService.deleteUser(1L);

        verify(userRepository).delete(user);
    }

    @Test
    void shouldThrowNotFound_whenDeletingNonExistentUser() {
        when(userRepository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.deleteUser(99L))
            .isInstanceOf(ResourceNotFoundException.class);
    }
}
```

## Testing exceptions

```java
@Test
void shouldThrowException_whenRepositoryFails() {
    when(userRepository.findById(anyLong())).thenThrow(new DataAccessException("DB down"));

    assertThatThrownBy(() -> userService.findById(1L))
        .isInstanceOf(DataAccessException.class);
}

@Test
void shouldMapRepositoryExceptionToServiceException() {
    when(userRepository.save(any())).thenThrow(new DataIntegrityViolationException("Duplicate"));

    assertThatThrownBy(() -> userService.createUser("john", "john@example.com"))
        .isInstanceOf(ServiceException.class)
        .hasCauseInstanceOf(DataIntegrityViolationException.class);
}
```

## Argument matchers

Use Mockito's argument matchers to write flexible stubs:

```java
// Any value
when(repository.findById(anyLong())).thenReturn(...);
when(repository.findByEmail(anyString())).thenReturn(...);

// Specific type
when(repository.save(any(User.class))).thenReturn(...);

// Capture arguments for further assertions
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(repository).save(captor.capture());
assertThat(captor.getValue().getEmail()).isEqualTo("john@example.com");
```

## Common pitfalls

- **Missing `MockitoExtension`**: without `@ExtendWith(MockitoExtension.class)`, `@Mock` and `@InjectMocks` are not processed and fields remain null
- **Mixing mocks and `@SpringBootTest` unnecessarily**: if you can test with pure Mockito (no Spring context), do so — it's faster and more focused
- **Not verifying interactions**: `verify()` is essential for void methods where no value is returned
- **Stubbing void methods with `when()`**: use `doThrow()`, `doNothing()`, `doAnswer()` instead of `when()` for void methods
