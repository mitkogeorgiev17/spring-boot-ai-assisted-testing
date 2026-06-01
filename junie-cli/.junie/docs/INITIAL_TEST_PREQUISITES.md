# Testing Setup and Patterns

This document defines the complete testing infrastructure setup, dependencies, configurations, and patterns for this Spring Boot project.

## Document Structure

### Part 1: Infrastructure Setup
Foundational setup that can be implemented without business logic knowledge:
- Dependencies and versions
- JaCoCo coverage configuration  
- Database test configuration
- Base annotations and extensions
- Basic project structure

### Part 2: Business Logic Testing Components
Components that require understanding of business logic:
- Factory patterns for domain objects
- Custom assertion helpers
- WireMock patterns for external services
- Integration test templates
- Security testing patterns

---

# PART 1: INFRASTRUCTURE SETUP

*This section contains configurations and dependencies that can be set up independently of business logic.*

## Required Dependencies

### Maven Properties
```xml
<properties>
    <java.version>17</java.version>
    <maven.test.skip>false</maven.test.skip>
    <component-test.skip>false</component-test.skip>
    <integration-test.skip>false</integration-test.skip>
    
    <!-- Testing Framework Versions -->
    <jacoco.version>0.8.10</jacoco.version>
    <org.mockito.version>5.5.0</org.mockito.version>
    <wiremock.version>2.1.2</wiremock.version>
    <awaitility.version>4.2.0</awaitility.version>
    <nl.jqno.equalsverifier.version>3.15.3</nl.jqno.equalsverifier.version>
</properties>
```

### Core Testing Dependencies
```xml
<!-- Spring Boot Test Starter (includes JUnit 5, AssertJ, Mockito, etc.) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Mockito Core for unit testing -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>${org.mockito.version}</version>
    <scope>test</scope>
</dependency>

<!-- Spring Security Test for security testing -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- WireMock for external service mocking -->
<dependency>
    <groupId>com.maciejwalkowiak.spring</groupId>
    <artifactId>wiremock-spring-boot</artifactId>
    <version>${wiremock.version}</version>
    <scope>test</scope>
</dependency>

<!-- Awaitility for async testing -->
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>${awaitility.version}</version>
    <scope>test</scope>
</dependency>

<!-- H2 Database for integration testing -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>

<!-- EqualsVerifier for equals/hashCode testing -->
<dependency>
    <groupId>nl.jqno.equalsverifier</groupId>
    <artifactId>equalsverifier</artifactId>
    <version>${nl.jqno.equalsverifier.version}</version>
    <scope>test</scope>
</dependency>
```

## JaCoCo Configuration for Test Coverage

### Maven Plugin Configuration

#### Compact Version (Recommended for SonarQube)
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${jacoco.version}</version>
    <executions>
        <!-- Prepare agent for unit tests -->
        <execution>
            <id>prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <!-- Prepare agent for integration tests -->
        <execution>
            <id>prepare-agent-integration</id>
            <goals>
                <goal>prepare-agent-integration</goal>
            </goals>
        </execution>
        <!-- Generate merged report after all tests -->
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>

<!-- Surefire Plugin for Unit Tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
</plugin>

