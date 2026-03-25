# Eureka Server (Service Registry) — Architecture & Flow
> Part of the infrastructure deep-dive series.

This document explains the role and flow of the **Eureka Server**, the central service
registry for the entire microservice architecture. 

---


## 1. The Core Problem It Solves

### Without Eureka (Static Routing)
If the API Gateway or `vault-service` wants to talk to `user-service`, they need to know
its exact IP address and port (e.g., `http://10.0.0.5:8081`). 
- **Problem:** If `user-service` crashes and restarts on a new IP, everything breaks.
- **Problem:** If you spin up 3 instances of `user-service` to handle high traffic, the Gateway doesn't know about them to distribute the load.

### With Eureka (Dynamic Discovery)
Instead of memorizing IP addresses, services only need to know **one thing**: the address
of the Eureka Server (`http://localhost:8761`). 
Every service registers its own IP and port with Eureka upon startup. When one service
wants to talk to another, it just asks Eureka for the address.

---

## 2. Configuration & Bootstrapping
(`EurekaServerApplication.java`, `application.yml`)

The Eureka Server is incredibly lightweight in terms of custom code. It relies almost
entirely on the `spring-cloud-starter-netflix-eureka-server` library.

### The Code
```java
@SpringBootApplication
@EnableEurekaServer // <-- This single annotation turns this app into a Service Registry
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### The Configuration (`application.yml`)
```yaml
server:
  port: 8761 # The standard default port for Eureka

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false # I am the server, don't register myself
    fetch-registry: false       # I am the server, don't fetch other nodes
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```
By default, every Eureka instance tries to find *another* Eureka instance to sync with
(for high availability). Because we are only running a single standalone Eureka node,
we set `register-with-eureka` and `fetch-registry` to `false` to prevent it from throwing
errors trying to connect to a nonexistent peer.

---

## 3. The Registration Flow

When you start up a microservice (like `user-service`), it automatically phones home
to Eureka to register itself.

```
[USER-SERVICE (Starting up on port 8081)]
    |
    |-- Checks its application.yml config
    |   spring.application.name=USER-SERVICE
    |   eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
    |
    |-- Sends HTTP POST to Eureka:
    |   "Hi! I am USER-SERVICE. You can reach me at IP 192.168.1.100, Port 8081"
    v
[EUREKA SERVER (:8761)]
    |
    |-- Adds entry to its internal registry table in memory:
    |   -------------------------------------------------
    |   | Application  | Status | IP Address    | Port  |
    |   -------------------------------------------------
    |   | USER-SERVICE | UP     | 192.168.1.100 | 8081  |
    |   -------------------------------------------------
    v
[USER-SERVICE]
    |-- Sends a "Heartbeat" (HTTP PUT) every 30 seconds: "I'm still alive!"
```

If the Eureka Server stops receiving heartbeats from `user-service` for 90 seconds,
it automatically removes `user-service` from the registry, assuming it crashed.

---

## 4. The Discovery Flow (Service-to-Service)

This is how `vault-service` uses Eureka to find `user-service` when it needs to derive
an AES encryption key.

### Flow Diagram

```
[VAULT-SERVICE]
    | Needs to fetch the masterPasswordHash from USER-SERVICE.
    |
    |-- Feign Client asks for "USER-SERVICE":
    |   @FeignClient(name = "USER-SERVICE")
    |
    |-- 1. Vault-service connects to Eureka
    |      "Hey Eureka, where is USER-SERVICE?"
    v
[EUREKA SERVER]
    |-- Looks at internal registry table.
    |-- "USER-SERVICE is currently alive at 192.168.1.100:8081"
    v
[VAULT-SERVICE]
    |
    |-- 2. Vault-service caches this address locally.
    |
    |-- 3. Vault-service makes a DIRECT HTTP call over the Docker network:
    |      GET http://192.168.1.100:8081/api/users/internal/by-username/john
    v
[USER-SERVICE]
    |-- Processes the request and returns JSON to Vault-service.
```

> **Crucial Concept:** Eureka does **NOT** proxy the traffic. It acts exactly like a
> phonebook. The vault-service looks up the number in the phonebook (Eureka), and then
> dials the number directly (Docker network over HTTP).

---

## 5. The Discovery Flow (API Gateway)

The `api-gateway` relies on Eureka to dynamically route traffic from the frontend browser
to the correct backend service.

```
FRONTEND BROWSER
    |
    | GET https://myapp.com/api/vault/1
    v
[API GATEWAY]
    |
    |-- Matches route: path=/api/vault/** -> uri=lb://VAULT-SERVICE
    |
    |-- The "lb://" prefix tells Spring Cloud Gateway to use the LoadBalancer.
    |
    |-- 1. Gateway asks Eureka: "Where are the active instances of VAULT-SERVICE?"
    |-- 2. Eureka returns a list (e.g., [10.0.0.6:8082, 10.0.0.7:8082])
    |-- 3. Gateway picks one instance (Round Robin load balancing).
    |-- 4. Gateway forwards the HTTP request to that specific IP.
```

---

## 6. Client-Side Caching & Resiliency

To prevent the Eureka Server from becoming a massive bottleneck (and a single point of failure), 
clients like the Gateway and `vault-service` use **client-side caching**.

### The Caching Flow

```
[API GATEWAY / VAULT-SERVICE (Eureka Client)]
    |
    |-- 1. STARTUP: Fetches the entire registry from Eureka 
    |      and saves it in local memory.
    |
    |-- 2. BACKGROUND TASK: Every 30 seconds, asks Eureka for delta updates
    |      to keep its local cache fresh.
    |
    |-- 3. INCOMING REQUEST: Needs to talk to USER-SERVICE.
    |      Instead of asking the Eureka Server over the network, 
    |      it looks at its OWN local in-memory cache.
    |
    |-- 4. Routes request directly to the cached IP.
```

### Why this is critical
If the Eureka Server container suddenly crashes and burns, **the entire microservice ecosystem stays online.**
The API Gateway and other microservices will simply continue routing traffic using the last
known local copy of the registry they downloaded. 

The only feature that breaks during a Eureka outage is *dynamic scaling*—if you spin up a brand 
new instance of `user-service`, the Gateway won't know about it until Eureka comes back online
to broadcast the update.

---

## 7. Security & Infrastructure Notes

- **Network Isolation:** The Eureka Server does not have any authentication (no JWTs, no username/password). It relies entirely on being isolated within a private Docker network. External traffic from the internet cannot reach port `8761`.
- **In-Memory Storage:** The registry is held completely in RAM. It does not use a database. If the Eureka Server crashes and reboots, it starts with an empty registry. (This is fine, because all connected microservices will immediately re-register themselves upon their next 30-second heartbeat check-in).
- **Dashboard UI:** Eureka provides a built-in web dashboard at `http://localhost:8761` showing all currently registered instances, their UP/DOWN status, and system memory metrics.
