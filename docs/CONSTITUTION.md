# Aura E-Commerce Platform Constitution
**Version:** 1.0.0

> **Authority Declaration:** This constitution is the supreme engineering authority for the Aura E-Commerce Platform. All code, architecture decisions, and operational procedures must comply strictly with the rules defined herein. Ignorance of this document is not an excuse for non-compliance.

*(Assumption: As the project brief, name, shape, and languages were skipped, this constitution assumes the creation of a production-grade, headless e-commerce backend API built with Node.js and TypeScript.)*

## Mission
To provide a highly available, secure, and scalable headless e-commerce backend that powers seamless shopping experiences across all web and mobile client platforms, ensuring data integrity and sub-second response times.

## Core Values
1. **Correctness over speed:** A working, bug-free system is prioritized over meeting arbitrary deadlines.
2. **Security over convenience:** Customer data and payment integrity are paramount; no security shortcuts are permitted.
3. **Simplicity over cleverness:** Code must be readable and maintainable by any engineer on the team.
4. **Maintainability over shortcuts:** Technical debt must be documented and paid down, not accumulated indefinitely.
5. **Observability over assumptions:** If it is not logged and monitored, it does not exist.
6. **Explicitness over magic:** Avoid hidden side-effects and implicit behaviors.
7. **Automation over manual processes:** CI/CD, testing, and deployments must be fully automated.
8. **Testing over trust:** All logic must be provably correct via automated tests.

## Technology Stack

| Category | Required / Approved Technologies |
| :--- | :--- |
| **Primary Language** | TypeScript (Strict Mode) |
| **Runtime** | Node.js (LTS) |
| **API Framework** | Express.js |
| **Database** | PostgreSQL |
| **ORM** | Prisma |
| **Caching** | Redis |
| **Validation** | Zod |
| **Testing** | Jest, Supertest |

**Forbidden Technologies / Practices:**
* Plain JavaScript in application code (TypeScript is mandatory).
* Unmaintained dependencies (libraries without updates in > 12 months).
* Experimental libraries in production without explicit architecture board approval.
* Direct database queries bypassing the ORM (unless explicitly approved for performance).

## Repository Structure
The repository follows a Domain-Driven Design (DDD) module structure to ensure separation of concerns.

```text
aura-ecommerce-api/
├── src/
│   ├── config/           # Environment and app configuration
│   ├── core/             # Shared utilities, middlewares, and base classes
│   │   ├── errors/       # Custom error classes
│   │   ├── logger/       # Logging configuration
│   │   └── types/        # Global TypeScript interfaces
│   ├── modules/          # Domain modules
│   │   ├── cart/         # Cart domain (controllers, services, models)
│   │   ├── catalog/      # Product catalog domain
│   │   ├── checkout/     # Checkout and payment domain
│   │   └── users/        # User management and auth domain
│   ├── app.ts            # Express application setup
│   └── server.ts         # Entry point
├── prisma/               # Database schema and migrations
├── tests/                # E2E and integration tests
├── Dockerfile
├── package.json
└── tsconfig.json
```

## Language/Code Standards
* **Typing:** TypeScript `strict: true` is mandatory. The use of `any` is strictly forbidden. Use `unknown` if the type is truly dynamic, followed by type narrowing.
* **Naming Conventions:**
  * Variables, functions, and methods: `camelCase`
  * Classes, Interfaces, and Types: `PascalCase`
  * Constants and Enum values: `UPPER_SNAKE_CASE`
  * File names: `kebab-case.ts`
* **Size Limits:**
  * Files should target **300 lines** maximum.
  * Files exceeding **500 lines** trigger a mandatory refactor blocking the PR.
  * Functions must not exceed 40 lines of logic.
* **Formatting:** Prettier and ESLint must be used. CI will fail on linting errors.

## Backend/API & Validation Standards
* **Architecture:** RESTful API design principles must be followed.
* **Validation:** All incoming requests (body, query, params) MUST be validated using Zod schemas at the controller boundary before reaching business logic.
* **Response Envelope:** All API responses must follow a standardized JSON envelope.

```json
// Success Response
{
  "success": true,
  "data": { ... },
  "meta": { "pagination": { ... } } // Optional
}

// Error Response
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [ ... ]
  },
  "requestId": "req-12345"
}
```

## Error Handling
Errors must be explicitly thrown and caught by a centralized error-handling middleware. Do not leak stack traces to the client in production.

| Error Category | HTTP Status | Description |
| :--- | :--- | :--- |
| `VALIDATION_ERROR` | 400 | Client provided invalid data (caught by Zod). |
| `AUTHENTICATION_ERROR` | 401 | Missing or invalid authentication credentials. |
| `AUTHORIZATION_ERROR` | 403 | Authenticated user lacks required permissions. |
| `BUSINESS_ERROR` | 409 / 422 | Domain rule violation (e.g., out of stock). |
| `EXTERNAL_SERVICE_ERROR`| 502 | Failure communicating with a third-party API (e.g., payment gateway). |
| `INFRASTRUCTURE_ERROR` | 503 | Database or cache connection failures. |
| `UNKNOWN_ERROR` | 500 | Unhandled exceptions. Triggers immediate PagerDuty alert. |

## Logging
* All logs must be output in structured JSON format to `stdout`.
* **Required Fields:** Every log entry must include: `event`, `timestamp`, `requestId`, `userId` (if authenticated), and `metadata` (contextual data).
* **Levels:** Use `info` for business events, `warn` for retries/degraded states, `error` for exceptions, and `debug` only in development.
* **PII/PHI:** Never log passwords, credit card numbers, or raw authorization tokens.