<!-- Failsafe Plugin for Integration Tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
</plugin>
```

#### Detailed Version (If separate reports are needed)
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${jacoco.version}</version>
    <executions>
        <!-- Unit Test Coverage -->
        <execution>
            <id>before-unit-test-execution</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
            <configuration>
                <destFile>${project.build.directory}/jacoco-output/jacoco-unit-tests.exec</destFile>
                <propertyName>surefire.jacoco.args</propertyName>
            </configuration>
        </execution>
        <execution>
            <id>after-unit-test-execution</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
            <configuration>
                <dataFile>${project.build.directory}/jacoco-output/jacoco-unit-tests.exec</dataFile>
                <outputDirectory>${project.reporting.outputDirectory}/jacoco-unit-test-coverage-report</outputDirectory>
            </configuration>
        </execution>
        
        <!-- Integration Test Coverage -->
        <execution>
            <id>before-integration-test-execution</id>
            <phase>pre-integration-test</phase>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
            <configuration>
                <destFile>${project.build.directory}/jacoco-output/jacoco-integration-tests.exec</destFile>
                <propertyName>failsafe.jacoco.args</propertyName>
            </configuration>
        </execution>
        <execution>
            <id>after-integration-test-execution</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>report</goal>
            </goals>
            <configuration>
                <dataFile>${project.build.directory}/jacoco-output/jacoco-integration-tests.exec</dataFile>
                <outputDirectory>${project.reporting.outputDirectory}/jacoco-integration-test-coverage-report</outputDirectory>
            </configuration>
        </execution>
        
        <!-- Merged Coverage Report -->
        <execution>
            <id>merge-unit-and-integration</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>merge</goal>
            </goals>
            <configuration>
                <fileSets>
                    <fileSet>
                        <directory>${project.build.directory}/jacoco-output/</directory>
                        <includes>
                            <include>*.exec</include>
                        </includes>
                    </fileSet>
                </fileSets>
                <destFile>${project.build.directory}/jacoco-output/merged.exec</destFile>
            </configuration>
        </execution>
        <execution>
            <id>create-merged-report</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>report</goal>
            </goals>
            <configuration>
                <dataFile>${project.build.directory}/jacoco-output/merged.exec</dataFile>
                <outputDirectory>${project.reporting.outputDirectory}/jacoco-merged-test-coverage-report</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>

<!-- Surefire Plugin for Unit Tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>${surefire.jacoco.args}</argLine>
    </configuration>
</plugin>

<!-- Failsafe Plugin for Integration Tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <configuration>
        <argLine>${failsafe.jacoco.args}</argLine>
    </configuration>
</plugin>
```

### Coverage Report Locations

#### Compact Configuration
- **Combined Report:** `target/site/jacoco/index.html`
- **XML Report (for SonarQube):** `target/site/jacoco/jacoco.xml`
- **Execution Data:** `target/jacoco.exec`

#### Detailed Configuration  
- **Unit Tests:** `target/site/jacoco-unit-test-coverage-report/index.html`
- **Integration Tests:** `target/site/jacoco-integration-test-coverage-report/index.html`
- **Merged Report:** `target/site/jacoco-merged-test-coverage-report/index.html`

#### SonarQube Integration
For SonarQube, add these properties to your `sonar-project.properties` or Maven configuration:
```properties
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
sonar.jacoco.reportPaths=target/jacoco.exec
```

## Test Database Configuration

### Integration Test Application Properties
**File:** `src/test/resources/application-it.yml`
```yaml
spring:
  liquibase:
    enabled: false  # Disable Liquibase for tests
  datasource:
    username: sa
    password: ''
    url: jdbc:h2:mem:testdb;Mode=Oracle  # Oracle compatibility mode
    driverClassName: org.h2.Driver
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: none  # Schema managed via SQL files

# External service URLs for WireMock mocking
baseUrls:
  keycloak: https://keycloak-keycloak.apps.bgsocpclt01.ucb.lan
  email: https://apigatetst.unicreditbulbank.bg/dev/internal/emails/v1/send-external
  workingDays: https://apigatetst.unicreditbulbank.bg/dev/internal/common/v1/workingDays
  toolboxRoles: https://uniappwebdev.ucb.lan/UniServe/ToolBox.svc/json/GetAppUserRoles
```

### Database Schema Management
**Structure:**
```
src/test/resources/db/
├── schema.sql      # CREATE TABLE statements
├── data.sql        # INSERT test data
└── drop-schema.sql # Cleanup script
```

**Usage Pattern:**
```java
@WithTestData  // Loads schema + data, cleans up after test
class SomeIntegrationTest {
    // Test methods
}
```

### Custom Test Data Annotation
```java
@Inherited
@Documented
@Retention(RUNTIME)
@Target({TYPE, METHOD})
@Sql(value = {"/db/schema.sql", "/db/data.sql"}, 
     config = @SqlConfig(transactionMode = ISOLATED))
@Sql(value = "/db/drop-schema.sql", 
     executionPhase = AFTER_TEST_METHOD,
     config = @SqlConfig(transactionMode = ISOLATED))
public @interface WithTestData {
}
```

### H2 Oracle Compatibility Extension
```java
@ExtendWith(H2OracleModeSetupExtension.class)
public class SomeIntegrationTest {
    // Ensures H2 runs in Oracle compatibility mode
}
```

### Base Test Annotations

#### @IntegrationTest (Composite Annotation)
```java
@Inherited
@Documented
@Target(TYPE)
@Retention(RUNTIME)
@Transactional
@ActiveProfiles("it")
@AutoConfigureTestEntityManager
@ExtendWith(H2OracleModeSetupExtension.class)
@SpringBootTest(webEnvironment = RANDOM_PORT, properties = "spring.main.banner-mode=off")
@MockBean({JwtDecoder.class, OutlookAppointmentService.class})
public @interface IntegrationTest {
}
```

