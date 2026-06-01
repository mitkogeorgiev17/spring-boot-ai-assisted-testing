# Controller Testing Standards

This document defines the authoritative testing conventions for controller layer tests in this Spring Boot project, based on existing implementation patterns.

## Purpose and Responsibility

Controller tests MUST test web layer components in isolation to verify HTTP request/response handling, input validation, and error responses. Controller tests MUST focus on validation logic, HTTP status codes, and error message formatting WITHOUT executing business logic.

Controller tests MUST include:
- HTTP endpoint validation testing
- Request body validation verification
- Error response format verification
- JSON serialization/deserialization testing

Controller tests MUST NOT include:
- Business logic execution
- Database operations
- External service calls  
- Security filter testing
- Authentication/authorization flows

## Typical Class Structure and Annotations

Controller test classes MUST follow this exact structure:

```java
@SuppressWarnings("java:S3577")
@WebMvcTest(controllers = [Controller].class)
@AutoConfigureMockMvc(addFilters = false)
class [Controller]CT {
    
    @MockBean
    private [Service] service;
    
    @MockBean  
    private AuthenticationService authenticationService;
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
}
```

### Required Annotations

#### `@SuppressWarnings("java:S3577")`
MUST be used to suppress SonarQube warnings about test class naming.

#### `@WebMvcTest(controllers = [Controller].class)`
MUST specify the exact controller class to test. This annotation:
- Loads only web layer components
- Disables full auto-configuration
- Enables MockMvc auto-configuration

#### `@AutoConfigureMockMvc(addFilters = false)`
MUST disable security filters since controller tests focus on validation, not authentication.

### Required Dependencies
- `@MockBean [Service] service` - Mock the domain service
- `@MockBean AuthenticationService authenticationService` - Always present (not configured in tests)
- `@Autowired MockMvc mockMvc` - HTTP request simulation
- `@Autowired ObjectMapper objectMapper` - JSON serialization

## Test Data Setup

### Factory Usage in Controller Tests
**Factory implementations in:** `docs/INITIAL_TEST_PREQUISITES.md`

#### Usage for Validation Testing
```java
@Test
void updateUserShouldReturnBadRequestWhenFirstNameIsBlank() throws Exception {
    // Given - Create invalid command using factory
    var command = UserFactory.createUpdateUserCommand().setFirstName(null);
    String requestContent = objectMapper.writeValueAsString(command);
    
    // When & Then
    mockMvc.perform(patch(userEmployeeId.getValue(), DEFAULT_EMPLOYEE_NUMBER)
            .contentType(APPLICATION_JSON)
            .content(requestContent))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.message", is("firstName: First name is mandatory.")));
    
    verifyNoInteractions(userService);
}
```

### HTTP Path Usage in Controller Tests
**HTTP utilities in:** `docs/INITIAL_TEST_PREQUISITES.md`

#### Usage Examples
```java
// Define paths using HttpPathValue
private static final HttpPath userEmployeeId = USER_OPERATIONS.append("/{employeeId}");
private static final HttpPath projectId = PROJECT_OPERATIONS.append("/{id}");

// Use in MockMvc requests
mockMvc.perform(patch(userEmployeeId.getValue(), DEFAULT_EMPLOYEE_NUMBER)
mockMvc.perform(post(PROJECT_OPERATIONS.getValue())
```

### Data Modification Pattern  
Invalid test data MUST be created by modifying valid commands:

```java
// REQUIRED: Start with valid command, then invalidate
var command = UserFactory.createUpdateUserCommand()
    .setFirstName(null);  // Make it invalid

var longNameCommand = ProjectFactory.createDefaultSaveProjectCommand()
    .setProjectName(repeat("Cool Project", 255));  // Exceed validation limit
```

### JSON Serialization
Request bodies MUST be serialized using `ObjectMapper`:

```java
// REQUIRED pattern
String requestContent = objectMapper.writeValueAsString(command);

mockMvc.perform(post(PROJECT_OPERATIONS.getValue())
    .contentType(APPLICATION_JSON)
    .content(requestContent))
```

