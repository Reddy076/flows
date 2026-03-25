# Microservices Dependencies — Detailed Breakdown
> Part of the infrastructure deep-dive series.

This document provides a highly detailed explanation of the dependencies (libraries and frameworks) used across the password manager ecosystem. 

Rather than duplicating standard Spring Boot boilerplate in every file, this project uses a **Parent POM (`password-manager-microservices`)** to manage versions globally (e.g., locking Java to version 21, and Spring Cloud to `2023.0.0`).

---

## 1. Global Dependencies (Shared by almost every service)

These are the backbone dependencies injected into almost every microservice module (User, Vault, Security, etc.).

### Spring Boot Core
- **`spring-boot-starter-web`**: The absolute core of the REST APIs. It provides the embedded Tomcat web server (running on ports like 8081, 8082) and the Spring MVC framework (`@RestController`, `@GetMapping`).
- **`spring-boot-starter-data-jpa`**: The Object-Relational Mapping (ORM) layer. It brings in Hibernate to automatically translate Java classes (Entities) into MySQL tables and handles SQL query generation (`JpaRepository`).
- **`spring-boot-starter-validation`**: Brings in the Hibernate Validator. Allows the use of `@Valid`, `@NotBlank`, and `@Email` in DTOs to automatically reject bad incoming HTTP requests before they hit the controller logic.
- **`spring-boot-starter-aop`**: Aspect-Oriented Programming. Used to create global cross-cutting concerns (like the `LoggingAspect` that automatically logs every method execution time without writing `.getLogger()` in every file).

### Spring Cloud Infrastructure
- **`spring-cloud-starter-netflix-eureka-client`**: The "phonebook client." Allows the service to automatically register its IP address with the Eureka Server on port `8761` and download the routing table every 30 seconds.
- **`spring-cloud-starter-config`**: The configuration client. Tells the microservice to pause its boot sequence and fetch `application.yml` properties (like database passwords) from the Config Server on port `8888`.
- **`spring-cloud-starter-openfeign`**: The internal communication library. Allows `vault-service` to talk to `user-service` simply by calling a Java interface method (`@FeignClient`), completely hiding the complexity of making raw HTTP requests.

### Database & Documentation Tools
- **`mysql-connector-j`**: The official JDBC driver required for Spring Data JPA to physically connect to the MySQL database containers.
- **`springdoc-openapi-starter-webmvc-ui`**: The Swagger/OpenAPI generator. It scans all `@RestController` classes and automatically generates the interactive API documentation web interface at `/swagger-ui.html`.
- **`lombok`**: A compile-time library that auto-generates repetitive Java code (Getters, Setters, Constructors, Builders) using annotations like `@Data` and `@RequiredArgsConstructor`.
- **`guava`**: Google's core libraries for Java. Used heavily in this project for advanced in-memory caching (like the API Gateway Rate Limiter cache) and complex string manipulation.

---

## 2. Service-Specific Dependencies

Some dependencies are heavy or highly specialized and are *only* included in the microservices that strictly need them.

### Security & User Identity (`user-service`, `vault-service`, `api-gateway`)
- **`spring-boot-starter-security`**: Specifically injects the `SecurityFilterChain`, `PasswordEncoder` (for BCrypt hashing of master passwords), and handles the `SecurityContext`.
- **`jjwt-api`, `jjwt-impl`, `jjwt-jackson` (v0.12.5)**: The industry-standard JSON Web Token library. 
  - *Why 3 pieces?* API is the interface, Impl does the heavy cryptographic signing/verification (HMAC SHA-256), and Jackson handles converting the JSON payload into Java objects.
- **`totp` (`dev.samstevens`)**: A specialized, lightweight Time-Based One-Time Password library. Used exclusively by `user-service` to generate the QR Codes for Google Authenticator/Authy and mathematically validate the 6-digit codes.

### Artificial Intelligence (`ai-service`)
- **`spring-boot-starter-webflux`**: The reactive, non-blocking version of Spring Web. Required by the `ai-service` because LLM text generation is often slow or streamed. WebFlux allows the server to keep the connection open without locking up Tomcat threads.
- **`okhttp`**: Square's highly optimized HTTP client. Used by the `LlmClientService` to make direct, fast HTTP POST calls out to the external Groq/OpenAI APIs (bypassing the heavier `RestTemplate` or `WebClient` for these specific low-level API calls).

### Notifications (`user-service`, `notification-service`)
- **`spring-boot-starter-mail`**: A wrapper around `JavaMailSender` and `Jakarta Mail`. Used to connect to SMTP servers (like Gmail or Twilio SendGrid) to dispatch OTPs for new device logins, welcome emails, and security breach alerts.

### Utilities (`vault-service`)
- **`commons-lang3`**: Apache Commons library. Used for robust, null-safe string manipulations and random string generation (e.g., generating secure sharing URLs or default recovery codes).