#### @WithTestData (Database Setup Annotation)
```java
@Inherited
@Documented
@Retention(RUNTIME)
@Target({TYPE, METHOD})
@Sql(value = {"/db/schema.sql", "/db/data.sql"}, 
     config = @SqlConfig(transactionMode = ISOLATED))
@Sql(value = "/db/drop-schema.sql", 
     executionPhase = AFTER_TEST_METHOD,
     config = @SqlConfig(transactionMode = ISOLATED))
public @interface WithTestData {
}
```

#### @WithHeaders (Custom Headers Annotation)
```java
@Inherited
@Documented
@Retention(RUNTIME)
@Target({METHOD, TYPE})
@ExtendWith(WithHeadersExtension.class)
public @interface WithHeaders {
    String[] headers() default {};
    boolean replace() default true;
}
```

#### @WithMockJwt (Security Testing Annotation)
```java
@Inherited
@Documented
@Retention(RUNTIME)
@Target({METHOD, TYPE})
@WithSecurityContext(factory = WithMockJwtSecurityContextFactory.class)
public @interface WithMockJwt {
    String username() default DEFAULT_USERNAME;
    String firstName() default DEFAULT_FIRST_NAME;
    String lastName() default DEFAULT_LAST_NAME;
    String email() default DEFAULT_EMAIL;
    String phone() default DEFAULT_PHONE;
    String company() default DEFAULT_COMPANY;
    String jobTitle() default DEFAULT_JOB_TITLE;
    String[] roles() default {ROLE_ADMIN, ROLE_MANAGER, ROLE_EMPLOYEE};
    TestExecutionEvent setupBefore() default TestExecutionEvent.TEST_METHOD;
}
```

---

# PART 2: BUSINESS LOGIC TESTING COMPONENTS

*This section contains patterns and components that require understanding of business logic and domain objects.*

## Factory Setup Patterns

### Factory Class Template
```java
@NoArgsConstructor(access = PRIVATE)
public class DomainObjectFactory {

    // Default value constants
    public static final Long DEFAULT_ID = 1L;
    public static final String DEFAULT_NAME = "Default Name";
    public static final String DEFAULT_EMAIL = "default@example.com";
    
    // Updated value constants  
    public static final String UPDATED_NAME = "Updated Name";
    public static final String UPDATED_EMAIL = "updated@example.com";

    // Entity factory methods
    public static DomainEntity createDefaultDomainEntity() {
        return new DomainEntity()
                .setId(DEFAULT_ID)
                .setName(DEFAULT_NAME)
                .setEmail(DEFAULT_EMAIL);
    }
    
    public static DomainEntity createUpdatedDomainEntity() {
        return createDefaultDomainEntity()
                .setName(UPDATED_NAME)
                .setEmail(UPDATED_EMAIL);
    }
    
    // DTO factory methods
    public static DomainDTO createDefaultDomainDto() {
        return new DomainDTO()
                .setId(DEFAULT_ID)
                .setName(DEFAULT_NAME)
                .setEmail(DEFAULT_EMAIL);
    }
    
    // Command factory methods
    public static SaveDomainCommand createSaveDomainCommand() {
        return new SaveDomainCommand()
                .setName(DEFAULT_NAME)
                .setEmail(DEFAULT_EMAIL);
    }
    
    public static UpdateDomainCommand createUpdateDomainCommand() {
        return new UpdateDomainCommand()
                .setName(UPDATED_NAME)
                .setEmail(UPDATED_EMAIL);
    }
    
    // Collection factory methods
    public static List<DomainDTO> createListDomainDTOResponse() {
        return List.of(createDefaultDomainDto());
    }
    
    public static Set<DomainEntity> createSetDomainEntityResponse() {
        return Set.of(createDefaultDomainEntity());
    }
}
```

### Factory Cross-References
```java
public class UserFactory {
    // Reference constants from other factories
    public static SaveUserCommand createSaveUserCommand() {
        return new SaveUserCommand()
                .setLabels(List.of(LabelFactory.DEFAULT_LABEL_ID))
                .setProjectId(ProjectFactory.DEFAULT_PROJECT_ID);
    }
}
```

