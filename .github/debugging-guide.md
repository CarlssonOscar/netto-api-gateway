# Systematic Debugging Guide - MuleSoft API Gateway

> **Reference document** - linked from main copilot-instructions.md  
> Use this when troubleshooting unexpected behavior or bugs

---

## Core Philosophy

> **Always start from the highest architectural level and move downward.**  
> Root causes are often in configuration or connectivity, not DataWeave transformations.

**The Debugging Mantra:**
```
Configuration → Connectivity → Flows → Transformations
```

**Never skip levels.** A "quick fix" in DataWeave often masks a configuration-level root cause.

---

## Quick Reference Checklist

Before diving into code, ask yourself:

1. ✅ Is `config.properties` loaded correctly?
2. ✅ Are HTTP listener/request configs using correct ports?
3. ✅ Is the backend API running and accessible?
4. ✅ Are the three API layers (8081, 8082, 8083) not conflicting?
5. ✅ Is RAML validation passing?

**Only after checking these** should you move to flow-specific debugging.

---

## The Debugging Hierarchy

### Level 1: Configuration (Check FIRST)

**What to inspect:**
- `config.properties` file
- Property placeholders (`${property}`)
- HTTP listener/request configurations
- Global configuration in `netto-api-gateway.xml`

**Questions to ask:**
- Are properties loaded correctly?
- Are ports configured correctly?
- Are timeouts appropriate?
- Are host addresses correct for the environment?

**When to investigate this level:**
- Application fails to start
- "Could not resolve placeholder" errors
- Port conflicts
- Configuration-related exceptions

**Common issues:**
```properties
# Wrong: Typo in property name
http.ports=8081  # Should be http.port

# Wrong: Incorrect host for environment
backend.api.host=production-server  # Works in prod, fails locally

# Wrong: Missing property
# process.api.port not defined but referenced
```

**Debugging commands:**
```bash
# Check if ports are in use
netstat -an | find "8081"
netstat -an | find "8082"
netstat -an | find "8083"

# Test backend connectivity
curl http://localhost:8080/api/v1/actuator/health
```

---

### Level 2: Connectivity

**What to inspect:**
- HTTP request configurations
- Backend API availability
- Network connectivity between layers
- Timeout settings

**Questions to ask:**
- Is the backend API running?
- Can the gateway reach the backend?
- Are timeouts too short?
- Are there firewall/network issues?

**When to investigate this level:**
- `HTTP:CONNECTIVITY` errors
- `HTTP:TIMEOUT` errors
- Requests to backend failing
- Intermittent failures

**Common issues:**
```xml
<!-- Wrong: Backend not running on this port -->
<http:request-connection host="localhost" port="8080" />

<!-- Wrong: Timeout too short for complex calculations -->
<http:request-config responseTimeout="1000" />  <!-- 1 second too short -->

<!-- Correct: Appropriate timeout -->
<http:request-config responseTimeout="30000" />  <!-- 30 seconds -->
```

**Debugging steps:**
```bash
# 1. Verify backend is running
curl http://localhost:8080/api/v1/actuator/health

# 2. Test direct call to backend
curl -X POST http://localhost:8080/api/v1/tax/calculate \
  -H "Content-Type: application/json" \
  -d '{"municipalityId":"550e8400-e29b-41d4-a716-446655440000","grossMonthlySalary":50000}'

# 3. Test gateway endpoint
curl -X POST http://localhost:8081/gateway/salary/calculate \
  -H "Content-Type: application/json" \
  -d '{"employeeId":"EMP001","baseSalary":50000,"currency":"SEK"}'
```

---

### Level 3: Flows

**What to inspect:**
- Flow routing (choice routers)
- Flow references (flow-ref)
- APIKit router configuration
- Error handlers

**Questions to ask:**
- Is the request reaching the correct flow?
- Are flow-refs pointing to existing flows?
- Is the choice router evaluating correctly?
- Are error handlers catching the right errors?

**When to investigate this level:**
- 404 errors from gateway
- Requests going to wrong flow
- Error responses not matching expectations
- Flow-ref failures

**Common issues:**
```xml
<!-- Wrong: Typo in flow-ref -->
<flow-ref name="process-salary-calculaiton" />  <!-- Typo! -->

<!-- Wrong: Incorrect path matching -->
<when expression="#[attributes.requestPath contains '/salary']">
    <!-- Won't match '/gateway/salary/calculate' -->
</when>

<!-- Correct: Proper path matching -->
<when expression="#[attributes.requestPath contains '/salary/calculate']">
```

**Debugging with logging:**
```xml
<!-- Add logging to trace flow execution -->
<logger level="INFO" 
        message="Flow reached: experience-api-main, path: #[attributes.requestPath]" />

<logger level="DEBUG" 
        message="Payload before transformation: #[payload]" />
```

---

### Level 4: Transformations (Check LAST)

**What to inspect:**
- DataWeave scripts
- Type coercion
- Null handling
- Field mapping

**Questions to ask:**
- Is the input payload what you expect?
- Are types being coerced correctly?
- Are null values handled?
- Does the output match the expected format?

**When to investigate this level:**
- Transformation errors
- Incorrect output format
- Type mismatch errors
- Null pointer issues

