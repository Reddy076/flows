# Object-Oriented Programming (OOP) in Our Microservices

## Overview
As a backend developer, understanding how Object-Oriented Programming principles are applied in a real-world, enterprise-level microservices architecture is critical. In our application (consisting of User Service, Vault Service, Security Service, Notification Service, AI Service, and API Gateway), OOP concepts are foundational to achieving clean, scalable, and maintainable code.

This document breaks down the four core pillars of OOP—Encapsulation, Inheritance, Polymorphism, and Abstraction—with concrete examples directly from our codebase. You can use this guide to confidently explain your technical expertise during interviews.

---

## 1. Encapsulation
**Definition:** Encapsulation is the bundling of data (attributes) and the methods that operate on that data into a single unit (class). It restricts direct access to some of the object's components, which is a means of preventing accidental interference and misuse of the data (data hiding).

**Where we use it:**
We use Encapsulation universally in our Data Models (Entities) and Data Transfer Objects (DTOs).

**Codebase Example:** `VaultEntry.java` (in `vault-service`)
```java
@Entity
@Table(name = "vault_entries")
@Data // Lombok annotation that automatically generates getters and setters
public class VaultEntry {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(name = "user_id", nullable = false)
  private Long userId;

  @Column(name = "encrypted_password", nullable = false, length = 1000)
  private String password;

  // Other private fields...
}
```

**Interview Explanation:**
> "In our microservices, we enforce Encapsulation primarily in our domain entities, like the `VaultEntry` class. All the fields matching our database columns (like the `id`, `userId`, and `password`) are declared as `private`. This means external classes cannot directly modify a vault entry's state. Instead, they must use public getter and setter methods provided. We use Lombok (`@Data`) to generate these methods cleanly. This ensures data integrity—for example, preventing a password from being incorrectly modified while bypassing our core encryption service."

---

## 2. Abstraction
**Definition:** Abstraction means hiding the complex implementation details and showing only the essential features of the object. It reduces complexity and allows the programmer to focus on interactions at a higher level.

**Where we use it:**
We use Abstraction extensively in our Repository layers and Service interfaces. It allows services to interact with data or external providers without needing to know *how* the data is queried or handled under the hood.

**Codebase Examples:** 
1. `UserRepository.java` (in `user-service`)
2. `LlmProvider.java` (in `ai-service`)

```java
// Abstract Interface for AI Integrations
public interface LlmProvider {
    String generateResponse(String prompt);
}

// Spring Data JPA Repository Abstraction
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

**Interview Explanation:**
> "We implement Abstraction everywhere to decouple our business logic from technical implementation details. A great example is our Data Access Layer utilizing Spring Data JPA. We declare interfaces like `UserRepository` that extend `JpaRepository`. Our service classes call methods like `findByEmail()`, without ever needing to write the underlying boilerplate SQL or manage database connections—Spring handles that complexity automatically. 
> 
> Similarly, in our AI microservice, we have an `LlmProvider` interface. The rest of the application just knows it can ask for and receive AI data, completely abstracting away the complex logic of calling different AI vendor APIs over HTTP."

---

## 3. Inheritance
**Definition:** Inheritance is a mechanism wherein a new class is derived from an existing class. The new class (child) inherits all properties and behaviors (methods) of the parent class, promoting code reusability.

**Where we use it:**
We use Inheritance heavily for creating structured Exception Handling architectures, and also extending core library functionalities like Spring Security or Spring Web components.

**Codebase Examples:** `ResourceNotFoundException.java` (across multiple services) and `JwtAuthenticationFilter.java` (in `user-service`).

```java
// Custom Exception Inheriting from standard Java RuntimeException
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