### Factory Usage in Tests
```java
@Test
void shouldCreateUser() {
    // Given
    var user = UserFactory.createDefaultUser();
    var command = UserFactory.createSaveUserCommand()
        .setEmail("custom@email.com");  // Modify as needed
    
    // Test execution...
}
```

## WireMock Server Setup Patterns

### Basic WireMock Configuration
```java
@IntegrationTest
@EnableWireMock({
    @ConfigureWireMock(name = "toolbox-server", property = "baseUrls.toolboxRoles"),
    @ConfigureWireMock(name = "emails-server", property = "email.uri"),
    @ConfigureWireMock(name = "days-server", property = "baseUrls.workingDays")
})
class SomeIntegrationTest {
    
    @InjectWireMock("toolbox-server")
    private WireMockServer toolboxServer;
    
    @InjectWireMock("emails-server")
    private WireMockServer emailsServer;
    
    @InjectWireMock("days-server")
    private WireMockServer daysServer;
}
```

### WireMock Helper Patterns
**Create helper classes for each external service:**

```java
@NoArgsConstructor(access = PRIVATE)
public class MockExternalWebServer_Toolbox {
    
    public static MockExternalWebServer_Toolbox givenToolboxServer(WireMockServer server) {
        return new MockExternalWebServer_Toolbox(server);
    }
    
    private final WireMockServer server;
    
    public MockExternalWebServer_Toolbox willGetEmployeeRole() {
        server.stubFor(post(urlEqualTo("/"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("[\"tis-employee\"]")));
        return this;
    }
    
    public MockExternalWebServer_Toolbox willGetManagerAdminRoles() {
        server.stubFor(post(urlEqualTo("/"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("[\"tis-admin\", \"tis-manager\", \"tis-employee\"]")));
        return this;
    }
    
    public MockExternalWebServer_Toolbox willGetAdminRole() {
        server.stubFor(post(urlEqualTo("/"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("[\"tis-admin\", \"tis-employee\"]")));
        return this;
    }
}
```

### WireMock Usage in Tests
```java
@Test
void shouldMockExternalServices() {
    // Given
    givenToolboxServer(toolboxServer).willGetEmployeeRole();
    givenEmailServer(emailsServer).willSendEmail();
    givenDaysServer(daysServer).willGetDays();
    
    // When & Then
    // Test execution with mocked external services
}
```

## Security Testing Infrastructure

### JWT Factory Base Pattern

#### JWT Factory Setup
```java
@NoArgsConstructor(access = PRIVATE)
public class JwtFactory {
    
    public static final String DEFAULT_USERNAME = "BB555555";
    public static final String DEFAULT_FIRST_NAME = "John";
    public static final String DEFAULT_LAST_NAME = "Smith";
    public static final String DEFAULT_EMAIL = "john.smith@email.com";
    public static final String DEFAULT_PHONE = "+359 88888880";
    public static final String DEFAULT_COMPANY = "Company Ltd.";
    public static final String DEFAULT_JOB_TITLE = "Vendor";
    
    public static final String ROLE_EMPLOYEE = "tis-employee";
    public static final String ROLE_MANAGER = "tis-manager";
    public static final String ROLE_ADMIN = "tis-admin";
    
    public static Jwt createDefaultJwt() {
        return createJwt(DEFAULT_USERNAME, DEFAULT_FIRST_NAME, DEFAULT_LAST_NAME, 
                        DEFAULT_EMAIL, DEFAULT_PHONE, DEFAULT_COMPANY, 
                        DEFAULT_JOB_TITLE, ROLE_EMPLOYEE);
    }
    
    public static Jwt createDefaultJwt(String username, String... roles) {
        var realmAccessClaim = Map.of("roles", List.of(roles));
        return Jwt.withTokenValue("token")
                .issuedAt(Instant.now())
                .expiresAt(Instant.MAX)
                .header("test", "test-value")
                .claim("preferred_username", username)
                .claim("realm_access", realmAccessClaim)
                .build();
    }
    
    public static Jwt createJwt(String username, String firstName, String lastName,
                               String email, String phone, String company, 
                               String jobTitle, String... roles) {
        var realmAccessClaim = Map.of("roles", List.of(roles));
        return Jwt.withTokenValue("token")
                .issuedAt(Instant.now())
                .expiresAt(Instant.MAX)
                .header("test", "test-value")
                .claim("preferred_username", username)
                .claim("realm_access", realmAccessClaim)
                .claim("given_name", firstName)
                .claim("family_name", lastName)
                .claim("email", email)
                .claim("phoneNumber", phone)
                .claim("company", company)
                .claim("job-title", jobTitle)
                .build();
    }
}
```

