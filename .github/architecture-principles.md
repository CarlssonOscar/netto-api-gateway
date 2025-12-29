# Architecture & Design Principles - MuleSoft API Gateway

> **Reference document** - linked from main copilot-instructions.md  
> Consult this when making architectural decisions or designing new components

---

## MuleSoft Three-Tier API Architecture

### Core Principles

MuleSoft's API-led connectivity pattern organizes APIs into three distinct layers:

```
┌─────────────────────────────────────┐
│     EXPERIENCE API (Port 8081)      │  ← Public entry point
│     - RAML validation               │
│     - Client-specific transforms    │
│     - Response formatting           │
├─────────────────────────────────────┤
│       PROCESS API (Port 8082)       │  ← Orchestration layer
│     - Aggregation logic             │
│     - Sequencing                    │
│     - Data composition              │
├─────────────────────────────────────┤
│       SYSTEM API (Port 8083)        │  ← Integration layer
│     - Backend connectivity          │
│     - Protocol translation          │
│     - System abstraction            │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│      BACKEND API (Port 8080)        │  ← Spring Boot (external)
│     - Business logic                │
│     - Tax calculations              │
│     - Data persistence              │
└─────────────────────────────────────┘
```

**Benefits:**
- **Reusability**: System APIs can be shared across multiple Process APIs
- **Agility**: Experience APIs can change without affecting backend systems
- **Governance**: Clear boundaries for security and monitoring
- **Maintainability**: Each layer has focused responsibilities

---

## Layer Responsibilities

### Experience API Layer (Gateway)

**Purpose**: Public-facing entry point for external clients.

**Responsibilities:**
- Request validation against RAML specification
- Authentication and authorization (policies)
- Rate limiting and throttling
- Response transformation for clients
- Error standardization
- CORS handling

**Does NOT contain:**
- Business logic
- Tax calculations
- Data processing
- Direct database access

```xml
<!-- Example: Experience API focuses on validation and routing -->
<flow name="experience-api-main">
    <http:listener config-ref="experience-api-httpListenerConfig" path="/gateway/*" />
    <apikit:router config-ref="experience-api-config" />
</flow>
```

### Process API Layer (Orchestration)

**Purpose**: Orchestrates business processes and coordinates calls to System APIs.

**Responsibilities:**
- Orchestrate calls to multiple System APIs
- Aggregate data from different sources
- Handle process-level sequencing
- Compose responses from multiple sources

**Does NOT contain:**
- Business logic (deferred to backend via System APIs)
- Direct external system access
- Client-specific transformations

```xml
<!-- Example: Process API coordinates multiple system calls -->
<flow name="process-salary-calculation">
    <flow-ref name="system-get-employee" />
    <flow-ref name="system-calculate-salary" />
    <!-- Aggregate results -->
</flow>
```

### System API Layer (Integration)

**Purpose**: Provides standardized interfaces to backend systems.

**Responsibilities:**
- Connect to backend systems (REST, SOAP, databases)
- Translate protocols
- Abstract backend complexity
- Handle system-specific error formats

**Does NOT contain:**
- Business logic (belongs in backend)
- Client-specific logic
- Process orchestration

```xml
<!-- Example: System API connects to backend -->
<flow name="system-calculate-salary">
    <http:request config-ref="backend-api-config" 
                  method="POST" 
                  path="/api/v1/tax/calculate" />
</flow>
```

---

## Design Principles

### Gateway is NOT Backend

> **CRITICAL**: The API Gateway is an integration layer, NOT a calculation engine.

| **Gateway Does** | **Backend Does** |
|------------------|------------------|
| Validate request format | Validate business rules |
| Transform data structures | Calculate tax rates |
| Route requests | Apply business logic |
| Handle connectivity | Persist data |
| Apply policies | Enforce domain rules |

### Single Responsibility for Flows

Each flow should have ONE clear purpose:

```xml
<!-- ❌ BAD - Flow doing too much -->
<flow name="do-everything">
    <!-- validation, transformation, multiple calls, error handling... -->
</flow>

<!-- ✅ GOOD - Focused flows -->
<flow name="experience-salary-calculate">
    <apikit:router />
</flow>

<sub-flow name="transform-request-to-backend">
    <ee:transform>...</ee:transform>
</sub-flow>

<sub-flow name="transform-response-to-client">
    <ee:transform>...</ee:transform>
</sub-flow>
```

