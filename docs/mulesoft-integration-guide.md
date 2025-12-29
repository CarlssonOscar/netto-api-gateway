# NettoApi - MuleSoft Integration Guide

> **Purpose**: Reference document for building MuleSoft API Gateway integration  
> **Backend API**: Spring Boot 4.x + Java 21  
> **Last Updated**: 2025-12-29

---

## Quick Start

### OpenAPI Specification

Import the OpenAPI spec directly into Anypoint Platform:

```
GET http://localhost:8080/api/v1/api-docs
```

**Format**: OpenAPI 3.0 JSON

---

## API Endpoints

### Base URL

```
http://localhost:8080/api/v1
```

### Tax Calculation

#### Calculate Net Salary (GET)

```http
GET /api/v1/tax/calculate?municipalityId={uuid}&grossSalary={decimal}&churchMember={boolean}&isPensioner={boolean}
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `municipalityId` | UUID | ✅ Yes | - | Municipality UUID |
| `grossSalary` | decimal | ✅ Yes | - | Monthly gross salary (SEK) |
| `churchMember` | boolean | No | `false` | Swedish Church member |
| `isPensioner` | boolean | No | `false` | 65+ years old |

**Example Request:**
```
GET /api/v1/tax/calculate?municipalityId=550e8400-e29b-41d4-a716-446655440000&grossSalary=50000&churchMember=false&isPensioner=false
```

#### Calculate Net Salary (POST)

```http
POST /api/v1/tax/calculate
Content-Type: application/json
```

**Request Body:**
```json
{
  "municipalityId": "550e8400-e29b-41d4-a716-446655440000",
  "grossMonthlySalary": 50000,
  "churchMember": false,
  "isPensioner": false
}
```

**Response (200 OK):**
```json
{
  "grossMonthlySalary": 50000.00,
  "grossYearlySalary": 600000.00,
  "municipalityName": "Umeå",
  "regionName": "Västerbotten",
  "taxableIncome": 558135.60,
  "basicDeduction": 41864.40,
  "municipalTax": 121673.53,
  "regionalTax": 60622.16,
  "stateTax": 0.00,
  "burialFee": 1548.00,
  "churchFee": 0.00,
  "totalTaxBeforeCredit": 183843.69,
  "jobTaxCredit": 38517.24,
  "totalTaxAfterCredit": 145326.45,
  "yearlyNetSalary": 454673.55,
  "monthlyNetSalary": 37889.46,
  "effectiveTaxRate": 24.22,
  "municipalTaxRate": 21.79,
  "regionalTaxRate": 10.86
}
```

### Municipality Data

#### List All Regions

```http
GET /api/v1/regions
```

**Response (200 OK):**
```json
[
  {
    "id": "uuid",
    "code": "24",
    "name": "Västerbotten"
  }
]
```

#### List All Municipalities

```http
GET /api/v1/municipalities
```

#### Get Municipalities by Region

```http
GET /api/v1/municipalities/by-region/{regionId}
```

#### Get Municipality by ID

```http
GET /api/v1/municipalities/{id}
```

---

## Health Endpoints

### Custom Health Checks

| Endpoint | Purpose | Response |
|----------|---------|----------|
| `GET /api/v1/health` | General health | `{"status": "UP", "timestamp": "..."}` |
| `GET /api/v1/health/ready` | Readiness probe | `{"status": "READY"}` |
| `GET /api/v1/health/live` | Liveness probe | `{"status": "ALIVE"}` |

### Spring Boot Actuator

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/actuator/health` | Actuator health with DB status |
| `GET /api/v1/actuator/info` | Application info |
| `GET /api/v1/actuator/metrics` | Application metrics |

**Recommended for MuleSoft:** Use `/api/v1/actuator/health` for load balancer health checks.

---

## Error Response Schema

All errors follow a consistent JSON structure:

### Standard Error

```json
{
  "timestamp": "2025-12-29T14:30:00.000",
  "status": 400,
  "error": "Bad Request",
  "message": "Description of the error"
}
```

### Validation Error (400)

```json
{
  "timestamp": "2025-12-29T14:30:00.000",
  "status": 400,
  "error": "Validation Failed",
  "messages": [
    "municipalityId: Municipality ID is required",
    "grossMonthlySalary: Gross salary must be a positive value"
  ]
}
```

### Error Codes