#### JWT Mocking Strategies

**1. Annotation-Based (@WithMockJwt)**
```java
@WithMockJwt  // Uses default JWT
@Test
void shouldReturnDataWhenUserIsAuthenticated() {
    // Test with default user (BB555555, admin/manager/employee roles)
}

@WithMockJwt(username = "BB666666", roles = JwtFactory.ROLE_EMPLOYEE)
@Test 
void shouldDenyAccessForEmployeeRole() {
    // Test with custom user and specific roles
}

@WithMockJwt(
    username = "custom-user",
    firstName = "Jane", 
    lastName = "Doe",
    email = "jane.doe@test.com",
    roles = {JwtFactory.ROLE_MANAGER, JwtFactory.ROLE_EMPLOYEE}
)
@Test
void shouldWorkWithCustomJwtClaims() {
    // Test with fully customized JWT
}
```

**2. Manual JWT Stubbing**
```java
@Test
void shouldHandleDifferentUsers() {
    // Given
    var jwt = JwtFactory.createDefaultJwt("BB777777", JwtFactory.ROLE_ADMIN);
    given(jwtDecoder.decode(any())).willReturn(jwt);
    
    // When & Then
    // Test execution with custom JWT
}
```

**3. Parameterized JWT Tests**
```java
@ParameterizedTest
@MethodSource("getJwtSource")
void shouldBehaveDifferentlyBasedOnUserRole(Jwt givenJwt) {
    // Given
    given(jwtDecoder.decode(any())).willReturn(givenJwt);
    
    // When & Then
    // Test behavior varies based on JWT roles
}

static Stream<Jwt> getJwtSource() {
    return Stream.of(
        JwtFactory.createDefaultJwt("user1", JwtFactory.ROLE_EMPLOYEE),
        JwtFactory.createDefaultJwt("user2", JwtFactory.ROLE_MANAGER),
        JwtFactory.createDefaultJwt("user3", JwtFactory.ROLE_ADMIN)
    );
}
```

### Custom Header Patterns

#### Header Value Definitions
```java
@RequiredArgsConstructor
public enum HttpHeaderValue implements HeaderEntry {
    REQUEST_ID("X-Input-Request-Id", "TIS-6774609896538112"),
    REQUEST_TIMESTAMP("X-Input-Timestamp", "901331921"),
    AUTHORIZATION("Authorization", "Bearer <default-token>"),
    CUSTOM_CLIENT_ID("X-Client-Id", "test-client-123"),
    CUSTOM_CORRELATION_ID("X-Correlation-Id", "correlation-456");
    
    private final String headerName;
    private final String defaultValue;
    
    @Override
    public String getName() { return headerName; }
    
    @Override
    public String getValue() { return defaultValue; }
    
    public HeaderEntry withValue(String value) {
        return new HeaderEntry() {
            @Override
            public String getName() { return headerName; }
            
            @Override
            public String getValue() { return value; }
        };
    }
}
```

#### @WithHeaders Annotation Pattern

**Create the Annotation:**
```java
package com.unicredit.tis.test.fixtures.http;

import org.junit.jupiter.api.extension.ExtendWith;

import java.lang.annotation.Documented;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.ElementType.TYPE;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

@Inherited
@Documented
@Retention(RUNTIME)
@Target({METHOD, TYPE})
@ExtendWith(WithHeadersExtension.class)
public @interface WithHeaders {
    
    /**
     * Headers to add to all requests in key=value format
     * Example: {"X-Client-Id=test-client", "X-Correlation-Id=test-correlation"}
     */
    String[] headers() default {};
    
    /**
     * Whether to replace existing headers with same name (true) or add additional (false)
     */
    boolean replace() default true;
}
```