**Common issues:**
```dataweave
// Wrong: Missing null handling
{
    salary: payload.grossSalary as Number  // Fails if null
}

// Correct: Handle nulls
{
    salary: payload.grossSalary as Number default 0
}

// Wrong: Wrong field name
{
    municipalityId: payload.municipality  // Backend expects municipalityId
}

// Correct: Match backend API contract
{
    municipalityId: payload.municipality_id
}
```

**Debugging DataWeave:**
```xml
<!-- Log input before transformation -->
<logger level="DEBUG" message="Input: #[payload]" />

<ee:transform>
    <ee:message>
        <ee:set-payload><!-- transformation --></ee:set-payload>
    </ee:message>
</ee:transform>

<!-- Log output after transformation -->
<logger level="DEBUG" message="Output: #[payload]" />
```

---

## Follow the Data Flow

Always trace the entire path through all API layers:

```
Client Request
    ↓
Experience API (8081) - Validate
    ↓
Process API (8082) - Orchestrate
    ↓
System API (8083) - Integrate
    ↓
Backend API (8080) - Calculate
    ↓
Response flows back
```

**For each layer, check:**
1. Is the request arriving?
2. Is it being routed correctly?
3. Is the transformation correct?
4. Is the response being returned?

---

## Common Issue Patterns

| **Symptom** | **Likely Level** | **What to Check** |
|-------------|------------------|-------------------|
| Application won't start | Configuration | Port conflicts, property errors |
| All requests timeout | Connectivity | Backend not running, wrong host/port |
| 404 on all requests | Flows | RAML path mismatch, wrong listener path |
| 400 Bad Request | Flows | RAML validation, missing required fields |
| Wrong response data | Transformations | DataWeave mapping, field names |
| 500 errors | Connectivity/Flows | Backend errors, unhandled exceptions |
| Intermittent failures | Connectivity | Timeouts, network issues, backend overload |

---

## MuleSoft-Specific Debugging Paths

### RAML/APIKit Issues
```
1. Is RAML file syntactically correct?
2. Does request match RAML schema?
3. Are required fields present?
4. Are types correct (string vs number)?
5. Is Content-Type header set correctly?
```

### HTTP Connector Issues
```
1. Is listener port available?
2. Is request host/port correct?
3. Are headers being passed correctly?
4. Is timeout configured appropriately?
5. Is response being parsed correctly?
```

### DataWeave Issues
```
1. Is input payload what you expect?
2. Are you accessing correct field paths?
3. Are types being coerced correctly?
4. Are nulls handled with 'default'?
5. Is output format correct (JSON/XML)?
```

### Error Handling Issues
```
1. Is error type matching correctly?
2. Is error-handler at correct scope?
3. Is on-error-propagate vs on-error-continue correct?
4. Is httpStatus variable being set?
5. Is error response being formatted correctly?
```

---

## Debugging Tools

### Logging Strategy

```xml
<!-- Add strategic logging points -->
<logger level="INFO" message="[ENTRY] Flow: #[flow.name]" />
<logger level="DEBUG" message="[PAYLOAD] #[payload]" />
<logger level="DEBUG" message="[ATTRIBUTES] #[attributes]" />
<logger level="ERROR" message="[ERROR] #[error.description]" />
```

### Test Backend Directly

```bash
# Health check
curl http://localhost:8080/api/v1/actuator/health

# Test calculation endpoint
curl -X POST http://localhost:8080/api/v1/tax/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "municipalityId": "550e8400-e29b-41d4-a716-446655440000",
    "grossMonthlySalary": 50000,
    "churchMember": false,
    "isPensioner": false
  }'
```

### Test Gateway Endpoint

```bash
# Test through gateway
curl -X POST http://localhost:8081/gateway/salary/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "employeeId": "EMP001",
    "baseSalary": 50000,
    "currency": "SEK"
  }'
```

---

## Documentation Template

For each major bug, document:

```markdown
## Bug: [Brief Description]

### Symptoms
- What was observed?
- What error messages appeared?

### Investigation Path
1. **Configuration Level**: What was checked?
2. **Connectivity Level**: What was tested?
3. **Flows Level**: What was traced?
4. **Transformations Level**: What was examined?

### Root Cause
- What was the actual problem?
- At which level was it found?

### Fix Applied
- What was changed?
- Why does this fix the root cause?

### Prevention
- How can this be prevented in the future?
```

---

## Red Flags - Stop and Check Configuration

🚩 **Immediate red flags that point to configuration issues:**
- Application fails to start
- Multiple endpoints failing with same error
- Works on one machine but not another
- "Could not resolve placeholder" in logs
- Port already in use errors
- Connection refused to backend

**Don't waste time debugging DataWeave when configuration is wrong.**

---

## Backend API Reference

When debugging integration issues, refer to:
- [`docs/mulesoft-integration-guide.md`](../docs/mulesoft-integration-guide.md)

Key backend details:
- Base URL: `http://localhost:8080/api/v1`
- Health: `GET /api/v1/actuator/health`
- Calculate: `POST /api/v1/tax/calculate`

---

*Last Updated: 2025-12-29*
