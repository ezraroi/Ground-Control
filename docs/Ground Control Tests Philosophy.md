# Ground Control Tests Philosophy

## Core Principle

> **We don't write tests to write tests — we test real logic.**

Tests exist to give us confidence that our code works correctly. Every test should protect against a real failure mode or validate meaningful behavior. If a test doesn't catch bugs or document important behavior, it's ceremony — not value.

---

## The Testing Diamond (Not Pyramid)

Ground Control follows a **testing diamond** approach — integration tests are our primary confidence layer:

```
        ╱╲
       ╱  ╲         E2E (rare, critical paths only)
      ╱    ╲
     ╱      ╲
    ╱────────╲
   ╱          ╲     Integration Tests (API + DB) ← PRIMARY
  ╱            ╲
 ╱──────────────╲
╱                ╲
╲                ╱   Unit Tests (when logic is complex/pure)
 ╲──────────────╱
```

### Why Diamond Over Pyramid?

| Pyramid Approach | Diamond Approach (Ours) |
|------------------|-------------------------|
| Many unit tests, few integration | Many integration tests, selective unit |
| Mocks everywhere | Real dependencies (Testcontainers) |
| Fast but fragile confidence | Slower but real confidence |
| Tests implementation details | Tests actual behavior |

**Our belief:** Integration tests catch real bugs. Unit tests are valuable when logic is complex and isolated, but testing through mocks often just tests that we configured mocks correctly.

| Layer | Purpose | When to Use |
|-------|---------|-------------|
| **Integration** | Test real flows through API + DB | **Default choice** — most tests |
| **Unit** | Test pure logic with complex branching | When logic is isolated, algorithmic, or has many edge cases |
| **E2E** | Validate critical user journeys | Sparingly, for smoke tests |

---

## What We Test (and Don't Test)

### ✅ DO Test

| What | Why | Test Type |
|------|-----|-----------|
| **API contracts** | Frontend depends on these, real confidence | Integration |
| **Database queries** | Complex joins, filters, edge cases | Integration |
| **Multi-component flows** | Controller → service → repository | Integration |
| **Error responses** | 404, 422, 500 — clients handle these | Integration |
| **URL/string parsing** | Many edge cases, pure function | Unit |
| **Algorithmic logic** | Scoring, matching, transformations | Unit |
| **Configuration validation** | Bad config = runtime crashes | Unit |

### ❌ DON'T Test

| What | Why |
|------|-----|
| **Third-party libraries** | Trust `@gitbeaker`, `prisma`, etc. to work |
| **Simple getters/setters** | No logic = no bugs |
| **Constructor wiring** | Covered by integration tests implicitly |
| **Private methods directly** | Test through public API |
| **Log statements** | Observability, not correctness |
| **Framework behavior** | Fastify routing works; don't test it |
| **Mocked behavior** | Testing mocks tests the mock, not the code |

---

## Test Infrastructure

### Testcontainers (Real Dependencies)

We use **Testcontainers** to spin up real PostgreSQL and Redis instances for tests. This gives us:

- **Confidence**: Tests run against real databases, not mocks
- **Isolation**: Each test run gets fresh containers
- **Parity**: Test environment matches production

```typescript
// test/setup/global-setup.ts
// Starts containers ONCE before all tests, tears down after
export async function setup() {
  postgresContainer = await new PostgreSqlContainer('postgres:16-alpine').start();
  redisContainer = await new RedisContainer('redis:7-alpine').start();
  // Run Prisma migrations
  execSync('npx prisma db push --skip-generate');
}
```

### Test App Builder

Integration tests use a shared test app builder that:

- Connects to testcontainer databases
- Loads real config (pillars, rulepacks)
- Skips background workers (catalog sync, scan worker)
- Provides a clean `inject()` API for HTTP requests

```typescript
// test/helpers/test-app.ts
const ctx = await buildTestApp();
const res = await ctx.app.inject({ method: 'GET', url: '/services' });
```

### Database Reset

Each test starts with a clean database:

```typescript
beforeEach(async () => {
  await resetDatabase(ctx.db);
});
```

This ensures test isolation without container restart overhead.

---

## Test File Organization

```
apps/api/test/
├── setup/
│   ├── global-setup.ts    # Testcontainer lifecycle
│   └── test-env.ts        # Env loader for tests
├── helpers/
│   ├── test-app.ts        # Fastify test app builder
│   ├── db.ts              # Database reset utilities
│   └── factories.ts       # Test data factories
├── unit/
│   ├── git/               # GitCollector unit tests
│   │   ├── gitlab-client.test.ts
│   │   ├── gitlab-provider.test.ts
│   │   ├── git-provider-registry.test.ts
│   │   └── git-collector.test.ts
│   └── catalog-sync-service.test.ts
└── integration/
    ├── services.test.ts   # /services API
    ├── teams.test.ts      # /teams API
    ├── leaderboard.test.ts
    └── scans.test.ts
```

---

## Test Factories

We use **factory functions** to create test data with sensible defaults:

```typescript
// Minimal: just the required fields
const team = createTeam();

// Customized: override specific fields
const team = createTeam({ name: 'platform-team', title: 'Platform Team' });

// Dependent: reference other entities
const service = createService({ ownerTeamId: team.id });
```