**Create the Extension:**
```java
package com.unicredit.tis.test.fixtures.http;

import org.junit.jupiter.api.extension.BeforeEachCallback;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.springframework.test.web.reactive.server.WebTestClient;

import java.util.HashMap;
import java.util.Map;

public class WithHeadersExtension implements BeforeEachCallback {
    
    private static final String HEADERS_KEY = "test.headers";
    
    @Override
    public void beforeEach(ExtensionContext context) {
        WithHeaders annotation = findAnnotation(context);
        if (annotation != null) {
            Map<String, String> headers = parseHeaders(annotation.headers());
            // Store headers in test context for WebClientVerify to use
            getStore(context).put(HEADERS_KEY, new HeaderConfig(headers, annotation.replace()));
        }
    }
    
    private WithHeaders findAnnotation(ExtensionContext context) {
        // Check method level first
        WithHeaders methodAnnotation = context.getRequiredTestMethod()
                .getAnnotation(WithHeaders.class);
        if (methodAnnotation != null) {
            return methodAnnotation;
        }
        
        // Check class level
        return context.getRequiredTestClass()
                .getAnnotation(WithHeaders.class);
    }
    
    private Map<String, String> parseHeaders(String[] headerStrings) {
        Map<String, String> headers = new HashMap<>();
        for (String header : headerStrings) {
            String[] parts = header.split("=", 2);
            if (parts.length == 2) {
                headers.put(parts[0].trim(), parts[1].trim());
            }
        }
        return headers;
    }
    
    private ExtensionContext.Store getStore(ExtensionContext context) {
        return context.getStore(ExtensionContext.Namespace.create(getClass(), context.getRequiredTestMethod()));
    }
    
    public static HeaderConfig getHeaderConfig(ExtensionContext context) {
        return context.getStore(ExtensionContext.Namespace.create(WithHeadersExtension.class, context.getRequiredTestMethod()))
                .get(HEADERS_KEY, HeaderConfig.class);
    }
    
    public record HeaderConfig(Map<String, String> headers, boolean replace) {}
}
```

**Integrate with WebClientVerify:**
```java
public class WebClientVerifyBuilder {
    
    // Add method to apply headers from @WithHeaders annotation
    public WebClientVerifyBuilder withTestHeaders(ExtensionContext context) {
        WithHeadersExtension.HeaderConfig config = WithHeadersExtension.getHeaderConfig(context);
        if (config != null) {
            config.headers().forEach((name, value) -> {
                HeaderEntry header = new HeaderEntry() {
                    @Override
                    public String getName() { return name; }
                    @Override
                    public String getValue() { return value; }
                };
                
                if (config.replace()) {
                    headerValues.removeIf(h -> h.getName().equals(name));
                }
                headerValues.add(header);
            });
        }
        return this;
    }
}
```

**Enhanced Verifications Helper:**
```java
@NoArgsConstructor(access = PRIVATE)
public class Verifications {
    
    public static WebClientVerifyBuilder verifyThat(WebTestClient webTestClient) {
        return WebClientVerify.builder()
                .withWebTestClient(webTestClient);
    }
    
    // New method that automatically applies @WithHeaders
    public static WebClientVerifyBuilder verifyThat(WebTestClient webTestClient, ExtensionContext context) {
        return WebClientVerify.builder()
                .withWebTestClient(webTestClient)
                .withTestHeaders(context);
    }
}
```

#### Header Usage Patterns

**Manual Header Addition:**
```java
@Test
void shouldHandleCustomHeaders() {
    // Given
    var customHeader = HttpHeaderValue.CUSTOM_CLIENT_ID.withValue("special-client");
    
    // When & Then
    verifyThat(webTestClient)
        .withHttpMethod(HttpMethod.GET)
        .withPath(USER_OPERATIONS)
        .headerValue(customHeader)  // Add custom header
        .headerValue(HttpHeaderValue.REQUEST_ID.withValue("custom-request-id"))
        .when()
        .authenticated()
        .thenShouldReturnResponse(HttpStatus.OK);
}
```

**Automatic Header Addition with @WithHeaders:**
```java
@Test
@WithHeaders(headers = {
    "X-Client-Id=integration-test-client",
    "X-Correlation-Id=test-correlation-123",
    "X-Custom-Header=custom-value"
})
void shouldAutomaticallyAddHeaders(ExtensionContext context) {
    // Headers automatically added to all requests in this test
    verifyThat(webTestClient, context)  // Pass context to apply headers
        .withHttpMethod(HttpMethod.GET)
        .withPath(USER_OPERATIONS)
        .when()
        .authenticated()
        .thenShouldReturnResponse(HttpStatus.OK);
}

@WithHeaders(headers = {
    "X-Client-Id=class-level-client",
    "X-Environment=test"
})
class SomeIntegrationTest {
    // Headers applied to all test methods in this class
    
    @Test
    void shouldInheritClassLevelHeaders(ExtensionContext context) {
        verifyThat(webTestClient, context)
            .withHttpMethod(HttpMethod.POST)
            .withPath(USER_OPERATIONS)
            .withRequestBody(CREATE_USER_REQ)
            .when()
            .authenticated()
            .thenShouldReturnResponse(HttpStatus.CREATED);
    }
    
    @Test
    @WithHeaders(headers = {"X-Override-Header=method-specific"})
    void shouldCombineClassAndMethodHeaders(ExtensionContext context) {
        // Method-level headers override class-level headers with same name
        verifyThat(webTestClient, context)
            .withHttpMethod(HttpMethod.GET)
            .withPath(USER_OPERATIONS)
            .when()
            .authenticated()
            .thenShouldReturnResponse(HttpStatus.OK);
    }
}
```