### DRY (Don't Repeat Yourself)

Extract common patterns into reusable sub-flows:

```xml
<!-- ✅ GOOD - Reusable error handling -->
<sub-flow name="handle-backend-error">
    <ee:transform>
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "BACKEND_ERROR",
    message: error.description,
    timestamp: now()
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</sub-flow>
```

### KISS (Keep It Simple)

- Prefer simple routing over complex logic
- Use sub-flows for clarity
- Avoid nested choice routers when possible
- Let the backend handle complexity

### Configuration over Hard-coding

```properties
# ✅ GOOD - Use config.properties
http.port=8081
backend.api.host=localhost
backend.api.port=8080

# Then reference in flows
# ${http.port}, ${backend.api.host}
```

---

## Data Flow Patterns

### Request Flow

```
Client Request
    ↓
Experience API (validate, transform)
    ↓
Process API (orchestrate)
    ↓
System API (integrate)
    ↓
Backend API (calculate)
    ↓
Response flows back up
```

### Error Handling Pattern

Each layer handles errors consistently:

```xml
<error-handler>
    <on-error-propagate type="APIKIT:BAD_REQUEST">
        <flow-ref name="handle-validation-error" />
    </on-error-propagate>
    <on-error-propagate type="HTTP:CONNECTIVITY">
        <flow-ref name="handle-backend-unavailable" />
    </on-error-propagate>
    <on-error-propagate type="ANY">
        <flow-ref name="handle-generic-error" />
    </on-error-propagate>
</error-handler>
```

### Standard Error Response

All errors should follow the same format:

```json
{
    "error": "ERROR_CODE",
    "message": "Human-readable message",
    "timestamp": "2025-12-29T14:30:00.000Z"
}
```

---

## Anti-Patterns to Avoid

❌ **Business Logic in Gateway**: Tax calculations, complex validations  
❌ **Direct Database Access**: Gateway should not have DB connections  
❌ **Hardcoded Values**: Use configuration properties  
❌ **God Flows**: Flows with too many responsibilities  
❌ **Skipping Layers**: Experience API calling System API directly  
❌ **Duplicated Transformations**: Extract to sub-flows  
❌ **Ignoring Error Handling**: Every flow needs error handling  
❌ **Mixed Concerns**: Validation and transformation in same block  

---

## File Organization

### Recommended Structure

```
src/main/mule/
├── netto-api-gateway.xml       # Global configuration
├── experience-api.xml          # Experience layer flows
├── process-api.xml             # Process layer flows
└── system-api.xml              # System layer flows

src/main/resources/
├── api/
│   └── experience-api.raml     # API specification
└── config/
    └── config.properties       # Environment configuration
```

### Configuration Separation

| **File** | **Purpose** |
|----------|-------------|
| `netto-api-gateway.xml` | Global config, properties loading |
| `experience-api.xml` | Public endpoints, RAML validation |
| `process-api.xml` | Orchestration flows |
| `system-api.xml` | Backend integration flows |
| `config.properties` | Environment-specific values |
| `experience-api.raml` | API contract definition |

---

## Integration with Backend API

### Backend API Reference

See [`docs/mulesoft-integration-guide.md`](../docs/mulesoft-integration-guide.md) for:
- Available endpoints
- Request/response formats
- Error response schemas
- Validation rules

### Key Integration Points

| **Gateway Endpoint** | **Backend Endpoint** |
|---------------------|---------------------|
| `POST /gateway/salary/calculate` | `POST /api/v1/tax/calculate` |
| Health checks | `/api/v1/actuator/health` |

### DataWeave Transformations

```dataweave
%dw 2.0
output application/json
---
// Transform from client format to backend format
{
    municipalityId: payload.municipality_id,
    grossMonthlySalary: payload.salary as Number,
    churchMember: payload.church_member default false,
    isPensioner: payload.is_pensioner default false
}
```

---

## When to Deviate

These principles are guidelines. Deviate when:
- **Performance requires it**: Caching at gateway level
- **Security mandates it**: Additional validation layers
- **Pragmatic trade-offs**: Documented exceptions

**But always:**
- Document why you're deviating
- Get team consensus
- Revisit if circumstances change

---

*Last Updated: 2025-12-29*