// Inheriting from Spring Web Filter
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    // Custom logic overriding parent's doFilterInternal() method...
}
```

**Interview Explanation:**
> "Inheritance allows us to adhere tightly to the DRY (Don't Repeat Yourself) principle. For instance, in our error handling, every service has custom exceptions like `ResourceNotFoundException` or `RateLimitExceededException`. Instead of writing entirely new exception frameworks from scratch, these classes simply `extends RuntimeException`. They inherit the base Java Exception hierarchy capabilities but allow us to write custom global exception handlers using Spring's `@ControllerAdvice`. 
> 
> Another major area is security; our custom `JwtAuthenticationFilter` inherits from Spring's `OncePerRequestFilter`. This means we seamlessly inherit robust web filtering lifecycles seamlessly, only needing to inject our custom JWT token validation logic."

---

## 4. Polymorphism
**Definition:** Polymorphism (meaning "many forms") allows us to perform a single action in different ways. In Java, this occurs mainly via Method Overriding (Dynamic/Runtime Polymorphism) and Method Overloading (Static/Compile-time Polymorphism).

**Where we use it:**
We frequently use Polymorphism via interface implementation (Overriding/Runtime Polymorphism) to support multiple variants of a specific feature, utilizing the highly-effective Strategy Design Pattern.

**Codebase Examples:** 
1. The Backup Importer System in `vault-service` (Runtime Polymorphism via Interfaces).
2. Custom JPA Attribute Converters in `user-service` (Runtime Polymorphism Overriding Library Methods).

**Example 1: The Strategy Pattern (Vault Importers)**
```java
// The standard interface contract
public interface Importer {
    List<VaultEntry> parse(String csvData, UserVaultDetails user, SecretKey key, EncryptionService encService) throws Exception;
    String getSupportedSource();
}

// Polymorphic implementations
@Component
public class LastPassImporter implements Importer {
    @Override
    public List<VaultEntry> parse(...) { /* LastPass specific CSV parsing logic */ }
}

@Component
public class ChromeImporter implements Importer {
    @Override
    public List<VaultEntry> parse(...) { /* Chrome specific CSV parsing logic */ }
}
```

**Example 2: Overriding Library Methods (JPA Converters)**
```java
// Implementing JPA's generic AttributeConverter interface
@Converter
public class StringListConverter implements AttributeConverter<List<String>, String> {

  @Override
  public String convertToDatabaseColumn(List<String> stringList) {
    return stringList != null ? String.join(";", stringList) : "";
  }

  @Override
  public List<String> convertToEntityAttribute(String string) {
    return string != null && !string.isEmpty() ? Arrays.asList(string.split(";")) : Collections.emptyList();
  }
}
```

**Interview Explanation:**
> "Polymorphism is heavily utilized across our application to build flexible and extensible systems. A prime example is our use of the Strategy pattern for importing vault data. We defined a single unified contract via the `Importer` interface with a `parse()` method. We then built polymorphic classes like `LastPassImporter` and `ChromeImporter` that implement this interface. At runtime, our core logic dynamically chooses which underlying object to use, but interacts with them generically as `Importer` objects. We can easily plug in a new importer later without modifying the core calling service.
> 
> Another great example is how we apply Polymorphism with standard library frameworks like JPA. Our `StringListConverter` class implements JPA's `AttributeConverter` interface. By `@Override`-ing `convertToDatabaseColumn` and `convertToEntityAttribute`, we polymorphically inject our custom string-joining logic directly into the automated database persistence cycle. When Hibernate tries to convert that specific column, it dynamically invokes our custom logic rather than the default."

---

## Summary for the Interview

When you are asked **"Can you explain how you've used OOP concepts in your projects?"**, you can confidently frame your answer like this:

> *"In our microservices architecture, clean code is essential, so OOP principles are heavily embedded into our design base.*
> 
> *First, we enforce **Encapsulation** across all domain entities (like User or Vault objects) by keeping properties strictly private and exposing them through controlled getters and setters to protect application state.*
> 
> *We utilize **Abstraction** heavily in our persistence layer via Spring Data repositories. Our services interact with databases through highly abstracted interfaces rather than raw SQL, shielding the core business logic from underlying database networking intricacies.*
> 
> *For **Inheritance**, we frequently extend framework-provided classes, such as creating custom exceptions that inherit from `RuntimeException`, or our security classes extending `OncePerRequestFilter` to inherit HTTP filtering lifecycles without writing boilerplate.*
> 
> *Finally, **Polymorphism** is key for features that require strategic flexibility. For example, our Vault CSV Import feature utilizes an `Importer` interface implemented variously by `LastPassImporter`, `ChromeImporter`, etc. The system can dynamically invoke the correct varying implementation behaviors at runtime through a single unified interface contract."*