**Advanced Header Configuration:**
```java
@WithHeaders(
    headers = {
        "X-Client-Id=test-client",
        "X-Request-Source=integration-test"
    },
    replace = false  // Add headers without replacing existing ones
)
@Test
void shouldAddWithoutReplacing(ExtensionContext context) {
    // Custom headers added alongside default REQUEST_ID, REQUEST_TIMESTAMP
    verifyThat(webTestClient, context)
        .withHttpMethod(HttpMethod.GET)
        .withPath(USER_OPERATIONS)
        .when()
        .authenticated()
        .thenShouldReturnResponse(HttpStatus.OK);
}
```

#### Automatic Header Injection
```java
public class WebClientVerify {
    
    public WebClientVerify() {
        // Automatically add required headers to every request
        this.headerValues.addAll(List.of(
            HttpHeaderValue.REQUEST_ID,
            HttpHeaderValue.REQUEST_TIMESTAMP
        ));
    }
    
    public WebClientVerify authenticated() {
        return headerValue(HttpHeaderValue.AUTHORIZATION);
    }
    
    public WebClientVerify headerValue(@NonNull HeaderEntry headerEntry) {
        headerValues.remove(headerEntry);  // Remove existing
        headerValues.add(headerEntry);     // Add new/updated
        return this;
    }
}
```

## Complete Integration Test Templates

### Full Integration Test Example
```java
@IntegrationTest
@EnableWireMock({
    @ConfigureWireMock(name = "toolbox-server", property = "baseUrls.toolboxRoles"),
    @ConfigureWireMock(name = "emails-server", property = "email.uri")
})
@WithTestData
@WithHeaders(headers = {
    "X-Client-Id=integration-test-suite",
    "X-Environment=test"
})
class CreateUserUseCaseIT {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private WebTestClient webTestClient;
    
    @MockBean
    private JwtDecoder jwtDecoder;
    
    @InjectWireMock("toolbox-server")
    private WireMockServer toolboxServer;
    
    @InjectWireMock("emails-server")
    private WireMockServer emailsServer;
    
    @Test
    @WithMockJwt(username = "BB555555", roles = JwtFactory.ROLE_ADMIN)
    void shouldCreateUserWhenUserIsAuthenticated(ExtensionContext context) {
        // Given
        givenToolboxServer(toolboxServer).willGetManagerAdminRoles();
        givenEmailServer(emailsServer).willSendEmail();
        
        // When & Then
        verifyThat(webTestClient, context)  // Automatic header injection
            .withHttpMethod(HttpMethod.POST)
            .withPath(USER_OPERATIONS)
            .withRequestBody(CREATE_USER_REQ)
            .when()
            .authenticated()
            .thenShouldReturnResponse(HttpStatus.CREATED);
            
        // Database verification
        assertThat(userRepository.existsByEmployeeNumber("BB555555")).isTrue();
    }
    
    @Test
    @WithMockJwt(username = "BB666666", roles = JwtFactory.ROLE_MANAGER)
    @WithHeaders(headers = {
        "X-Special-Request=manager-operation",
        "X-Priority=high"
    })
    void shouldHandleManagerSpecificHeaders(ExtensionContext context) {
        // Given
        givenToolboxServer(toolboxServer).willGetManagerAdminRoles();
        
        // When & Then - combines class-level and method-level headers
        verifyThat(webTestClient, context)
            .withHttpMethod(HttpMethod.GET)
            .withPath(USER_OPERATIONS)
            .when()
            .authenticated()
            .thenShouldReturnResponse(HttpStatus.OK);
    }
    
    @ParameterizedTest
    @MethodSource("getUnauthorizedJwtSource")
    void shouldDenyAccessForUnauthorizedUsers(Jwt givenJwt, ExtensionContext context) {
        // Given
        given(jwtDecoder.decode(any())).willReturn(givenJwt);
        givenToolboxServer(toolboxServer).willGetEmployeeRole();
        
        // When & Then
        verifyThat(webTestClient, context)
            .withHttpMethod(HttpMethod.POST)
            .withPath(USER_OPERATIONS)
            .withRequestBody(CREATE_USER_REQ)
            .when()
            .authenticated()
            .thenShouldReturnResponse(HttpStatus.FORBIDDEN);
    }
    
    static Stream<Jwt> getUnauthorizedJwtSource() {
        return Stream.of(
            JwtFactory.createDefaultJwt("employee1", JwtFactory.ROLE_EMPLOYEE),
            JwtFactory.createDefaultJwt("manager1", JwtFactory.ROLE_MANAGER)
        );
    }
}
```

