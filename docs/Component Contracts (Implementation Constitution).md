# Component Contracts (Implementation Constitution)

> **Foundation Documents:** This specification defines component-level contracts for Ground Control implementation. For immutable core definitions, see the [Constitution](./Ground%20Control%20Constitution.md). For product guidance, see the [Product Definition](./Product%20Definition_%20Ground%20Control%20-%20V4.md). For technical architecture, see the [Technical Specification](./Master%20Technical%20Design%20Specification_%20Ground%20Control%20Platform_v3.md).

---

# Orchestrator (Engine Core)

### **Description** The immutable pipeline coordinator that runs Collection → Evaluation → Enrichment → Persistence. It is the "CPU" of Ground Control.

### Role & Responsibilities

* Loads active Pillars/Rulepacks config.  
* Requests a context from CollectorRegistry.  
* **Evaluates rulepack applicability gates** (`applicableWhen`) before running checks.
* Executes rulepacks by dispatching checks via CheckRegistry.  
* Emits raw results into Enricher to produce EvaluationSnapshot.  
* Persists snapshots and scan status.

> **Note:** Spotlight decoration is applied at view-time by SpotlightDecorator and NextBestUpgradeSelector (see View Layer sections). The Orchestrator does not resolve or pass spotlights.

### Design Guidelines

* Keep it "dumb": no domain logic, no thresholds, no narrative.  
* Never hardcode check IDs, collector keys, or scoring rules.  
* Fail-safe: no collector/check failure may crash a scan.  
* Support concurrency safely (stateless, idempotent).
* Honor `applicableWhen` gates: if context is `NOT_APPLICABLE`, skip entire rulepack (all checks become `NOT_APPLICABLE`).
* **Baseline is rulepack-owned:** Each rulepack declares `badge.baseline` in its YAML. The orchestrator does not resolve baselines — the enricher reads them directly from rulepack config.

### Must Not Break

* **OCP:** new checks/rulepacks must not require modifications here.  
* **Truth/Meaning separation:** orchestrator must not attach narrative.  
* **Universal resilience:** scan completes even with partial context.
* **Applicability contracts:** `applicableWhen` must be evaluated before checks run.

---

# CollectorRegistry (Universal Collection Runner)

### **Description** Runs all registered collectors in parallel and produces a resilient EvaluationContext.

### Role & Responsibilities

* Executes collectors via `Promise.allSettled`.  
* Writes results into ContextRegistry (`ctx`) by key.  
* Records failures and N/A into `_errors` (COLLECTOR\_FAILED vs NOT\_APPLICABLE).  
* Enforces collection timeout and safe defaults.

### Design Guidelines

* Each collector must be isolated; one failure never blocks others.  
* Standardize error metadata (kind \+ message).  
* Keep deterministic behavior: same inputs should produce the same keys.

### Must Not Break

* **Graceful degradation:** never throw; always return a context.  
* **Signal integrity:** collector failure must be explicit (not silent).  
* **NOT\_APPLICABLE correctness:** absence-by-design must not appear as failure.

---

# ContextRegistry (Typed Context Store)

### **Description** The typed container for collected data. The only access point for checks.

### Role & Responsibilities

* Stores per-key context payloads (`git`, `k8s`, `datadog`, `sonic`, etc.).  
* Provides typed access: `ctx.get<GitData>('git')`.  
* Cooperates with `_errors` semantics (missing contexts are explainable).

### Design Guidelines

* Keep API minimal: `get()`, optionally `has()` (but avoid extra coupling).  
* Do not leak internal storage; treat it as immutable after collection.

### Must Not Break

* **Typed usage:** checks depend on `ctx.get<T>` being safe and consistent.  
* **No implicit magic:** absence must remain representable and explainable.

---

# Collector (e.g., GitCollector, K8sCollector, DatadogCollector)

### **Description** A plugin that knows how to gather a domain's "reality" and return a typed context object.

### Role & Responsibilities

* Fetch domain signals (repo tree, k8s manifests/metrics, observability signals).  
* Normalize results into a strict typed payload.  
* Decide and emit: success vs COLLECTOR\_FAILED vs NOT\_APPLICABLE.

### Design Guidelines

* IO happens here (not in checks).  
* Prefer caching and bounded queries; be rate-limit aware.  
* Emit NOT\_APPLICABLE deliberately when integration is absent by design.

### Must Not Break

* **Universal collection contract:** must not throw; must report failures via registry.  
* **No business meaning:** no scoring, no narrative, no thresholds.  
* **Isolation:** A timeout in the `DatadogCollector` must **never** prevent the `GitCollector` from returning data.

---

# GitProviderRegistry (Git Provider Selection)

### **Description** Runtime registry that holds git providers and selects the appropriate one based on repository URL.

### Role & Responsibilities

* Register git providers (GitLab, GitHub, etc.) at startup.
* Resolve the correct provider for a given repository URL.
* Enforce provider ID uniqueness.
* Support provider priority via registration order (first match wins).

### Design Guidelines

* Keep API minimal: `register()`, `resolve()`, `getProviders()`, `hasProviders()`.
* Fail-fast on duplicate provider IDs.
* Return `undefined` when no provider matches (let collector decide NOT_APPLICABLE).

### Must Not Break

* **OCP:** adding new providers = register only, no registry code changes.
* **Determinism:** same URL must resolve to same provider across calls.
* **No business logic:** registry is a pure lookup table, no scoring or evaluation.

---

# GitProvider (e.g., GitLabProvider, GitHubProvider)

### **Description** A provider implementation that knows how to talk to a specific git hosting service.

### Role & Responsibilities

* Implement `GitProvider` interface: `canHandle()`, `listFiles()`, `getMetadata()`.
* Determine if it can handle a given repository URL via `canHandle()`.
* Normalize provider-specific API responses to `GitData` shape.
* Encapsulate authentication and API client configuration.

### Design Guidelines

* `canHandle()` must be fast and side-effect-free (no API calls).
* Use URL pattern matching based on configured host.
* Delegate actual API calls to a dedicated client class.
* Handle both HTTPS and SSH URL formats.

### Must Not Break

* **Single responsibility:** provider handles one git hosting service.
* **No collector logic:** provider does not decide PASS/FAIL, only fetches data.
* **Error transparency:** API errors must propagate cleanly for collector to classify.

> **Note:** The original design described a separate Adapter layer between collectors and providers. In practice, providers encapsulate both provider logic and API abstraction directly via the `GitProvider` interface. This is a deliberate simplification. When GitHub support is added, a `GitHubProvider` follows the same `GitProvider` interface — no separate adapter layer is needed.

---

# GitLabClient (GitLab API Wrapper)

### **Description** Thin wrapper around `@gitbeaker/rest` that provides typed access to GitLab API.

### Role & Responsibilities

* Configure `@gitbeaker` client with host URL and token.
* Provide typed methods: `listRepositoryTree()`, `getProject()`, `getLatestCommit()`.
* Parse repository URLs to extract project paths.
* Handle pagination for large repositories.

### Design Guidelines

* Keep methods focused on data retrieval (no business logic).
* Normalize `@gitbeaker` types to simpler internal types.
* URL parsing logic (`parseProjectPath`) must handle HTTPS, SSH, nested groups.

### Must Not Break

* **Type safety:** always return typed data, never raw API responses.
* **Isolation:** client failures must not crash the collector.
* **Testability:** URL parsing logic must be pure and unit-testable.

---

# CheckRegistry

### **Description** The runtime mapping of `checkId -> DefinedCheck`. It is the dispatch table for YAML rulepacks.

### Role & Responsibilities

* Loads all DefinedCheck plugins at startup.  
* Provides lookup and execution hooks to the engine.

### Design Guidelines

* Fast lookup (Map).  
* Ensure uniqueness of check IDs (fail-fast at startup).  
* Encourage domain grouping by files/modules, but registration remains atomic.

### Must Not Break

* **YAML contract:** `checkId` must always resolve or fail predictably (config error).  
* **OCP:** new checks must be register-only, no core changes.

---

# DefinedCheck (Check Implementation Unit)

### **Description** A reusable, typed measurement unit: `(EvaluationContext, params) -> MeasurementResult`.

> **CRITICAL: TRUTH vs MEANING SEPARATION**
> Check = TRUTH: Measures and returns observed facts (observedValue, displayValue).
> Checks NEVER decide pass/fail. They only measure. Status is computed by Enricher applying thresholds from YAML.

### Role & Responsibilities

* Validate rule parameters via `schema`.  
* Measure using only context data (no IO).  
* Return observedValue + displayValue (MEASUREMENT ONLY, no status/judgment).
* Declare `windowType`: STATIC (point-in-time) or ROLLING (time/event window).
* **Format `displayValue` for human readability** (the check owns display logic).
* Return error info when measurement cannot be completed (COLLECTOR_FAILED or NOT_APPLICABLE).

### Design Guidelines

* Prefer generic parameterized checks for common patterns (file exists, threshold, regex).  
* Return explainable results: observedValue + displayValue.  
* Treat missing context explicitly via error field:  
  * COLLECTOR\_FAILED → error with missingContextKey  
  * NOT\_APPLICABLE → error kind NOT\_APPLICABLE (neutral)  
* Avoid state; checks must be safe under concurrency.
* **displayValue formatting:**
  * Use human-readable values: `6m 30s` not `390` seconds
  * Include context: show thresholds when relevant
  * Counts with totals: `18 of 20` not just `18`
  * Percentages with meaning: `90%` with what it represents

### Must Not Break

* **No judgment:** Check must NOT return status (PASS/FAIL). That's the Enricher's job.
* **windowType required:** Every check must declare STATIC or ROLLING.
* **No DSL:** logic must live in TS, not YAML expressions.  
* **No IO rule:** checks must not call external systems directly.  
* **Signal integrity:** missing context must use error field, not fake observedValue.
* **Display ownership:** check must provide meaningful `displayValue`, not raw data dumps.

---

# RulepackLoader (Config Loader)

### **Description** Loads YAML rulepacks/pillars into validated runtime objects.

### Role & Responsibilities

* Parse YAML configs.  
* Validate via Zod schemas (pillars, rulepacks, badge configs, applicability gates).  
* Validate `spotlights.yaml` via `SpotlightSchemas` (matchAny semantics, multiplier ranges 1.0-2.0, sealed enum IDs).
* Validate `focus-modes.yaml` via `FocusModeSchemas` (category multipliers, tag boost lists, sealed focus mode IDs).
* Validate **`badge.baseline`** is present on every rulepack (required field, fail-fast at startup).
* Produce efficient lookup structures.

### Design Guidelines

* Strict validation: fail fast on invalid configs, crash the web-server (do not start with invalid config).  
* Preserve order and IDs; enforce uniqueness:  
  * rulepackId unique  
  * ruleId unique within rulepack  
  * pillarId referenced exists
* `applicableWhen.contextKey` should reference a known collector key.

### Must Not Break

* **Trust boundary:** invalid config must never run partially.  
* **Determinism:** same YAML must produce the same runtime model.

### Configuration Constraints (Validated at Startup)

| Badge Type | Max Rules | `blocking` Property | badge.thresholds | badge.baseline |
|------------|-----------|---------------------|------------------|----------------|
| GATEKEEPER | Unlimited | Allowed | Per-rule (optional) | **Required** |
| LADDER | Exactly 1 | Forbidden | Required (numbers) | **Required** |

These constraints are enforced by Zod schema validation, `validateLadderRulepacks()`, and `validateGatekeeperRulepacks()`.
Invalid configuration fails startup with a clear error message.

**Rationale:**
- GATEKEEPER = checklist model (pass all blocking rules) → `blocking` controls if rule affects badge
- LADDER = single-metric model (numeric value → tier) → one rule, no blocking concept

### Baseline Contract (Explicit Requirement)

Every rulepack MUST declare `badge.baseline` in its YAML config. This is the **single source of truth** for whether a rulepack blocks Flight Readiness.

| Baseline Value | Meaning | `baselineRequired` on result |
|----------------|---------|------------------------------|
| `NONE` | Informational only, does not block Flight Readiness | `undefined` |
| `PASS` / `BRONZE` / `SILVER` / `GOLD` | Minimum badge level required for Flight Readiness | The declared level |

**Design decisions:**
- Baseline lives on the **rulepack** (not the pillar). Each rulepack owns its readiness contract.
- `pillars.yaml` defines organizational grouping and display order only — no `readiness` section.
- The orchestrator does not resolve baselines. The enricher reads `rulepack.badge.baseline` directly.
- `NONE` maps to `baselineRequired: undefined` at enrichment time, maintaining the existing consumer contract (`baselineRequired !== undefined` = required).

---

# Enricher (Threshold Application + Badge Facts — NO Config, NO Scoring)

### **Description** Applies thresholds from YAML to measurements and produces badge facts. Produces `TruthCheckResult` (truth only).

> **CRITICAL BOUNDARY: TRUTH vs MEANING**
> Check = TRUTH: Returns measurement (observedValue, displayValue) - NO status/judgment.
> Enricher = MEANING: Applies thresholds from YAML to determine status (PASS/FAIL) or tier (GOLD/SILVER/BRONZE).
> This separation ensures: checks measure, config judges.
>
> The Enricher also:
> - Must NOT attach config fields (implication, remediation, etc.) — that is ConfigResolver's job at view-time.
> - Must NOT compute scores — that is ScoringPolicyV1's responsibility.

