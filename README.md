# Netto API Gateway

A MuleSoft-based API Gateway implementing a three-tier architecture following MuleSoft best practices.

## Architecture Overview

This API Gateway implements the **API-Led Connectivity** approach with three distinct API layers:

```
┌─────────────────────────────────────────────────────────┐
│                   External Clients                      │
│              (Mobile Apps, Web Apps, etc.)              │
└────────────────────┬────────────────────────────────────┘
                     │ HTTPS
                     ▼
┌─────────────────────────────────────────────────────────┐
│              EXPERIENCE API (Gateway)                   │
│                    Port: 8081                           │
│                                                         │
│  Responsibilities:                                      │
│  • Request validation (RAML-based)                      │
│  • Response transformation                              │
│  • Routing to Process API                               │
│  • Error handling                                       │
│                                                         │
│  Does NOT contain:                                      │
│  • Business logic                                       │
│  • Tax calculations                                     │
│  • Data processing                                      │
└────────────────────┬────────────────────────────────────┘
                     │ Internal
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   PROCESS API                           │
│                    Port: 8082                           │
│                                                         │
│  Responsibilities:                                      │
│  • Orchestrate business processes                       │
│  • Coordinate multiple System API calls                 │
│  • Aggregate data from different sources                │
│  • Process-level sequencing                             │
│                                                         │
│  NOT directly accessible from external clients          │
└────────────────────┬────────────────────────────────────┘
                     │ Internal
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   SYSTEM API                            │
│                    Port: 8083                           │
│                                                         │
│  Responsibilities:                                      │
│  • Placeholder for backend system integration           │
│  • Abstract backend system complexity                   │
│  • Standardized interfaces to backends                  │
│                                                         │
│  Backend systems would contain:                         │
│  • Business logic                                       │
│  • Tax calculations                                     │
│  • Data persistence                                     │
│                                                         │
│  NOT directly accessible from external clients          │
└─────────────────────────────────────────────────────────┘
```

## Project Structure

```
netto-api-gateway/
├── pom.xml                                 # Maven project configuration
├── src/
│   ├── main/
│   │   ├── mule/
│   │   │   ├── netto-api-gateway.xml       # Main configuration
│   │   │   ├── experience-api.xml          # Experience API (Gateway layer)
│   │   │   ├── process-api.xml             # Process API (Orchestration layer)
│   │   │   └── system-api.xml              # System API (Integration layer)
│   │   └── resources/
│   │       ├── api/
│   │       │   └── experience-api.raml     # API specification with validation
│   │       ├── config/
│   │       │   └── config.properties       # Configuration properties
│   │       └── mule-artifact.json          # Mule application descriptor
│   └── test/
│       └── munit/                          # MUnit test files
└── README.md                               # This file
```

## API Layers Explained

### 1. Experience API (Gateway - Port 8081)

**Purpose**: The only public entry point for all client requests.

**Key Features**:
- RAML-based request validation
- Automatic error handling for invalid requests
- Response transformation to client-friendly format
- Routes all requests to Process API
- Does NOT contain business logic

**Endpoints**:
- `POST /gateway/v1/salary/calculate` - Calculate employee net salary

**Example Request**:
```bash
curl -X POST http://localhost:8081/gateway/v1/salary/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "employeeId": "EMP001",
    "baseSalary": 50000,
    "currency": "USD"
  }'
```

**Example Response**:
```json
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

### 2. Process API (Port 8082)

**Purpose**: Orchestrate business processes and coordinate System API calls.

**Key Features**:
- Orchestrates calls to multiple System APIs
- Aggregates data from different sources
- Handles process-level logic and sequencing
- NOT directly accessible from external clients

**Flow for Salary Calculation**:
1. Receives request from Experience API
2. Calls Employee System API to get employee data
3. Calls Salary Calculation System API with employee data
4. Aggregates results from multiple System APIs
5. Returns aggregated response to Experience API

### 3. System API (Port 8083)

**Purpose**: Provide integration placeholders for backend systems.

**Key Features**:
- Abstracts backend system complexity
- Provides standardized interfaces
- Contains placeholders for actual backend integration
- NOT directly accessible from external clients

**Placeholder Endpoints**:
- `POST /system/api/employee/get` - Get employee data (placeholder)
- `POST /system/api/salary/calculate` - Calculate salary (placeholder)

**Implementation Notes**:
The System API contains extensive comments showing how to integrate with:
- SQL Databases (using Database connector)
- REST APIs (using HTTP connector)
- SOAP Web Services (using Web Service connector)
- Message Queues (using JMS connector)

## Configuration

Configuration is managed in `src/main/resources/config/config.properties`:

```properties
# Experience API Configuration (Gateway)
http.port=8081
http.host=0.0.0.0

