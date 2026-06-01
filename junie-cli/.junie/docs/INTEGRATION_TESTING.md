# Integration Testing Standards

This document defines the authoritative testing conventions for integration tests in this Spring Boot project, based on existing implementation patterns.

## Purpose and Responsibility

Integration tests MUST test complete use case flows through HTTP endpoints with real Spring application context, database, and mocked external services. Integration tests MUST verify end-to-end functionality including HTTP request/response handling, authentication, validation, business logic, and database persistence.

Integration tests MUST include:
- Full Spring Boot application context
- HTTP endpoint testing via WebTestClient
- Database operations with H2 in-memory database
- External service mocking via WireMock
- Security and authentication flows
- JSON request/response verification

Integration tests MUST NOT include:
- Real external service calls
- Real databases (production databases)
- File system operations outside of classpath resources

## Typical Class Structure and Annotations

Integration test classes MUST follow this exact structure:

```java
@IntegrationTest
@EnableWireMock({
    @ConfigureWireMock(name = "server-name", property = "config.property.name")
})
@WithTestData
class [UseCase]UseCaseIT {
    
    @Autowired
    private [Repository] repository;
    
    @Autowired
    private WebTestClient webTestClient;
    
    @MockBean
    private JwtDecoder jwtDecoder;
    
    @InjectWireMock("server-name")
    private WireMockServer serverName;
}
```

### Required Annotations

#### `@IntegrationTest` (Composite Annotation)
**Complete implementation in:** `docs/INITIAL_TEST_PREQUISITES.md`
MUST be used at class level for all integration tests.

#### `@EnableWireMock`
MUST be used when external services need mocking:
```java
@EnableWireMock({
    @ConfigureWireMock(name = "toolbox-server", property = "baseUrls.toolboxRoles"),
    @ConfigureWireMock(name = "emails-server", property = "email.uri")
})
```

#### `@WithTestData`
**Complete implementation in:** `docs/INITIAL_TEST_PREQUISITES.md`
MUST be used for database setup. Can be applied to class or method level.

### Dependency Injection
Required dependencies MUST be injected:
- `@Autowired WebTestClient webTestClient` - HTTP client
- `@Autowired [Repository] repository` - Database verification
- `@MockBean JwtDecoder jwtDecoder` - JWT authentication (already provided by `@IntegrationTest`)
- `@InjectWireMock("server-name") WireMockServer serverName` - External service mocks

## Test Data Setup

### Database Setup Pattern
Database test data MUST be managed via SQL files:

**Files Location:** `src/test/resources/db/`
- `schema.sql` - CREATE TABLE statements
- `data.sql` - INSERT statements for test data  
- `drop-schema.sql` - DROP TABLE cleanup

**Usage:**
```java
// Applied to class (all tests use same data)
@WithTestData
class SomeUseCaseIT {

// Applied to method (specific test data)
@Test
@WithTestData
void specificTest() {
```

### HTTP Request/Response Data
HTTP fixtures MUST use `HttpBodyValue` enum constants:

**Files Location:** `src/test/resources/http.*/`
- Pattern: `[operation]-[entity]-[request|response].json`
- Examples:
  - `create-absence-request.json`
  - `get-user-by-id-response.json`
  - `update-project-request.json`

**Usage:**
```java
// REQUIRED: Use enum constants
HttpBodyValue.CREATE_ABSENCE_REQ
HttpBodyValue.GET_USER_BY_ID_RESP

// Dynamic content substitution
CREATE_ABSENCE_REQ.replace("START_DATE", now().toString())
```

### JWT Test Data
**Implementation details in:** `docs/INITIAL_TEST_PREQUISITES.md`

#### Usage in Integration Tests
```java
// Annotation-based JWT (recommended)
@Test
@WithMockJwt(username = "BB555555", roles = ROLE_ADMIN)
void shouldAllowAdminOperations(ExtensionContext context) {
    verifyThat(webTestClient, context)
        .withHttpMethod(HttpMethod.DELETE)
        .withPath(USER_OPERATIONS.append("/BB666666"))
        .when()
        .authenticated()
        .thenShouldReturnResponse(HttpStatus.NO_CONTENT);
}

// Manual JWT for parameterized tests
@ParameterizedTest
@MethodSource("getUserRoleSource")
void shouldBehaveDifferentlyByRole(Jwt givenJwt) {
    given(jwtDecoder.decode(any())).willReturn(givenJwt);
    // Test execution varies by role
}
```

