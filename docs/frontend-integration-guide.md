# Frontend Integration Guide (API Gateway)

> **Purpose**: API reference for frontend developers integrating via the MuleSoft API Gateway  
> **Base URL**: `http://localhost:8081/gateway`  
> **Last Updated**: 2025-12-30
>
> **Note**: For direct backend access (demos without MuleSoft), see the backend project's integration guide.

---

## Quick Start

### Calculate Net Salary

```javascript
const response = await fetch('http://localhost:8081/gateway/tax/calculate', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    municipalityId: '550e8400-e29b-41d4-a716-446655440000',
    grossMonthlySalary: 50000,
    churchMember: false,
    isPensioner: false
  })
});

const result = await response.json();
console.log(`Net salary: ${result.monthlyNetSalary} SEK`);
```

---

## API Endpoints

### POST /gateway/tax/calculate

Calculate net salary with full Swedish tax breakdown.

**Request Body:**

```json
{
  "municipalityId": "550e8400-e29b-41d4-a716-446655440000",
  "grossMonthlySalary": 50000,
  "churchMember": false,
  "isPensioner": false
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `municipalityId` | UUID | ✅ | Municipality identifier |
| `grossMonthlySalary` | number | ✅ | Monthly salary before tax (SEK) |
| `churchMember` | boolean | ❌ | Swedish Church member (default: false) |
| `isPensioner` | boolean | ❌ | Age 65+ (default: false) |

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

---

### GET /gateway/regions

List all Swedish regions (län).

**Response:**

```json
[
  {
    "id": "uuid",
    "code": "24",
    "name": "Västerbotten"
  }
]
```

---

### GET /gateway/municipalities

List all municipalities in Sweden.

**Response:**

```json
[
  {
    "id": "uuid",
    "code": "2480",
    "name": "Umeå",
    "municipalTaxRate": 21.79,
    "regionId": "uuid",
    "regionName": "Västerbotten"
  }
]
```

---

### GET /gateway/municipalities/by-region/{regionId}

Get municipalities in a specific region.

**Parameters:**
- `regionId` (path) - UUID of the region

---

### GET /gateway/municipalities/{id}

Get a single municipality by ID.

**Parameters:**
- `id` (path) - UUID of the municipality

---

## Error Handling

### Error Response Format

```json
{
  "timestamp": "2025-12-30T10:00:00.000",
  "status": 400,
  "error": "Validation Failed",
  "messages": [
    "municipalityId: Municipality ID is required"
  ]
}
```

### HTTP Status Codes

| Status | Meaning | Action |
|--------|---------|--------|
| 200 | Success | Use response data |
| 400 | Validation error | Check request fields |
| 404 | Municipality not found | Verify municipalityId |
| 500 | Server error | Retry or contact support |

---

## Vue Integration Example

### Composable

```typescript
// composables/useTaxCalculation.ts
import { ref } from 'vue'

interface TaxRequest {
  municipalityId: string
  grossMonthlySalary: number
  churchMember?: boolean
  isPensioner?: boolean
}

interface TaxResult {
  grossMonthlySalary: number
  monthlyNetSalary: number
  effectiveTaxRate: number
  municipalTax: number
  regionalTax: number
  stateTax: number
  jobTaxCredit: number
  // ... other fields
}

export function useTaxCalculation() {
  const loading = ref(false)
  const error = ref<string | null>(null)
  const result = ref<TaxResult | null>(null)

  const calculate = async (request: TaxRequest) => {
    loading.value = true
    error.value = null
    
    try {
      const response = await fetch('http://localhost:8081/gateway/tax/calculate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(request)
      })
      
      if (!response.ok) {
        const err = await response.json()
        throw new Error(err.messages?.join(', ') || err.message)
      }
      
      result.value = await response.json()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error'
    } finally {
      loading.value = false
    }
  }

  return { calculate, loading, error, result }
}
```

### Component Example

```vue
<!-- components/SalaryCalculator.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { useTaxCalculation } from '@/composables/useTaxCalculation'

