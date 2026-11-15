# E-commerce Microservices

A full-stack application for managing customers, products, and invoices using Spring Boot microservices and an Angular frontend.

The project explores how independent services work together through service discovery, an API gateway, declarative HTTP clients, centralized configuration, and circuit breakers. A responsive French-language interface connects these backend capabilities to practical business workflows.

## What I Implemented

- Built separate customer, inventory, and billing services with their own persistence layers.
- Exposed customer and product resources through Spring Data REST, including pagination and sorting.
- Implemented invoice creation, listing, customer filtering, detailed views, and deletion.
- Connected billing to customer and inventory services using OpenFeign and service discovery.
- Added an API gateway, starting with static routes and extending it to discovery-based routing.
- Centralized shared, service-specific, and environment-specific settings with Spring Cloud Config.
- Added circuit breakers and fallback responses for invoice detail lookups during downstream outages.
- Developed a responsive Angular interface with validated forms, loading states, error feedback, and recovery actions.
- Added backend integration tests and desktop/mobile browser tests for the main workflows.

## Architecture

```text
Angular frontend
       |
       | Development proxy: /api/*
       v
API Gateway
       |
       +-- Customer Service --> Customer H2 database
       +-- Inventory Service --> Inventory H2 database
       +-- Billing Service --> Billing H2 database
                    |
                    +-- OpenFeign --> Customer / Inventory services

Eureka: service registration and discovery
Config Server: configuration from the Git-backed config-repo/ directory
```

The gateway resolves service routes through Eureka. Customer, inventory, billing, and gateway services load centralized settings from Config Server.

| Component | Responsibility | Default port |
| --- | --- | --- |
| `discovery-service` | Eureka service registry | `8761` |
| `config-service` | Git-backed configuration server | `9999` |
| `customer-service` | Customer data and REST resources | `8081` |
| `inventory-service` | Product catalog, prices, and stock data | `8082` |
| `billing-service` | Invoices, line items, and service integration | `8083` |
| `gateway-service` | Discovery-based API routing | `8888` |
| `frontend` | Angular user interface | `4200` |

## Technology Stack

| Layer | Technologies |
| --- | --- |
| Backend | Java 21, Spring Boot 3.5.7, Spring Cloud 2025.0.0 |
| Service communication | OpenFeign, Spring Cloud LoadBalancer, Eureka |
| Routing and configuration | Spring Cloud Gateway, Spring Cloud Config |
| Persistence | Spring Data JPA, Spring Data REST, H2 |
| Resilience and validation | Resilience4j, Jakarta Bean Validation, Spring Boot Actuator |
| Frontend | Angular 22.2, TypeScript 6, RxJS, CSS |
| Testing | Spring Boot Test, JUnit, Playwright |
| Build tools | Maven, npm |

## Main Features

### Customer and Product Management

- Create, edit, delete, and browse customers and products.
- Use server-side pagination and name-based sorting.
- Validate customer forms, product prices, and stock quantities in the interface.
- Recover from stale pagination and failed refreshes.

### Invoice Management

- Create invoices with multiple product lines and positive quantities.
- Retrieve product prices on the server when creating an invoice.
- Store the unit price on each invoice line and calculate monetary totals with `BigDecimal`.
- Browse invoices, filter by customer, and display enriched customer and product details.
- Print invoice details and delete invoices with confirmation.

### Frontend Experience

- Business overview with service-backed metrics and recent invoices.
- Responsive navigation and layouts for desktop and mobile.
- Lazy-loaded pages and typed API models.
- Shared feedback and pagination components, with clear loading and empty states.
- French-language labels and MAD currency presentation.

## Technical Details

### REST and Gateway Integration

Customer and product APIs return HAL resources. The Angular API layer converts these responses into typed models and extracts identifiers from resource links when necessary.

During development, the frontend forwards `/api/*` requests to the gateway on port `8888` and removes the leading `/api` prefix.

