# ecommerce-spring-cloud-local-dev

Docker Compose environment for running the full Spring Cloud microservices stack locally.

## What This Service Does

This repository contains no application code. It provides a `docker-compose.yml` that orchestrates all six application containers and their three PostgreSQL databases, wiring them together on a shared Docker network. Each application service pulls its configuration from the Config Server at startup via `SPRING_CLOUD_CONFIG_URI`, registers with Eureka, and exposes its port on localhost. Startup order is enforced through `depends_on` with `condition: service_healthy`, so no service starts before its dependencies are ready.

## Services

| Container | Image | Port (host) | Depends On |
|---|---|---|---|
| sc-config-server | kappa9/config-server:1.0.0 | 8888 | — |
| sc-discovery-server | kappa9/discovery-server:1.0.0 | 8761 | config-server |
| sc-api-gateway | kappa9/api-gateway:1.0.0 | 8080 | config-server, discovery-server |
| sc-product-service | kappa9/product-service:1.0.0 | 8081 | config-server, discovery-server, postgres |
| sc-inventory-service | kappa9/inventory-service:1.0.0 | 8082 | config-server, discovery-server, postgres |
| sc-order-service | kappa9/order-service:1.0.0 | 8083 | config-server, discovery-server, postgres |
| sc-product-service-postgres | postgres:14.22 | 5432 | — |
| sc-inventory-service-postgres | postgres:14.22 | 5433 | — |
| sc-order-service-postgres | postgres:14.22 | 5434 | — |

## Key Configuration

| Property | Value | Why |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | `docker` | Activates docker-specific datasource config from config-repo |
| `SPRING_CLOUD_CONFIG_URI` | `http://sc-config-server:8888` | Overrides localhost URL baked in bootstrap.yml |
| `LOGGING_LEVEL_IO_GITHUB_RESILIENCE4J` | `DEBUG` | Surfaces circuit breaker and retry state transitions in order-service logs |
| healthcheck `start_period` | 30s (app), 20s (postgres) | Gives JVM time to start before Docker marks the container unhealthy |
| `condition: service_healthy` | all critical deps | Prevents race conditions on startup |

## How to Run

```bash
docker compose up -d
```

Prerequisites: Docker Desktop running, all images available on Docker Hub under `kappa9/`.

To tear down and remove volumes:

```bash
docker compose down -v
```

## Stack

| Component | Version |
|---|---|
| Docker Compose | 3.x |
| PostgreSQL | 14.22 |
| Java (in images) | 21 |
| Spring Boot (in images) | 4.0.6 |
| Spring Cloud (in images) | 2025.1.1 |

## Startup Order

```
1. sc-config-server           :8888   <- this repo starts this first
2. sc-discovery-server        :8761
3. sc-product-service         :8081
   sc-inventory-service       :8082
   sc-order-service           :8083
4. sc-api-gateway             :8080   <- entry point for all HTTP traffic
```

---

# Project Overview — Spring Cloud Microservices Demo

## Goal

A didactic project to learn Spring Cloud through a minimal e-commerce system built as microservices. Each repository isolates one Spring Cloud concept so it can be studied and tested independently.

## Architecture

```
[Client HTTP]
      |
[API Gateway :8080]
      |
      +---> [Product Service :8081]
      +---> [Inventory Service :8082]
      +---> [Order Service :8083]
                  |
                  +---> [Product Service]   (via OpenFeign)
                  +---> [Inventory Service] (via OpenFeign)

All services register on  --> [Eureka Discovery Server :8761]
All services read config from --> [Config Server :8888]
Config Server reads from      --> [config-repo on GitHub]
```

## Repository Structure (Polyrepo)

| # | Repository | Purpose |
|---|---|---|
| 1 | `spring-cloud-config-repo` | YAML configuration files, read by Config Server via Git |
| 2 | `spring-cloud-config-server` | Reads config-repo and exposes it to other services |
| 3 | `spring-cloud-discovery-server` | Eureka — service registry |
| 4 | `spring-cloud-api-gateway` | Single entry point, routes to microservices |
| 5 | `spring-cloud-product-service` | Product CRUD |
| 6 | `spring-cloud-inventory-service` | Inventory CRUD |
| 7 | `spring-cloud-order-service` | Order orchestration, calls product and inventory |
| — | `ecommerce-spring-cloud-local-dev` | Docker Compose for running the full stack locally |

## Spring Cloud Concepts Covered

| Concept | Component | Repository |
|---|---|---|
| Centralized configuration | Spring Cloud Config | config-server + config-repo |
| Service discovery | Eureka | discovery-server |
| Client-side load balancing | Spring Cloud LoadBalancer | built into Feign and Gateway |
| Inter-service communication | OpenFeign | order-service |
| API Gateway / routing | Spring Cloud Gateway | api-gateway |
| Circuit Breaker | Resilience4j | order-service |
| Retry | Resilience4j | order-service |

## Common Stack

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>4.0.6</version>
</parent>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2025.1.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**Java:** 21