# Code Quality Standards - MuleSoft API Gateway

> **Reference document** - linked from main copilot-instructions.md  
> Follow these standards for all code contributions

---

## Naming Conventions

### Mule XML Flows

| **Type** | **Convention** | **Example** |
|----------|---------------|-------------|
| Main Flows | `{layer}-{resource}-{action}` | `experience-salary-calculate` |
| Sub-flows | `{action}-{description}` | `transform-request-to-backend` |
| Error Handlers | `handle-{error-type}` | `handle-validation-error` |
| Configurations | `{layer}-{type}Config` | `experience-api-httpListenerConfig` |
| Variables | `camelCase` | `httpStatus`, `originalPayload` |

### Flow Naming Examples

```xml
<!-- Main flows - include layer prefix -->
<flow name="experience-api-main">
<flow name="experience-salary-calculate">
<flow name="process-salary-calculation">
<flow name="system-calculate-salary">

<!-- Sub-flows - action-focused -->
<sub-flow name="transform-request-to-backend">
<sub-flow name="transform-response-to-client">
<sub-flow name="validate-employee-id">
<sub-flow name="handle-backend-error">
```

### RAML Naming

| **Type** | **Convention** | **Example** |
|----------|---------------|-------------|
| Types | `PascalCase` | `SalaryCalculationRequest` |
| Properties | `camelCase` | `employeeId`, `baseSalary` |
| Resources | `lowercase-kebab` | `/salary/calculate` |
| Query params | `camelCase` | `?municipalityId=...` |

---

## DataWeave Standards

### General Principles

✅ **DO:**
- Use meaningful variable names
- Add comments for complex transformations
- Use default values for optional fields
- Format output consistently

❌ **DON'T:**
- Use single-letter variable names
- Leave magic values unexplained
- Mix naming conventions in output

### DataWeave Examples

```dataweave
%dw 2.0
output application/json
---
// ✅ GOOD - Clear, documented transformation
{
    // Map from client format to backend API format
    municipalityId: payload.municipality_id,
    grossMonthlySalary: payload.salary as Number,
    
    // Optional fields with sensible defaults
    churchMember: payload.church_member default false,
    isPensioner: payload.is_pensioner default false
}
```

```dataweave
%dw 2.0
output application/json
---
// ❌ BAD - Unclear transformation
{
    m: payload.mid,
    g: payload.s as Number,
    c: payload.cm default false
}
```

### Response Transformations

```dataweave
%dw 2.0
output application/json
---
// Transform backend response to client-friendly format
{
    gross_salary: payload.grossMonthlySalary,
    net_salary: payload.monthlyNetSalary,
    total_tax: payload.totalTaxAfterCredit / 12,
    tax_rate: payload.effectiveTaxRate,
    breakdown: {
        municipal: payload.municipalTax / 12,
        regional: payload.regionalTax / 12,
        state: payload.stateTax / 12,
        church: payload.churchFee / 12,
        burial: payload.burialFee / 12,
        job_credit: payload.jobTaxCredit / 12
    }
}
```

### Error Response Format

```dataweave
%dw 2.0
output application/json
---
{
    error: vars.errorCode default "UNKNOWN_ERROR",
    message: error.description default "An unexpected error occurred",
    timestamp: now()
}
```

---

## XML Structure Standards

### Flow Organization

Keep flows focused and well-organized:

```xml
<!-- ✅ GOOD - Clear structure -->
<flow name="experience-salary-calculate">
    <!-- 1. Listener -->
    <http:listener config-ref="experience-api-httpListenerConfig" path="/salary/calculate" />
    
    <!-- 2. Validation/Router -->
    <apikit:router config-ref="experience-api-config" />
    
    <!-- 3. Error handling at the end -->
    <error-handler>
        <on-error-propagate type="APIKIT:BAD_REQUEST">
            <flow-ref name="handle-validation-error" />
        </on-error-propagate>
    </error-handler>
</flow>
```

### Sub-flow Usage

Extract reusable logic into sub-flows:

```xml
<!-- ✅ GOOD - Reusable transformation -->
<sub-flow name="transform-request-to-backend">
    <ee:transform>
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    municipalityId: payload.municipality_id,
    grossMonthlySalary: payload.salary as Number
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</sub-flow>

<!-- Use in multiple flows -->
<flow name="experience-salary-calculate">
    <flow-ref name="transform-request-to-backend" />
    <flow-ref name="call-process-api" />
</flow>
```

### Comments in XML

```xml
<!-- 
========================================
EXPERIENCE API LAYER (GATEWAY)
========================================

This is the public-facing API Gateway - the only entry point for external clients.

Responsibilities:
- Request validation against RAML specification
- Routing requests to Process API
- Response transformation
- Error handling

Does NOT contain:
- Business logic
- Tax calculations
- Data processing
-->
```

---

## Error Handling Standards

### Consistent Error Structure

All errors must follow this format:

```json
{
    "error": "ERROR_CODE",
    "message": "Human-readable description",
    "timestamp": "2025-12-29T14:30:00.000Z"
}
```

### Error Codes

| **Code** | **HTTP Status** | **Usage** |
|----------|----------------|-----------|
| `VALIDATION_ERROR` | 400 | RAML/APIKit validation failures |
| `NOT_FOUND` | 404 | Resource not found |
| `METHOD_NOT_ALLOWED` | 405 | Wrong HTTP method |
| `BACKEND_ERROR` | 500/502 | Backend API errors |
| `TIMEOUT_ERROR` | 504 | Backend timeout |
| `INTERNAL_ERROR` | 500 | Unexpected errors |