Examples through the gateway:

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/customer-service/customers?page=0&size=8` | Paginated customers |
| `POST` | `/customer-service/customers` | Create a customer |
| `GET` | `/inventory-service/products?page=0&size=8` | Paginated products |
| `POST` | `/inventory-service/products` | Create a product |
| `GET` | `/billing-service/api/bills` | List invoice summaries |
| `GET` | `/billing-service/api/bills?customerId=1` | Filter invoices by customer |
| `GET` | `/billing-service/api/bills/1` | Retrieve an enriched invoice |
| `POST` | `/billing-service/api/bills` | Create an invoice |
| `DELETE` | `/billing-service/api/bills/1` | Delete an invoice |

Example invoice request body:

```json
{
  "customerId": 1,
  "items": [
    { "productId": 1, "quantity": 2 },
    { "productId": 2, "quantity": 1 }
  ]
}
```

### Resilience

Invoice detail lookups use separate customer and inventory circuit breakers. Fallback responses allow an existing invoice to remain readable when a downstream service is unavailable.

Invoice creation uses direct service lookups rather than fallback values, so an unavailable customer or product service cannot silently create an invoice using placeholder data or a zero fallback price.

### Centralized Configuration

The `config-repo/` directory contains shared settings, service-specific settings, and `dev`/`prod` profile overrides. Config Server reads this directory from Git.

Supported environment variables include:

| Variable | Purpose |
| --- | --- |
| `CONFIG_GIT_URI` | Override the configuration repository URI |
| `CONFIG_GIT_LABEL` | Select its branch or revision; defaults to `main` |
| `CONFIG_SEARCH_PATHS` | Select its configuration directory; defaults to `config-repo` |
| `CONFIG_SERVER_URL` | Override the clients' Config Server URL |
| `EUREKA_URL` | Override the Eureka registration URL |
| `SPRING_PROFILES_ACTIVE` | Activate a profile such as `dev` or `prod` |

Refreshable customer configuration can be reloaded through `POST /actuator/refresh`. The `requests/configuration.http` file provides executable examples for configuration profiles, refresh, and gateway requests.

The configured H2 databases are in-memory, including the example `prod` profiles. Application data is recreated when a service restarts.

## Run Locally

### Requirements

- JDK 21.
- Maven 3.9 or later.
- Git, available to Config Server and configuration integration tests.
- Node.js compatible with Angular 22: `^22.22.3`, `^24.15.0`, or `>=26.0.0`.
- npm 11.

### Start the Backend

From the repository root, build and run the backend tests:

```bash
mvn clean verify
```

Start each service in a separate terminal, in this order:

```bash
mvn -pl discovery-service spring-boot:run
mvn -pl config-service spring-boot:run
mvn -pl customer-service spring-boot:run
mvn -pl inventory-service spring-boot:run
mvn -pl billing-service spring-boot:run
mvn -pl gateway-service spring-boot:run
```

Wait for the services to register in the Eureka dashboard at `http://localhost:8761` before using gateway routes. Config Server needs access to its configured Git repository; `CONFIG_GIT_URI` can point to a dedicated local clone instead.

### Start the Frontend

From the `frontend/` directory:

```bash
npm ci
npm start
```

Open `http://localhost:4200`. The development proxy connects the interface to the gateway.

## Testing

Backend tests cover REST behavior, invoice enrichment and lifecycle, validation, downstream failures, circuit recovery, configuration merging, profiles, and refresh.

From the repository root:

```bash
mvn test
```

From `frontend/`, install the browser used by the end-to-end tests and run the frontend checks:

```bash
npx playwright install chromium
npm run typecheck
npm run build
npm test
```

The default Playwright suite uses mocked APIs and runs desktop and mobile scenarios. It starts its own frontend server on port `4201`.

With the backend services running, execute the real gateway integration scenario:

```bash
npm run test:live
```

## Project Structure

```text
.
|-- billing-service/      Invoice persistence and business logic
|-- config-repo/          Shared and profile-specific configuration
|-- config-service/       Spring Cloud Config Server
|-- customer-service/     Customer REST resources
|-- discovery-service/    Eureka registry
|-- frontend/             Angular application and browser tests
|-- gateway-service/      API gateway
|-- inventory-service/    Product REST resources
|-- requests/             Executable HTTP examples
`-- pom.xml               Multi-module backend build
```
