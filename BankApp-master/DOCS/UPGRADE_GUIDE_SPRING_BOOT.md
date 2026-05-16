# Spring Boot 3.2.0 Upgrade Guide - Complete Migration

**Version:** 2.0.0  
**Date:** 2026-05-16  
**From:** Spring Boot 2.1.4 + Java 8  
**To:** Spring Boot 3.2.0 + Java 17  

---

## Table of Contents

1. [Pre-Upgrade Checklist](#pre-upgrade-checklist)
2. [Step-by-Step Upgrade Instructions](#step-by-step-upgrade-instructions)
3. [Common Issues & Solutions](#common-issues--solutions)
4. [Testing & Verification](#testing--verification)
5. [Performance Tuning](#performance-tuning)
6. [Rollback Procedure](#rollback-procedure)

---

## Pre-Upgrade Checklist

### Environment Requirements

```bash
# Check Java version (must be 11+, recommend 17+)
java -version
# Expected: openjdk version "17.x.x" or higher

# Check Maven version (must be 3.8.1+)
mvn -version
# Expected: Maven 3.8.1 or higher

# Check Git status (clean working directory)
git status
# Expected: working tree clean

# Check current branch
git branch -v
# Expected: on a feature or develop branch, not main
```

### Pre-Upgrade Backup

```bash
# Create backup of current state
git checkout -b backup/before-spring-boot-upgrade
git push origin backup/before-spring-boot-upgrade

# Return to main upgrade branch
git checkout feature/issue-2-spring-boot-upgrade
```

### Code Review Checklist

- [ ] All changes are committed
- [ ] No uncommitted changes
- [ ] All tests passing on current version
- [ ] Code review approved
- [ ] Backup created
- [ ] Team notified

---

## Step-by-Step Upgrade Instructions

### Step 1: Update Java Version

#### 1.1 Update Local Java Installation

**macOS (using Homebrew):**
```bash
brew uninstall java
brew install java@17
```

**Windows (using Chocolatey):**
```bash
choco uninstall jdk17
choco install openjdk17
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get update
sudo apt-get install openjdk-17-jdk
sudo update-alternatives --config java
```

#### 1.2 Configure IDE

**IntelliJ IDEA:**
1. File → Project Structure → Project
2. Set Project SDK to Java 17
3. Set Language Level to 17
4. Apply and OK

**Eclipse:**
1. Window → Preferences → Java → Installed JREs
2. Add new JRE pointing to Java 17 installation
3. Set as default

**Visual Studio Code:**
1. Install Extension Pack for Java
2. Configure JAVA_HOME: `export JAVA_HOME=/path/to/java/17`
3. Run command palette: "Java: Configure Java Runtime"

#### 1.3 Verify Installation

```bash
java -version
# Output should show: openjdk version "17.x.x"

javac -version
# Output should show: javac 17.x.x

# Verify Maven uses Java 17
mvn -version
# Output should show: Java version: 17.x.x
```

### Step 2: Update pom.xml

#### 2.1 Update Spring Boot Version

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <!-- OLD: <version>2.1.4.RELEASE</version> -->
    <version>3.2.0</version>
    <relativePath/>
</parent>
```

#### 2.2 Update Java Version

```xml
<properties>
    <!-- OLD: <java.version>1.8</java.version> -->
    <java.version>17</java.version>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
</properties>
```

#### 2.3 Replace Swagger with SpringDoc OpenAPI

```xml
<!-- REMOVE these dependencies -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>

<!-- ADD these dependencies -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.0.4</version>
</dependency>
```

#### 2.4 Add New Dependencies

```xml
<!-- JWT Support -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>

<!-- PostgreSQL Driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.1</version>
    <scope>runtime</scope>
</dependency>

<!-- Flyway Database Migrations -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>9.22.3</version>
</dependency>
```

#### 2.5 Update Build Plugins

```xml
<build>
    <plugins>
        <!-- Add explicit compiler plugin configuration -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>17</source>
                <target>17</target>
                <release>17</release>
            </configuration>
        </plugin>

        <!-- Add JaCoCo for code coverage -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.10</version>
        </plugin>
    </plugins>
</build>
```

### Step 3: Create/Update Configuration Files

#### 3.1 Create application.yml

Create file: `src/main/resources/application.yml`

```yaml
spring:
  application:
    name: bank-app
  jpa:
    hibernate:
      ddl-auto: validate
  datasource:
    url: jdbc:h2:mem:testdb
    hikari:
      maximum-pool-size: 10

server:
  port: 8989
  servlet:
    context-path: /bank-api

logging:
  level:
    root: INFO
    com.coding.exercise: DEBUG

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

#### 3.2 Update/Create SecurityConfig.java

Replace deprecated WebSecurityConfigurerAdapter:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/bank-api/swagger-ui/**").permitAll()
                .requestMatchers("/bank-api/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(basic -> {})
            .build();
        return http.build();
    }
}
```

#### 3.3 Create OpenApiConfig.java

```java
@Configuration
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Banking Application API")
                .version("2.0.0")
            );
    }
}
```

### Step 4: Update Dependencies

```bash
# Clean previous builds
mvn clean

# Update dependencies
mvn dependency:tree

# Check for dependency conflicts
mvn dependency:analyze

# Install dependencies
mvn install -DskipTests
```

### Step 5: Fix Compilation Issues

#### 5.1 Identify Issues

```bash
mvn clean compile
```

#### 5.2 Common Issues & Fixes

**Issue: `cannot find symbol` for Swagger classes**
```java
// OLD (Swagger 2.x)
import springfox.documentation.swagger2.annotations.EnableSwagger2;

// NEW (SpringDoc)
// No annotation needed - auto-configured
```

**Issue: WebSecurityConfigurerAdapter not found**
```java
// OLD (Spring Boot 2.x)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception { }
}

// NEW (Spring Boot 3.x)
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http.build();
    }
}
```

**Issue: `No bean of type AuthenticationManager`**
```java
// OLD
@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration config) {
    return config.getAuthenticationManager();
}

// NEW (Auto-configured in Spring Boot 3.x)
// No need to create bean - use constructor injection
@Component
public class AuthService {
    private final AuthenticationManager authenticationManager;
    
    public AuthService(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }
}
```

### Step 6: Run Tests

```bash
# Run all unit tests
mvn test

# Run with coverage
mvn clean test jacoco:report

# Run integration tests
mvn verify

# Run specific test class
mvn test -Dtest=CustomerServiceTest
```

### Step 7: Start Application

```bash
# Start with Maven
mvn spring-boot:run

# Or start with Java directly
mvn clean package
java -jar target/bank-app-2.0.0.jar
```

### Step 8: Verify API Endpoints

```bash
# Test health endpoint
curl http://localhost:8989/bank-api/actuator/health

# Test customer endpoint
curl -u sa: http://localhost:8989/bank-api/customers

# Test API documentation
curl http://localhost:8989/bank-api/v3/api-docs | jq .

# Open Swagger UI in browser
open http://localhost:8989/bank-api/swagger-ui.html
```

---

## Common Issues & Solutions

### Issue 1: Java 8 Compatibility

**Symptom:** `ERROR] COMPILATION ERROR`

**Cause:** Code uses Java 8 features incompatible with Spring Boot 3.x

**Solution:**
```bash
# Verify Java version
javac -version

# Update pom.xml java.version to 17
# Clean and rebuild
mvn clean compile
```

### Issue 2: Swagger Endpoints Not Found

**Symptom:** 404 errors when accessing Swagger UI

**Cause:** Swagger 2.x endpoints don't exist in SpringDoc

**Solution:**
```bash
# OLD endpoints (no longer valid)
# /v2/api-docs
# /swagger-ui.html (redirects)

# NEW endpoints
# /v3/api-docs       ← Use this
# /swagger-ui/index.html

# Or just use the context path aware URLs
# /bank-api/v3/api-docs
# /bank-api/swagger-ui/index.html
```

### Issue 3: SecurityFilterChain Bean Not Found

**Symptom:** `NoSuchBeanDefinitionException: No qualifying bean of type 'SecurityFilterChain'`

**Cause:** Missing SecurityConfig bean definition

**Solution:**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean  // ← Must have @Bean annotation
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(authorize -> authorize.anyRequest().authenticated());
        return http.build();
    }
}
```

### Issue 4: Flyway Migration Error

**Symptom:** `org.flywaydb.core.api.FlywayException`

**Cause:** Flyway tries to create migration table but fails

**Solution:**
```yaml
spring:
  flyway:
    enabled: true
    baselineOnMigrate: true
```

### Issue 5: PostgreSQL Connection Fails

**Symptom:** `Unable to connect to PostgreSQL database`

**Cause:** PostgreSQL driver not configured or server not running

**Solution:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/bankdb
    username: postgres
    password: password
    driver-class-name: org.postgresql.Driver
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
```

---

## Testing & Verification

### Unit Test Verification

```bash
mvn clean test

# Expected output:
# [INFO] Tests run: 45, Failures: 0, Errors: 0, Skipped: 0
```

### Integration Test Verification

```bash
mvn verify

# Expected output:
# [INFO] Integration tests: PASSED
```

### Code Coverage Verification

```bash
mvn clean test jacoco:report

# Open report
open target/site/jacoco/index.html

# Expected: >80% coverage
```

### API Endpoint Verification

| Endpoint | Method | Status | Response |
|----------|--------|--------|----------|
| /bank-api/customers | GET | 200 | JSON array |
| /bank-api/customers | POST | 201 | Created customer |
| /bank-api/actuator/health | GET | 200 | {status: UP} |
| /bank-api/v3/api-docs | GET | 200 | OpenAPI spec |
| /bank-api/swagger-ui/index.html | GET | 200 | HTML page |
| /bank-api/h2-console | GET | 200 | H2 console |

### Performance Verification

```bash
# Check startup time
time mvn spring-boot:run

# Expected: < 15 seconds

# Check memory usage
jps -l -m
# Expected: < 500MB for basic startup
```

---

## Performance Tuning

### Optimized Configuration

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 20000
      idle-timeout: 300000
      max-lifetime: 1200000
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 20
          fetch_size: 50
        order_inserts: true
        order_updates: true

server:
  tomcat:
    threads:
      max: 200
      min-spare: 10
```

### JVM Tuning

```bash
# Run with optimized JVM settings
java -Xms256m -Xmx512m \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -jar target/bank-app-2.0.0.jar
```

---

## Rollback Procedure

### If Major Issues Occur

```bash
# Stash current changes
git stash

# Switch to backup branch
git checkout backup/before-spring-boot-upgrade

# Restart application
mvn spring-boot:run

# Investigate issues
# Fix issues
# Create new branch from main
git checkout -b feature/spring-boot-upgrade-v2 main
```

### Database Rollback (if needed)

```bash
# If using Flyway migrations
# Downgrade scripts are in src/main/resources/db/migration/
# Flyway will detect and prevent accidental downgrades

# Manual rollback steps:
1. Stop application
2. Backup current database
3. Run previous version
4. Verify data integrity
```

---

## Success Criteria Checklist

- [ ] Java 17 installed and verified
- [ ] pom.xml updated to Spring Boot 3.2.0
- [ ] All dependencies updated
- [ ] Configuration files created/updated
- [ ] No compilation errors
- [ ] All unit tests passing
- [ ] All integration tests passing
- [ ] Code coverage > 80%
- [ ] Application starts successfully
- [ ] All API endpoints responding
- [ ] Swagger UI accessible at new path
- [ ] No security warnings
- [ ] Performance acceptable (startup < 15s)
- [ ] Documentation updated
- [ ] Team notified and approved

---

## Post-Upgrade Tasks

### 1. Update Documentation

- [ ] Update README.md with new Java/Spring Boot versions
- [ ] Update API documentation links
- [ ] Update development setup guide
- [ ] Update deployment guide

### 2. Update CI/CD Pipelines

- [ ] Update GitHub Actions to use Java 17
- [ ] Update build configurations
- [ ] Test full CI/CD pipeline

### 3. Monitor Production

- [ ] Monitor logs for errors
- [ ] Monitor performance metrics
- [ ] Monitor memory usage
- [ ] Monitor response times
- [ ] Be ready to rollback if issues arise

### 4. Team Communication

- [ ] Notify team of successful upgrade
- [ ] Schedule migration training
- [ ] Document lessons learned
- [ ] Update team wiki/documentation

---

## Support & Resources

### Official Documentation
- Spring Boot 3.2.0: https://spring.io/projects/spring-boot
- Java 17: https://www.oracle.com/java/technologies/javase/jdk17-relnotes.html
- SpringDoc OpenAPI: https://springdoc.org/

### Helpful Links
- Migration Guide: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide
- Spring Framework 6: https://spring.io/blog/2022/11/16/spring-framework-6-0-goes-ga
- OpenAPI 3.0 Spec: https://spec.openapis.org/oas/v3.0.3

### Getting Help
- GitHub Issues: https://github.com/ajju853/banking-application-/issues
- Stack Overflow: Tag `spring-boot` and `java-17`
- Spring Community: https://spring.io/community

---

## Version History

| Version | Date | Status | Notes |
|---------|------|--------|-------|
| 2.0.0 | 2026-05-16 | COMPLETED | Spring Boot 3.2.0, Java 17 |
| 1.0.0 | 2025-08-04 | ARCHIVED | Spring Boot 2.1.4, Java 8 |

---

**Upgrade Completed Successfully! 🎉**

**Next Steps:**
1. ✅ Issue #2 Complete: Spring Boot Upgrade
2. 🚀 Issue #3: PostgreSQL Migration  
3. 📝 Issue #4: Testing Framework
4. 🔐 Issue #5: JWT Authentication