### Role & Responsibilities

* **Apply thresholds** from YAML to measurement observedValue:
  * GATEKEEPER: Apply `rule.threshold` to determine PASS/FAIL
  * LADDER: Apply `badge.thresholds` to determine GOLD/SILVER/BRONZE/FAIL
* **Derive achievementType** from check's `windowType`:
  * STATIC → CORE (permanent achievement)
  * ROLLING → CIRCUIT (temporary, must be maintained)
* **Produce TruthCheckResult** with:
  * Identity keys: ruleId, checkId, pillarId, rulepackId
  * Measurement from check: observedValue, displayValue
  * Status computed from threshold: status (PASS/FAIL/WARN)
  * Blocking semantics: blockingStatus, errorKind, missingContextKey
  * Evidence: evidence (links, raw, version)
* **Compute `blockingStatus`** from measurement error:
  * `EVALUABLE` — measurement successful, threshold applied
  * `BLOCKED` — COLLECTOR_FAILED (telemetry missing, blocks readiness)
  * `NEUTRAL` — NOT_APPLICABLE (excluded from evaluation, never blocks)
* Evaluate badge outcomes via `BadgeEvaluatorRegistry`:
  * GATEKEEPER (binary pass/fail from threshold)
  * LADDER (tiered from tier thresholds)
* **Derive `RulepackApplicability`** from check-level `blockingStatus` (single derivation point):
  * `APPLICABLE` — at least one check is EVALUABLE
  * `NOT_APPLICABLE` — all checks are NEUTRAL (context absent by design)
  * `BLOCKED` — any check is BLOCKED (telemetry missing)
* **Stamp `applicability`** on each `RulepackResult` at enrichment time.
* **Read `baseline`** from `rulepack.badge.baseline` to resolve `baselineRequired` (`NONE` → `undefined`).
* Produce RulepackBadgeResult with:
  * Badge level (NOT score)
  * Achievement type (derived from windowType: CORE/CIRCUIT)
  * Baseline comparison (baselineMet)
  * Rationale (explainability)

### Decomposed Components

The Enricher delegates to these focused components:

* `ThresholdApplicator` — Applies thresholds from YAML to observedValue
* `BaselineComparator` — Computes baselineMet from badge level and required baseline
* `BadgeEvaluatorRegistry` — Strategy pattern for badge type evaluation
  * `GatekeeperEvaluator` — Applies rule thresholds for binary PASS/FAIL
  * `LadderEvaluator` — Applies tier thresholds for GOLD/SILVER/BRONZE

### Design Guidelines

* **Output is TruthCheckResult[] only** — no config fields (name, implication, remediation, impact, effort, tags, blocking, missingSignalText).
* **Threshold application is core responsibility** — enricher owns status determination from observedValue + YAML thresholds.
* **blockingStatus is the single source of truth** for error semantics:
  * `BLOCKED` → telemetry missing narrative, blocks evaluation
  * `NEUTRAL` → excluded from evaluation, NOT blocked
* Enrichment must be pure and deterministic.
* Use ruleId as primary identity in output.
* **Baseline is rulepack-owned:** Enricher reads `badge.baseline` from rulepack config. `NONE` → `undefined` (informational).
* **Applicability is a single derivation point:** Computed once at enrichment time from check blockingStatus, stamped on `RulepackResult`, consumed by all view-layer components.
* Achievement type is derived from check windowType (STATIC → CORE, ROLLING → CIRCUIT).

### Must Not Break

* **THRESHOLD APPLICATION:** Enricher MUST apply thresholds from YAML to determine status. Checks never decide PASS/FAIL.
* **NO CONFIG:** Enricher must NOT attach config fields. That is ConfigResolver's job at view-time.
* **NO SCORING:** Enricher must NOT compute rulepackScore. That is ScoringPolicyV1's job.
* **Achievement derivation:** CORE/CIRCUIT must be derived from check windowType, not declared in YAML.
* **Explainability:** badge rationale must be derivable.
* **Baseline from config:** `baselineMet` is computed from `rulepack.badge.baseline`, not from an external parameter.
* **blockingStatus semantics:** BLOCKED blocks readiness, NEUTRAL is excluded (never blocks).
* **Applicability stamped:** Every `RulepackResult` must have `applicability` stamped at enrichment time.

---

# ConfigResolver (View-Time Config Lookup)

### **Description** Provides O(1) lookup of rule/rulepack configuration at view-time for hydrating TruthCheckResult into ViewCheckResult.

> **CRITICAL BOUNDARY:** ConfigResolver owns all config field attachment.
> Snapshots store TruthCheckResult (truth only). When rendering views, ConfigResolver hydrates config fields from YAML.
> This ensures engineers always see the current best guidance, even for historical scans.

### Role & Responsibilities

* **Build indexed lookup** from rulepacks at startup:
  * Rule index: `"rulepackId:ruleId" → RuleConfig`
  * Rulepack index: `rulepackId → RulepackConfig`
* **Provide `getRule(rulepackId, ruleId)`** for config lookup.
* **Provide `hydrateCheck(TruthCheckResult)`** to produce ViewCheckResult:
  * Merge truth with config fields: name, implication, remediation, impact, effort, blocking, tags, missingSignalText
* **Provide `hydrateRulepack(RulepackResult)`** for full rulepack hydration.
* **Graceful degradation:** if rule not found, return truth unchanged (config fields undefined).

### Design Guidelines

* **Pure and deterministic:** same input always produces same output.
* **O(1) lookup:** use Map for indexed access.
* **Default blocking to true** when not specified in YAML.
* **No side effects:** resolver is stateless after construction.
* **Instantiated once at startup** with loaded rulepacks.

### Must Not Break

* **View-time only:** ConfigResolver is called by view layer components, not Enricher.
* **Graceful degradation:** unknown rules return truth unchanged (no errors).
* **Config freshness:** config is always from current YAML, not stored in snapshots.
* **No business logic:** resolver is pure lookup, no scoring or evaluation.

---

# Achievement Type Derivation

