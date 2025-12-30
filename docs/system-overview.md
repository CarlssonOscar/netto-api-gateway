# NettoApiGateway - System Overview

> **Purpose**: Architecture documentation for MuleSoft API Gateway  
> **Status**: Development  
> **Last Updated**: 2025-12-30

---

## Executive Summary

NettoApiGateway is a MuleSoft-based API Gateway that provides a unified entry point for Swedish net salary calculations. It follows MuleSoft's API-led connectivity pattern with three distinct layers, routing requests to a Spring Boot backend that handles all business logic.

**Key Principle**: The gateway is an integration layer, NOT a calculation engine. All tax calculations happen in the backend.

---

## System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                  │
│              (Web Apps, Mobile Apps, Partners)                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 EXPERIENCE API (Port 8081)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  • RAML Validation (APIKit)                               │  │
│  │  • Client-friendly request/response format                │  │
│  │  • Error standardization                                  │  │
│  │  • Future: Authentication, Rate Limiting                  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   PROCESS API (Port 8082)                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  • Request orchestration                                  │  │
│  │  • Multi-service aggregation (future)                     │  │
│  │  • Process-level routing                                  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SYSTEM API (Port 8083)                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  • Backend HTTP integration                               │  │
│  │  • Protocol translation                                   │  │
│  │  • System abstraction                                     │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  BACKEND API (Port 8080)                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Spring Boot 4.x + Java 21                                │  │
│  │  • Swedish tax calculations                               │  │
│  │  • Municipality/Region data                               │  │
│  │  • Business logic & validation                            │  │
│  │  • PostgreSQL database                                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    NettoApiGateway (MuleSoft)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ experience-api  │  │  process-api    │  │   system-api    │  │
│  │     .xml        │  │     .xml        │  │     .xml        │  │
│  │                 │  │                 │  │                 │  │
│  │ • APIKit Router │  │ • Orchestration │  │ • HTTP Client   │  │
│  │ • RAML Valid.   │  │ • Routing       │  │ • Backend Calls │  │
│  │ • Error Handler │  │ • Aggregation   │  │ • Response Map  │  │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  │
│           │                    │                    │            │
│           └────────────────────┼────────────────────┘            │
│                                │                                 │
│  ┌─────────────────────────────┴─────────────────────────────┐  │
│  │              netto-api-gateway.xml                         │  │
│  │         (Global Config, HTTP Listener/Request)             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────────────────────────┐   │
│  │ experience-api  │  │        config.properties            │   │
│  │     .raml       │  │   (Ports, Hosts, Timeouts)          │   │
│  └─────────────────┘  └─────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Flow

### Tax Calculation Request Flow

```
1. Client POST /gateway/tax/calculate
         │
         ▼
2. Experience API (8081)
   ├── RAML validates request schema
   ├── Extracts: municipalityId, grossMonthlySalary, churchMember, isPensioner
   └── Forwards to Process API
         │
         ▼
3. Process API (8082)
   ├── Routes to tax calculation flow
   └── Calls System API
         │
         ▼
4. System API (8083)
   ├── Transforms request (if needed)
   ├── POST to Backend: /api/v1/tax/calculate
   └── Returns backend response
         │
         ▼
5. Backend API (8080)
   ├── Validates business rules
   ├── Calculates all tax components
   ├── Returns full breakdown
   └── Response flows back through layers
         │
         ▼
6. Client receives TaxCalculationResponse
   └── 19 fields with complete tax breakdown
```

---

## API Endpoints

### Experience API (Public Gateway)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/gateway/tax/calculate` | Calculate net salary with full tax breakdown |
| GET | `/gateway/regions` | List all Swedish regions (län) |
| GET | `/gateway/municipalities` | List all municipalities |
| GET | `/gateway/municipalities/by-region/{regionId}` | Municipalities in a region |
| GET | `/gateway/municipalities/{id}` | Single municipality details |

### Backend API (Internal)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/tax/calculate` | Tax calculation engine |
| GET | `/api/v1/regions` | Region data |
| GET | `/api/v1/municipalities` | Municipality data |
| GET | `/api/v1/municipalities/by-region/{regionId}` | Filtered municipalities |
| GET | `/api/v1/municipalities/{id}` | Single municipality |
| GET | `/api/v1/actuator/health` | Health check |

---

