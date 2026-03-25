# Project Dependencies List
> A comprehensive cheat-sheet of all Maven dependencies used across the microservices ecosystem.

---

## 1. Global & Parent POM (`password-manager-microservices`)
**Spring Boot Version:** `3.2.3`
**Spring Cloud Version:** `2023.0.0`
**Java Version:** `21`

### Shared Libraries (Injected into almost all modules)
- `org.springframework.boot:spring-boot-starter-web` (REST APIs & Tomcat)
- `org.springframework.boot:spring-boot-starter-data-jpa` (Hibernate & Database)
- `org.springframework.boot:spring-boot-starter-validation` (Bean Validation)
- `org.springframework.boot:spring-boot-starter-aop` (Aspect-Oriented Logs)
- `org.springframework.cloud:spring-cloud-starter-netflix-eureka-client` (Service Registration)
- `org.springframework.cloud:spring-cloud-starter-config` (Central Config Loading)
- `org.springframework.cloud:spring-cloud-starter-openfeign` (Internal HTTP Clients)
- `com.mysql:mysql-connector-j` (JDBC Driver)
- `org.projectlombok:lombok` (v1.18.30 - Code generation)
- `org.mapstruct:mapstruct` (v1.5.5.Final - Object mapping)
- `org.springdoc:springdoc-openapi-starter-webmvc-ui` (v2.5.0 - Swagger UI)

---

## 2. Security & Auth (`user-service`, `vault-service`, `api-gateway`)
- `org.springframework.boot:spring-boot-starter-security` (Spring Security Core)
- `io.jsonwebtoken:jjwt-api` (v0.12.5 - JWT Interface)
- `io.jsonwebtoken:jjwt-impl` (v0.12.5 - JWT Cryptography runtime)
- `io.jsonwebtoken:jjwt-jackson` (v0.12.5 - JWT JSON parsing runtime)
- `dev.samstevens.totp:totp` (v1.7.1 - 2FA / Google Authenticator)

---

## 3. API Gateway (`api-gateway`)
- `org.springframework.cloud:spring-cloud-starter-gateway` (WebFlux Gateway Core)
- `io.jsonwebtoken:jjwt-api` (JWT Validation at the Edge)
- `io.jsonwebtoken:jjwt-impl`
- `io.jsonwebtoken:jjwt-jackson`
- `com.google.guava:guava` (v33.0.0-jre - Rate Limiter InMemory Cache)

---

## 4. Artificial Intelligence (`ai-service`)
- `org.springframework.boot:spring-boot-starter-webflux` (Reactive Web for streaming responses)
- `com.squareup.okhttp3:okhttp` (v4.12.0 - Fast HTTP client for external LLMs)
- `com.fasterxml.jackson.core:jackson-databind` (Manual JSON passing)
- `org.springframework.boot:spring-boot-starter-actuator` (Service health metrics)
- `com.google.guava:guava` (Caching & string tools)

---

## 5. Notifications & Utilities (`user-service`, `notification-service`)
- `org.springframework.boot:spring-boot-starter-mail` (JavaMailSender / SMTP)
- `org.apache.commons:commons-lang3` (Random string generation / Null-safe utils)
- `com.google.guava:guava` (v33.0.0-jre)

---

## 6. Infrastructure Core (`eureka-server`, `config-server`)
- `org.springframework.cloud:spring-cloud-starter-netflix-eureka-server` (Standalone Eureka Node)
- `org.springframework.cloud:spring-cloud-config-server` (Standalone Config Node)
- `org.springframework.boot:spring-boot-starter-actuator` (Required for `/actuator/health` checks)

---

## 7. Testing (All Modules)
- `org.springframework.boot:spring-boot-starter-test` (JUnit 5, Mockito, SpringBootTest)
