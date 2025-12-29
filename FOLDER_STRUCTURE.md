# Folder Structure

```
netto-api-gateway/
│
├── pom.xml                                    # Maven project configuration
│                                              # - Mule application packaging
│                                              # - MuleSoft connectors (HTTP, APIKit)
│                                              # - CloudHub deployment config
│
├── README.md                                  # Main documentation
│                                              # - Architecture overview
│                                              # - Setup and running instructions
│                                              # - API usage examples
│
├── ARCHITECTURE.md                            # Detailed architecture docs
│                                              # - Three-tier architecture explanation
│                                              # - Request flow diagrams
│                                              # - Deployment strategies
│
├── .gitignore                                 # Git ignore file
│                                              # - Ignores Maven target directory
│                                              # - Ignores IDE files
│
└── src/
    ├── main/
    │   ├── mule/                              # Mule configuration files
    │   │   │
    │   │   ├── netto-api-gateway.xml          # Main configuration
    │   │   │                                  # - Global configuration
    │   │   │                                  # - Property file references
    │   │   │                                  # - Architecture documentation
    │   │   │
    │   │   ├── experience-api.xml             # Experience API (Gateway Layer)
    │   │   │                                  # PORT: 8081 (PUBLIC)
    │   │   │                                  # - HTTP listener configuration
    │   │   │                                  # - APIKit router for validation
    │   │   │                                  # - POST /salary/calculate endpoint
    │   │   │                                  # - Request validation
    │   │   │                                  # - Response transformation
    │   │   │                                  # - Routing to Process API
    │   │   │                                  # - Error handling
    │   │   │
    │   │   ├── process-api.xml                # Process API (Orchestration Layer)
    │   │   │                                  # PORT: 8082 (INTERNAL)
    │   │   │                                  # - HTTP listener configuration
    │   │   │                                  # - Salary calculation orchestration
    │   │   │                                  # - Calls Employee System API
    │   │   │                                  # - Calls Salary Calculation System API
    │   │   │                                  # - Data aggregation logic
    │   │   │                                  # - Fallback handling
    │   │   │
    │   │   └── system-api.xml                 # System API (Integration Layer)
    │   │                                      # PORT: 8083 (INTERNAL)
    │   │                                      # - HTTP listener configuration
    │   │                                      # - Employee data endpoint (placeholder)
    │   │                                      # - Salary calculation endpoint (placeholder)
    │   │                                      # - Backend integration examples
    │   │                                      # - Mock responses for testing
    │   │
    │   └── resources/
    │       │
    │       ├── api/
    │       │   └── experience-api.raml        # API specification
    │       │                                  # - RAML 1.0 specification
    │       │                                  # - Request/response schemas
    │       │                                  # - Data type definitions
    │       │                                  # - Validation rules
    │       │                                  # - API documentation
    │       │                                  # - Example requests/responses
    │       │
    │       ├── config/
    │       │   └── config.properties          # Configuration properties
    │       │                                  # - HTTP ports for each layer
    │       │                                  # - API endpoints
    │       │                                  # - Timeout configurations
    │       │                                  # - Retry settings
    │       │
    │       └── mule-artifact.json             # Mule application descriptor
    │                                          # - Minimum Mule version
    │                                          # - Configuration file list
    │                                          # - Application metadata
    │
    └── test/
        └── munit/                             # MUnit test directory
                                               # (Tests can be added here)
```

## API Layer Separation

### Experience API (Port 8081) - PUBLIC
- **File**: `experience-api.xml`
- **Entry Point**: `/gateway/v1/*`
- **Access**: External clients (web apps, mobile apps)
- **Purpose**: 
  - Validate requests against RAML
  - Transform responses for clients
  - Route to Process API
  - Handle errors gracefully

### Process API (Port 8082) - INTERNAL
- **File**: `process-api.xml`
- **Entry Point**: `/process/api/*`
- **Access**: Only from Experience API
- **Purpose**: 
  - Orchestrate business processes
  - Call multiple System APIs
  - Aggregate data from different sources
  - Manage process flow

### System API (Port 8083) - INTERNAL
- **File**: `system-api.xml`
- **Entry Point**: `/system/api/*`
- **Access**: Only from Process API
- **Purpose**: 
  - Integrate with backend systems
  - Provide placeholders for real integrations
  - Abstract backend complexity
  - Standardize backend interfaces

## Configuration Flow

```
config.properties
    ↓
netto-api-gateway.xml (loads properties)
    ↓
├── experience-api.xml (uses http.port, process.api.*)
├── process-api.xml (uses process.api.*, system.api.*)
└── system-api.xml (uses system.api.*)
```

## RAML Validation Flow

```
Client Request
    ↓
experience-api.xml (HTTP Listener)
    ↓
APIKit Router
    ↓
experience-api.raml (validation rules)
    ↓
✓ Valid → Continue to flow
✗ Invalid → Return 400 Bad Request
```

## Data Flow for Salary Calculation

```
1. Client
   POST /gateway/v1/salary/calculate
   └─> experience-api.xml (Port 8081)

2. Experience API validates and routes
   POST /process/api/salary/calculate
   └─> process-api.xml (Port 8082)

3. Process API orchestrates
   ├─> POST /system/api/employee/get
   │   └─> system-api.xml (Port 8083)
   │       └─> Returns employee data (mock or real)
   │
   └─> POST /system/api/salary/calculate
       └─> system-api.xml (Port 8083)
           └─> Returns calculation (mock or real)

4. Process API aggregates results
   └─> Returns to Experience API

5. Experience API transforms
   └─> Returns to Client
```

## Key Files Purpose

| File | Purpose | Contains Business Logic? |
|------|---------|-------------------------|
| `pom.xml` | Maven build configuration | ❌ No |
| `experience-api.raml` | API contract & validation | ❌ No |
| `experience-api.xml` | Gateway implementation | ❌ No |
| `process-api.xml` | Orchestration logic | ❌ No |
| `system-api.xml` | Backend integration | ❌ No (just integration) |
| `config.properties` | Configuration values | ❌ No |
| Backend Systems | Actual implementations | ✅ YES |

**Note**: Business logic (tax calculations, salary rules) resides in backend systems, not in the gateway.