## Assertion Strategy

### HTTP Response Verification (Primary)
HTTP responses MUST be verified using `WebClientVerify` fluent API:

```java
// REQUIRED pattern
verifyThat(webTestClient)
    .withHttpMethod(HttpMethod.GET)
    .withPath(USER_OPERATIONS.append("/BB555555"))
    .when()
    .authenticated()
    .thenShouldReturnResponse(GET_USER_BY_ID_RESP);
```

### JSON Response Verification
JSON verification MUST use lenient comparison (ignores order and extra fields):

```java
// Status code + JSON body
.thenShouldReturnResponse(HttpBodyValue.EXPECTED_RESPONSE)

// Status code only
.thenShouldReturnResponse(HttpStatus.CREATED)

// Status code + custom JSON
.thenShouldRespondWith(HttpStatus.OK, customJsonBody)
```

### Database State Verification
Database assertions MUST use AssertJ on repository methods:

```java
// Existence verification
assertThat(absenceRepository.existsById(2L)).isTrue();

// Field verification  
assertThat(userRepository.findByEmployeeNumberIgnoreCase("BB555555").get())
    .hasFieldOrPropertyWithValue("email", expectedEmail);

// Collection size verification
assertThat(userRepository.findByEmployeeNumberIgnoreCase("BB555555").get().getLabels())
    .hasSizeGreaterThan(1);
```

### Authentication/Authorization Verification
Authentication failures MUST be verified using specific methods:

```java
// 401 Unauthorized
.thenShouldDeniedAccess()

// 403 Forbidden (for role-based access)
.thenShouldReturnResponse(HttpStatus.FORBIDDEN)
```

### Static Import Requirements
These static imports MUST be used:
```java
import static com.unicredit.tis.test.fixtures.verifies.Verifications.verifyThat;
import static com.unicredit.tis.test.fixtures.http.HttpPathValue.*;
import static com.unicredit.tis.test.fixtures.http.HttpBodyValue.*;
```

### Custom Headers with @WithHeaders Annotation

**Implementation details in:** `docs/INITIAL_TEST_PREQUISITES.md`

#### Usage in Integration Tests

#### Usage Patterns

**Class-Level Headers (Applied to all test methods):**
```java
@IntegrationTest
@WithTestData
@WithHeaders(headers = {
    "X-Client-Id=integration-test-client", 
    "X-Environment=test"
})
class UserIntegrationTest {
    
    @Test
    void shouldInheritClassHeaders(ExtensionContext context) {
        verifyThat(webTestClient, context)  // Headers automatically added
            .withHttpMethod(HttpMethod.GET)
            .withPath(USER_OPERATIONS)
            .when()
            .authenticated()
            .thenShouldReturnResponse(GET_USERS_RESP);
    }
}
```

**Method-Level Headers (Override class-level):**
```java
@Test
@WithHeaders(headers = {
    "X-Special-Operation=create-admin-user",
    "X-Priority=critical"
})
void shouldCreateAdminUserWithSpecialHeaders(ExtensionContext context) {
    verifyThat(webTestClient, context)
        .withHttpMethod(HttpMethod.POST)
        .withPath(USER_OPERATIONS)
        .withRequestBody(CREATE_ADMIN_USER_REQ)
        .when()
        .authenticated()
        .thenShouldReturnResponse(HttpStatus.CREATED);
}
```

**Combined with Security Annotations:**
```java
@Test
@WithMockJwt(username = "BB555555", roles = ROLE_ADMIN)
@WithHeaders(headers = {
    "X-Admin-Operation=user-management",
    "X-Audit-Required=true"
})
void shouldCombineSecurityAndHeaders(ExtensionContext context) {
    verifyThat(webTestClient, context)
        .withHttpMethod(HttpMethod.DELETE)
        .withPath(USER_OPERATIONS.append("/BB666666"))
        .when()
        .authenticated()
        .thenShouldReturnResponse(HttpStatus.NO_CONTENT);
}
```