const { calculate, loading, error, result } = useTaxCalculation()

const municipalityId = ref('')
const salary = ref<number>(0)
const churchMember = ref(false)
const isPensioner = ref(false)

const handleSubmit = () => {
  calculate({
    municipalityId: municipalityId.value,
    grossMonthlySalary: salary.value,
    churchMember: churchMember.value,
    isPensioner: isPensioner.value
  })
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <!-- Form fields -->
    
    <p v-if="loading">Calculating...</p>
    <p v-if="error" class="error">{{ error }}</p>
    
    <div v-if="result" class="result">
      <h3>Monthly Net Salary: {{ result.monthlyNetSalary.toLocaleString() }} SEK</h3>
      <p>Effective Tax Rate: {{ result.effectiveTaxRate }}%</p>
    </div>
  </form>
</template>
```

---

## Municipality Dropdown

```typescript
// composables/useMunicipalities.ts
import { ref, onMounted } from 'vue'

interface Municipality {
  id: string
  name: string
  code: string
  municipalTaxRate: number
  regionName: string
}

export function useMunicipalities() {
  const municipalities = ref<Municipality[]>([])
  const loading = ref(true)

  onMounted(async () => {
    const response = await fetch('http://localhost:8081/gateway/municipalities')
    municipalities.value = await response.json()
    loading.value = false
  })

  return { municipalities, loading }
}
```

```vue
<!-- components/MunicipalitySelect.vue -->
<script setup lang="ts">
import { useMunicipalities } from '@/composables/useMunicipalities'

const modelValue = defineModel<string>()
const { municipalities, loading } = useMunicipalities()
</script>

<template>
  <select v-model="modelValue" :disabled="loading">
    <option v-if="loading" value="">Loading...</option>
    <template v-else>
      <option value="">Select municipality...</option>
      <option v-for="m in municipalities" :key="m.id" :value="m.id">
        {{ m.name }} ({{ m.regionName }}) - {{ m.municipalTaxRate }}%
      </option>
    </template>
  </select>
</template>
```

---

## Development Setup

### CORS

The backend allows CORS from localhost during development. For production, configure appropriate origins.

### Health Check

```javascript
// Check if gateway is running (requires MuleSoft runtime)
// For backend health, use port 8080 directly
const health = await fetch('http://localhost:8080/api/v1/actuator/health');
const status = await health.json();
console.log(status); // { "status": "UP" }
```

---

## Response Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `grossMonthlySalary` | number | Input: Monthly salary before tax |
| `grossYearlySalary` | number | Yearly gross (monthly × 12) |
| `municipalityName` | string | Name of selected municipality |
| `regionName` | string | Name of region (län) |
| `taxableIncome` | number | Income after basic deduction |
| `basicDeduction` | number | Grundavdrag |
| `municipalTax` | number | Kommunalskatt (yearly) |
| `regionalTax` | number | Landstingsskatt (yearly) |
| `stateTax` | number | Statlig skatt (yearly) |
| `burialFee` | number | Begravningsavgift (yearly) |
| `churchFee` | number | Kyrkoavgift if member (yearly) |
| `totalTaxBeforeCredit` | number | Sum of all taxes |
| `jobTaxCredit` | number | Jobbskatteavdrag (yearly) |
| `totalTaxAfterCredit` | number | Final tax amount |
| `yearlyNetSalary` | number | Net income per year |
| `monthlyNetSalary` | number | Net income per month |
| `effectiveTaxRate` | number | Percentage of gross paid in tax |
| `municipalTaxRate` | number | Municipal tax rate % |
| `regionalTaxRate` | number | Regional tax rate % |

---

## Tips

1. **Cache municipality data** - It rarely changes, fetch once on app load
2. **Show loading states** - Tax calculation takes ~100-500ms
3. **Format currency** - Use `toLocaleString('sv-SE')` for Swedish formatting
4. **Validate input** - Ensure salary is positive before sending

---

*Last Updated: 2025-12-30*
