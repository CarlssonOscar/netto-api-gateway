# Netto API Gateway

A MuleSoft-based API Gateway implementing a three-tier architecture following MuleSoft best practices.

> **Demo Note**: This gateway is designed but requires MuleSoft Anypoint Studio to run. For demo purposes, connect your frontend directly to the backend API at `http://localhost:8080/api/v1`. The gateway design demonstrates API-led connectivity architecture.

## Documentation

| Document | Purpose |
|----------|---------|
| [System Overview](docs/system-overview.md) | Architecture, design decisions, how the gateway works |
| [Frontend Integration Guide](docs/frontend-integration-guide.md) | API reference for frontend development |
| [MuleSoft Integration Guide](docs/mulesoft-integration-guide.md) | Backend API details for gateway implementation |

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
│  Connects to backend Spring Boot API:                   │
│  • POST /api/v1/tax/calculate                           │
│  • GET /api/v1/regions                                  │
│  • GET /api/v1/municipalities                           │
│                                                         │
│  NOT directly accessible from external clients          │
└─────────────────────────────────────────────────────────┘
                     │ HTTP
                     ▼
┌─────────────────────────────────────────────────────────┐
│              BACKEND API (Spring Boot)                  │
│                    Port: 8080                           │
│                                                         │
│  Responsibilities:                                      │
│  • Swedish tax calculation (business logic)             │
│  • Municipality and region data                         │
│  • Data persistence                                     │
│                                                         │
│  For demo: Connect frontend directly here               │
│  Base URL: http://localhost:8080/api/v1                 │
└─────────────────────────────────────────────────────────┘
```

## Project Structure

```
netto-api-gateway/
├── pom.xml                                 # Maven project configuration
├── docs/
│   ├── frontend-integration-guide.md       # Frontend API reference
│   └── mulesoft-integration-guide.md       # Backend API reference
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
- Routes all requests to Process API
- Does NOT contain business logic

**Endpoints**:
- `POST /gateway/tax/calculate` - Calculate Swedish net salary with full tax breakdown
- `GET /gateway/regions` - List all Swedish regions (län)
- `GET /gateway/municipalities` - List all municipalities (kommuner)
- `GET /gateway/municipalities/by-region/{regionId}` - Get municipalities in a region
- `GET /gateway/municipalities/{id}` - Get a specific municipality

**Example Request** (Tax Calculation):
```bash
curl -X POST http://localhost:8081/gateway/tax/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "municipalityId": "550e8400-e29b-41d4-a716-446655440000",
    "grossMonthlySalary": 50000,
    "churchMember": false,
    "isPensioner": false
  }'
```

**Example Response**:
```json
{
  "grossMonthlySalary": 50000.00,
  "grossYearlySalary": 600000.00,
  "municipalityName": "Umeå",
  "regionName": "Västerbotten",
  "monthlyNetSalary": 37889.46,
  "yearlyNetSalary": 454673.55,
  "effectiveTaxRate": 24.22,
  "municipalTax": 121673.53,
  "regionalTax": 60622.16,
  "stateTax": 0.00,
  "jobTaxCredit": 38517.24
}
```

### 2. Process API (Port 8082)

**Purpose**: Orchestrate business processes and coordinate System API calls.

**Key Features**:
- Orchestrates calls to System APIs
- Passes through data (no transformation needed)
- NOT directly accessible from external clients

**Flow for Tax Calculation**:
1. Receives request from Experience API
2. Calls System API for tax calculation
3. Returns response to Experience API

### 3. System API (Port 8083)

**Purpose**: Integrate with the backend Spring Boot API.

**Key Features**:
- Connects to backend at `http://localhost:8080/api/v1`
- Abstracts backend system complexity
- Provides standardized interfaces
- NOT directly accessible from external clients

**Backend Endpoints Called**:
- `POST /api/v1/tax/calculate` - Swedish net salary calculation
- `GET /api/v1/regions` - List all regions
- `GET /api/v1/municipalities` - List all municipalities
- `GET /api/v1/municipalities/by-region/{regionId}` - Municipalities by region
- `GET /api/v1/municipalities/{id}` - Single municipality

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

# Backend API Configuration (Spring Boot)
backend.api.host=localhost
backend.api.port=8080
backend.api.basePath=/api/v1
backend.api.connection.timeout=5000
backend.api.response.timeout=30000

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

### Option 1: Using Anypoint Studio (Recommended)
1. Download and install [Anypoint Studio](https://www.mulesoft.com/platform/studio) (free)
2. Import the project: File → Import → Anypoint Studio → Maven-based Mule Project
3. Right-click on the project → Run As → Mule Application

### Option 2: Demo Mode (Direct Backend Access)
For demos and development, skip the gateway and connect directly to the backend:

```
Backend URL: http://localhost:8080/api/v1
Swagger UI:  http://localhost:8080/api/v1/swagger-ui.html
```

See [Frontend Integration Guide](docs/frontend-integration-guide.md) for API details.

### Option 3: Deploy to CloudHub
```bash
mvn clean deploy -DmuleDeploy \
  -Danypoint.username=<your-username> \
  -Danypoint.password=<your-password>
```

## Testing

### Test via Gateway (requires Anypoint Studio)

```bash
# Calculate net salary
curl -X POST http://localhost:8081/gateway/tax/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "municipalityId": "550e8400-e29b-41d4-a716-446655440000",
    "grossMonthlySalary": 50000
  }'

# Get all regions
curl http://localhost:8081/gateway/regions

# Get municipalities in a region
curl http://localhost:8081/gateway/municipalities/by-region/{regionId}
```

### Test via Backend Directly (for demos)

```bash
# Calculate net salary
curl -X POST http://localhost:8080/api/v1/tax/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "municipalityId": "550e8400-e29b-41d4-a716-446655440000",
    "grossMonthlySalary": 50000
  }'

# Get all regions
curl http://localhost:8080/api/v1/regions

# Interactive testing
open http://localhost:8080/api/v1/swagger-ui.html
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
4. **Add backend call in System API** (`src/main/mule/system-api.xml`)

### Connecting to a Different Backend

Update `src/main/resources/config/config.properties`:

```properties
backend.api.host=your-backend-host
backend.api.port=8080
backend.api.basePath=/api/v1
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

---

## Related Projects

- **NettoApi** - Spring Boot backend with Swedish tax calculation logic
- **NettoFrontend** - Frontend application (planned)