| HTTP Status | Scenario |
|-------------|----------|
| `200` | Success |
| `400` | Validation error, missing parameter, malformed JSON |
| `404` | Municipality not found |
| `500` | Internal server error |

---

## Request Validation

### Required Fields (POST)

| Field | Validation | Error Message |
|-------|------------|---------------|
| `municipalityId` | Not null, valid UUID | "Municipality ID is required" |
| `grossMonthlySalary` | Not null, positive number | "Gross salary must be a positive value" |

### Optional Fields

| Field | Type | Default |
|-------|------|---------|
| `churchMember` | boolean | `false` |
| `isPensioner` | boolean | `false` |

---

## MuleSoft Configuration Recommendations

### API Policies

| Policy | Recommendation |
|--------|----------------|
| **Rate Limiting** | 100 requests/minute per client |
| **Client ID Enforcement** | Enable for production |
| **Spike Control** | 10 requests/second max burst |
| **IP Allowlist** | Restrict to known consumers |

### Caching

| Endpoint | Cache TTL | Notes |
|----------|-----------|-------|
| `/api/v1/regions` | 24 hours | Rarely changes |
| `/api/v1/municipalities` | 24 hours | Rarely changes |
| `/api/v1/tax/calculate` | No cache | Real-time calculation |

### Timeouts

| Setting | Recommended Value |
|---------|-------------------|
| Connection timeout | 5 seconds |
| Response timeout | 30 seconds |
| Health check interval | 30 seconds |

### Load Balancer Configuration

```yaml
health_check:
  path: /api/v1/actuator/health
  interval: 30s
  timeout: 5s
  healthy_threshold: 2
  unhealthy_threshold: 3
```

---

## Authentication

The backend API has **no built-in authentication**. Authentication should be handled in MuleSoft:

### Recommended Flow

```
Client → MuleSoft (auth) → Backend API (no auth)
```

### Options

1. **API Key** - Simple, stateless
2. **OAuth 2.0 Client Credentials** - For service-to-service
3. **OAuth 2.0 + JWT** - For end-user authentication

---

## CORS

CORS is **not configured** on the backend. MuleSoft should handle CORS headers for browser clients:

```yaml
cors:
  allow_origins: ["https://your-frontend.com"]
  allow_methods: ["GET", "POST", "OPTIONS"]
  allow_headers: ["Content-Type", "Authorization"]
```

---

## Sample MuleSoft DataWeave

### Transform Request

```dataweave
%dw 2.0
output application/json
---
{
  municipalityId: payload.municipality_id,
  grossMonthlySalary: payload.salary as Number,
  churchMember: payload.church_member default false,
  isPensioner: payload.is_pensioner default false
}
```

### Transform Response

```dataweave
%dw 2.0
output application/json
---
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

---

## Testing

### Verify Backend is Running

```bash
curl http://localhost:8080/api/v1/actuator/health
```

**Expected:**
```json
{"status":"UP"}
```

### Test Calculation

```bash
curl -X POST http://localhost:8080/api/v1/tax/calculate \
  -H "Content-Type: application/json" \
  -d '{"municipalityId":"550e8400-e29b-41d4-a716-446655440000","grossMonthlySalary":50000}'
```

### Test Validation Error

```bash
curl -X POST http://localhost:8080/api/v1/tax/calculate \
  -H "Content-Type: application/json" \
  -d '{"grossMonthlySalary":-1000}'
```

**Expected (400):**
```json
{
  "timestamp": "...",
  "status": 400,
  "error": "Validation Failed",
  "messages": [
    "municipalityId: Municipality ID is required",
    "grossMonthlySalary: Gross salary must be a positive value"
  ]
}
```

---

## Swagger UI

For interactive API testing:

```
http://localhost:8080/api/v1/swagger-ui.html
```

---

## Environment URLs

| Environment | Backend URL |
|-------------|-------------|
| Local | `http://localhost:8080` |
| Dev | TBD |
| Staging | TBD |
| Production | TBD |

---

## Changelog

| Date | Changes |
|------|---------|
| 2025-12-29 | Initial version |

## API Contract Ownership

Backend API:
- Technical, domain-oriented
- Stable but not frontend-optimized
- May change internally without affecting frontend

MuleSoft Public API:
- Frontend-optimized
- Stable and versioned
- May differ in naming, structure and aggregation