# Process API Configuration
process.api.host=localhost
process.api.port=8082
process.api.basePath=/process/api

# System API Configuration
system.api.host=localhost
system.api.port=8083
system.api.basePath=/system/api

# API Gateway Configuration
gateway.timeout=30000
gateway.max.retries=3
```

## Building the Project

### Prerequisites
- Java 8 or higher
- Maven 3.3.9 or higher
- MuleSoft Anypoint Studio (optional, for development)

### Build Commands

```bash
# Clean and package the application
mvn clean package

# Install dependencies
mvn clean install

# Deploy to CloudHub (requires configuration)
mvn clean deploy -DmuleDeploy
```

## Running the Application

### Option 1: Using Maven
```bash
mvn mule:run
```

### Option 2: Using Anypoint Studio
1. Import the project into Anypoint Studio
2. Right-click on the project
3. Select "Run As" > "Mule Application"

### Option 3: Deploy to CloudHub
```bash
mvn clean deploy -DmuleDeploy \
  -Danypoint.username=<your-username> \
  -Danypoint.password=<your-password>
```

## Testing

### Test Salary Calculation

```bash
# Valid request
curl -X POST http://localhost:8081/gateway/v1/salary/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "employeeId": "EMP001",
    "baseSalary": 50000,
    "currency": "USD"
  }'

# Invalid request (missing required field)
curl -X POST http://localhost:8081/gateway/v1/salary/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "employeeId": "EMP001",
    "baseSalary": 50000
  }'

# Invalid request (negative salary)
curl -X POST http://localhost:8081/gateway/v1/salary/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "employeeId": "EMP001",
    "baseSalary": -1000,
    "currency": "USD"
  }'
```

## Key Design Principles

### 1. Separation of Concerns
- **Experience API**: Client interaction, validation, transformation
- **Process API**: Orchestration and aggregation
- **System API**: Backend integration

### 2. Gateway Does NOT Contain Business Logic
All business logic, including tax calculations and salary processing, is delegated to backend systems accessed through System APIs. The gateway only:
- Validates requests
- Routes requests
- Transforms responses

### 3. Security Through Layering
- Only Experience API is publicly accessible
- Process and System APIs are internal only
- Each layer can have its own security policies

### 4. Scalability
- Each API layer can be scaled independently
- System APIs can be reused across multiple Process APIs
- Process APIs can be reused across multiple Experience APIs

### 5. Maintainability
- Clear folder structure
- Comprehensive documentation in code
- Placeholders clearly marked for backend integration

## Extending the Gateway

### Adding a New Endpoint

1. **Update RAML specification** (`src/main/resources/api/experience-api.raml`)
2. **Add flow in Experience API** (`src/main/mule/experience-api.xml`)
3. **Add orchestration in Process API** (`src/main/mule/process-api.xml`)
4. **Add System API placeholder** (`src/main/mule/system-api.xml`)

### Integrating with Backend Systems

Replace the placeholder transforms in `system-api.xml` with actual connector calls:

**Database Example**:
```xml
<db:select config-ref="Database_Config">
    <db:sql>SELECT * FROM employees WHERE id = :employeeId</db:sql>
    <db:input-parameters>#[{'employeeId': payload.employeeId}]</db:input-parameters>
</db:select>
```

**REST API Example**:
```xml
<http:request config-ref="Backend_API_Config" 
              path="/payroll/calculate" 
              method="POST">
    <http:body>#[payload]</http:body>
</http:request>
```

## Error Handling

The gateway implements comprehensive error handling:

- **400 Bad Request**: Invalid request data (validation errors)
- **404 Not Found**: Resource not found
- **405 Method Not Allowed**: HTTP method not supported
- **500 Internal Server Error**: Unexpected errors
- **503 Service Unavailable**: Backend service unavailable
- **504 Gateway Timeout**: Backend service timeout

All errors return a consistent format:
```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable error message",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## Monitoring and Logging

The application includes comprehensive logging:
- Request/response logging at Experience API
- Orchestration logging at Process API
- Backend call logging at System API

Log levels can be configured in Anypoint Studio or CloudHub.

## License

This is a sample implementation for educational purposes.