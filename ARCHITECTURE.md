# API Gateway Architecture Documentation

## Overview

This document provides detailed architecture documentation for the Netto API Gateway, a MuleSoft-based implementation following API-Led Connectivity principles.

## Three-Tier API Architecture

### Layer 1: Experience API (Gateway)

**Port**: 8081  
**Public Access**: YES  
**Entry Point**: `/gateway/v1/*`

#### Responsibilities

1. **Request Validation**
   - Validates all incoming requests against RAML specification
   - Ensures required fields are present
   - Validates data types and formats (e.g., currency codes)
   - Returns 400 Bad Request with detailed error messages for invalid requests

2. **Response Transformation**
   - Transforms backend responses to client-friendly format
   - Standardizes response structure
   - Adds metadata (timestamps, status codes)
   - Ensures consistent error response format

3. **Routing**
   - Routes all validated requests to Process API
   - Handles routing errors (timeouts, connectivity issues)
   - Implements retry logic (configurable)
   - Returns appropriate HTTP status codes

#### What Experience API Does NOT Do

- ❌ Business logic processing
- ❌ Tax calculations
- ❌ Database operations
- ❌ Complex data transformations
- ❌ Direct backend system calls

#### API Contract (RAML)

The Experience API is defined by `experience-api.raml`:
- Defines request/response schemas
- Enforces data validation rules
- Documents API endpoints
- Provides example requests/responses

### Layer 2: Process API

**Port**: 8082  
**Public Access**: NO (Internal only)  
**Entry Point**: `/process/api/*`

#### Responsibilities

1. **Process Orchestration**
   - Coordinates calls to multiple System APIs
   - Manages the sequence of operations
   - Handles process-level error recovery
   - Implements business process flows

2. **Data Aggregation**
   - Combines data from multiple System APIs
   - Creates unified response from disparate sources
   - Handles partial failures gracefully
   - Returns aggregated results to Experience API

3. **Process Logic**
   - Determines which System APIs to call
   - Decides the order of operations
   - Handles conditional flows
   - Manages process state

#### Example: Salary Calculation Process

```
1. Receive request from Experience API
   ↓
2. Call System API: Get Employee Data
   ↓
3. Store employee data in variables
   ↓
4. Call System API: Calculate Salary (with employee data)
   ↓
5. Store calculation results
   ↓
6. Aggregate data from steps 2 and 4
   ↓
7. Return combined result to Experience API
```

#### Error Handling

- Falls back to mock data if System APIs are unavailable
- Logs all System API failures
- Continues processing with available data
- Returns partial results when possible

### Layer 3: System API

**Port**: 8083  
**Public Access**: NO (Internal only)  
**Entry Point**: `/system/api/*`

#### Responsibilities

1. **Backend System Integration**
   - Provides standardized interface to backend systems
   - Abstracts backend-specific protocols and formats
   - Handles backend authentication and authorization
   - Manages backend-specific error handling

2. **Data Access**
   - Retrieves data from backend systems
   - Performs CRUD operations
   - Handles database connections
   - Manages transaction boundaries

3. **System-Specific Logic**
   - Converts API requests to backend-specific formats
   - Transforms backend responses to standard format
   - Handles backend-specific quirks and limitations
   - Manages backend system timeouts and retries

#### Placeholders for Backend Integration

The System API includes placeholders that demonstrate how to integrate with:

**1. SQL Database**
```xml
<db:select config-ref="Database_Config">
    <db:sql>SELECT * FROM employees WHERE id = :employeeId</db:sql>
    <db:input-parameters>#[{'employeeId': payload.employeeId}]</db:input-parameters>
</db:select>
```

**2. REST API**
```xml
<http:request config-ref="Backend_API_Config" 
              path="/api/v1/employee" 
              method="GET">
    <http:query-params>#[{'id': payload.employeeId}]</http:query-params>
</http:request>
```

**3. SOAP Web Service**
```xml
<ws:consumer config-ref="SOAP_Service_Config" 
             operation="GetEmployee">
    <ws:message>#[payload]</ws:message>
</ws:consumer>
```

**4. Message Queue (JMS)**
```xml
<jms:publish config-ref="JMS_Config" 
             destination="employee.data.queue">
    <jms:message>#[payload]</jms:message>
</jms:publish>
```