Factories generate unique IDs automatically, preventing collision between tests.

---

## Test Style Guide

### Naming

Tests should read like specifications:

```typescript
// ✅ Good: describes behavior
it('returns NOT_APPLICABLE when service has no repoUrl', async () => { ... });
it('filters services by teamId', async () => { ... });

// ❌ Bad: describes implementation
it('calls parseProjectPath', async () => { ... });
it('test service filter', async () => { ... });
```

### Structure (Arrange-Act-Assert)

```typescript
it('returns services filtered by lifecycle', async () => {
  // Arrange
  const team = await ctx.db.group.create({ data: createTeam() });
  await ctx.db.service.create({ 
    data: createService({ ownerTeamId: team.id, lifecycle: 'production' }) 
  });
  await ctx.db.service.create({ 
    data: createService({ ownerTeamId: team.id, lifecycle: 'experimental' }) 
  });

  // Act
  const res = await ctx.app.inject({
    method: 'GET',
    url: '/services?lifecycle=production',
  });

  // Assert
  expect(res.statusCode).toBe(200);
  expect(res.json().data).toHaveLength(1);
});
```

### Mocking Strategy

> **Prefer real dependencies. Mock only what you can't control.**

| Scenario | Approach |
|----------|----------|
| **Database** | ✅ Real DB via Testcontainers — never mock |
| **Redis** | ✅ Real Redis via Testcontainers — never mock |
| **External APIs** (GitLab, Backstage) | Mock at boundary (client level) |
| **Time** | Use `vi.useFakeTimers()` when testing time-dependent logic |
| **Random values** | Seed or mock when determinism is required |

**When mocking external APIs:**

```typescript
// Mock at the CLIENT level, not the provider/collector
// This keeps the real logic intact, only stubbing the HTTP layer
vi.mock('../../../src/engine/collectors/git/providers/gitlab/gitlab-client.js', () => ({
  GitLabClient: vi.fn().mockImplementation(() => ({
    listRepositoryTree: vi.fn().mockResolvedValue([{ path: 'README.md' }]),
    getProject: vi.fn().mockResolvedValue({ defaultBranch: 'main' }),
  })),
}));
```

**Why we don't mock the database:**
- Prisma queries are complex — mocking them is fragile
- Testcontainers startup is fast enough (~3-5s once per test run)
- Real queries catch real bugs (schema mismatches, constraint violations)

---

## When to Write Each Test Type

### Integration Tests (Default Choice)

**Start here.** Write integration tests for:

- **API endpoints** — request/response contracts, status codes, pagination
- **Database behavior** — queries, filters, constraints, relationships
- **Multi-component flows** — controller → service → repository
- **Real-world scenarios** — how users actually use the system

**Examples in Ground Control:**

- `GET /services` with filters and pagination
- `POST /scans` triggers async job correctly
- Leaderboard aggregation returns correct rankings
- Service snapshot includes all expected fields

### Unit Tests (When Logic Demands It)

Write unit tests **only when**:

- Logic is **pure** and **algorithmic** (same input → same output)
- Function has **many edge cases** that are tedious to test via integration
- Code is **deeply nested or branching** (complex conditionals)
- Testing through integration would be **slow or awkward**

**Examples in Ground Control:**

- `parseProjectPath()` — URL parsing with 10+ edge cases
- `canHandle()` — URL pattern matching (fast, pure, many cases)
- `resolve()` — provider selection order matters
- Scoring algorithms — mathematical correctness

### Skip Tests When

- Code is **trivial wiring** (DI constructor, exports)
- Behavior is **covered by integration tests** already
- Testing would require **complex mocking** that just tests the mock
- You're testing **framework behavior** (Fastify routing works)

---

## Running Tests

```bash
# Run all tests
npm test

# Run unit tests only
npm test -- --run test/unit/

# Run specific test file
npm test -- --run test/unit/git/gitlab-client.test.ts

# Run with debug output
DEBUG=true npm test

# Watch mode (development)
npm test -- --watch
```

---

## Coverage Philosophy

We don't chase coverage percentages. Coverage is a **side effect** of good testing, not a goal.

**Good coverage targets:**

- Business logic: high coverage (80%+)
- API controllers: medium coverage (contracts, error cases)
- Wiring/glue code: low coverage (integration tests cover implicitly)

**Red flags:**

- 100% coverage with meaningless tests
- Low coverage on critical paths
- Tests that test mocks, not behavior

---

## Summary

| Principle | Practice |
|-----------|----------|
| **Diamond over Pyramid** | Integration tests first, unit tests when logic is complex |
| **Real dependencies** | Testcontainers for DB/Redis, mock only external APIs |
| **Confidence over speed** | Integration tests are slower but catch real bugs |
| **Unit when pure** | Reserve unit tests for algorithmic/pure logic with many edge cases |
| **Readable tests** | Behavior-driven names, AAA structure, factories |
| **Pragmatic coverage** | High on API contracts, selective on internal logic |

> Tests are documentation that runs. Integration tests document how the system actually works. Unit tests document algorithmic edge cases.
