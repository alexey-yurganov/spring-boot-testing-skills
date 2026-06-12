# Testing Spring Boot REST Controllers

Use `@WebMvcTest` to test controllers in isolation. This loads only the web layer — controller, advice, filters, converters — without services or repositories. Use `@MockBean` for the controller's dependencies.

## Basic structure

```java
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.assertj.core.api.Assertions.*;

@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    // tests go here
}
```

- `@WebMvcTest(UserController.class)` loads only beans needed for the web layer, scoped to the specified controller
- `MockMvc` sends HTTP requests and inspects responses without starting a server
- `@MockBean` creates a mock and adds it to the Spring application context

## Testing GET requests

```java
@Test
void shouldReturnUser_whenValidId() throws Exception {
    User user = new User(1L, "john@example.com");
    when(userService.findById(1L)).thenReturn(user);

    mockMvc.perform(get("/api/users/1")
            .accept(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.email").value("john@example.com"))
        .andExpect(jsonPath("$.id").value(1));
}

@Test
void shouldReturn404_whenUserNotFound() throws Exception {
    when(userService.findById(99L)).thenThrow(new ResourceNotFoundException("User not found"));

    mockMvc.perform(get("/api/users/99")
            .accept(MediaType.APPLICATION_JSON))
        .andExpect(status().isNotFound());
}
```

## Testing POST requests

```java
@Test
void shouldCreateUser_whenValidRequest() throws Exception {
    CreateUserRequest request = new CreateUserRequest("john", "john@example.com");
    User created = new User(1L, "john@example.com");
    when(userService.createUser(anyString(), anyString())).thenReturn(created);

    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"name\":\"john\",\"email\":\"john@example.com\"}"))
        .andExpect(status().isCreated())
        .andExpect(header().exists("Location"))
        .andExpect(jsonPath("$.id").value(1));
}

@Test
void shouldReturn400_whenRequestBodyInvalid() throws Exception {
    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"name\":\"\",\"email\":\"not-an-email\"}"))
        .andExpect(status().isBadRequest());
}
```

## Testing PUT requests

```java
@Test
void shouldUpdateUser_whenValidRequest() throws Exception {
    User updated = new User(1L, "newemail@example.com");
    when(userService.updateEmail(eq(1L), anyString())).thenReturn(updated);

    mockMvc.perform(put("/api/users/1")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"email\":\"newemail@example.com\"}"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.email").value("newemail@example.com"));
}
```

## Testing DELETE requests

```java
@Test
void shouldDeleteUser_whenValidId() throws Exception {
    doNothing().when(userService).deleteUser(1L);

    mockMvc.perform(delete("/api/users/1"))
        .andExpect(status().isNoContent());

    verify(userService).deleteUser(1L);
}

@Test
void shouldReturn404_whenDeletingNonExistentUser() throws Exception {
    doThrow(new ResourceNotFoundException("User not found")).when(userService).deleteUser(99L);

    mockMvc.perform(delete("/api/users/99"))
        .andExpect(status().isNotFound());
}
```

## Testing request parameters and path variables

```java
@Test
void shouldSearchUsers_withQueryParams() throws Exception {
    when(userService.searchUsers("john", 0, 20)).thenReturn(List.of(new User(1L, "john@example.com")));

    mockMvc.perform(get("/api/users/search")
            .param("query", "john")
            .param("page", "0")
            .param("size", "20"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$").isArray())
        .andExpect(jsonPath("$.length()").value(1));
}
```

## Testing response with custom headers

```java
@Test
void shouldSetCustomHeader() throws Exception {
    when(userService.findById(1L)).thenReturn(new User(1L, "john@example.com"));

    mockMvc.perform(get("/api/users/1"))
        .andExpect(status().isOk())
        .andExpect(header().string("X-API-Version", "1.0"));
}
```

## Testing exception handling

```java
@Test
void shouldReturn500_whenServiceThrowsUnexpectedError() throws Exception {
    when(userService.findById(1L)).thenThrow(new RuntimeException("Unexpected"));

    mockMvc.perform(get("/api/users/1"))
        .andExpect(status().isInternalServerError())
        .andExpect(jsonPath("$.error").value("Internal Server Error"));
}
```

## Complete example

**Source controller:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest request) {
        User created = userService.createUser(request.getName(), request.getEmail());
        URI location = URI.create("/api/users/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}
```

**Test class:**
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser_whenValidId() throws Exception {
        User user = new User(1L, "john@example.com");
        when(userService.findById(1L)).thenReturn(user);

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }

    @Test
    void shouldReturn404_whenUserNotFound() throws Exception {
        when(userService.findById(99L)).thenThrow(new ResourceNotFoundException("not found"));

        mockMvc.perform(get("/api/users/99"))
            .andExpect(status().isNotFound());
    }

    @Test
    void shouldCreateUser_whenValidRequest() throws Exception {
        User created = new User(1L, "john@example.com");
        when(userService.createUser("John", "john@example.com")).thenReturn(created);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"John\",\"email\":\"john@example.com\"}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1));
    }

    @Test
    void shouldDeleteUser_whenValidId() throws Exception {
        doNothing().when(userService).deleteUser(1L);

        mockMvc.perform(delete("/api/users/1"))
            .andExpect(status().isNoContent());
    }
}
```

## Common pitfalls

- **Missing `@WebMvcTest` controller class**: always specify the controller class, otherwise all controllers load (slow)
- **Using `@Mock` instead of `@MockBean`**: `@Mock` creates a plain Mockito mock not visible to the Spring context; `@MockBean` registers it as a Spring bean replacement
- **Missing `throws Exception`**: MockMvc calls throw checked exceptions; each test method signature must include `throws Exception`
- **Hardcoded JSON strings**: for complex request bodies, consider using `JacksonTester` or `ObjectMapper` to serialize test objects rather than hand-writing JSON
- **Not verifying service interactions**: use `verify()` on `@MockBean` services to confirm the controller delegates correctly