## Security
* **Authentication & Authorization:** Must occur server-side. JWTs must be verified on every protected route.
* **Secrets Handling:**
  * Secrets must be injected via Environment Variables at runtime.
  * In production, secrets must be fetched from a Secret Manager (e.g., AWS Secrets Manager).
  * **NEVER** commit secrets, `.env` files, or credentials to version control.
* **Dependency Policy:**
  * All dependencies must pass automated security scans (e.g., `npm audit`, Snyk) in CI.
  * Must pass license review (no copyleft licenses like GPL in proprietary code).
  * Must be actively maintained.
  * Prefer building over adding a dependency when the required functionality is trivial (< 50 lines of code).

## Performance
* **API Latency:** 95th percentile (p95) response time must be under 200ms for all read operations.
* **Caching:** The product catalog and pricing endpoints must utilize Redis caching. Cache invalidation strategies must be documented per domain.
* **Database:** N+1 query problems are forbidden. Use Prisma's `include` or explicit batching.

## Testing
* **Minimum Coverage:** 80% minimum overall line coverage; **95% mandatory** for critical business logic (Cart, Checkout, Pricing).
* **Required Test Types:**
  * **Unit Tests:** For all business logic, services, and utilities (mocking DB/external services).
  * **Integration Tests:** For database queries and API endpoints (using a test database).
  * **E2E Tests:** Critical user journeys (e.g., add to cart -> checkout -> payment).

## CI/CD
* **Platform:** GitHub Actions.
* **CI Gates per PR:** Every pull request must successfully pass the following sequential gates:
  1. Linting (ESLint/Prettier)
  2. Typecheck (`tsc --noEmit`)
  3. Unit tests
  4. Integration tests
  5. Security scan
* **Deployment:** Merges to `main` automatically deploy to the Staging environment. Production deployments require manual approval via GitHub Environments.

## Documentation
* **API Documentation:** All endpoints must be documented using OpenAPI (Swagger) 3.0 specifications. The spec must be kept in sync with the code.
* **Architecture:** Significant architectural changes require an Architecture Decision Record (ADR) stored in `docs/adr/`.
* **Setup:** The `README.md` must contain exact, working instructions to spin up the project locally via Docker Compose in under 5 minutes.

## Observability
* **Tracing:** Distributed tracing (OpenTelemetry) must be implemented across all services.
* **Metrics:** Track API request rates, error rates, and latency (RED metrics).
* **Monitoring:** Datadog is the standard for APM, log aggregation, and alerting.

## AI Development Rules
* **AI-generated code policy:** AI code is UNTRUSTED. It must be reviewed, tested, and validated by a human engineer before merging. The author assumes full responsibility for any AI-generated code committed.
* **Agent restrictions (without human approval):**
  * May NOT deploy to production.
  * May NOT rotate credentials.
  * May NOT modify infrastructure.
  * May NOT approve pull requests.

## Prompt/MCP/RAG Standards
* **Prompts:** Must be version-controlled, documented, and tested alongside the code that invokes them. Prompt changes require standard code review.
* **MCP Integrations:** Must operate on the principle of least-privilege, be fully auditable, and easily revocable.
* **RAG Sources:** Must be trusted, versioned, and source-attributed in the output.

## Code Review Standards
Every Pull Request must explicitly answer the following questions in its description:
1. **What changed?** (Brief summary of the technical implementation)
2. **Why?** (Link to Jira ticket or business requirement)
3. **Risks?** (What could break? Security implications?)
4. **Rollback plan?** (How do we revert if this fails in production?)
5. **Testing evidence?** (Screenshots, test output, or explanation of how it was verified)

## Git Standards
* **Branching Strategy:** Trunk-based development with short-lived feature branches.
* **Branch Naming:** `feature/*`, `bugfix/*`, `hotfix/*`, `chore/*`
* **Commit Conventions:** Conventional Commits are mandatory.
  * Types: `feat`, `fix`, `refactor`, `test`, `docs`, `perf`, `chore`.
  * Example: `feat(cart): add support for discount codes`

## Dependency Rules
* Lockfiles (`package-lock.json`) must be committed and strictly respected (`npm ci` in CI/CD).
* Upgrades to major versions of core frameworks (Express, Prisma, TypeScript) require an ADR and dedicated testing sprint.

## Definition of Done
A feature is not considered "Done" until ALL of the following are true:
- [ ] Requirements implemented exactly as specified.
- [ ] Unit and Integration tests written.
- [ ] All tests passing in CI.
- [ ] Typecheck passing.
- [ ] Linting passing.
- [ ] Security review completed (no new vulnerabilities introduced).
- [ ] API Documentation (OpenAPI) updated.
- [ ] Performance validated (no degradation in p95 latency).
- [ ] Code reviewed and approved by at least one senior engineer.

## Non-Negotiable Rules (NEVER / ALWAYS)
* **NEVER** commit secrets, API keys, or `.env` files to the repository.
* **NEVER** bypass CI checks or force-push to the `main` branch.
* **NEVER** use `any` in TypeScript.
* **ALWAYS** validate external input at the system boundary.
* **ALWAYS** write tests for bug fixes to prevent regressions.
* **ALWAYS** leave the codebase cleaner than you found it.

## Amendment Process
This constitution is a living document but requires consensus to change. The process is:
1. **Written proposal:** Submit a PR modifying this document with a detailed rationale.
2. **Architecture review:** The proposal is reviewed by the Principal Engineering group.
3. **Team approval:** Requires a majority upvote from the core engineering team.
4. **Version increment:** Upon merge, the version number at the top of this document must be incremented (SemVer).