### Error Handler Pattern

```xml
<error-handler>
    <on-error-propagate type="APIKIT:BAD_REQUEST">
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "VALIDATION_ERROR",
    message: error.description,
    timestamp: now()
}]]></ee:set-payload>
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus">400</ee:set-variable>
            </ee:variables>
        </ee:transform>
    </on-error-propagate>
    
    <on-error-propagate type="HTTP:CONNECTIVITY">
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "BACKEND_UNAVAILABLE",
    message: "Backend service is temporarily unavailable",
    timestamp: now()
}]]></ee:set-payload>
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus">503</ee:set-variable>
            </ee:variables>
        </ee:transform>
    </on-error-propagate>
    
    <on-error-propagate type="ANY">
        <logger level="ERROR" message="Unexpected error: #[error.description]" />
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: "INTERNAL_ERROR",
    message: "An unexpected error occurred",
    timestamp: now()
}]]></ee:set-payload>
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus">500</ee:set-variable>
            </ee:variables>
        </ee:transform>
    </on-error-propagate>
</error-handler>
```

---

## RAML Standards

### Type Definitions

```raml
types:
  SalaryCalculationRequest:
    type: object
    properties:
      employeeId:
        type: string
        required: true
        description: Unique identifier for the employee
        example: "EMP001"
      baseSalary:
        type: number
        required: true
        description: Base salary amount
        example: 50000
      currency:
        type: string
        required: true
        pattern: ^[A-Z]{3}$
        description: ISO 4217 currency code
        example: "USD"
```

### Error Response Types

```raml
types:
  ErrorResponse:
    type: object
    properties:
      error:
        type: string
        description: Error code
      message:
        type: string
        description: Human-readable error message
      timestamp:
        type: datetime
        description: When the error occurred
```

### Resource Definitions

```raml
/salary:
  /calculate:
    post:
      description: Calculate net salary for an employee
      body:
        application/json:
          type: SalaryCalculationRequest
      responses:
        200:
          body:
            application/json:
              type: SalaryCalculationResponse
        400:
          body:
            application/json:
              type: ErrorResponse
        500:
          body:
            application/json:
              type: ErrorResponse
```

---

## Configuration Standards

### config.properties Structure

```properties
# ===========================================
# Experience API Configuration (Gateway)
# ===========================================
http.port=8081
http.host=0.0.0.0

# ===========================================
# Process API Configuration
# ===========================================
process.api.host=localhost
process.api.port=8082
process.api.basePath=/process/api

# ===========================================
# System API Configuration
# ===========================================
system.api.host=localhost
system.api.port=8083
system.api.basePath=/system/api

# ===========================================
# Backend API Configuration
# ===========================================
backend.api.host=localhost
backend.api.port=8080
backend.api.basePath=/api/v1

# ===========================================
# Gateway Settings
# ===========================================
gateway.timeout=30000
gateway.max.retries=3
```

### Using Configuration in Flows

```xml
<!-- ✅ GOOD - Reference configuration -->
<http:listener-config name="experience-api-httpListenerConfig">
    <http:listener-connection host="${http.host}" port="${http.port}" />
</http:listener-config>

<!-- ❌ BAD - Hardcoded values -->
<http:listener-config name="experience-api-httpListenerConfig">
    <http:listener-connection host="0.0.0.0" port="8081" />
</http:listener-config>
```

---

## Logging Standards

### When to Log

| **Level** | **When to Use** |
|-----------|-----------------|
| `INFO` | Request received, response sent, key milestones |
| `WARN` | Recoverable errors, retries, fallbacks |
| `ERROR` | Failures, exceptions, unrecoverable errors |
| `DEBUG` | Detailed diagnostic (development only) |

### Logging Examples

```xml
<!-- ✅ GOOD - Meaningful log with context -->
<logger level="INFO" 
        message="Processing salary calculation for employee: #[payload.employeeId]" />

<logger level="ERROR" 
        message="Backend API call failed: #[error.description], endpoint: #[vars.backendUrl]" />

<!-- ❌ BAD - No context -->
<logger level="INFO" message="Processing request" />
```

---

## Things to Avoid

### Business Logic in Gateway

```xml
<!-- ❌ WRONG - Tax calculation in gateway -->
<ee:transform>
    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    netSalary: payload.grossSalary * 0.75,  // WRONG: Business logic!
    taxAmount: payload.grossSalary * 0.25   // WRONG: Belongs in backend!
}]]></ee:set-payload>
</ee:transform>

<!-- ✅ CORRECT - Gateway transforms, backend calculates -->
<http:request method="POST" path="/api/v1/tax/calculate" />
```

### Hardcoded Values

```xml
<!-- ❌ WRONG - Hardcoded -->
<http:request-config name="backend-config">
    <http:request-connection host="localhost" port="8080" />
</http:request-config>

<!-- ✅ CORRECT - Configuration -->
<http:request-config name="backend-config">
    <http:request-connection host="${backend.api.host}" port="${backend.api.port}" />
</http:request-config>
```

### Skipping API Layers

```xml
<!-- ❌ WRONG - Experience calling System directly -->
<flow name="experience-api-main">
    <flow-ref name="system-calculate-salary" />  <!-- WRONG! -->
</flow>

<!-- ✅ CORRECT - Proper layer flow -->
<flow name="experience-api-main">
    <flow-ref name="call-process-api" />
</flow>
```

---

*Last Updated: 2025-12-29*