## Request Flow Example

### Salary Calculation Request Flow

```
Client Request
    |
    | POST /gateway/v1/salary/calculate
    | {
    |   "employeeId": "EMP001",
    |   "baseSalary": 50000,
    |   "currency": "USD"
    | }
    |
    ▼
┌─────────────────────────────────────────┐
│  EXPERIENCE API (Port 8081)             │
│                                         │
│  1. HTTP Listener receives request      │
│  2. APIKit validates against RAML       │
│  3. Store original request              │
│  4. Log incoming request                │
│  5. Route to Process API                │
└─────────────────┬───────────────────────┘
                  │
                  | POST /process/api/salary/calculate
                  |
                  ▼
┌─────────────────────────────────────────┐
│  PROCESS API (Port 8082)                │
│                                         │
│  1. Receive request                     │
│  2. Store request variables             │
│  3. Call Employee System API            │
│     POST /system/api/employee/get       │
│  4. Store employee data                 │
│  5. Call Salary System API              │
│     POST /system/api/salary/calculate   │
│  6. Store calculation result            │
│  7. Aggregate all data                  │
│  8. Return to Experience API            │
└─────────────────┬───────────────────────┘
                  │
                  | Multiple System API calls
                  |
                  ▼
┌─────────────────────────────────────────┐
│  SYSTEM API (Port 8083)                 │
│                                         │
│  Endpoint 1: Get Employee               │
│  - Placeholder for backend call         │
│  - Returns employee data                │
│                                         │
│  Endpoint 2: Calculate Salary           │
│  - Placeholder for backend call         │
│  - Returns calculation result           │
│                                         │
│  Note: In production, these would call │
│  actual backend systems where business  │
│  logic and tax calculations reside      │
└─────────────────┬───────────────────────┘
                  │
                  | Return aggregated data
                  |
                  ▼
┌─────────────────────────────────────────┐
│  EXPERIENCE API (Port 8081)             │
│                                         │
│  1. Receive Process API response        │
│  2. Transform to client format          │
│  3. Add metadata (timestamp, status)    │
│  4. Log outgoing response               │
│  5. Return to client                    │
└─────────────────┬───────────────────────┘
                  │
                  | HTTP 200 OK
                  |
                  ▼
Client Response
{
  "employeeId": "EMP001",
  "netSalary": 37500,
  "grossSalary": 50000,
  "currency": "USD",
  "calculationDate": "2024-01-15T10:30:00Z",
  "status": "SUCCESS",
  "message": "Salary calculated successfully"
}
```

## Security Considerations

### 1. Network Segmentation
- Experience API: Public-facing (with security policies)
- Process API: Internal network only
- System API: Internal network only

### 2. Authentication & Authorization
- Experience API: Can implement OAuth 2.0, JWT, or API keys
- Process API: Internal authentication (mTLS recommended)
- System API: Backend-specific authentication

### 3. Data Validation
- RAML-based validation at Experience API
- Additional business validation at Process API
- Data sanitization at System API

### 4. Rate Limiting
- Implement at Experience API level
- Protect backend systems from overload
- Return 429 Too Many Requests when exceeded

## Scalability Strategy

### Horizontal Scaling
Each API layer can be scaled independently:

```
Load Balancer
    |
    ├── Experience API Instance 1 (Port 8081)
    ├── Experience API Instance 2 (Port 8081)
    └── Experience API Instance N (Port 8081)
            |
            ├── Process API Instance 1 (Port 8082)
            ├── Process API Instance 2 (Port 8082)
            └── Process API Instance N (Port 8082)
                    |
                    ├── System API Instance 1 (Port 8083)
                    ├── System API Instance 2 (Port 8083)
                    └── System API Instance N (Port 8083)
```

### Caching Strategy
- Cache employee data at Process API level
- Cache calculation rules at System API level
- Implement cache invalidation policies
- Use distributed cache (Redis) for multi-instance deployments

## Monitoring and Observability

### Metrics to Track
1. **Experience API**
   - Request rate (requests/second)
   - Response time (p50, p95, p99)
   - Error rate (%)
   - Validation failure rate

2. **Process API**
   - Orchestration time
   - System API call count
   - System API failure rate
   - Aggregation time

