# Config Server — Architecture & Flow
> Part of the infrastructure deep-dive series.

This document explains the role and flow of the **Spring Cloud Config Server**, the central configuration authority for the entire microservice ecosystem.

---

## 1. The Core Problem It Solves

### Without a Config Server
Normally, if `user-service` connects to a database, the credentials are hardcoded in its own `application.yml`. If `vault-service` connects to an LLM provider, its API key is hardcoded in its own `application.yml`.
- **Problem:** If you change a database password, you have to individually locate, edit, rebuild, and redeploy 5 different microservices.
- **Problem:** Secrets are scattered locally across multiple repositories, increasing the risk of accidental leakage.

### With a Config Server
Every microservice boots up almost completely "empty". Their only configuration is: "My name is X, go ask the Config Server for my actual settings." 
The Config Server is entirely responsible for storing and serving environment variables, passwords, API URLs, and database connections to all other services centrally.

---

## 2. Server Configuration & Bootstrapping
(`ConfigServerApplication.java`, `application.yml`)

The Config Server itself is a lightweight Spring Boot app powered by `spring-cloud-config-server`.

### Server Configuration (`application.yml`)
```yaml
server:
  port: 8888  # The standard default port for Spring Cloud Config Server

spring:
  application:
    name: config-server
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config-repo
```

### What does "Native" mean here?
Spring Cloud Config Server usually pulls configuration files directly from a remote **Git Repository** (e.g., GitHub). However, to make this project easier to run locally without needing SSH keys or Git tokens, it is using the `native` profile.

This means it reads the `.yml` configuration files directly from an internal folder: `src/main/resources/config-repo/`. Inside that folder, you will find files like:
- `user-service.yml`
- `vault-service.yml`
- `api-gateway.yml`

---

## 3. The Client Bootstrapping Flow (How Services Load Properties)

When a microservice (like `user-service`) starts up, it must contact the Config Server **before** it tries to connect to a database or Eureka.

### The Flow Diagram

```
[USER-SERVICE] (Starting up on port 8081)
    |
    |-- 1. Reads its local boostrap.yml:
    |      spring.application.name = user-service
    |      spring.config.import = optional:configserver:http://localhost:8888
    |
    |-- 2. Halts booting and makes HTTP GET request:
    |      GET http://localhost:8888/user-service/default
    v
[CONFIG SERVER] (:8888)
    |
    |-- 3. Receives request for "user-service" (default profile).
    |
    |-- 4. Searches `classpath:/config-repo` for `user-service.yml`.
    |      (Or falls back to `application.yml` for shared global properties).
    |
    |-- 5. Combines the files and returns a massive JSON response 
    |      containing all database URLs, JWT secrets, and port configs.
    v
[USER-SERVICE]
    |
    |-- 6. Injects the fetched properties into its `Environment`
    |      (resolves all @Value("${...}") and @ConfigurationProperties).
    |
    |-- 7. Connects to MySQL using the fetched DB credentials.
    |
    |-- 8. Connects to Eureka using the fetched Eureka URL.
    |
    |-- 9. Fully started and ready for traffic!
```

---

## 4. How the "Global" configuration works

Not every property is unique to a specific service. For example, every service needs to know where Eureka is, and every service needs the global JWT Secret to validate tokens.

Inside `config-repo`, there is usually a file named `application.yml` (the default). 

1. When `vault-service` asks for its config, the Config Server reads both `application.yml` (global) **and** `vault-service.yml` (specific).
2. It merges them.
3. If there is a conflict, the properties in `vault-service.yml` **override** the global ones.

---

## 5. Security & Infrastructure Notes

- **Network Isolation:** Just like Eureka, the Config Server is only exposed internally on the Docker network (`8888`). External users hitting the API Gateway cannot query the Config Server to extract DB passwords.
- **Fail Fast:** Microservices are configured to fail immediately if the Config Server is offline. If `vault-service` cannot reach `localhost:8888` on startup, it will intentionally crash rather than booting with missing or dangerous default configurations.
- **Startup Order:** Because every service depends on the Config Server to know how to connect to the database or Eureka, the `config-server` must always be the **very first** container to boot up in a `docker-compose` environment.