## Design Decisions

### 1. No Authentication (Current Phase)

**Decision**: Open system without authentication for demo/development.

**Rationale**:
- Simplifies initial development and testing
- Authentication can be added as MuleSoft policy later
- Backend has no auth - gateway will handle it when needed

**Future**: Add API Key or OAuth 2.0 via MuleSoft policies.

### 2. Full Tax Breakdown Passthrough

**Decision**: Pass all 19 tax calculation fields from backend to client.

**Rationale**:
- Frontend needs complete transparency for Swedish tax display
- No data hiding or simplification at gateway level
- Backend is source of truth for all calculations

### 3. Three-Tier Despite Simple Use Case

**Decision**: Maintain Experience → Process → System layers even for straightforward passthrough.

**Rationale**:
- Follows MuleSoft best practices
- Enables future orchestration (e.g., multiple backend calls)
- Clear separation of concerns
- Easier to add features like caching, aggregation

### 4. No Business Logic in Gateway

**Decision**: Gateway transforms data format only, never calculates.

**Rationale**:
- Single source of truth (backend)
- Easier testing and debugging
- Gateway remains stateless
- Tax law changes only affect backend

---

## Configuration

### Port Allocation

| Component | Port | Purpose |
|-----------|------|---------|
| Experience API | 8081 | Public gateway entry point |
| Process API | 8082 | Internal orchestration |
| System API | 8083 | Internal backend integration |
| Backend API | 8080 | Spring Boot application |

### Key Configuration (config.properties)

```properties
# Gateway Ports
http.port=8081
process.api.port=8082
system.api.port=8083

# Backend Connection
backend.api.host=localhost
backend.api.port=8080
backend.api.basePath=/api/v1

# Timeouts
backend.api.timeout=30000
```

---

## Error Handling

### Standard Error Response

```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable description",
  "timestamp": "2025-12-30T10:00:00.000Z"
}
```

### Error Code Mapping

| Backend Status | Gateway Error Code | HTTP Status |
|----------------|-------------------|-------------|
| 400 | VALIDATION_ERROR | 400 |
| 404 | NOT_FOUND | 404 |
| 500 | BACKEND_ERROR | 502 |
| Timeout | TIMEOUT_ERROR | 504 |
| Connection refused | BACKEND_UNAVAILABLE | 503 |

---

## Technology Stack

| Layer | Technology | Version |
|-------|------------|---------|
| API Gateway | MuleSoft Anypoint | 4.4.x |
| API Specification | RAML | 1.0 |
| Validation | APIKit | 2.x |
| Backend | Spring Boot | 4.x |
| Runtime | Java | 21 |
| Database | PostgreSQL | (backend) |

---

## File Structure

```
NettoApiGateway/
├── src/main/mule/
│   ├── netto-api-gateway.xml    # Global config, HTTP configs
│   ├── experience-api.xml       # Public API layer
│   ├── process-api.xml          # Orchestration layer
│   └── system-api.xml           # Backend integration layer
├── src/main/resources/
│   ├── api/
│   │   └── experience-api.raml  # API contract
│   └── config/
│       └── config.properties    # Environment config
├── docs/
│   ├── mulesoft-integration-guide.md  # Backend API reference
│   ├── frontend-integration-guide.md  # Frontend integration
│   └── system-overview.md             # This document
└── pom.xml                      # Maven build (MuleSoft)
```

---

## Glossary

| Term | Definition |
|------|------------|
| **Experience API** | Public-facing API layer that handles client requests |
| **Process API** | Middle layer for orchestration and aggregation |
| **System API** | Backend integration layer |
| **APIKit** | MuleSoft module for RAML-based API validation |
| **RAML** | RESTful API Modeling Language |
| **Gateway** | The complete MuleSoft application acting as API proxy |
| **Backend** | Spring Boot application with business logic |

---

## Related Documentation

- [Frontend Integration Guide](frontend-integration-guide.md) - API reference for frontend developers
- [MuleSoft Integration Guide](mulesoft-integration-guide.md) - Backend API documentation
- [Architecture Principles](../.github/architecture-principles.md) - Design guidelines
- [Code Standards](../.github/code-standards.md) - Coding conventions
- [Debugging Guide](../.github/debugging-guide.md) - Troubleshooting

---

*Last Updated: 2025-12-30*
