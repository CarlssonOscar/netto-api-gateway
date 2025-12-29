# GitHub Copilot Instructions - MuleSoft API Gateway

> **Streamlined version** - detailed standards are in separate reference documents  
> Last Updated: 2025-12-29

---

## 📋 Quick Navigation

**Primary Documentation:**
- 📖 [`docs/mulesoft-integration-guide.md`](../docs/mulesoft-integration-guide.md) - Backend API integration reference
- 🏗️ [Architecture & Design Principles](architecture-principles.md) - MuleSoft three-tier API patterns
- 📐 [Code Quality Standards](code-standards.md) - XML, DataWeave, RAML best practices
- 🐛 [Debugging Guide](debugging-guide.md) - Systematic troubleshooting approach

---

## ⚠️ CRITICAL RULES - Always Apply

### 1. Always Re-read Files
- **Never assume** that a previously shown file is still up to date
- **When modifying a file**: re-read it first, then show the new version with changes
- **Don't rely on memory** or earlier context for file contents

### 2. Think Before Claiming Correctness
- **If told you are wrong**: re-evaluate your answer critically
- **Check** against code, documentation, and established patterns
- **Respond** with facts and reasoning, not assumptions
- **Acknowledge mistakes** when wrong; explain reasoning when correct

### 3. Compare Solutions
- **Analyze** your own answers vs. previous solutions
- **If the old solution is better**: explicitly suggest reverting to it
- **Don't assume** newer is always better

### 4. Gateway is NOT Backend - NO BUSINESS LOGIC
**The API Gateway is an integration layer, NOT a calculation engine.**

✅ **Gateway Responsibilities:**
- Request validation (RAML/APIKit)
- Request/Response transformation (DataWeave)
- Routing between API layers
- Error handling and standardization
- Authentication/Authorization policies

❌ **FORBIDDEN in Gateway:**
- Tax calculations
- Business rule implementation
- Data persistence
- Domain logic

```
Business logic belongs in Backend API → See docs/mulesoft-integration-guide.md
```

---

## 🎯 Core Principles (Quick Reference)

### MuleSoft Three-Tier Architecture
```
┌─────────────────────────────────────┐
│     Experience API (Port 8081)      │  ← Public entry point, RAML validation
├─────────────────────────────────────┤
│       Process API (Port 8082)       │  ← Orchestration, aggregation
├─────────────────────────────────────┤
│       System API (Port 8083)        │  ← Backend integration
└─────────────────────────────────────┘
```

### Layer Separation
- **Experience API**: Client-facing, validates requests, transforms responses
- **Process API**: Orchestrates calls, aggregates data from multiple sources
- **System API**: Integrates with backend systems (Spring Boot API, databases)
- **Full details**: See [architecture-principles.md](architecture-principles.md)

### Code Quality
- **Single Responsibility**: One flow per concern
- **Meaningful Names**: Clear flow and sub-flow naming
- **Reusable Components**: Extract common transformations
- **Full standards**: See [code-standards.md](code-standards.md)

### Debugging
- **Top-Down Approach**: Configuration → Connectivity → Flows → Transformations
- **Never skip levels**: Root cause often in configuration, not DataWeave
- **Full guide**: See [debugging-guide.md](debugging-guide.md)

---

## 🤖 AI Behavior Rules

### General Behavior
- **Explain why** for architectural choices, not only how
- **Challenge layer violations** - business logic must stay in backend
- **Ask for clarification** instead of guessing when requirements are ambiguous
- **Stay focused**: Implement only what has been requested

### Working with Files
- **For large changes**: Describe what changed and why
- **For refactoring**: Explain before/after and reasoning
- **For big changes**: Explain differences clearly

### What NOT to Do (Unless Asked)
- ❌ Add business logic to gateway
- ❌ Extra refactorings
- ❌ Additional logging beyond essentials
- ❌ "Nice to have" improvements
- ❌ Unrelated fixes

---

## 📚 Documentation Standards

### Integration Reference
**Primary Documentation**: [`docs/mulesoft-integration-guide.md`](../docs/mulesoft-integration-guide.md)  
*(Reference for backend API endpoints, request/response formats, error schemas)*

### When to Update Documentation
| **Priority** | **Trigger** |
|--------------|-------------|
| **MUST** | New API layers, endpoint changes, RAML updates, backend contract changes |
| **SHOULD** | Configuration changes, new policies, deployment changes |

### Documentation Awareness
- **For integration questions**: Use `docs/mulesoft-integration-guide.md` as primary reference
- **Propose updates**: If changes affect API contracts
- **Follow patterns**: Established in existing flows unless strong reason to deviate

---

## 🔧 Code Standards (Quick Reference)

### Naming Conventions
```xml
<!-- Mule Flows -->
experience-api-main             <!-- Main entry point flows -->
process-salary-calculation      <!-- Process orchestration flows -->
system-get-employee             <!-- System integration flows -->

<!-- Sub-flows -->
transform-request-to-backend    <!-- Transformation sub-flows -->
handle-validation-error         <!-- Error handling sub-flows -->
```

### DataWeave Conventions
```dataweave
%dw 2.0
output application/json
---
{
  // Use camelCase for output to match backend API
  municipalityId: payload.municipality_id,
  grossMonthlySalary: payload.salary as Number,
  
  // Use default for optional fields
  churchMember: payload.church_member default false,
  isPensioner: payload.is_pensioner default false
}
```

### Essential Rules
- **Comments**: English only, explain "why" not "what"
- **Flows**: Keep thin - validation + routing + transformation only
- **Error Handling**: Consistent error response format
- **Configuration**: Use `config.properties` for environment-specific values

**Full standards**: See [code-standards.md](code-standards.md)

---

## 🐛 Debugging (Quick Reference)

### Before Diving Into Code - Check:
1. ✅ Is `config.properties` loaded correctly?
2. ✅ Are HTTP listener/request configs correct for environment?
3. ✅ Is the backend API running and accessible?
4. ✅ Are ports not conflicting (8081, 8082, 8083)?
5. ✅ Is RAML validation passing?

### Hierarchy (Top → Down)
```
1. Configuration Level (config.properties, HTTP configs)
2. Connectivity Level (Backend reachable? Ports correct?)
3. Flow Level (Routing correct? Flow-refs valid?)
4. Transformation Level (DataWeave correct? Types matching?)
```

**Full guide**: See [debugging-guide.md](debugging-guide.md)

---

## 📝 Change Management

### For Every Code Change
- Implement only what was explicitly requested
- For large changes: explain what, why, and potential side effects
- For API contract changes: update RAML and documentation

### Code Review Mindset
- Challenge any business logic in gateway (belongs in backend)
- If old solution is better: say so clearly and suggest reverting
- Avoid "rewrite everything" unless absolutely necessary

---

## Version Information

**Copilot Instructions Version**: `1.0`  
**Last Updated**: `2025-12-29`  
**Technology Stack**: MuleSoft 4.4.x + APIKit

---

*This instruction file must be followed by both developers and AI assistants. It is a central tool for maintaining architecture, code quality, and living documentation in this project.*