## Test Execution Commands

### Maven Commands
```bash
# Run unit tests only
mvn test

# Run integration tests only  
mvn failsafe:integration-test

# Run all tests with coverage
mvn clean verify

# Generate coverage reports
mvn jacoco:report

# Skip specific test types
mvn test -Dintegration-test.skip=true
mvn test -Dcomponent-test.skip=true
```

### Generated Reports

#### Compact Configuration
- **Coverage Report:** `target/site/jacoco/`
- **Test Results:** `target/surefire-reports/` and `target/failsafe-reports/`

#### Detailed Configuration
- **Unit Test Coverage:** `target/site/jacoco-unit-test-coverage-report/`
- **Integration Test Coverage:** `target/site/jacoco-integration-test-coverage-report/`
- **Merged Coverage:** `target/site/jacoco-merged-test-coverage-report/`
- **Test Results:** `target/surefire-reports/` and `target/failsafe-reports/`

---

## Infrastructure Setup Checklist

### Phase 1: Basic Infrastructure (Required First)

✅ **Dependencies Added:**
- Spring Boot Test Starter
- Mockito Core
- WireMock Spring Boot
- H2 Database
- Spring Security Test
- Awaitility

✅ **Maven Configuration:**
- JaCoCo plugin configured (compact or detailed)
- Surefire plugin configured
- Failsafe plugin configured
- Properties for test execution

✅ **Test Application Configuration:**
- `application-it.yml` created with H2 configuration
- Database test profiles configured
- External service URLs defined

✅ **Database Test Structure:**
- `src/test/resources/db/` directory created
- `schema.sql` template prepared
- `data.sql` template prepared  
- `drop-schema.sql` template prepared

✅ **Base Annotations Implemented:**
- `@IntegrationTest` composite annotation
- `@WithTestData` database annotation
- `@WithHeaders` custom headers annotation
- `@WithMockJwt` security annotation

### Phase 2: Business Logic Components (After Phase 1)

✅ **Factory Infrastructure:**
- Factory base pattern established
- Constants naming conventions
- Cross-factory dependency patterns

✅ **Assertion Infrastructure:**
- Custom assertion base classes
- ModelAssertions entry point
- Fluent assertion patterns

✅ **WireMock Infrastructure:**
- Helper class patterns
- External service mocking utilities
- Request/response stubbing patterns

✅ **Security Testing Infrastructure:**
- JWT factory for token creation
- Parameterized authentication patterns
- Role-based testing scenarios

## Key Patterns Summary

1. **@IntegrationTest** - Composite annotation for full Spring Boot context
2. **@WithTestData** - Database schema/data management
3. **@WithMockJwt** - Declarative JWT authentication
4. **@WithHeaders** - Automatic custom header injection
5. **WireMock Helpers** - Fluent external service mocking
6. **Custom Assertions** - Domain-specific fluent assertions
7. **Factory Pattern** - Consistent test data creation
8. **Parameterized Security** - Testing multiple authentication scenarios
9. **JaCoCo Integration** - Comprehensive coverage reporting

## Ready-to-Code Indicators

Your project is ready for automated test generation when:
- ✅ All Phase 1 infrastructure is complete and working
- ✅ Test execution commands work (`mvn test`, `mvn verify`)
- ✅ Coverage reports generate successfully
- ✅ Sample test using annotations compiles and runs
- ✅ Database test setup/cleanup cycle works
- ✅ WireMock configuration loads properly