#### Header Precedence Rules
1. **Method-level @WithHeaders** overrides **class-level @WithHeaders** (same header names)
2. **Manual .headerValue()** overrides **@WithHeaders** (same header names)
3. **Default headers** (REQUEST_ID, REQUEST_TIMESTAMP) are always added unless explicitly overridden
4. **replace = false** preserves existing headers with same names

## Mocking Strategy

### External Service Mocking (WireMock)
External services MUST be mocked using WireMock helper methods:

```java
// Toolbox service (roles)
givenToolboxServer(toolboxServer)
    .willGetEmployeeRole()           // Returns ["tis-employee"]
    .willGetManagerAdminRoles()      // Returns multiple roles
    .willGetAdminRole()              // Returns admin + employee
    .willGetManagerRole();           // Returns manager + employee

// Email service
givenEmailServer(emailsServer)
    .willSendEmail();                // Returns {"Status": "Ok"}

// Calendar/Days service
givenDaysServer(daysServer)
    .willGetDays();                  // Returns calendar dates array
```

### JWT Decoder Mocking
JWT decoding MUST be stubbed for each test:

```java
// Single JWT for test
given(jwtDecoder.decode(any())).willReturn(createDefaultJwt());

// Parameterized tests with different JWTs
@ParameterizedTest
@MethodSource("getJwtSource")
void testWithDifferentUsers(Jwt givenJwt) {
    given(jwtDecoder.decode(any())).willReturn(givenJwt);
    // ... test execution
}

static Stream<Jwt> getJwtSource() {
    return Stream.of(
        createDefaultJwt(),
        createDefaultJwt("BB44444", ROLE_ADMIN),
        createDefaultJwt("BB44444", ROLE_MANAGER)
    );
}
```

### What MUST Be Mocked
- External HTTP services (via WireMock)
- `JwtDecoder` (via `@MockBean`, already configured in `@IntegrationTest`)
- `OutlookAppointmentService` (via `@MockBean`, already configured in `@IntegrationTest`)

### What MUST NOT Be Mocked
- Spring Boot application context
- Controllers and use cases
- Services and repositories
- Database operations
- Security filters and authentication flow
- JSON serialization/deserialization

## Naming Conventions

### Class Names  
- Pattern: `[UseCase]UseCaseIT`
- Examples: `GetUserUseCaseIT`, `CreateAbsenceUseCaseIT`, `UpdateUserUseCaseIT`

### Test Method Names
- Pattern: `should[ExpectedBehavior]When[Condition]`
- Examples:
  - `shouldReturnUsersWhenUserIsAuthenticated`
  - `shouldFailWhenUserIsNotAuthenticated`
  - `shouldCreateAbsenceWithSubstitute`
  - `shouldNotCreateAbsenceWhenIsOverlappingAnotherOne`

### WireMock Server Names
- Pattern: `[service-name]-server`
- Examples: `toolbox-server`, `emails-server`, `days-server`

## Common Patterns

### Test Method Structure
Tests MUST follow Given-When-Then structure:

```java
@Test
@WithMockJwt(username = "BB555555", roles = ROLE_ADMIN)
@WithHeaders(headers = {
    "X-Request-Source=integration-test",
    "X-Operation-Type=absence-creation"
})
void shouldCreateAbsenceWhenUserIsAuthenticated(ExtensionContext context) {
    // Given
    givenToolboxServer(toolboxServer).willGetManagerAdminRoles();
    givenEmailServer(emailsServer).willSendEmail();
    
    // When & Then
    verifyThat(webTestClient, context)  // Context provides access to @WithHeaders
        .withHttpMethod(HttpMethod.POST)
        .withPath(ABSENCE_OPERATIONS)
        .withRequestBody(CREATE_ABSENCE_REQ.replace("START_DATE", now().toString()))
        .when()
        .authenticated()
        .thenShouldReturnResponse(HttpStatus.CREATED);
        
    // Database verification
    assertThat(absenceRepository.existsById(2L)).isTrue();
}
```