## Assertion Strategy

### HTTP Status Verification (Primary)
Status codes MUST be verified using MockMvc matchers:

```java
// REQUIRED pattern for validation errors
.andExpect(status().isBadRequest())

// Other common patterns
.andExpect(status().isOk())
.andExpect(status().isCreated())
```

### JSON Response Verification (Primary)
JSON responses MUST be verified using `jsonPath()` with Hamcrest matchers:

```java
// REQUIRED pattern for error responses
.andExpect(jsonPath("$.status", is("BAD_REQUEST")))
.andExpect(jsonPath("$.errorCode", is("ERR009")))
.andExpect(jsonPath("$.reason", is("Invalid field(s).")))
.andExpected(jsonPath("$.message", is("firstName: First name is mandatory.")));
```

### Service Interaction Verification  
Service layer MUST NOT be called for validation failures:

```java
// REQUIRED pattern - verify no service interaction
verifyNoInteractions(userService);
verifyNoInteractions(projectService);
verifyNoInteractions(absenceService);
```

### Static Import Requirements
These static imports MUST be used:
```java
import static org.hamcrest.Matchers.is;
import static org.mockito.Mockito.verifyNoInteractions;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static com.unicredit.tis.test.fixtures.http.HttpPathValue.*;
```

## Mocking Strategy

### Service Layer Mocking
All service layer dependencies MUST be mocked with `@MockBean`:

```java
@MockBean
private UserService userService;

@MockBean  
private ProjectService projectService;

@MockBean
private AbsenceService absenceService;

// Always present (not configured)
@MockBean
private AuthenticationService authenticationService;
```

### What MUST Be Mocked
- Domain services (`UserService`, `ProjectService`, `AbsenceService`)
- `AuthenticationService` (standard requirement, not configured in tests)

### What MUST NOT Be Mocked  
- Controllers (tested via `@WebMvcTest`)
- Spring validation framework (`@Valid` processing)
- Exception handling (`GlobalExceptionHandler`)
- JSON serialization/deserialization (`ObjectMapper`)
- HTTP request/response processing

### Mock Configuration
Service mocks MUST NOT be configured in controller tests since tests verify validation failures before service execution:

```java
// FORBIDDEN: No service stubbing in controller tests
given(userService.updateUser(...)).willReturn(...); // DON'T DO THIS

// REQUIRED: Verify no service interaction
verifyNoInteractions(userService); // CORRECT
```

## Naming Conventions

### Class Names
- Pattern: `[Controller]CT`  
- Examples: `UserControllerCT`, `ProjectControllerCT`, `AbsenceControllerCT`

### Test Method Names
- Pattern: `{operation}ShouldReturn{Status}When{ValidationCondition}()`
- Examples:
  - `updateUserShouldReturnBadRequestWhenFirstNameIsBlank`
  - `saveProjectShouldReturnBadRequestWhenProjectNameIsNull`
  - `saveAbsenceShouldReturnBadRequestWhenAbsenceStartDateIsBlank`

### HTTP Path Variables
- Pattern: `private static final HttpPath [resource][Id] = [BASE_PATH].append("/{id}")`
- Examples:
  - `userEmployeeId = USER_OPERATIONS.append("/{employeeId}")`
  - `projectId = PROJECT_OPERATIONS.append("/{id}")`
  - `absenceId = ABSENCE_OPERATIONS.append("/{id}")`

## Common Patterns

### Test Method Structure
Tests MUST follow Given-When-Then structure with comments:

```java
@Test
void updateUserShouldReturnBadRequestWhenFirstNameIsBlank() throws Exception {
    // Given
    var command = UserFactory.createUpdateUserCommand()
        .setFirstName(null);
    String requestContent = objectMapper.writeValueAsString(command);
    
    // When & Then
    mockMvc.perform(patch(userEmployeeId.getValue(), DEFAULT_EMPLOYEE_NUMBER)
            .accept(APPLICATION_JSON)
            .contentType(APPLICATION_JSON)
            .content(requestContent))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.status", is("BAD_REQUEST")))
        .andExpect(jsonPath("$.errorCode", is("ERR009")))
        .andExpect(jsonPath("$.reason", is("Invalid field(s).")))
        .andExpect(jsonPath("$.message", is("firstName: First name is mandatory.")));
    
    verifyNoInteractions(userService);
}
```

