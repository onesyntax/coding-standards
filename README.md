# Laravel Clean Architecture Project Structure

This document outlines a comprehensive project structure for Laravel applications following Clean Architecture principles with a domain-driven design approach.

## Key Principles

1. **Domain-Focused Organization**: Top-level directories represent business domains rather than technical concerns
2. **Clean Architecture Layers**: Each domain follows the standard Clean Architecture layers
3. **Co-located Specifications**: Tests reside with the code they verify
4. **Testing Support**: Each domain includes testing support utilities
5. **Explicit IO Layer**: Clear separation of external integrations and frameworks

## Directory Structure

```
project-root/
├── app/
│   ├── Foundation/               # Shared kernel, base classes, traits
│   │   ├── Entities/             # Base entities, value objects, shared domain logic
│   │   ├── UseCases/             # Base interactors, shared application services
│   │   ├── Adapters/             # Base controllers, presenters, interface adapters
│   │   ├── IO/                   # Shared IO components
│   │       ├── Database/         # Common database concerns, base repositories
│   │       ├── Http/             # Shared controllers, middleware, base API resources
│   │       ├── Web/              # Common layouts, shared components, base templates
│   │       └── ExternalServices/ # Shared service clients, integrations
│   │   ├── Specs/                # Foundation specifications
│   │   └── Testing/              # Foundation testing support API
│   │
│   ├── Booking/                  # Booking domain module
│   │   ├── Entities/             # Domain entities and business rules
│   │   ├── UseCases/             # Application business logic
│   │   ├── Adapters/             # Interface adapters (controllers, presenters)
│   │   ├── IO/                   # Frameworks, drivers, external services
│   │       ├── Database/         # Booking-specific database implementations
│   │       ├── Http/             # Booking-specific HTTP interfaces
│   │       ├── Web/              # Booking-specific UI elements
│   │       └── ExternalServices/ # Booking-specific external services
│   │   ├── Specs/                # Booking behavior specifications
│   │   └── Testing/              # Booking-specific testing utilities
│   │
│   ├── Payment/                  # Payment domain module
│   │   ├── Entities/             # Payment-specific domain entities
│   │   ├── UseCases/             # Payment-specific application logic
│   │   ├── Adapters/             # Payment-specific interface adapters
│   │   ├── IO/                   # Payment-specific frameworks and drivers
│   │       ├── Database/         # Payment-specific database implementations
│   │       ├── Http/             # Payment-specific HTTP interfaces
│   │       ├── Web/              # Payment-specific UI elements
│   │       └── ExternalServices/ # Payment gateways, providers
│   │   ├── Specs/                # Payment behavior specifications
│   │   └── Testing/              # Payment-specific testing utilities
│   │
│   └── User/                     # User domain module
│       ├── Entities/             # User-specific domain entities
│       ├── UseCases/             # User-specific application logic
│       ├── Adapters/             # User-specific interface adapters
│       ├── IO/                   # User-specific frameworks and drivers
│           ├── Database/         # User-specific database implementations
│           ├── Http/             # User-specific HTTP interfaces
│           ├── Web/              # User-specific UI elements
│           └── ExternalServices/ # Authentication services, etc.
│       ├── Specs/                # User behavior specifications
│       └── Testing/              # User-specific testing utilities
```

## Layer Descriptions

### Entities Layer

Contains the enterprise-wide business rules and entities:
- Domain entities
- Value objects
- Domain events
- Business rule validators

This layer has no dependencies on other layers or external frameworks.

### UseCases Layer

Contains application-specific business rules:
- Interactors/services
- Command handlers
- Input/output ports
- DTOs (Data Transfer Objects)

This layer can depend on the Entities layer but not on outer layers.

### Adapters Layer

Contains interface adapters that convert data between the use cases and external formats:
- Controllers
- Presenters
- API resources/transformers
- Gateway interfaces

This layer can depend on UseCases but not on the IO layer.

### IO Layer

Contains frameworks, drivers, and external services:
- Database implementations
- Web interfaces
- External API integrations
- Framework-specific code

This is the outermost layer and can depend on all inner layers.

## Specifications

Each domain includes a `Specs` directory containing behavior specifications for that domain. Specs serve as both tests and living documentation, describing the expected behavior of components.

## Testing Support

Each domain includes a `Testing` directory that provides utilities to support testing:
- Test data factories
- Mock implementations
- Testing helpers
- Fixtures and datasets

This approach provides domain-specific testing tools tailored to each domain's needs.

## Benefits

1. **Clear Domain Focus**: The structure clearly communicates what the application does
2. **Maintainability**: Co-located specs make it easy to find and update tests
3. **Autonomy**: Each domain can evolve independently
4. **Scalability**: New domains can be added without affecting existing ones
5. **Clean Dependencies**: Dependencies flow inward, maintaining architectural integrity

This structure is designed to support large, complex applications while keeping the codebase maintainable, testable, and aligned with business domains.