> **See [Constitution Section 4](./Ground%20Control%20Constitution.md#4-achievement-types) for canonical definitions.**

### **Description** Achievement type (CORE vs CIRCUIT) is **derived from the check's `windowType`**, not from badge type or YAML declaration.

### Derivation Rule

| Check windowType | Achievement Type | Meaning |
|------------------|------------------|---------|
| `STATIC` | CORE | Permanent — one-time structural improvement that stays built |
| `ROLLING` | CIRCUIT | Temporary — ongoing operational health that must be maintained |

### Classification Examples

| Rulepack | Badge Type | Check windowType | Achievement | Rationale |
|----------|------------|------------------|-------------|-----------|
| `cloud_native` | GATEKEEPER | STATIC | CORE | Config-based cloud-native readiness (config, probes, requests, HPA) |
| `ci_stability` | GATEKEEPER | ROLLING | CIRCUIT | Event-windowed vitality (rate threshold) |
| `ci_performance` | LADDER | ROLLING | CIRCUIT | Rolling performance |
| `safety_net` | LADDER | ROLLING | CIRCUIT | Rolling stability (24h) |
| `safe_releases` | GATEKEEPER | STATIC | CORE | Config-based gate |

**Rule of thumb:** If the check measures a rolling window (time or events), it's a CIRCUIT. If it measures point-in-time state, it's a CORE.

### User-Facing Semantics

* **Core (Foundation):** Permanent achievement. Once earned, remains visible as part of service foundation.
* **Circuit (Vitality):** Temporary achievement that must be carried. Represents current operational vitality over rolling window. When window breaks, Circuit degrades.

### Design Guidelines

* Classification is determined at enrichment time from `check.windowType`.
* Badge type (GATEKEEPER/LADDER) determines **how** evaluation happens, not achievement type.
* The UI must make Core vs Circuit distinction visually obvious.
* Circuits must visually communicate window truth (nominal, degrading, down).

### Visual Identity Contract

* **Recommendation type visuals** are centralized in `apps/web/src/lib/recommendation-visuals.ts`. Components must import from this single source — no local TYPE_CONFIG maps or per-component icon/color definitions.
* **Achievement type (Core/Circuit) must be visible** wherever upgrade recommendations are shown, including NextBestUpgrade alternatives, UpgradeCard, and UpgradeBrief.
* **Recommendation type colors** must use `--gc-rec-*` CSS variables defined in `apps/web/src/index.css`, never raw Tailwind color classes (e.g., `emerald-500`, `orange-500`).
* **Impact/effort display** is centralized in `apps/web/src/lib/impact-effort.ts`. Same single-source pattern applies.
* **Unit tests** in `apps/web/src/lib/__tests__/recommendation-visuals.test.ts` enforce exhaustiveness, CSS variable usage, and icon uniqueness. Adding a new UpgradeType without updating the config will fail these tests.

### Must Not Break

* **Derivation source:** Achievement type comes from `check.windowType`, never from badge type or YAML declaration.
* **Classification stability:** Same check windowType must always produce same achievement type.
* **User comprehension:** Terms must remain Core/Circuit—no alternative terminology.
* **Visual centralization:** Recommendation type icon/color configs must not be duplicated across components.

---

# BadgeEvaluatorRegistry (Strategy Pattern)

### **Description** Runtime mapping of BadgeType → BadgeEvaluator. Implements strategy pattern for badge evaluation.

### Role & Responsibilities

* Register evaluators for each badge type (GATEKEEPER, LADDER).
* Provide lookup and dispatch to correct evaluator.
* Enforce uniqueness of badge type registrations.

### Design Guidelines

* Keep API minimal: `register()`, `get()`, `has()`.
* Fail-fast on duplicate registrations.
* Each evaluator reads pre-computed `blockingStatus` field for consistent error handling.

### Must Not Break

* **OCP:** Adding new badge types = add new evaluator, no registry changes.
* **No scoring:** Evaluators produce badge facts only, NOT scores.
* **Determinism:** Same inputs must produce same badge evaluation.

---

# BadgeEvaluator Interface

### **Description** Strategy interface for badge type evaluation.

```typescript
interface BadgeEvaluator {
  readonly badgeType: BadgeType;
  evaluate(
    rulepack: RulepackConfig,
    checks: EnrichedCheckResult[],
    baselineRequired: BadgeLevel | undefined,
    achievementType: AchievementType,
  ): RulepackBadgeResult;
}
```

### Implementations

* **GatekeeperEvaluator** — Binary pass/fail evaluation
* **LadderEvaluator** — Tiered evaluation (lowest tier wins)

### Must Not Break

* **No scoring:** Evaluators must NOT compute rulepackScore
* **Read blockingStatus:** All evaluators read pre-computed `blockingStatus` field for error semantics
* **Badge facts only:** Return badge level, baselineMet, rationale

---

# ScoringPolicyV1

### **Description** The explicit fairness policy that converts badge facts into scores and readiness status.

> **CRITICAL BOUNDARY:** ScoringPolicyV1 owns ALL score computation.
> The Enricher produces badge facts only (level, achievementType, baselineMet).
> ScoringPolicyV1 takes these facts and computes rulepackScore, pillarScores, and all V2 metrics.

### Role & Responsibilities

* **Compute rulepackScore (0–100)** from badge facts:
  * LADDER: GOLD=100, SILVER=80, BRONZE=60, FAIL/NOT_EVALUATED=0
  * GATEKEEPER: 100 if baselineMet, 0 otherwise
* Compute pillarScores as weighted aggregates.
* Compute pillarReadiness based on rulepack baseline requirements (`badge.baseline`).
* Compute telemetryCoverage for UI trust and fairness.
* Compute **coreCount** and **circuitCount** from rulepack badge results (V2).
* Compute **flightHoursEarned** for this scan (V2):
  * +100 per newly earned Core (one-time burst)
  * +10/day per nominal Circuit
  * No negative Momentum (per V2 guardrails)
* Compute **riskLevel** (NOMINAL | WATCHLIST | HOLD) based on (V2):
  * HOLD: Critical rulepacks in TELEMETRY_MISSING or multiple high-severity FAILs
  * WATCHLIST: Some Circuits down or approaching baseline gaps
  * NOMINAL: Default healthy state
* Compute **goNoGoPending** list when pillar is within N items of readiness (V2).

### Design Guidelines

* Error neutrality via `blockingStatus`:  
  * `BLOCKED` → no score penalty, can block readiness  
  * `NEUTRAL` → excluded from evaluation  
* Versioned output: always set `scoringPolicyVersion = 'v1'`.  
* Keep policy logic isolated (so v2 can coexist later).
* Momentum must remain secondary feedback—never affects readiness or scores.
* Risk assessment must be explainable: always include `riskDrivers` array with evidence.
* Go/No-Go threshold (N) should be configurable (default: 1-2 items).

### Must Not Break

* **Fairness contract:** missing telemetry must never reduce score.  
* **Stability:** policy versioning prevents historical score reinterpretation.
* **Momentum guardrails:** Never negative momentum; never used for ranking or comparison.
* **Risk evidence:** HOLD/WATCHLIST states must always link to specific rulepack/check evidence.

---

# SnapshotRepository (Persistence Layer)

### **Description** Stores and retrieves snapshots and scan statuses. Provides immutable evaluation history and query-ready indexes for team/org aggregation views.

### Role & Responsibilities

* Write EvaluationSnapshot atomically per scan completion.  
* Fetch latest snapshot per service.  
* Fetch time-series history.  
* Upsert `ServiceState` (current-truth table) at the end of each scan for fast aggregation reads.
* `ScanQueryService` provides the read-model layer on top of this persistence (see ScanQueryService section).

### Design Guidelines

* Treat snapshots as immutable.  
* Make reads fast (UI must be sub-100ms).  
* Support pagination and filtering at DB level.

### Must Not Break

* **Immutability:** never mutate past snapshots.  
* **Performance:** endpoints must not recompute heavy logic on request.

---

# ScanQueryService (Read-Model Layer)

### **Description** Centralizes cross-service aggregate queries by reading from `ServiceState` (current truth) instead of running expensive `DISTINCT ON` queries against the `Scan` table.

**File:** `apps/api/src/db/queries/scan-query-service.ts`

### Role & Responsibilities

* **`getLatestScansForOrg()`** — Current state of every service across org (lightweight, reads from ServiceState).
* **`getLatestScansWithData()`** — Current state with full snapshot data (joins ServiceState → Scan for data JSON).
* **`getLatestScansForTeam(teamId)`** — Current state for a specific team's services.
* **`getLatestScansForSpotlight()`** — Current state for spotlight view.
* **`getLatestEvalTimestamp()`** — Timestamp of most recent evaluation.
* **`countEvaluatedServices()`** — Count services evaluated within freshness window.

### Design Guidelines

* Read from `ServiceState` (upserted by worker after each scan), not from `Scan` with `DISTINCT ON`.
* All queries enforce a **3-day freshness guard** on `lastEvaluatedAt` — excludes services not evaluated recently.
* Two query variants: lightweight (ServiceState columns only) and detailed (joins Scan for full snapshot JSON).
* Uses parameterized raw SQL (`$queryRaw`) for performance.
* **Domain boundary:** ScanQueryService returns all fresh scans regardless of catalog `isActive` status. It stays in the scan domain and has no awareness of the catalog domain. Active-service filtering is the responsibility of the consumer (aggregator), which receives `activeServiceIds` from the controller via `ServiceDirectory`. This preserves single source of truth — only `ServiceDirectory` defines what "active" means.

### Must Not Break

* **Freshness guard:** All queries must enforce the 3-day `lastEvaluatedAt` filter. Stale services must not appear in active counts.
* **ServiceState as read source:** Never fall back to `DISTINCT ON Scan` for aggregate queries.
* **No catalog coupling:** ScanQueryService must NOT join to `Service` table or filter by `isActive`. Active-service filtering belongs to the consumer (aggregator), using `activeServiceIds` from `ServiceDirectory`.

---

# MomentumQueryService

### **Description** Historical momentum aggregation queries. Reads from the `Scan` table to compute per-service and aggregate momentum deltas across configurable timeframes.

**File:** `apps/api/src/db/queries/momentum-query-service.ts`

### Role & Responsibilities

* **`getOrgHistory(timeframe, activeServiceIds)`** — Org-wide daily circuit totals for sparkline display.
* **`getTeamHistory(teamId, timeframe, activeServiceIds)`** — Single-team daily circuit totals.
* **`getHistoryByTeam(teamIds, timeframe, activeServiceIds)`** — Multi-team momentum history for org-level team sparklines.
* **`getServiceDeltas(timeframe, activeServiceIds)`** — Per-service momentum totals as `Map<string, number>`. Preserves service identity for the domain layer to compute gaining fleet percentage.
* **`getServiceHistory(serviceId, timeframe)`** — Single-service history (no fleet scope needed — already scoped to one service).
* **`getMultiServiceHistory(serviceIds, timeframe)`** — Multi-service breakdown (already scoped to explicit service IDs).

### Design Guidelines

* **Fleet scope contract:** All aggregate methods (org, team, historyByTeam, serviceDeltas) accept `activeServiceIds: ReadonlySet<string>` as a **required** parameter. This is the fleet boundary defined by `ServiceDirectory`. Queries never define their own fleet scope.
* **Domain boundary:** Same pattern as ScanQueryService — no catalog coupling, no awareness of `Service.isActive`. The fleet boundary is received as input, not derived from the Scan table.
* All queries enforce `isActiveAtScan = true` as a domain invariant (historical correctness).
* `getServiceDeltas` returns `Map<string, number>` (preserving service identity) so the domain layer can derive both numerator and denominator from the same scoped dataset.
* Uses parameterized raw SQL (`$queryRaw`) for performance.

### Must Not Break

* **Fleet scope is required:** All aggregate methods must accept `activeServiceIds`. Optional fleet scope is forbidden — it allows universe mismatch between numerator and denominator.
* **Identity preservation:** `getServiceDeltas` must return `Map<string, number>`, never `number[]`. Discarding service identity makes correct fleet filtering impossible.
* **No catalog coupling:** Must NOT join to `Service` table or filter by `isActive`. Active-service filtering uses the `activeServiceIds` parameter.
* **isActiveAtScan invariant:** All queries must filter `isActiveAtScan = true`.

---

# Recommendation Kernel

### **Description** Single source of truth for shared recommendation predicates, canonical priority order, and focus mode boosting across all recommendation engines.

**File:** `apps/api/src/view-layer/recommendation-kernel.ts`

> **Architecture:** Two engines consume this kernel — NBU (service-level) and Fleet engines (team/org-level). See Tech Spec Section 15 for full architecture.

### Role & Responsibilities

* Define **CANONICAL_PRIORITY** — authoritative priority order for all 5 recommendation types.
* Provide **needsVitalityProtection()** — time-based vitality detection using `deriveCircuitState()`.
* Provide **isBaselineGapCandidate()** — baseline gap detection for CORE rulepacks (CIRCUITs excluded).
* Provide **isHighImpactLowEffort()** — shared predicate for friction detection.
* Provide **classifyRulepack()** — mutually exclusive rulepack classifier for fleet engines. Returns `BASELINE`, `VITALITY`, or `RULE_LEVEL`.
* Provide **applyFocusModeBoost()** — focus mode priority boost for flat recommendation lists.
* Support **Focus Modes** (YAML-configured via `focus-modes.yaml`):
  * REDUCE_FRICTION: Boost `better-dx`/`cost-efficiency` tags
  * SHIP_FASTER: Boost `faster-delivery`/`change-confidence` tags
  * BREAK_LESS: Boost `faster-recovery`/`safer-deploys` tags
  * BASELINE_READY: Boost telemetry and baseline recommendations

### Design Guidelines

* No lifecycle state (not a task system).
* Recommendations must always be explainable and evidence-backed.
* Default to encouragement: focus on "next step" not "failure list."
* Focus Modes reorder and filter recommendations — they don't change underlying logic.
* REDUCE_FRICTION targets high-impact/low-effort items that are NOT blocking baseline.
* **Mutual exclusion by design:** `classifyRulepack()` ensures each rulepack maps to exactly one category (BASELINE, VITALITY, or RULE_LEVEL). Rulepack-level types (BASELINE, VITALITY) subsume all rules from that rulepack — no rule-level recommendations (REDUCE_FRICTION, OPTIMIZE) can coexist from the same rulepack.
* Adding a new recommendation type requires updating: UpgradeType enum, vocabulary, CANONICAL_PRIORITY, and enumeration logic. Steps 1-3 are compiler-enforced.

### Must Not Break

* **Canonical priority order:** PROTECT_VITALITY > UNBLOCK_TELEMETRY > REACH_BASELINE > REDUCE_FRICTION > OPTIMIZE.
* **Single source of truth:** All engines must consume CANONICAL_PRIORITY from the kernel — no hardcoded priority values.
* **Vitality detection:** All engines must use `needsVitalityProtection()` — no inline `!baselineMet` checks.
* **Rulepack classification:** Fleet engines must use `classifyRulepack()` for mutual exclusion — no independent `if` statements that allow rulepacks to appear in multiple categories.
* **Intrinsic motivation:** avoid spam/noise and avoid shame framing.
* **No managed quests:** do not introduce ownership/time tracking in v1.

---

# VocabularyProvider (Backend-Owned Display Text)

### **Description** Single source of truth for all domain display vocabulary — labels, descriptions, and tooltips for every domain enum and coaching topic. Enforces the contract that the frontend is a pure renderer with no hardcoded domain text.

**File:** `apps/api/src/view-layer/vocabulary-provider.ts`

> **DESIGN PRINCIPLE: Backend-Owned Vocabulary**
> The frontend renders backend-provided vocabulary and does not define domain text. All labels, descriptions, and tooltips come from the API — either embedded directly in response payloads or fetched from `/config/vocabulary`. See Technical Spec Section 1, Principle 7.

### Role & Responsibilities

* **Enum vocabulary** (keyed by domain enum values):
  * `CheckStatus` — PASS, NOT_MET, WARN, ERROR
  * `UpgradeType` — UNBLOCK_TELEMETRY, REACH_BASELINE, PROTECT_VITALITY, REDUCE_FRICTION, OPTIMIZE
  * `ReadinessState` — flight_ready, preflight, telemetry_missing
  * `CircuitState` — nominal, degrading, down
  * `CoreState` — earned, not_earned, telemetry_missing, not_applicable
  * `ProtocolState` — met, below, telemetry_missing, not_applicable
  * `RulepackApplicability` — APPLICABLE, NOT_APPLICABLE, BLOCKED
  * `BadgeType` — LADDER, GATEKEEPER
  * `ImpactLevel` — HIGH, MEDIUM, LOW
  * `EffortLevel` — SMALL, MODERATE, SIGNIFICANT
  * `EvidenceConclusionState` — passed, not_met, warning, blocked, not_applicable
* **Coaching topic vocabulary** (keyed by topic ID):
  * `signalCoverage` — Org/team-level: percentage of services with connected telemetry
  * `evaluationCoverage` — Service-level: percentage of rules that could be evaluated
  * `closestWins` — Services one step from Flight Ready
  * `nominalCircuits` — Nominal circuits count
  * `evidenceBadgeAffectingRules` — Rules that affect badge outcome
  * `evidenceAdvisoryRules` — Non-blocking advisory rules
* **Mapping functions:** `mapCheckStatusToConclusion()`, `mapProtocolStateToConclusion()`, `mapReadinessToConclusion()` — map domain states to evidence conclusion states.
* **Formatting functions:** `formatRuleCriteria()`, `formatLadderCriteria()`, `formatGatekeeperProtocolCriteria()` — human-readable criteria display.
* **Delivery:** `getFullVocabulary()` for `/config/vocabulary` endpoint, `getOrgCoachingLabels()` for Mission Map, `buildClosestWinsCoaching()` for team closest wins.

### Design Guidelines

* **OCP gate:** Adding any new enum value causes a TypeScript compile error until the vocabulary entry is added. This prevents shipping domain values without display text.
* **Coverage distinction:** `signalCoverage` (org/team) and `evaluationCoverage` (service) are distinct concepts with separate vocabulary entries. They must never be conflated.
* **Non-alarming copy:** Use neutral language (e.g., "easing" not "declining", "not yet met" not "failing").

### Must Not Break

* **OCP enforcement:** New enum values must produce compile errors until vocabulary is added.
* **Frontend purity:** Frontend components must NOT hardcode domain labels, descriptions, or tooltip text.
* **API self-containment:** Every API response containing a domain metric must include the vocabulary for that metric.
* **Coverage distinction:** `signalCoverage` and `evaluationCoverage` must have separate topic IDs.

---

# SpotlightDecorator (Spotlight Application)

### **Description** Shared utility that applies spotlight metadata to recommendations using matchAny semantics. Single source of truth for spotlight matching logic.

**File:** `apps/api/src/view-layer/spotlight-decorator.ts`

### Role & Responsibilities

* **`spotlightMatchesCandidate(spotlight, candidate)`** — Match a recommendation candidate against active spotlight using matchAny semantics: checks pillarId, rulepackId, ruleId, tags, or unblocks (for UNBLOCK_TELEMETRY).
* **`decorateWithSpotlight(rec, activeSpotlight)`** — Decorate a single recommendation with spotlight info if matched. Adds `{ id, title, kind: 'org_focus' }`.
* **`decorateListWithSpotlight(recs, activeSpotlight)`** — Decorate all recommendations in a list.

### Matching Semantics

* **matchAny:** Any selector match succeeds. If the spotlight specifies `rulepackId: ['ci_stability']` and `tags: ['faster-recovery']`, a candidate matching either condition is decorated.
* **UNBLOCK_TELEMETRY special case:** For telemetry recommendations with `unblocks`, checks if ANY underlying rulepack or rule matches the spotlight selectors.
* **No match / no spotlight:** Returns recommendations unchanged.

### Design Guidelines

* Used by `ReadinessAggregator` and `TeamConsoleAggregator` for org/team-level recommendation decoration.
* Used by `NextBestUpgradeSelector` for service-level spotlight integration (via separate multiplier logic).
* Pure functions — no side effects.

### Must Not Break

* **matchAny semantics:** Any selector match succeeds — never require all selectors to match.
* **Unchanged on no match:** Recommendations without spotlight match must be returned unmodified.
* **Single source of truth:** All spotlight matching must use this decorator's logic.

---

# RecommendationKeyBuilder (Pure Utility)

### **Description** Single source of truth for building recommendation keys used in API routing and drawer navigation.

### Role & Responsibilities

* Build type-prefixed recommendation keys for API routing
* Ensure format matches `parseRecommendationKey` in readiness-aggregator
* Provide consistent key format across all view-layer aggregators

### Key Format Contract

| Type | Format | Example |
|------|--------|---------|
| UNBLOCK_TELEMETRY | `telemetry:<missingContextKey>` | `telemetry:ci:gitlab` |
| PROTECT_VITALITY | `protect_vitality:<rulepackId>` | `protect_vitality:ci_stability` |
| REACH_BASELINE | `reach_baseline:<rulepackId>` | `reach_baseline:safety_net` |
| REDUCE_FRICTION | `reduce_friction:<rulepackId>:<ruleId>` | `reduce_friction:safety_net:probes_configured` |
| OPTIMIZE | `optimize:<rulepackId>:<ruleId>` | `optimize:safety_net:hpa_configured` |

### Design Guidelines

* Use `buildRecommendationKey(type, identifiers)` for all recommendationKey generation
* Use `buildRuleKey(rule)` ONLY for deduplication logic (not API keys)
* Never construct recommendation keys inline—always use the builder
* Adding new recommendation types requires only a new case in the switch (OCP)

### Must Not Break

* **Parser compatibility:** Key format must match `parseRecommendationKey` expectations
* **DRY:** All aggregators must use the same builder function
* **OCP:** Adding new types requires only a new case in the switch

### UNBLOCK_TELEMETRY Title Contract

All UNBLOCK_TELEMETRY recommendations **must** use the signal-centric title format:

| Field | Format | Example |
|-------|--------|---------|
| title | `Connect telemetry: {missingContextKey}` | `Connect telemetry: git` |

**Rationale:**
* The recommendation identity is the **signal** (missingContextKey), not the rulepack
* One missing signal can block multiple rulepacks
* The brief API drawer shows which rulepacks are unblocked via `unblocks.rulepacks[]`
* Title consistency ensures clicking any UNBLOCK_TELEMETRY recommendation shows matching drawer content

**Use `buildTelemetryTitle(missingContextKey)` for all UNBLOCK_TELEMETRY title generation.**

### Aggregation Key Identity Contract

When aggregating issues across services in `aggregateIssuesAcrossServices`, the aggregation key determines deduplication. **The aggregation key must align with the recommendation's identity components.**

| Type | Identity Components | Aggregation Key |
|------|-------------------|-----------------|
| UNBLOCK_TELEMETRY | `missingContextKey` only | `buildTelemetryKey(missingContextKey)` |
| PROTECT_VITALITY | `rulepackId` | `${rulepackId}:circuit` |
| REACH_BASELINE | `rulepackId` | `${rulepackId}:baseline` |
| REDUCE_FRICTION | `rulepackId:ruleId` | `${rulepackId}:${ruleId}:qw` |
| OPTIMIZE | `rulepackId:ruleId` | `${rulepackId}:${ruleId}:opt` |

**Why UNBLOCK_TELEMETRY is Special:**
* A single missing signal (e.g., `ci`) can block **multiple rulepacks**
* The recommendation identity is the **signal**, not the rulepack
* Aggregating by `rulepackId:missingContextKey` creates duplicates and crowds out other recommendation types
* **Use `buildTelemetryKey(missingContextKey)` for telemetry aggregation**

### Rulepack-Level Subsumption

Rulepack-level types (REACH_BASELINE, PROTECT_VITALITY) subsume all rules from that rulepack. When a rulepack is classified as BASELINE or VITALITY via `classifyRulepack()`, no individual rules from that rulepack may appear as REDUCE_FRICTION or OPTIMIZE. This is enforced structurally by the `if/else if` classification chain in fleet engines, not by post-hoc cleanup.

### Must Not Break (Aggregation)

* **Identity alignment:** Aggregation key must match recommendation identity
* **Single utility:** Use `buildTelemetryKey()` for telemetry aggregation
* **No inline keys:** Never construct aggregation keys inline for telemetry
* **Rulepack subsumption:** BASELINE and VITALITY classifications must prevent rule-level leakage into FRICTION/OPTIMIZE

---

# NextBestUpgradeSelector (View Logic) [V3]

### **Description** Selects a single primary upgrade with alternatives using leverage-based scoring.

### Role & Responsibilities

* Generate candidates from five sources:
  * **Telemetry candidates**: `COLLECTOR_FAILED` errors blocking required baselines
    * **Telemetry candidate dedup:** `generateTelemetryCandidates` deduplicates by `missingContextKey` (signal), not by `rulepackId`. One missing signal = one candidate, regardless of how many rulepacks it blocks. The first encountered rulepack provides routing context (rulepackId/ruleId).
  * **Vitality candidates**: DEGRADING or DOWN Circuits (see Vitality Candidate Generation below)
  * **Baseline candidates**: Required rulepacks with `baselineMet=false`
  * **Friction candidates**: High-impact/low-effort items NOT blocking baseline
  * **Optimize candidates**: Remaining FAIL/WARN checks
* Score candidates using leverage formula: `(impact/effort) * confidence * multipliers`
* Apply multipliers:
  * Telemetry blocking baseline: **10x** (when service is TELEMETRY_MISSING)
  * Vitality protection (DEGRADING/DOWN Circuit): **2.2x**
  * Closest win: **2.5x**
  * Completes Go/No-Go: **2.0x**
  * Focus mode multipliers (from `focus-modes.yaml`):
    * Category multipliers: boost/dampen specific upgrade types (e.g., PROTECT_VITALITY 2.2x for BREAK_LESS)
    * Tag boost multipliers: 1.3x when candidate has matching outcome tag (exact match)
  * Spotlight (when active and matched): **1.0x - 2.0x**
* Apply deterministic tie-breakers for stable ordering
* Build explainability output (`explainWhyChosen[]`) for coaching
* **Spotlight integration:** When an active spotlight is configured, matching candidates receive the spotlight multiplier and are decorated with a `spotlight:{id}` tag.

### Vitality Candidate Generation (V3)

**Purpose:** Detect DEGRADING or DOWN Circuits and generate PROTECT_VITALITY candidates.

**Implementation:**
```typescript
function needsVitalityProtection(
  pack: RulepackResult,
  rulepackConfig: RulepackConfig | undefined,
): boolean {
  if (pack.badge?.achievementType !== 'CIRCUIT') return false;
  
  // Edge case: baselineMet=false AND no lastPassedAt = DOWN (never passed in history)
  // This handles Circuits that have been failing since before we started tracking,
  // or new checks that have never passed. Treat as DOWN - needs protection.
  if (!pack.badge.lastPassedAt && !pack.badge.baselineMet) {
    return true;
  }
  
  const thresholds = getVitalityThresholdsForRulepack(rulepackConfig);
  const circuitState = deriveCircuitState(pack.badge.lastPassedAt, thresholds);
  
  return circuitState === 'DEGRADING' || circuitState === 'DOWN';
}
```

**Key Properties:**
- **Truth vs Meaning**: Calls `deriveCircuitState()` (pure function from enrichment layer)
- **View-Time Computation**: Same pattern as `FlightDeckAggregator.buildCircuits()`
- **Per-Rulepack Thresholds**: Different Circuits have different urgency profiles
- **Scan-Cadence Independent**: Time-based (hours), not scan-count based
- **Never-Passed Edge Case**: Circuits with `lastPassedAt=undefined` AND `baselineMet=false` are treated as DOWN

**"Save the Baby" Philosophy:** When any Circuit is DEGRADING or DOWN, all non-vitality candidates receive a 0.1x penalty (vitality damper). This ensures PROTECT_VITALITY always wins regardless of other multipliers.

### Vitality Protection Detection (Scoring)

**Purpose:** Determine if the vitality damper (0.1x) and vitality boost (2.2x) should be applied during scoring.

**Implementation:**
```typescript
function computeAnyCircuitNeedsProtection(
  snapshot: EvaluationSnapshot,
  rulepacks: RulepackConfig[],
): boolean {
  return snapshot.rulepackResults.some(pack => {
    const config = rulepacks.find(r => r.id === pack.rulepackId);
    return needsVitalityProtection(pack, config);
  });
}
```

**Logic:**
- Reuses `needsVitalityProtection` (which delegates to `deriveCircuitState` in enrichment)
- Single source of truth: same function determines "does any circuit need protection" for both the 0.1x damper and vitality candidate generation
- Returns `true` only when a Circuit is genuinely DEGRADING or DOWN (time-based, respects grace period)
- When `true`: non-vitality candidates receive 0.1x penalty; PROTECT_VITALITY candidates receive 2.2x boost

### Design Guidelines

* **Determinism:** Same inputs must always produce same output.
* **Explainability:** Every recommendation must have `explainWhyChosen[]` bullets.
* **Category order:** telemetry → vitality → baseline → friction → optimize (as tie-breaker).
* **V3 Vitality logic:** Uses time-based `deriveCircuitState()` to detect DEGRADING and DOWN Circuits.
* **Confidence floor:** Minimum confidence weight is 0.3 (never collapse to zero).

### Output Contract

```typescript
interface NextBestUpgradeResult {
  primary: CandidateUpgrade | null;  // Highest-leverage upgrade
  alternatives: CandidateUpgrade[];  // 2-3 alternatives
  computedAtTs: string;              // ISO timestamp
}

interface CandidateUpgrade extends UpgradeRecommendation {
  score: number;           // Computed leverage score
  isClosestWin: boolean;   // Only remaining blocker for pillar
  completesGoNoGo: boolean; // Completes Go/No-Go → Flight Ready
  moves: UpgradeMove;      // Move metadata
  explainWhyChosen: string[]; // Human-readable explanations
  confidence: number;      // 0.3 - 1.0
}
```

### Tie-Breaker Order

When scores are within threshold (0.01), break ties by:
1. `completesGoNoGo` (true first)
2. `isClosestWin` (true first)
3. Impact (HIGH > MEDIUM > LOW)
4. Effort (S < M < L)
5. Confidence (higher first)
6. Lexicographic: `(categoryPriority, pillarId, rulepackId, ruleId)`

**Category Priority Order:** telemetry → vitality → baseline → friction → optimize

### Must Not Break

* **Determinism:** Same inputs = same output (no randomness).
* **Explainability:** Every recommendation must have `explainWhyChosen[]`.
* **Category order:** Tie-breaker must respect telemetry → vitality → baseline → friction → optimize.
* **No stickiness:** Recommendations always reflect current truth (no retention logic).
* **Confidence bounds:** Confidence must be in [0.3, 1.0] range.
* **V3 Vitality:** `needsVitalityProtection()` must use `deriveCircuitState()` for time-based detection.
* **OCP compliance:** View layer imports enrichment function, no direct threshold logic.

---

# ReadinessAggregator (View Logic — Org Level)

### **Description** Produces all organization-level views for the Mission Map — team readiness, org summaries, org-wide upgrade recommendations, vitality briefs, and upgrade drawer data.

**File:** `apps/api/src/view-layer/readiness-aggregator.ts`

### Role & Responsibilities

* **`getTeamReadiness(options)`** — Aggregate team readiness with pillar scores, risk distribution, momentum, closest wins, and top blocker per team.
* **`getSummary()`** — Org-wide Mission Map summary including vitality brief and momentum trajectory.
* **`getOrgUpgrades(options?)`** — Org-level upgrade recommendations with telemetry volume threshold, spotlight decoration, and focus mode boost.
* **`getVitalityBrief(state, options?)`** — Circuit vitality brief for degrading/down states across the org.
* **`getUpgradeBrief(recommendationKey, options?)`** — Upgrade brief with scope and affected services for drawer navigation.
* **`getUpgradeServices(recommendationKey, options?)`** — Paginated list of affected services for a given recommendation.

### Constructor Dependencies

`PrismaClient`, `PillarConfig[]`, `RulepackConfig[]`, `ConfigResolver`, `SpotlightConfig | null`, `ScanQueryService`, `MomentumQueryService?`

### Design Guidelines

* Reads from `ServiceState` (current-truth table), never from raw `Scan` with DISTINCT ON.
* **Active-service filtering:** All public methods accept `activeServiceIds: ReadonlySet<string>` (**required**, provided by the controller from `ServiceDirectory`). Scan results **and momentum queries** are filtered to active services only before computing any metrics. This ensures readiness distributions, coverage percentages, momentum counts, and service counts use the same population as `ServiceDirectory.countServices()`.
* **Fleet scope passthrough:** `activeServiceIds` is forwarded to all `MomentumQueryService` calls (org history, service deltas, team history). Momentum queries never define their own fleet boundary.
* Telemetry volume threshold: UNBLOCK_TELEMETRY recommendations only surfaced when >= 30% of evaluated fleet is affected (prevents isolated noise at org level).
* Spotlight decoration via `SpotlightDecorator` (see SpotlightDecorator section).
* Momentum trajectory via `MomentumQueryService` at org altitude (gaining fleet %). The `FleetDeltas` domain object ensures numerator and denominator derive from the same scoped dataset.
* Fleet coverage via `computeFleetCoveragePct()` domain function — single source of truth for `coveragePct` at both org and team level.
* View-time hydration via `ConfigResolver` — never stores config fields.
* **Mutual exclusion by design:** Uses `classifyRulepack()` from the kernel to classify each rulepack into exactly one category (BASELINE, VITALITY, or RULE_LEVEL). Telemetry is always processed independently per-check. No post-hoc deduplication needed.
* Recommendation key generation uses `buildRecommendationKey()` and `buildTelemetryKey()`.
* **Recommendation key passthrough:** `buildNextBestUpgrade` uses the algorithm's pre-computed `recommendationKey` directly (from `buildRecommendationKey`). It does not rebuild keys with `buildRuleKey`. Alternatives are filtered to exclude candidates sharing the primary's `recommendationKey`.

### Must Not Break

* **Volume threshold (30%):** Telemetry recommendations must respect fleet-wide threshold at org level.
* **View-time hydration:** Config fields come from ConfigResolver, never from snapshots.
* **Deterministic aggregation:** Same inputs must produce same org-level views.
* **Mutual exclusion:** Must use `classifyRulepack()` for rulepack classification — no independent `if` statements that allow rulepacks to appear in multiple categories.
* **Fairness:** Teams must not be penalized for telemetry gaps.
* **Honest UI:** Readiness/coverage must always be visible alongside aggregated views.
* **No Momentum ranking:** Momentum must never be used for team comparison or sorting.
* **Fleet scope required:** `activeServiceIds` is required (not optional) for all public methods. Momentum queries must receive fleet scope — they must never define their own fleet boundary.

### Momentum Exposure Contract

> **Momentum is trajectory feedback, NOT a ranking metric.**
>
> The API exposes momentum at team/org level for trajectory visualization only.
> This data MUST NOT be used for:
> - Team comparison or ranking
> - Performance evaluation
> - Sorting by momentum
>
> UI renders Momentum as sparklines and trend labels only.
> If misuse is detected, this field may be removed from aggregate endpoints.

---

# FlightDeckAggregator (View Logic — Service Level)

### **Description** Produces the complete service-level Flight Deck view — cockpit data (Cores + Circuits with state), next best upgrade, flight readiness, protocols list, and evidence payloads. The primary consumer of ConfigResolver and deriveCircuitState at view-time.

**File:** `apps/api/src/view-layer/flight-deck-aggregator.ts`

### Role & Responsibilities

* **`getDeck(serviceId, service, snapshot, options)`** — Build complete `ServiceDeck` with cockpit, next best upgrade, flight readiness, and protocols.
* **`getProtocolDetail(snapshot, protocolId)`** — Detailed protocol view with all rules and their evidence.
* **`getEvidence(snapshot, options)`** — Evidence payload for Evidence Drawer (rule-level, protocol-level, or service-level).
* Private builders:
  * `buildCockpit()` — Service cockpit with readiness, cores gallery, circuits panel, momentum sparkline.
  * `buildCores()` — All cores with state derived via `deriveCoreState()` (earned/not_earned/telemetry_missing/not_applicable).
  * `buildCircuits()` — Circuits with time-based vitality states derived via `deriveCircuitState()`.
  * `buildNextBestUpgrade()` — Selects next best upgrade using leverage scoring and focus mode.
  * `buildFlightReadiness()` — Flight readiness progress from required rulepacks.
  * `buildProtocols()` — Filtered and sorted protocol summaries.

### Constructor Dependencies

`PrismaClient`, `PillarConfig[]`, `RulepackConfig[]`, `CheckRegistry`, `ConfigResolver`, `MomentumQueryService`

### Design Guidelines

* **View-time hydration:** Hydrates `TruthCheckResult` via `ConfigResolver.hydrateCheck()` to produce `ViewCheckResult`. Never access config fields on un-hydrated truth objects.
* **Domain logic from enrichment layer:** Derives CoreState via `deriveCoreState()`, ProtocolState via `deriveProtocolState()`, and CircuitState via `deriveCircuitState()` — all pure functions from enrichment layer. View layer consumes, never defines, state derivation logic.
* **Applicability consumption:** Uses `RulepackApplicability` (stamped at enrichment time) as input to state derivation. Never re-derives applicability from check data.
* **Evidence building:** Check-scoped evidence is assembled from TruthCheckResult.evidence with view-time context (config fields, vocabulary).
* **Evidence completeness:** `buildSignalsForCheck` produces `SignalEvidence` for both evaluable AND blocked checks. Blocked checks use `raw` to carry collector error context (`errorKind`, `message`, `missingContextKey`). Only NEUTRAL checks are excluded (NOT_APPLICABLE is not evidence).
* **Momentum:** Uses `MomentumQueryService` at service altitude (raw delta display).

### Must Not Break

* **View-time hydration:** Never store config fields in snapshots. Always hydrate via ConfigResolver.
* **Enrichment layer imports:** CoreState, ProtocolState, and CircuitState derivation must come from enrichment layer functions, not inline logic.
* **Circuit state derivation:** Must use `deriveCircuitState()` with per-rulepack thresholds from config.
* **Evidence integrity:** Evidence must be traceable to source signals and check results.

---

# TeamConsoleAggregator (View Logic — Team Level)

### **Description** Produces team-level Squad Console views — huddle data, team upgrades, services table, vitality, and closest wins. The team-level counterpart to ReadinessAggregator.

**File:** `apps/api/src/view-layer/team-console-aggregator.ts`

### Role & Responsibilities

* **`getHuddle(teamId, teamName, totalServices, timeframe)`** — Team huddle data with readiness distribution (spine), closest wins count, top blocker, and momentum trajectory.
* **`getUpgrades(teamId, options)`** — Team-level upgrade recommendations with telemetry volume threshold, deduplication, spotlight decoration, and focus mode boost.
* **`getServicesTable(teamId, options)`** — Enriched services table with filtering (by segment), sorting, and text search.
* **`getVitality(teamId)`** — Team vitality with degrading/down circuit counts and top degrading circuits sorted by urgency.
* **`getClosestWinsServices(teamId, teamName, options)`** — Services exactly 1 failing required rulepack away from Flight Ready, with progress context and common blockers.

### Constructor Dependencies

`PrismaClient`, `PillarConfig[]`, `RulepackConfig[]`, `ConfigResolver`, `SpotlightConfig | null`, `ScanQueryService`, `MomentumQueryService?`

### Design Guidelines

* **Active-service filtering:** All public methods accept `activeServiceIds?: ReadonlySet<string>` (provided by the controller from `ServiceDirectory`). Scan results are filtered to active services only before computing any metrics. This ensures team readiness distributions, vitality, closest wins, and service counts exclude deactivated services.
* **Mutual exclusion by design:** Uses `classifyRulepack()` from the kernel to classify each rulepack into exactly one category (BASELINE, VITALITY, or RULE_LEVEL). Telemetry is always processed independently per-check. No post-hoc deduplication needed — the `if/else if` chain prevents cross-category leakage structurally.
* **Degrading circuit sorting:** Circuits sorted by urgency (DOWN before DEGRADING, then by hours since last pass descending).
* **Closest wins definition:** Services with exactly 1 failing required rulepack. These are the team's best "path to completion" targets.
* **Telemetry volume threshold:** At team level, UNBLOCK_TELEMETRY only surfaced when >= 30% of team fleet affected (or >= 3 absolute services).
* **Momentum:** Uses `MomentumQueryService` at team altitude (per-service normalized delta).

### Must Not Break

* **Mutual exclusion:** Must use `classifyRulepack()` for rulepack classification — no independent `if` statements that allow rulepacks to appear in multiple categories.
* **Closest wins definition:** Exactly 1 failing required rulepack — no more, no less.
* **View-time hydration:** Config fields come from ConfigResolver, never from snapshots.
* **Volume threshold:** Telemetry recommendations must respect team-level threshold.
* **Risk transparency:** Team risk is derived from service risks, never hidden or smoothed.

---

# REST API Controllers

### **Description** Resource-oriented interface between UI and backend. Controllers delegate to aggregators and domain services — they contain no domain logic themselves.

### Controller Inventory

| Controller | File | Scope |
|------------|------|-------|
| **Services** | `controllers/services.ts` | Flight Deck: service deck, protocols, evidence, upgrades, next best upgrade |
| **Teams** | `controllers/teams.ts` | Squad Console: huddle, upgrades, services table, vitality, closest wins |
| **Mission Map** | `controllers/org.ts` | Mission Map: team readiness, summary, org upgrades, vitality/upgrade briefs |
| **Scans** | `controllers/scans.ts` | Scan triggers (POST 202), scan status polling |
| **Spotlight** | `controllers/spotlight.ts` | Active spotlight, spotlight brief, spotlight scope with affected services |
| **Evidence** | `controllers/evidence.ts` | Evidence drawer data for rule/protocol/service-level evidence |
| **Config** | `controllers/config.ts` | Configuration metadata: pillars, rulepacks, vocabulary, focus modes |

### Role & Responsibilities

* Serve read-only snapshot-based data quickly.  
* Trigger scans asynchronously (202 Accepted).  
* Validate inputs and configs (422 on validation issues).

### Design Guidelines

* Controllers must not run heavy logic synchronously or contain actual domain logic.  
* Use standard query params for filtering/pagination.  
* Return enriched snapshot and recommendations directly (UI is dumb).
* Delegate to the appropriate aggregator: services → FlightDeckAggregator, teams → TeamConsoleAggregator, mission-map → ReadinessAggregator.
* **Active-service gate:** Controllers fetch `activeServiceIds` from `ServiceDirectory.getAllServiceIds()` and pass the resulting `Set<string>` into aggregator methods. This bridges the catalog domain (what exists) with the scan domain (what was observed), ensuring all metrics use the same population. The controller is the orchestrator — aggregators never access `ServiceDirectory` directly.

### Must Not Break

* **Async contract:** POST /scans must never block on evaluation.  
* **Schema correctness:** invalid YAML/config must fail fast and loud.
* **No domain logic:** Controllers are thin — aggregators own the view logic.

---

# useViewState (Frontend URL State Hook)

### **Description** React hook that provides URL-driven state management for all frontend views. Enables deep-linking, shareability, and view-specific defaults.

> **DESIGN PRINCIPLE: The View Owns Its Defaults**
> Each view declares its own defaults via the `defaults` parameter.
> This ensures navigation callers (like `navigateToTeam`) don't need to know what defaults each view requires.
> Encapsulation is preserved: the view's default behavior is defined in the view itself.

### Role & Responsibilities

* Parse URL search params into typed `ViewState` object.
* Provide state setters that update URL (enabling deep-linking).
* Accept view-specific `defaults` parameter for customized default behavior.
* Apply priority: **URL param > view default > global default**.
* Provide navigation helpers (`navigateToService`, `navigateToTeam`).

### View-Specific Defaults Contract

| View | Default `showVitality` | Rationale |
|------|------------------------|-----------|
| Mission Map (org) | `false` | Operational health is optional context at org level |
| Squad Console (team) | `true` | Team leads need operational context by default |
| Flight Deck (service) | always visible | No toggle; vitality is always shown for service view |

### Usage Example

```typescript
// Mission Map (org view) - vitality OFF by default
const { showVitality } = useViewState({ defaults: { showVitality: false } });

// Squad Console (team view) - vitality ON by default
const { showVitality } = useViewState({ defaults: { showVitality: true } });
```

### Design Guidelines

* **URL is source of truth:** All view state is reflected in URL for shareability.
* **View-owned defaults:** Each view declares its own defaults, not navigation callers.
* **Explicit over implicit:** Even when a view's default matches the global default, declare it explicitly for clarity.
* **No magic:** If a view needs different behavior, it must explicitly pass `defaults`.

### Must Not Break

* **Priority order:** URL param must always win over view default, which wins over global default.
* **Caller ignorance:** Navigation functions (`navigateToTeam`) must NOT need to know view defaults.
* **Encapsulation:** View-specific behavior lives in the view component, not in shared hooks or navigation utilities.
* **Deep-linking:** Any URL must produce deterministic view state when loaded directly.

---

# Scan Producer (API → Queue)

### **Description** Resolves target services and enqueues one BullMQ job per service via `addBulk`. Pure fire-and-forget fan-out — no status tracking.

### Role & Responsibilities

* Accept scan requests (service/team/org).  
* Resolve service IDs from ServiceDirectory.  
* Enqueue individual per-service jobs to BullMQ.

### Design Guidelines

* Idempotency (dedupe optional, but avoid spamming).  
* Fast response (always 202).

### Must Not Break

* **UI responsiveness:** scan trigger must be immediate.

---

# Worker (Consumer)

### **Description** Evaluates a single service per BullMQ job: orchestrator run + persistence. Also handles scheduled fan-out triggers (resolve all services, re-enqueue individual jobs).

### Role & Responsibilities

* Evaluate exactly one service per job (collect, evaluate, enrich, persist).  
* After `orchestrator.evaluate()`, enrich with `lastPassedAt` for CIRCUIT rulepacks via VitalityHistoryService.
* Upsert `ServiceState` (current-truth table) for fast aggregation reads.
* Handle scheduled fan-out: resolve all org services, enqueue individual jobs via `addBulk`.
* Log failures with serviceId context.

### Design Guidelines

* Stateless and horizontally scalable (HPA on queue depth).  
* Do not crash on partial failures; honor universal collection.
* Scheduler (see Scheduler section) registers repeatable cron jobs that trigger org-wide scans via this worker.

### Must Not Break

* **Reliability:** worker must never leave scans in "hung" state.  
* **Determinism:** same inputs yield consistent evaluation behavior.

---

# Scheduler (Repeatable Scan Registration)

### **Description** Registers repeatable org-wide scans using BullMQ's native repeatable jobs feature. Jobs persist in Redis across server restarts.

**File:** `apps/api/src/queue/scheduler.ts`

### Role & Responsibilities

* **`startScheduler(config)`** — Register repeatable org-wide scans with BullMQ cron expression (default: `"0 6,18 * * *"` — 6 AM and 6 PM UTC).
* **`stopScheduler()`** — Stop scheduler. Intentionally does NOT remove repeatable jobs from Redis.
* On startup: removes any existing repeatable job with the same name (clean slate), then registers new repeatable job with configured cron pattern.

### Design Guidelines

* Uses BullMQ's native repeatable jobs — no custom scheduling logic.
* Fixed `jobId` prevents duplicate registrations.
* Job registration is idempotent (same jobId = same job).
* **Does NOT remove jobs on shutdown** — this prevents breaking scheduling during rolling deploys where old instances stop before new instances start.

### Must Not Break

* **Idempotent registration:** Multiple instances starting simultaneously must not create duplicate jobs.
* **No job removal on shutdown:** Rolling deploy safety requires jobs to persist across restarts.
* **Cron configurability:** Cron expression must be configurable, not hardcoded.

---

# Catalog Layer

The catalog layer manages the integration with external service catalogs (e.g., Backstage) and provides a stable read model for service/team metadata. This layer is **separate from the Engine** and never participates in scan-time execution.

---

# Backstage Entity Mapping Convention

### **Description** Defines how Backstage entities are mapped to Ground Control's internal data model with simplified, consistent naming.

> **DESIGN PRINCIPLE: Simplified Entity Naming**
> Every entity has exactly two naming properties: `id` and `displayName`.
> - `id`: Stable identifier for routing, linking, and database keys (e.g., `"platform-team"`)
> - `displayName`: Human-readable name for UI display (e.g., `"Platform Engineering"`)
>
> This eliminates the previous redundancy of `name` + `title` fields and provides a clear contract.

### Mapping Rules

| Backstage Source | Ground Control Field | Notes |
|------------------|---------------------|-------|
| `metadata.name` | `id` | Stable identifier, used as primary key |
| `spec.profile.displayName` | `displayName` (priority 1) | Best source for teams/groups |
| `metadata.title` | `displayName` (priority 2) | Fallback when profile.displayName absent |
| `metadata.name` | `displayName` (priority 3) | Final fallback when no title exists |

### Service Entity Mapping

```typescript
interface ServiceInput {
  id: string;           // Backstage metadata.name
  displayName: string;  // metadata.title ?? metadata.name
  ownerTeamId: string;  // Extracted from spec.owner relation
  // ... other fields
}
```

### Team/Group Entity Mapping

```typescript
interface GroupInput {
  id: string;           // Backstage metadata.name
  kindType: 'TEAM' | 'GROUP';
  displayName: string;  // spec.profile.displayName ?? metadata.title ?? metadata.name
  parentId: string | null;
}
```

### API Response Contract

All API responses use `displayName` for human-readable names:

```typescript
// GET /services/:id
{
  id: "payment-service",
  displayName: "Payment Service",  // NOT name + title
  ownerTeamId: "platform-team"
}

// GET /teams/:id
{
  id: "platform-team",
  displayName: "Platform Engineering"  // NOT name + title
}

// ServiceCockpit (Flight Deck)
{
  serviceId: "payment-service",
  serviceName: "Payment Service",      // Resolved from displayName
  ownerTeamId: "platform-team",        // ID for routing/linking
  ownerTeamName: "Platform Engineering" // Resolved displayName for display
}
```

### Must Not Break

* **Two-field contract:** Every entity has exactly `id` + `displayName`, never `name` + `title`.
* **Stable IDs:** `id` comes from `metadata.name`, never changes after initial sync.
* **Display priority:** `spec.profile.displayName` > `metadata.title` > `metadata.name`.
* **Explicit resolution:** Owner team names are resolved at view-time (ServiceCockpit.ownerTeamName).
* **No redundancy:** API responses never return both `name` and `title` fields.

---

# BackstageCatalogClient (API Adapter)

### **Description** Abstraction over the Backstage Catalog API. Talks to Backstage and maps entities to internal types.

### Role & Responsibilities

* Authenticate with Backstage (token, mTLS, service account).
* List/get entities (Components, Groups).
* Map Backstage entities → internal `ServiceInput` / `GroupInput` using the Entity Mapping Convention above.
* Handle pagination, retries, rate limits.
* Encapsulate Backstage-specific details (annotations, labels, refs).

### Design Guidelines

* Keep interface minimal: `listComponents()`, `listGroups()`, `getEntityByRef()`.
* Isolate Backstage version differences here.
* Implement robust retry/backoff and clear error mapping.
* Make it swappable: future support for ServiceNow, custom catalogs, etc.

### Must Not Break

* **Scan-time isolation:** must NEVER be called during scan execution.
* **Single responsibility:** no business logic, scoring, or evaluation awareness.
* **Stable mapping:** entity-to-model mapping must be consistent and deterministic.
* **Error transparency:** API failures must surface clearly (not swallowed).

---

# CatalogSyncService (ETL Job)

### **Description** The ETL job that keeps the database in sync with Backstage. Runs on startup and periodically.

### Role & Responsibilities

* Run initial sync on application startup (block until complete).
* Run periodic sync at configurable intervals (default: 60 minutes).
* Upsert services/teams by stable ID (Backstage entity ref).
* Soft-delete removed entities (`isActive=false`), never hard-delete.
* Log sync statistics (added, updated, deactivated, total).
* Expose health status for observability.

### Design Guidelines

* Sync job is **independent** from scan execution.
* Implement idempotent upserts (same input = same DB state).
* Use transactions for atomicity where practical.
* Emit metrics for sync duration, entity counts, failure rates.
* Support manual trigger for testing/debugging.

### Must Not Break

* **Startup guarantee:** API must not serve requests until initial sync completes.
* **Soft-delete rule:** never hard-delete entities (preserve history, avoid broken refs).
* **Non-blocking scans:** slow or failed sync must not block scan workers.
* **Resilience:** Backstage downtime must not crash the sync service—retry and log.

---

# ServiceDirectory (Read-Only Resolver)

### **Description** The single source of truth for service metadata during scan execution. Reads from the DB mirror, never from Backstage directly.

### Role & Responsibilities

* Get service by ID (fast, from DB).
* List services with filters (team, tier, lifecycle).
* Provide canonical `ServiceIdentity` to scan workers.
* Get team by ID.
* List all teams.
* **`getAllServiceIds()`** — Get all active service IDs. Used by controllers to build the `activeServiceIds` set that aggregators use to filter scan results. This is the canonical source of "which services count" for all metrics.
* **`countTeamsWithActiveServices()`** — Count teams that own at least one active service (the honest "teams Ground Control can see" count).
* **`countServicesByTeam()`** — Count active services grouped by owning team (used as the honest denominator for coverage and momentum).
* Provide counts for pagination.

### Design Guidelines

* Keep API read-only: no mutations.
* Optimize for speed: scans must not wait for metadata lookups.
* Return `null` or throw `ServiceNotFoundError` for missing/inactive services.
* Never fall back to Backstage API—DB is the truth during scans.

### Must Not Break

* **No Backstage dependency:** ServiceDirectory must NEVER call Backstage API.
* **Scan stability:** same serviceId must return same metadata during a scan.
* **Fast reads:** lookup must be O(1) or indexed query—no full scans.
* **Inactive handling:** inactive services must not appear in normal queries.

---

# Catalog Layer — Critical Invariant

> **The Engine must never require Backstage at scan-time.**
> 
> Only the CatalogSyncService talks to Backstage. The scan worker relies exclusively on the DB mirror (via ServiceDirectory) for service metadata. This preserves:
> 
> * **Resilience** — Backstage downtime doesn't stop scans
> * **Performance** — No API latency in the scan hot path  
> * **Determinism** — Scan input is stable throughout execution
> * **Testability** — Mock ServiceDirectory, not Backstage API

---

# Observability Layer

The observability layer integrates with external metrics/monitoring systems (e.g., Datadog) to collect runtime signals. Like the Catalog Layer, it is **separate from evaluation logic** and provides typed context data for checks.

---

# DatadogClient (Datadog API Wrapper)

### **Description** Thin wrapper around Datadog Metrics API. Handles authentication, query construction, and response normalization.

### Role & Responsibilities

* Configure API client with API key, app key, and site.
* Build scoped metric query strings with proper escaping.
* Execute time series queries with explicit `from/to` time bounds.
* Normalize API responses to typed internal structures.
* Handle pagination for large result sets.

### Design Guidelines

* Keep methods focused on data retrieval (no business logic).
* Use native `fetch` to avoid heavy dependencies.
* Query builder must be pure and unit-testable.
* Implement timeout and retry with exponential backoff.
* Never interpret metric values—return raw data.

### Must Not Break

* **Type safety:** always return typed data, never raw API responses.
* **Isolation:** client failures must not crash the collector.
* **Testability:** query construction logic must be pure and unit-testable.
* **No evaluation logic:** client does not decide PASS/FAIL, only fetches data.

---

# DatadogCollector (Runtime Metrics Collector)

### **Description** Collects runtime stability signals from Datadog for a service. Returns `DatadogData` with restarts, OOM kills, crash loops, and fleet context.

### Role & Responsibilities

* Map service identity to Datadog metric scope (cluster, namespace, workload, container).
* Query multiple metrics in parallel (restarts, OOM, crash loops, replica count).
* Enforce 24-hour rolling window at query time (`now - 24h` to `now`).
* Aggregate per-pod breakdowns for scale-aware analysis.
* Decide and emit: success vs COLLECTOR_FAILED vs NOT_APPLICABLE.

### Design Guidelines

* Window is computed at collection time, not configured.
* Scope queries strictly: cluster + namespace + workload + `container:main`.
* Return NOT_APPLICABLE when:
  * Datadog integration not configured (no API keys)
  * Service has no k8s metadata (no cluster/namespace)
* Return COLLECTOR_FAILED on API errors, timeouts, or missing required metrics.
* Never hardcode thresholds—collector returns raw counts.

### Must Not Break

* **Universal collection contract:** must not throw; must report failures via registry.
* **No business meaning:** no scoring, no tiers, no badge logic.
* **Window integrity:** every query must use explicit `from/to` bounds, not rollup defaults.
* **Isolation:** Datadog API slowness must never block other collectors.
* **Scope correctness:** exclude init containers, sidecars, and canary workloads.

---

# DatadogCollector — Window Contract

> **The 24-hour contract is enforced at collection time.**
>
> * Window: `[now - 24h, now]` computed at the moment of collection.
> * Scan frequency must not affect outcomes—same service at same "now" scores the same.
> * API time bounds define the window; rollups are for aggregation efficiency only.
> * The collector is stateless—it does not remember previous scans.

---

# RuntimeStabilityCheck (Crashes Composite Check)

### **Description** Evaluates all crash signals (restarts, OOM, CrashLoopBackOff) and returns a tier level for the LADDER badge.

### Role & Responsibilities

* Read `DatadogData` from context.
* Compute composite crash count across all signal types.
* Calculate scale-aware metrics: `maxCrashesPerPod`, `podsAffectedPct`.
* Determine tier (GOLD/SILVER/BRONZE/FAIL) based on configured thresholds.
* Return PASS only for GOLD (badge earned), FAIL for all other tiers.
* Include all dimensions in `observedValue` for explainability.

### Design Guidelines

* Thresholds come from YAML params, never hardcoded.
* Tier calculation is a pure function (same inputs → same tier).
* `observedValue` must include: tier, totalCrashes, maxCrashesPerPod, podsAffected, podsAffectedPct, desiredReplicas, window.
* Handle missing context explicitly (ERROR with appropriate kind).
* Any crash blocks GOLD—this is the "badge purity" contract.

### Must Not Break

* **No IO rule:** check reads only from context, never calls Datadog directly.
* **Badge purity:** GOLD requires zero crashes of ANY type (restart, OOM, crashloop).
* **Scale fairness:** use `podsAffectedPct` to normalize across fleet sizes.
* **Explainability:** `observedValue` must provide all evidence for UI rendering.
* **Tier determinism:** same DatadogData + same thresholds = same tier.

---

# RuntimeStabilityCheck — Badge Philosophy

> **"If you had one crash, you can't have the badge."**
>
> * GOLD = badge earned (zero crashes in 24h window)
> * SILVER/BRONZE = progress indicators (improving, but no badge)
> * Below BRONZE = actively failing
>
> The check returns `status: PASS` only for GOLD. All other tiers return `status: FAIL` with `observedValue.tier` set to the achieved tier. The Enricher reads this tier to set the badge level and score.
>
> This preserves the product philosophy: the badge is binary (earned or not), but scores show progress.

---

# Enricher — LADDER Badge Evaluation (Extension)

### **Additional Responsibilities for LADDER Badge Type**

* For LADDER rulepacks, read `observedValue.tier` from check results.
* Map tier to badge level: GOLD → `GOLD`, SILVER → `SILVER`, BRONZE → `BRONZE`, else → `FAIL`.
* Compute rulepack score: GOLD=100, SILVER=80, BRONZE=60, FAIL=0.
* If multiple checks in a LADDER rulepack, use the **lowest tier** across all checks.
* Badge is "earned" only at GOLD level for minBaseline=GOLD rulepacks.

### Design Guidelines (LADDER-specific)

* Tier must be explicitly present in `observedValue`—do not infer from status.
* Score mapping is configurable per rulepack (default: 100/80/60/0).
* Rationale must explain which tier was achieved and why.

### Must Not Break (LADDER-specific)

* **Tier source:** badge level must come from check's `observedValue.tier`, not check status.
* **Lowest tier wins:** multiple checks in rulepack → take minimum tier.
* **Score consistency:** tier → score mapping must be deterministic and documented.

---

# CI/CD Layer

The CI/CD layer integrates with GitLab to collect pipeline performance and stability signals. It evaluates the **last N merged MRs** (event window) rather than a fixed time window, aligning with process-oriented metrics.

---

# CiCollector (CI/CD Metrics Collector)

### **Description** Collects CI/CD signals from GitLab by resolving merged MRs to their post-merge pipelines.

### Role & Responsibilities

* Map service identity to GitLab project.
* Fetch last N merged MRs targeting default branch.
* Resolve each MR to its merge-result commit SHA (merge_commit_sha ?? squash_commit_sha ?? sha).
* Find the pipeline that ran on the default branch for that SHA.
* Fetch pipeline jobs to detect retries (flakiness signal).
* Decide and emit: success vs COLLECTOR_FAILED vs NOT_APPLICABLE.

### Design Guidelines

* Event window is MR-based (last N merges), not time-based.
* Non-real-time: if pipeline not yet visible, skip MR (next scan picks it up).
* Use bounded concurrency when hydrating MRs (avoid rate limits).
* Never hardcode thresholds—collector returns raw durations and counts.

### Must Not Break

* **Universal collection contract:** must not throw; must report failures via registry.
* **No business meaning:** no scoring, no tiers, no badge logic.
* **Non-real-time contract:** missing pipeline = skip, not ERROR.
* **Isolation:** GitLab API slowness must never block other collectors.

---

# CiCollector — Event Window Contract

> **The event window is defined by merged MRs, not calendar time.**
>
> * Window: Last N merged MRs targeting the default branch (default: 20).
> * Each MR is resolved to its merge-result commit SHA.
> * Only MRs with discoverable default-branch pipelines are included.
> * If a recent merge's pipeline isn't visible yet, skip it (non-real-time).
> * This approach avoids noise from non-release pushes (readme fixes, schedules).

---

# ChangeStabilityCheck (GATEKEEPER Check)

### **Description** Evaluates CI stability over the event window. Returns clean rate for GATEKEEPER threshold evaluation.

### Role & Responsibilities

* Read `CiData` from context.
* Count clean merges (pipeline.isClean === true).
* Calculate clean rate (cleanMerges / totalMerges).
* Return cleanRate in observedValue for threshold evaluation by Enricher.
* Include flakyEvents in observedValue for explainability.

### Design Guidelines

* Thresholds come from YAML params, never hardcoded.
* observedValue must include: cleanRate, cleanMerges, totalMerges, flakyEvents.
* Handle missing context explicitly (ERROR with appropriate kind).

### Must Not Break

* **No IO rule:** check reads only from context, never calls GitLab directly.
* **Threshold evaluation:** PASS/FAIL determined by Enricher comparing cleanRate to threshold.
* **Explainability:** observedValue must list which MRs failed.

---

# DeploymentSpeedCheck (LADDER Check)

### **Description** Evaluates deployment speed via P90 pipeline duration. Returns tier for LADDER badge.

### Role & Responsibilities

* Read `CiData` from context.
* Filter to successful pipelines only.
* Calculate P90 of durationSeconds using nearest-rank method.
* Determine tier (GOLD/SILVER/BRONZE/FAIL) based on thresholds.
* Return PASS only for GOLD, FAIL for other tiers.

### Design Guidelines

* Require minimum sample count (e.g., 5) before evaluating.
* Thresholds come from YAML params.
* observedValue must include: tier, p90DurationSeconds, sampleCount, thresholds.

### Must Not Break

* **No IO rule:** check reads only from context.
* **LADDER contract:** check must return tier in observedValue for enricher.
* **Sample requirement:** insufficient samples = WARN, not arbitrary tier.

---

# GitOps Layer

The GitOps layer discovers and parses Kubernetes manifests from Git repositories using convention-based paths. Like other collectors, it provides typed context data for checks without interpreting results.

---

# GitOpsCollector (K8s Manifest Discovery)

### **Description** Discovers and parses Kubernetes manifests from Git using a convention-based directory structure. Returns typed `GitOpsData` with workload analysis, autoscaler config, and resource definitions.

**File:** `apps/api/src/engine/collectors/gitops/gitops-collector.ts`

### Role & Responsibilities

* Map service identity to Git repository and discover manifest path.
* Search `.af/deployments/{serviceId}/environments/{env}/{region}/*.yaml` directory structure.
* List available environments, prioritize configured environment (default: production), fall back to first available.
* Parse YAML files into typed K8s resources (workloads, configMaps, secrets, services).
* Build analysis: workload type, autoscaler presence, container images, resource requests/limits, config externalization.
* Decide and emit: success vs COLLECTOR_FAILED vs NOT_APPLICABLE.

### Discovery Convention

```
.af/deployments/{serviceId}/environments/{env}/{region}/*.yaml
```

The collector automatically discovers available environments and regions — no hardcoded paths. Environment priority: configured environment (e.g., `production`) > first available.

### Design Guidelines

* Discovery is convention-based: if directory structure doesn't match, emit NOT_APPLICABLE.
* Never hardcode environment names or region lists.
* Parser handles multi-document YAML files (separated by `---`).
* Return NOT_APPLICABLE when Git provider is not configured or repo has no `.af/` directory.

### Must Not Break

* **Universal collection contract:** Must not throw; must report failures via registry.
* **No business meaning:** No scoring, no tiers, no pass/fail logic.
* **Discovery convention:** `.af/deployments/{serviceId}/environments/` path pattern is the contract.
* **Isolation:** Git API slowness must never block other collectors.

---

# SonicCollector (Sonic Namespace Collector)

### **Description** Gathers Sonic namespace configuration and dependency analysis from `.af/namespaces/default.json`. Returns `SonicData` with namespace config, dependency graph, and vulnerability score.

**File:** `apps/api/src/engine/collectors/sonic/sonic-collector.ts`

### Role & Responsibilities

* Read `.af/namespaces/default.json` from Git repository via git provider.
* Parse Sonic config JSON to extract basic config (team, namespace, envType).
* Extract dependencies from join array (services and resources with URLs).
* Calculate vulnerability score: `(direct_services × 3) + (direct_resources × 2)`.
* Decide and emit: success vs COLLECTOR_FAILED vs NOT_APPLICABLE.

### Design Guidelines

* Return NOT_APPLICABLE when Git provider is not configured or file doesn't exist.
* Direct dependencies only — no recursive fetching.
* Parse join URLs to extract dependency type and name.

> **Known concern:** The `calculateVulnerabilityScore()` formula borders on "business meaning" — it applies weights to dependency counts. This may be better suited as check-level logic rather than collector-level. Flagged for architectural review.

### Must Not Break

* **Universal collection contract:** Must not throw; must report failures via registry.
* **No tiers/badges:** Collector does not determine pass/fail or badge levels.
* **Isolation:** Slow Git API must never block other collectors.

---

# VitalityHistoryService (V3 — Last Passed Timestamp Finder)

### **Description** Queries historical scans to find `lastPassedAt` timestamp for CIRCUIT rulepacks. Produces the "temporal truth" used to derive CircuitState at view-time.

> **CRITICAL: SCAN-CADENCE INDEPENDENCE**
> By storing a **timestamp** (not a scan count), the derived CircuitState is independent of how frequently we scan. The same real-world situation produces the same CircuitState regardless of scan frequency.

> **TRUTH vs MEANING**
> - `lastPassedAt` is TRUTH: ISO timestamp stored in snapshot
> - `CircuitState` is MEANING: derived at view-time from `(now - lastPassedAt)` + hour-based VitalityThresholds

### Role & Responsibilities

* Query historical scans for a service (bounded by `vitalityLookbackScans` engine config).
* For each CIRCUIT rulepack currently failing, find the most recent scan where `baselineMet=true`.
* Return a map of `rulepackId → lastPassedAt`.
* For rulepacks currently passing, return `undefined` (Circuit is NOMINAL).

### Design Guidelines

* This is a scan-time computation—runs in the worker after orchestrator.evaluate().
* Query is bounded by `vitalityLookbackScans` (engine config, default 10).
* Pure timestamp finding logic—no interpretation or state derivation.
* Resilient to missing data: if no pass found in history, return `undefined`.

### Must Not Break

* **Truth only:** Must NOT derive CircuitState—that's the view layer's job.
* **Bounded query:** Must respect `vitalityLookbackScans` limit.
* **Timestamp, not count:** Must return timestamp for scan-cadence independence.
* **Graceful degradation:** Empty history or no pass found = `undefined`.

---

# deriveCircuitState (V3 — View-Time CircuitState Derivation)

### **Description** A pure function that derives CircuitState from `lastPassedAt` timestamp using hour-based `VitalityThresholds`. This is called at view-time, NOT stored.

> **THE CORE TRUTH**
> DEGRADING/DOWN answers: "How long has this problem existed?" — not how many scans failed.
> This is fair: same problem duration = same state, regardless of activity level.

### Function Signature

```typescript
function deriveCircuitState(
  lastPassedAt: string | undefined,
  thresholds: VitalityThresholds,
  now: Date = new Date()
): CircuitState;
```

### Role & Responsibilities

* Calculate `hoursSinceLastPass = (now - lastPassedAt)` in hours.
* Compare against `degradingAfterHours` and `downAfterHours` thresholds.
* Return CircuitState: `NOMINAL`, `DEGRADING`, or `DOWN`.
* Pure function—no side effects, no database access.

### CircuitState Derivation Logic

* `lastPassedAt === undefined` → `NOMINAL` (currently passing or first scan)
* `hoursSinceLastPass >= downAfterHours` → `DOWN`
* `hoursSinceLastPass >= degradingAfterHours` → `DEGRADING`
* Otherwise → `NOMINAL`

### Design Guidelines

* Called at view-time by `FlightDeckAggregator.buildCircuits()`.
* Thresholds come from `getVitalityThresholdsForRulepack()` for per-rulepack support.
* If thresholds change, ALL historical views update immediately.
* Scan-cadence independent: same duration = same state.

### Must Not Break

* **Pure function:** No side effects, no database access.
* **View-time only:** Never called during scan—only when building API responses.
* **Config-driven:** Must use thresholds from config, not hardcoded values.
* **Undefined is NOMINAL:** `lastPassedAt === undefined` must always return `NOMINAL`.
* **Hour-based comparison:** Must use hours for scan-cadence independence.

---

# VitalityThresholds (V3 — CircuitState Interpretation Config)

### **Description** Configuration interface defining hour-based thresholds for interpreting time since last pass into CircuitState.

> **SCAN-CADENCE INDEPENDENCE**
> By using hours (not scan counts), the same real-world situation produces the same CircuitState regardless of how often we scan. This ensures fairness: same problem duration = same state.

> **TWO DIMENSIONS (Conceptually Separate)**
> - Check's window (e.g., 24h crashes, 20 MRs) = measurement scope (truth)
> - VitalityThresholds (hours) = coaching urgency (meaning)

### Interface

```typescript
interface VitalityThresholds {
  degradingAfterHours: number;  // default: 12 (hours until DEGRADING)
  downAfterHours: number;       // default: 36 (hours until DOWN)
}
```

### Role & Responsibilities

* Define the hour-based interpretation rules for CircuitState derivation.
* Per-rulepack configurable via `rulepack.vitalityThresholds` in YAML.
* Falls back to global defaults via `getVitalityThresholdsForRulepack()`.
* Separate from engine config (`vitalityLookbackScans`).

### Per-Rulepack Urgency Profiles

Different Circuits can have different urgency:

```yaml
# High urgency (runtime crashes): shorter thresholds
vitalityThresholds:
  degradingAfterHours: 6
  downAfterHours: 24

# Lower urgency (CI flakiness): longer thresholds
vitalityThresholds:
  degradingAfterHours: 24
  downAfterHours: 72
```

### Config Separation

| Config Type | Name | Controls | When Applied |
|-------------|------|----------|--------------|
| Engine | `vitalityLookbackScans` | How many scans to search for lastPassedAt | Scan-time |
| Meaning | `VitalityThresholds` | How to interpret (now - lastPassedAt) | View-time |

### Must Not Break

* **View-time application:** Thresholds must be applied at view-time, not stored.
* **Per-rulepack with fallback:** Check rulepack config first, then global defaults.
* **Config change = instant update:** Changing thresholds updates all historical views.
* **Hour-based:** Must use hours for scan-cadence independence.

---

# CoreEvaluator (View-Time Core State Derivation)

### **Description** A pure function that derives CoreState from badge evaluation truth and rulepack applicability. Complements `deriveCircuitState()` for Circuits, providing symmetric state derivation for Core achievements.

**File:** `apps/api/src/enrichment/core-evaluator.ts`

### Function Signature

```typescript
function deriveCoreState(
  applicability: RulepackApplicability,
  baselineMet: boolean,
  hasBlockedChecks: boolean,
): CoreState;  // 'earned' | 'not_earned' | 'telemetry_missing' | 'not_applicable'
```

### Role & Responsibilities

* Determine Core achievement state at view-time from truth values.
* Priority logic (strict order):
  * `applicability === 'NOT_APPLICABLE'` → `not_applicable`
  * `applicability === 'BLOCKED'` or `hasBlockedChecks === true` → `telemetry_missing`
  * `baselineMet === true` → `earned`
  * Otherwise → `not_earned`

### Design Guidelines

* Pure function — no side effects, no database access.
* Called at view-time by `FlightDeckAggregator.buildCores()`.
* Domain logic isolation: defines what CoreState MEANS; belongs in enrichment layer, not view layer.
* View aggregators consume this logic; they do not define it.
* **Applicability-first:** `not_applicable` takes precedence over all other states. A rulepack that doesn't apply to a service must never appear as `not_earned`.

### Must Not Break

* **Pure function:** No side effects, no database access.
* **Priority order:** `not_applicable` > `telemetry_missing` > `earned` > `not_earned`.
* **Applicability source:** Must accept `RulepackApplicability` from enrichment layer, not derive it inline.
* **View-time only:** Called when building API responses, not during scan.
* **Enrichment layer ownership:** This logic defines domain meaning and must live in enrichment, not views.

---

# ProtocolState Derivation (View-Time Protocol State)

### **Description** A pure function that derives ProtocolState from rulepack applicability and badge evaluation truth. Provides consistent state derivation for the Flight Readiness protocol table.

**File:** `apps/api/src/enrichment/core-evaluator.ts`

### Function Signature

```typescript
function deriveProtocolState(
  applicability: RulepackApplicability,
  baselineMet: boolean,
): ProtocolState;  // 'met' | 'below' | 'telemetry_missing' | 'not_applicable'
```

### Role & Responsibilities

* Determine protocol state at view-time from truth values.
* Priority logic (strict order):
  * `applicability === 'NOT_APPLICABLE'` → `not_applicable`
  * `applicability === 'BLOCKED'` → `telemetry_missing`
  * `baselineMet === true` → `met`
  * Otherwise → `below`

### Design Guidelines

* Pure function — no side effects, no database access.
* **Single derivation point:** Both cockpit cores and protocol table derive state from the same enrichment-layer applicability. No duplicate logic.
* Domain logic isolation: view layer consumes, never defines, state derivation logic.

### Must Not Break

* **Pure function:** No side effects, no database access.
* **Priority order:** `not_applicable` > `telemetry_missing` > `met` > `below`.
* **Applicability source:** Must accept `RulepackApplicability` from enrichment layer.
* **Enrichment layer ownership:** This logic must live in enrichment, not views.

---

# Momentum System (Domain Layer)

The Momentum system computes trajectory feedback from badge states. It is a full domain subsystem with its own calculator, history service, trajectory builder, and constants. Momentum is **secondary to readiness and upgrades** and must never be used for ranking.

> **See [Constitution Section 9](./Ground%20Control%20Constitution.md#9-momentum-trajectory) for canonical Momentum definitions and guardrails.**

---

# MomentumCalculator (Pure Earning Logic)

### **Description** Pure domain logic for momentum calculation using the Hybrid Earning Model: Core Burst + Circuit Daily.

**File:** `apps/api/src/domain/momentum/calculator.ts`

### Role & Responsibilities

* **`calculateDailyMomentum(state)`** — Calculate daily momentum from badge state: `(newCoresEarned × CORE_BURST) + (circuitCount × CIRCUIT_DAILY)`.
* **`buildDailyMomentumSeries(dailyStates)`** — Build daily momentum series from an array of badge states.
* **`detectNewCores(currentCoreCount, previousCoreCount)`** — Detect new cores earned by comparing consecutive scans.
* **`calculateDailyRate(circuitCount)`** — Calculate current daily rate from latest state (Circuits only, no burst).

### Earning Formula

| Event | Momentum Earned |
|-------|-----------------|
| Core first earned | +100 (one-time burst, `CORE_BURST`) |
| Circuit active per day | +10/day (`CIRCUIT_DAILY`) |

### Design Guidelines

* Pure functions — no IO, no database access.
* Scan-cadence independent: same operational state produces same momentum regardless of scan frequency.
* Core burst is awarded once when a Core is first earned (detected by comparing coreCount across scans), not repeatedly per scan.

### Must Not Break

* **Earning rates:** Must use constants from `MomentumConstants` — never hardcode values.
* **Scan-cadence independence:** Same badge state must produce same momentum regardless of how many times scanned.
* **No negative momentum:** Momentum can never go below zero.
* **Pure functions:** No side effects, no database access.

---

# MomentumHistoryService (Series Construction)

### **Description** Transforms sparse database rows into contiguous daily or weekly arrays with gap-filling. Shared history-array construction for all aggregators.

**File:** `apps/api/src/domain/momentum/history-service.ts`

### Role & Responsibilities

* **`buildHistoryArray(rows, timeframe)`** — Build contiguous daily history array from sparse DB rows, filling missing days with 0. Aggregates to weekly buckets for 90d timeframe.
* **`aggregateToWeekly(dailyData, targetWeeks)`** — Aggregate daily data into weekly buckets (sums daily values).

### Design Guidelines

* Gap-filling: missing days get 0 momentum (not interpolated).
* Today's data is always excluded (incomplete day would skew trends).
* For 90d timeframe: daily array is aggregated into ~12 weekly buckets for readable sparklines.

### Must Not Break

* **Gap-filling with zero:** Missing days must be 0, not interpolated or omitted.
* **Today excluded:** Today's incomplete data must never appear in series.
* **Correct bucketing:** Weekly aggregation must sum daily values, not average.

---

# Readiness Domain (Pure Functions)

### **Description** Pure domain functions for service readiness derivation and fleet-level coverage computation. No dependencies — easy to test, impossible to couple to infrastructure.

**File:** `apps/api/src/domain/readiness.ts`

### Role & Responsibilities

* **`deriveServiceReadiness(pillarReadiness)`** — Derive overall service readiness from per-pillar states (FLIGHT_READY / PRE_FLIGHT / TELEMETRY_MISSING).
* **`computeFleetCoveragePct(withTelemetry, totalServices)`** — Single source of truth for fleet-level telemetry coverage percentage. "With telemetry" = FLIGHT_READY + PRE_FLIGHT (not TELEMETRY_MISSING). Result is clamped to `Math.min(100, ...)` as a safety net against data inconsistencies. Used by ReadinessAggregator (org), ReadinessAggregator (team), and TeamConsoleAggregator (team huddle).

### Must Not Break

* **Single source of truth:** All coverage computations must call `computeFleetCoveragePct` — never inline the formula.
* **Pure functions:** No side effects, no database access.
* **Zero denominator safety:** Returns 0 when `totalServices <= 0`.

---

# TrajectoryBuilder (Multi-Altitude Trajectory)

### **Description** Builds `MomentumData` objects with series, delta, trend, daily rate, and altitude-specific labels. Three altitude-aware builders answer different questions at each view level.

**File:** `apps/api/src/domain/momentum/trajectory.ts`

### Role & Responsibilities

* **`buildMomentumData(series, timeframe, circuitCount?)`** — Service altitude: "Is my work compounding?" Raw delta display (e.g., "+280 · 7d").
* **`buildTeamMomentumData(series, serviceCount, timeframe)`** — Team altitude: "Is the team improving?" Per-service normalized delta (e.g., "+11 /svc · 7d").
* **`buildOrgMomentumData(series, fleet: FleetDeltas, timeframe)`** — Org altitude: "Is the culture spreading?" Gaining fleet percentage (e.g., "Gaining: 57%"). Accepts a single `FleetDeltas` object — denominator is derived from the same scoped dataset as the numerator. Separate denominator parameters are forbidden.
* **`FleetDeltas`** — Domain interface: `{ deltas: ReadonlyMap<string, number>, fleetSize: number }`. Pre-scoped fleet momentum deltas where numerator and denominator come from the same universe. `deltas` is the per-service delta map from `MomentumQueryService.getServiceDeltas()`, `fleetSize` is the total active fleet count from `ServiceDirectory`.
* **`calculateTrend(series)`** — Calculate trend direction by comparing recent vs earlier averages: `up` (> 10% higher), `down` (< 90%), `flat` (within 10%).
* **`directionWord(trend)`** — Map trend to display word: up → "gaining", down → "easing", flat → "steady".

### Design Guidelines

* Each altitude answers exactly one question — see Constitution Section 9.
* Trend uses a 10% threshold for direction detection.
* Direction words use neutral language: "easing" not "declining" (per non-alarming copy pattern).
* **Fleet scope contract:** `buildOrgMomentumData` uses a single `FleetDeltas` input — `gainingServiceCount` is derived from `fleet.deltas.values()` and `serviceCount` from `fleet.fleetSize`. This structurally prevents the numerator/denominator universe mismatch that caused "1243 of 1240 services".

### Must Not Break

* **Trend threshold:** 10% comparison between recent and earlier halves of series.
* **Altitude-specific display:** Service = raw delta, Team = per-service normalized, Org = gaining fleet %.
* **Non-alarming language:** "easing" for down trend, never "declining" or "dropping".
* **Pure functions:** No side effects, no database access.
* **Single-input fleet contract:** `buildOrgMomentumData` must accept `FleetDeltas`, never separate `serviceDeltas: number[]` and `totalServiceCount: number` parameters. Separate parameters allow universe mismatch.

---

# MomentumConstants (Immutable Rates)

### **Description** Immutable earning rates and trajectory window configuration for the Momentum system.

**File:** `apps/api/src/domain/momentum/constants.ts`

### Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `CORE_BURST` | 100 | Momentum awarded when a Core is first earned (one-time) |
| `CIRCUIT_DAILY` | 10 | Momentum awarded per nominal Circuit per day |

### Trajectory Window Configuration

| Window | Days | Granularity | Data Points |
|--------|------|-------------|-------------|
| `7d` | 7 | daily | 7 |
| `30d` | 30 | daily | 30 |
| `90d` | 90 | weekly | ~12 |

### Must Not Break

* **Immutability:** Earning rates are constants — changing them changes all historical trajectory views.
* **Configuration consistency:** Trajectory window config must match what MomentumHistoryService and TrajectoryBuilder expect.

---

# Flight Manual Components

### ManualOverlay

- **Role:** Centered HUD overlay container for Flight Manual
- **Location:** `apps/web/src/components/manual/ManualOverlay.tsx`
- **Renders when:** `manual` URL param is present
- **Behavior:** Fixed centered overlay (~80vw x 80vh), backdrop with blur, Esc to close, focus trap. Split pane: left nav (280px) + right content.
- **Owns:** Body scroll lock, keyboard escape handler
- **Children:** ManualNav (left), ManualContent (right)
- **State source:** useFlightManual hook (URL-driven)

### ManualNav

- **Role:** Left pane navigation — search, protocol tree, legend tab
- **Location:** `apps/web/src/components/manual/ManualNav.tsx`
- **Behavior:** Typeahead search with client-side fuzzy matching, accordion tree (Pillar > Rulepack), tab toggle (Protocols / Legend)
- **State source:** useFlightManual (navigation), useProtocols (data + search)

### ManualContent

- **Role:** Right pane content router — dispatches to target-specific renderers
- **Location:** `apps/web/src/components/manual/ManualContent.tsx`
- **Dispatches to:** PillarDefinition, RulepackDefinition, RuleDefinition, LegendDefinition based on ManualTarget.type

### useFlightManual Hook

- **Role:** Manual-specific URL state management
- **Location:** `apps/web/src/hooks/useFlightManual.ts`
- **Contract:** parseManualTarget/buildManualTarget for URL <-> ManualTarget conversion. openManual(target), closeManual(), navigateManual(target), isOpen, currentTarget.
- **URL format:** `?manual=type:id` (e.g., `?manual=rulepack:ci_performance`)

### useProtocols Hook

- **Role:** Protocol data fetching and client-side search
- **Location:** `apps/web/src/hooks/useProtocols.ts`
- **Contract:** Fetches extended /config/rulepacks + /config/pillars, builds protocolTree, provides search(query) and getProtocol(target).
- **Cache:** staleTime: Infinity (config is static)

### ProtocolMetadataResolver (Backend)

- **Role:** Config-layer resolver for Flight Manual evaluation metadata
- **Location:** `apps/api/src/config/protocol-metadata-resolver.ts`
- **Built at:** Bootstrap time (in createEngine or index.ts)
- **Contract:** Combines CheckRegistry + GroundControlConfig to resolve per-rulepack: achievementType, signalSources, badge semantics, editUrl, rule criteria. O(1) lookup at request time.
- **Signal resolution chain:** rule.params.contextKey > rulepack.applicableWhen.contextKey > 'unknown' (derived from YAML config only — no scattered config on check implementations)
- **Vocabulary:** Badge semantics use canonical terms from vocabulary-provider (e.g., "Not Yet Met", "Telemetry Missing") — no freeform strings.

---