3. **System API**
   - Backend call success rate
   - Backend response time
   - Connection pool usage
   - Circuit breaker status

### Logging Strategy
- Correlation ID across all layers
- Structured logging (JSON format)
- Log levels: INFO for flows, ERROR for failures
- Sensitive data masking

## Deployment Options

### 1. CloudHub (Recommended for Production)
```bash
mvn clean deploy -DmuleDeploy \
  -Danypoint.username=${USERNAME} \
  -Danypoint.password=${PASSWORD}
```

### 2. On-Premises (Standalone Mule Runtime)
```bash
# Build the application
mvn clean package

# Copy to Mule apps directory
cp target/netto-api-gateway-1.0.0-SNAPSHOT.jar \
   $MULE_HOME/apps/
```

### 3. Docker Container
```dockerfile
FROM mulesoft/mule:4.4.0
COPY target/netto-api-gateway-1.0.0-SNAPSHOT.jar /opt/mule/apps/
```

### 4. Kubernetes (with Runtime Fabric)
```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: netto-api-gateway
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: mule-runtime
        image: mulesoft/mule:4.4.0
```

## Performance Considerations

### 1. Connection Pooling
Configure HTTP connectors with connection pooling:
```xml
<http:request-config name="process-api-requestConfig">
    <http:request-connection host="${process.api.host}" 
                            port="${process.api.port}">
        <pooling-profile maxActive="50" 
                        maxIdle="10" 
                        exhaustedAction="WHEN_EXHAUSTED_WAIT"/>
    </http:request-connection>
</http:request-config>
```

### 2. Timeout Configuration
- Experience API timeout: 30 seconds (configurable)
- Process API timeout: 25 seconds
- System API timeout: 20 seconds

### 3. Async Processing
For long-running operations, consider:
- Implementing async request/response pattern
- Using JMS queues for decoupling
- Providing status check endpoints

## Business Logic Separation

### IMPORTANT: Where Business Logic Resides

This gateway implementation follows the principle that **business logic does NOT belong in the gateway**. Instead:

#### Backend Systems (External to Gateway)
✅ Tax calculation logic  
✅ Salary computation rules  
✅ Business validation rules  
✅ Data persistence  
✅ Complex data processing  

#### Gateway (This Implementation)
✅ Request validation (format, required fields)  
✅ Routing to appropriate backend  
✅ Response transformation (format conversion)  
✅ Error handling and standardization  
❌ Business logic  
❌ Tax calculations  
❌ Salary computation  

### Example Clarification

When the System API returns salary calculation:
```json
{
  "netSalary": 37500,
  "deductions": {
    "taxes": 10000,
    "insurance": 1500,
    "retirement": 1000
  }
}
```

These values are calculated by the **backend payroll system**, not by the gateway. The System API is just an integration point that calls the backend system and returns the result.

## Future Enhancements

### 1. Additional Endpoints
- Employee management (CRUD)
- Payroll history retrieval
- Tax document generation
- Benefits administration

### 2. Advanced Features
- OAuth 2.0 authentication
- API versioning strategy
- GraphQL support
- WebSocket support for real-time updates

### 3. Integration Patterns
- Event-driven architecture with JMS/Kafka
- Saga pattern for distributed transactions
- Circuit breaker for fault tolerance
- Bulkhead pattern for resource isolation

## Troubleshooting

### Common Issues

1. **Port Already in Use**
   - Change ports in `config.properties`
   - Kill processes using the ports

2. **Backend Service Unavailable**
   - Check System API is running on port 8083
   - Verify network connectivity
   - Review error logs for details

3. **Validation Errors**
   - Ensure request matches RAML specification
   - Check all required fields are present
   - Verify data types and formats

4. **Timeout Errors**
   - Increase timeout values in `config.properties`
   - Check backend system performance
   - Review connection pool settings

## Support and Maintenance

### Documentation Updates
- Keep RAML specification in sync with implementation
- Update README for new endpoints
- Document configuration changes
- Maintain API changelog

### Code Quality
- Follow MuleSoft naming conventions
- Keep flows focused and single-purpose
- Use sub-flows for reusability
- Comment complex logic

### Testing Strategy
- Unit tests with MUnit
- Integration tests for end-to-end flows
- Performance testing under load
- Security testing for vulnerabilities