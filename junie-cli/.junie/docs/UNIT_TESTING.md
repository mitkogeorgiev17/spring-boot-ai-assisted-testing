# Unit Testing Standards

Unit tests MUST test service layer components in isolation. Focus on business logic validation and error handling.

## Scope
- Service layer components only
- NO Spring context, database, external services, or network operations

## Class Structure

```java
@MockitoSettings
class [ServiceName]Tests {
    @Mock private [Dependency] dependency;
    private [ServiceName] serviceUnderTest;
    
    @BeforeEach
    void setUp() {
        serviceUnderTest = new [ServiceName](dependency);
    }
}
```

**Required:** `@MockitoSettings`, `@Mock`, manual instantiation in `@BeforeEach`  
**Forbidden:** `@InjectMocks`, `@SpringBootTest`, `@ExtendWith`

## Naming Conventions

### Class Names
Pattern: `[ServiceName]Tests`  
Examples: `UserServiceTests`, `ProjectServiceTests`, `AbsenceServiceTests`

### Test Method Names
**Happy Path:** `{methodName}ShouldReturnExpectedResponse()`
```java
void getUserShouldReturnExpectedResponse()
void saveUserShouldReturnExpectedResponse() 
void updateUserShouldReturnExpectedResponse()
```

**Error Cases:** `{methodName}ShouldThrowErrorWhen{Condition}()`
```java
void getUserShouldThrowErrorWhenUserIsNotFound()
void assignLabelInSystemShouldThrowErrorWhenLabelHasProjectScope()
void saveUserShouldThrowErrorWhenEmployeeNumberAlreadyExists()
```

### Variable Names
- Service under test: `serviceUnderTest`
- Results: `actual[DomainObject]` (e.g., `actualUser`, `actualProject`)

## Test Data

**Factory Pattern:**
```java
User user = UserFactory.createDefaultUser();
UpdateUserCommand command = UserFactory.createUpdateUserCommand();
```

**Modifications:** Use fluent setters
```java
User user = UserFactory.createDefaultUser().setLabels(new HashSet<>()).setColor(null);
```

**Variables:** Use `var` for local variables
```java
var actualUser = serviceUnderTest.getUser(employeeNumber);
```

## Assertions

**Domain Objects:** Use custom fluent assertions
```java
assertThat(actualUserDto)
    .hasFirstName("Johnny")
    .hasLastName("Terrory")
    .hasEmployeeNumber("BB055555");
```

**Collections/Primitives:** Direct AssertJ
```java
Assertions.assertThat(actualResult.getLabels()).isEmpty();
```

**Exceptions:** Use `verifyThat()` wrapper
```java
verifyThat(() -> serviceUnderTest.methodCall())
    .shouldThrowRuntimeException("Expected error message");
```

**Required Imports:**
```java
import static com.unicredit.tis.test.fixtures.assertions.ModelAssertions.assertThat;
import static com.unicredit.tis.test.fixtures.verifies.Verifications.verifyThat;
```

## Mocking

**BDD-Style Mockito:** 
```java
given(userRepository.findByEmployeeNumber(employeeNumber)).willReturn(Optional.of(user));
given(userRepository.save(any(User.class))).willAnswer(x -> x.getArgument(0));
```

**Mapper Setup:**
```java
@BeforeEach
void setUp() {
    var userMapper = Mappers.getMapper(UserMapper.class);
    var absenceMapper = new AbsenceMapperImpl();
    ReflectionTestUtils.setField(absenceMapper, "userMapper", userMapper);
    serviceUnderTest = new AbsenceService(absenceRepository, userService, absenceMapper);
}
```

**Mock:** Service dependencies, repositories, external clients  
**Don't Mock:** MapStruct mappers, domain objects, DTOs, collections, primitives

**Verification:**
```java
verify(absenceRepository, times(1)).delete(absence);
```

## Test Structure

**Given-When-Then Pattern:**
```java
@Test
void getUserShouldReturnExpectedResponse() {
    // Given
    var user = UserFactory.createDefaultUser();
    given(userRepository.findByEmployeeNumber(DEFAULT_EMPLOYEE_NUMBER)).willReturn(Optional.of(user));
    
    // When
    var actualUserDto = serviceUnderTest.getUser(DEFAULT_EMPLOYEE_NUMBER);
    
    // Then
    assertThat(actualUserDto).hasFirstName(DEFAULT_FIRSTNAME).hasLastName(DEFAULT_LASTNAME);
}
```

## Forbidden Practices
- `@SpringBootTest`, `@InjectMocks`, classic Mockito (`when()`, `thenReturn()`)
- Mocking mappers, manual object construction, builder pattern
- Direct AssertJ exception testing

## Using Factories in Unit Tests

**Factory implementations in:** `docs/INITIAL_TEST_PREQUISITES.md` (Part 2)

#### Usage Examples
```java
@Test
void getUserShouldReturnExpectedResponse() {
    // Given - Use factories for test data
    var user = UserFactory.createDefaultUser();
    var expectedDto = UserFactory.createDefaultUserDto();
    given(userRepository.findByEmployeeNumber(DEFAULT_EMPLOYEE_NUMBER))
        .willReturn(Optional.of(user));
    
    // When
    var actualDto = serviceUnderTest.getUser(DEFAULT_EMPLOYEE_NUMBER);
    
    // Then
    assertThat(actualDto).hasFirstName(DEFAULT_FIRSTNAME).hasLastName(DEFAULT_LASTNAME);
}

// Modify factory data for specific scenarios
var invalidCommand = UserFactory.createUpdateUserCommand().setEmail(null);
```

## Using Custom Assertions in Unit Tests

**Assertion implementations in:** `docs/INITIAL_TEST_PREQUISITES.md` (Part 2)

#### Usage Examples  
```java
// Use custom fluent assertions for domain objects
assertThat(actualUserDto)
    .hasFirstName("John")
    .hasLastName("Doe")
    .hasEmployeeNumber("BB123456");

// Use direct AssertJ for primitives/collections
Assertions.assertThat(result.getLabels()).isEmpty();
Assertions.assertThat(result.getProjects()).hasSize(2);
```

## Reference
`UserServiceTests.java` - Complete implementation example