### MockMvc Request Pattern
HTTP requests MUST follow this pattern:

```java
// POST requests
mockMvc.perform(post(PROJECT_OPERATIONS.getValue())
    .accept(APPLICATION_JSON)
    .contentType(APPLICATION_JSON)
    .content(requestContent))

// PATCH with path variables
mockMvc.perform(patch(userEmployeeId.getValue(), DEFAULT_EMPLOYEE_NUMBER)
    .accept(APPLICATION_JSON)  
    .contentType(APPLICATION_JSON)
    .content(requestContent))

// PATCH with query parameters
mockMvc.perform(patch(projectId.getValue(), DEFAULT_PROJECT_ID)
    .queryParam("projectOwnerId", DEFAULT_EMPLOYEE_NUMBER)
    .accept(APPLICATION_JSON)
    .contentType(APPLICATION_JSON)
    .content(requestContent))
```

### Parameterized Validation Tests
Length and format validation MUST use parameterized tests:

```java
@ParameterizedTest
@NullSource
@ValueSource(strings = "a")  // Too short
void saveProjectShouldReturnBadRequestWhenProjectNameIsInvalid(String invalidName) throws Exception {
    // Given
    var command = ProjectFactory.createDefaultSaveProjectCommand()
        .setProjectName(invalidName);
    String requestContent = objectMapper.writeValueAsString(command);
    
    // When & Then
    mockMvc.perform(post(PROJECT_OPERATIONS.getValue())
            .accept(APPLICATION_JSON)
            .contentType(APPLICATION_JSON) 
            .content(requestContent))
        .andExpect(status().isBadRequest());
        
    verifyNoInteractions(projectService);
}
```

### Error Response Structure
All validation error responses MUST have this JSON structure:

```json
{
  "status": "BAD_REQUEST",
  "errorCode": "ERR009", 
  "reason": "Invalid field(s).",
  "message": "fieldName: Validation error message."
}
```

## Anti-Patterns

### Forbidden Practices
- Testing successful operations (use integration tests)
- Stubbing service methods (`given()`, `when()`)
- Testing business logic (use unit tests)
- Including security filters (`addFilters = false` is required)
- Manual JSON string construction
- Testing multiple controllers in one test class

### Service Interaction Anti-Patterns
```java
// FORBIDDEN: Stubbing services in controller tests  
given(userService.updateUser(...)).willReturn(...); // DON'T DO THIS

// FORBIDDEN: Verifying service calls for successful operations
verify(userService).updateUser(...); // DON'T DO THIS (no successful operations tested)

// REQUIRED: Verify no service interaction for validation failures
verifyNoInteractions(userService); // CORRECT
```

### Path Construction Anti-Patterns
```java
// FORBIDDEN: Hardcoded paths
mockMvc.perform(get("/tis/users/v1.0.0/users")); // DON'T DO THIS

// REQUIRED: Use HttpPathValue enum
mockMvc.perform(get(USER_OPERATIONS.getValue())); // CORRECT
```

## Representative Examples

### Primary Reference: `UserControllerCT.java`
Demonstrates:
- Proper `@WebMvcTest` setup with security filters disabled
- Factory-based test data creation and modification
- MockMvc request patterns with path variables
- JSON path assertions for error responses
- Service non-interaction verification

### Secondary References:
- `ProjectControllerCT.java` - POST operations and parameterized validation tests
- `AbsenceControllerCT.java` - PATCH operations with Given-When-Then comments

## Test Configuration Requirements

### Testing Focus
Controller tests MUST focus exclusively on input validation failures resulting in `400 Bad Request`.