### Parameterized Authentication Tests
Authentication scenarios MUST use parameterized tests:

```java
@ParameterizedTest
@MethodSource("getJwtSource")
void shouldBehaveDifferentlyBasedOnUserRole(Jwt givenJwt) {
    // Given
    given(jwtDecoder.decode(any())).willReturn(givenJwt);
    givenToolboxServer(toolboxServer).willGetManagerAdminRoles();
    
    // When & Then
    verifyThat(webTestClient)
        .withHttpMethod(HttpMethod.GET)
        .withPath(USER_OPERATIONS)
        .when()
        .authenticated()
        .thenShouldReturnResponse(GET_USERS_RESP);
}
```

### Dynamic Content Pattern
Request bodies with dynamic content MUST use replacement methods:

```java
// Single replacement
.withRequestBody(CREATE_ABSENCE_REQ.replace("START_DATE", now().toString()))

// Multiple replacements
.withRequestBody(TEMPLATE_REQ
    .replace("FIELD1", value1)
    .replace("FIELD2", value2))
```

## Anti-Patterns

### Forbidden Practices
- Using real external service endpoints
- Using production databases
- Mocking Spring components (controllers, services, repositories)
- Manual JSON string construction
- Using `@MockBean` for internal components
- Direct HTTP client usage instead of `WebClientVerify`

### Authentication Anti-Patterns
```java
// FORBIDDEN: Manual security context setup
SecurityContextHolder.setAuthentication(...); // DON'T DO THIS

// REQUIRED: Use @WithMockJwt or JWT stubbing
@WithMockJwt(username = "BB555555", roles = ROLE_ADMIN) // CORRECT
```

## Representative Examples

### Primary Reference: `GetUserUseCaseIT.java`
Demonstrates:
- Complete `@IntegrationTest` setup with WireMock
- `@WithTestData` database setup
- Parameterized authentication testing
- HTTP endpoint verification with `WebClientVerify`
- External service mocking with toolbox roles
- Custom header management with `@WithHeaders`

### Header Management Examples:

**Basic Header Setup:**
```java
@IntegrationTest
@WithTestData
@WithHeaders(headers = {"X-Client-Id=test-client", "X-Environment=integration"})
class GetUserUseCaseIT {
    
    @Test
    void shouldReturnUsersWithCustomHeaders(ExtensionContext context) {
        verifyThat(webTestClient, context)
            .withHttpMethod(HttpMethod.GET)
            .withPath(USER_OPERATIONS)
            .when()
            .authenticated()
            .thenShouldReturnResponse(GET_USERS_RESP);
    }
}
```

**Dynamic Header Configuration:**
```java
@Test
@WithHeaders(headers = {
    "X-Request-Id=dynamic-" + System.currentTimeMillis(),
    "X-Trace-Id=trace-12345",
    "X-User-Context=admin-operation"
})
void shouldHandleDynamicHeaders(ExtensionContext context) {
    verifyThat(webTestClient, context)
        .withHttpMethod(HttpMethod.POST)
        .withPath(USER_OPERATIONS)
        .withRequestBody(CREATE_USER_REQ)
        .when()
        .authenticated()
        .thenShouldReturnResponse(HttpStatus.CREATED);
}
```

### Secondary References:
- `CreateAbsenceUseCaseIT.java` - POST operations with multiple external services
- `UpdateUserUseCaseIT.java` - PATCH operations with database state verification

## Test Configuration

### Test Configuration
**Database and application configuration details in:** `docs/INITIAL_TEST_PREQUISITES.md`

## Test Execution Requirements

Tests MUST run with:
- Full Spring Boot application context
- Real database operations (H2 in-memory)
- Mocked external HTTP services
- Real security and authentication flow
- Real JSON serialization/deserialization

Tests MUST provide:
- HTTP endpoint coverage
- Authentication/authorization verification
- Database state verification
- External service integration verification