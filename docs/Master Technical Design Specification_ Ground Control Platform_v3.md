# Master Technical Design Specification: Ground Control Platform

> **Foundation Documents:** This specification provides technical implementation details. For immutable core definitions (hierarchy, states, relationships), see the [Ground Control Constitution](./Ground_Control_Constitution.md). For product guidance (UX, guardrails, cultural intent), see the [Product Definition](./Product_Definition_Ground_Control_V4.md).

Always prefer long term right solution over quick wins or shortcuts. 

# 1\. Architectural Principles

1. ***Open-Closed Principle (OCP):***  
   * *The **Engine** (Orchestrator) is immutable. It does not know specific metrics.*  
   * ***Extensions** (Checks, Collectors, Rulepacks) are added as plugins.*  
   * *Rule: You never modify it to add a new check.*  
2. ***Separation of Truth & Meaning (Pure Measurement):***  
   * ***Code (Check):** Measures and returns observed facts (observedValue, displayValue). Checks NEVER decide pass/fail. They only measure.*  
   * ***Config (Rulepack):** Defines Thresholds (what values mean pass/fail) AND Meaning (Implication, Risk, Effort). It is contextual.*  
   * ***Enricher:** Applies thresholds from config to observedValue to determine status (PASS/NOT_MET) or tier (GOLD/SILVER/BRONZE).*  
   * *Benefit: The same check can have different thresholds in different rulepacks. Threshold changes don't require code changes.*  
3. ***Signal Integrity & Universal Reality:***  
   * *We distinguish between **Service Quality** (NOT_MET) and **Signal Quality** (ERROR).*  
   * *Rule: A missing tool/signal required by a rule results in an **ERROR (Telemetry Missing / Blocked)**, not NOT_MET.*  
      *Rule: Collection is Universal. We collect available reality. If a collector fails, the system degrades gracefully (Partial Context); the scan continues.*  
      *Rule: **Not Applicable is not an Error.** If a collector is not configured or not relevant for a given service type, it marks the context as **NOT\_APPLICABLE** (neutral), not ERROR.*  
4. ***Typed Contexts (OCP-Safe):***  
   * *Collectors return strict Types (GitData), not generic Maps.*  
   * *The Registry provides access via string keys, but usage is typed: ctx.get\<GitData\>('git').*
5. ***Domain Logic Isolation:***  
   * *Domain logic (state derivation, scoring, evaluation) must live in the domain/enrichment layer.*  
   * *View layer aggregators consume domain logic, they do not define it.*  
   * *Example: `deriveCoreState()` and `deriveCircuitState()` are pure functions in enrichment layer.*  
   * *Rule: If the logic defines what a concept MEANS (earned, nominal, etc.), it belongs in enrichment, not views.*  
   * *Benefit: Future views can reuse the same logic; testability improves; single source of truth.*
6. ***View-Layer Hydration Contract:***  
   * *Snapshots contain `TruthCheckResult` (truth only). Config fields (`implication`, `remediation`, `impact`, `effort`, `tags`) do not exist on this interface.*  
   * *View aggregators MUST inject `ConfigResolver` as a required constructor dependency.*  
   * *Before accessing config fields, MUST call `configResolver.hydrateCheck(truthCheck)` to produce `ViewCheckResult`.*  
   * *Fallbacks that mask undefined config values are forbidden - they hide architectural bugs.*  
   * *Violation: `check.implication ?? 'default'` where check is `TruthCheckResult`*  
   * *Correct: `const hydrated = configResolver.hydrateCheck(check); hydrated.implication`*  
   * ***Evidence Uniformity:** All truth check results that represent real observations (evaluable or blocked) produce `SignalEvidence` entries. The `raw` field serves dual purpose: for evaluable checks it holds query/sampling provenance; for blocked checks it holds collector error details (`errorKind`, `message`, `missingContextKey`). NEUTRAL checks (NOT_APPLICABLE) produce no evidence. This ensures evidence drawers show complete truth without special-casing.*
7. ***Topic-Based Vocabulary Contract:***  
   * *All domain vocabulary (labels, descriptions, tooltips) is backend-owned. The frontend is a pure renderer — it never hardcodes domain text.*  
   * ***VocabularyProvider** (`view-layer/vocabulary-provider.ts`) is the single source of truth for all display text. It defines two kinds of vocabulary:*  
     * ***Enum vocabulary:** keyed by domain enum values (CheckStatus, UpgradeType, ReadinessState, CircuitState, CoreState, ProtocolState, BadgeType, ImpactLevel, EffortLevel). OCP: adding a new enum value causes a TypeScript compile error until the vocabulary entry is added.*  
     * ***Topic vocabulary:** keyed by coaching topic IDs (`signalCoverage`, `evaluationCoverage`, `closestWins`, `nominalCircuits`). Each topic provides `label`, `description`, and `shortDescription`.*  
   * ***Delivery:** Vocabulary is embedded in API responses where the data appears (self-contained rendering: `coachingLabels` in `MissionMapSummary`, `TeamHuddle`; `label`/`description` in `CoverageInfo`). Also served via `/config/vocabulary` for general-purpose use (sort buttons, filter dropdowns, column headers).*  
   * ***Coverage distinction:** `signalCoverage` (org/team: percentage of services with connected telemetry) and `evaluationCoverage` (service: percentage of rules that could be evaluated) are distinct concepts with separate vocabulary entries. They must never be conflated.*  
   * ***Rules:***  
     * *Frontend components MUST NOT hardcode domain labels, descriptions, or tooltip text.*  
     * *Every API response that contains a domain metric MUST include the vocabulary for that metric.*  
     * *Different concepts that share a name (e.g., "coverage" at org-level vs service-level) MUST have distinct topic IDs.*  

# 2\. The Domain Model Hierarchy

### ***Level 1: The Pillar (Strategy)***

* ***Definition:** A high-level strategic bucket (e.g., "Production Health").*  
* ***Role:** Aggregates Scores, defines "Readiness" (Flight Ready/Pre-flight), and acts as the UI container.*

### ***Level 2: The Rulepack (Tactics)***

* ***Definition:** A specific skill or module (e.g., "The Safety Net").*  
* ***Role:***  
  * *Belongs to **One** Pillar (1:1 ownership via `pillarId`).*  
  * *Configures a list of Checks with thresholds and narrative.*  
  * *Awards a Badge (GATEKEEPER or LADDER).*
  * *May define `applicableWhen` to gate evaluation on context availability (e.g., `applicableWhen: { contextKey: sonic }` - skip if Sonic not configured).*
* ***Badge Types:***
  * ***GATEKEEPER:** Binary pass/fail based on threshold applied to observedValue from check.*
  * ***LADDER:** Tiered achievement (GOLD/SILVER/BRONZE) based on tier thresholds.*
* ***Achievement Type (CORE vs CIRCUIT)** - DERIVED from check windowType:*
  * *Achievement type is **derived** from check's `windowType`, NOT declared in YAML: STATIC → CORE, ROLLING → CIRCUIT.*
  * ***CORE** (Foundation): Permanent achievement. Once earned, stays earned. For static config/hygiene checks.*

  * ***CIRCUIT** (Vitality): A **living, windowed achievement** representing the service’s **current operational posture** (“how is it doing *now*?”). It must be **maintained** and is expected to fluctuate as reality fluctuates.
    * *Truth is **window-based** (time window or event window) and deterministic at scan time; scan cadence must not change outcomes.*
    * *To support coaching without shame, the view layer may derive a **Circuit State** from the achieved badge level for CIRCUIT rulepacks:*
      * *`NOMINAL` — meeting the desired tier*
      * *`DEGRADING` — degraded but not yet broken (typically an intermediate LADDER tier such as Silver/Bronze; “near-miss” heuristics may be added later without changing stored truth)*
      * *`DOWN` — below minimum tier*
    * *Derived Circuit State is **presentation-only**; the stored truth remains `badgeLevel` + `observedValue` + evidence.*
  * ***Validation rules** (fail-fast at startup):*
    * *Check's windowType must be valid: STATIC or ROLLING.*
    * *Derivation is automatic: STATIC window → CORE, ROLLING window → CIRCUIT.*
* ***Design Principle: Capability vs Authority***
  * *Rulepack defines **capability** (what can be achieved: badge type, tiers, thresholds).*
  * *Pillar defines **authority** (what is required: which rulepacks, what baseline for each).*
  * *A rulepack does NOT know what baseline is required for readiness—that's the pillar's decision.*
  * *This enables: optional rulepacks, flexible strictness per pillar, progressive standards.*

* ***Design Principle: Applicability Contract (Loose Coupling)***

  *`applicableWhen.contextKey` implements a **demand-side declaration** pattern:*

  * ***Collectors** register what they provide (e.g., `git`, `gitops`, `datadog`, `ci`, `sonic`).*
  * ***Rulepacks** declare what they need (e.g., `applicableWhen: { contextKey: gitops }`).*
  * ***Neither side knows about the other.** The Orchestrator is the only component that resolves the relationship at scan-time.*

  *Context keys are **dynamic** — they evolve with the product and depend on runtime configuration (e.g., Datadog collector is only registered when API keys are configured). The set of available keys is never hardcoded; it is whatever the registered collectors emit at startup.*

  * ***At scan-time**, the Orchestrator evaluates the gate: if the collector returned `NOT_APPLICABLE` for the service, the rulepack is skipped entirely (all rules become NEUTRAL). If the collector returned data or `COLLECTOR_FAILED`, the rulepack runs (checks handle errors individually).*
  * ***At startup**, the bootstrap validates that referenced context keys match registered collectors. Mismatches produce a warning (not a hard failure) because collector registration is conditional.*

### ***Level 3: The Check (Measurement)***

* ***Definition:** A reusable logic unit (e.g., MaxRestartsCheck).*  
* ***Role:** Receives EvaluationContext, returns CheckResult. Reusable across rulepacks.*

# 3\. High-Level Data Flow

*Code snippet*

```

graph LR
    Input[Trigger Service A] --> A
    subgraph Engine
    A[Collector Phase] -->|Universal Context| B[Evaluation Phase]
    B -->|CheckResults| C[Enrichment Phase]
    C -->|Snapshot| DB[(Database)]
    end
    DB --> D[View Logic]
    D --> UI[Frontend]

```

### ***Phase A: Universal Collection (The Reality)***

* ***Input:** ServiceIdentity (from Backstage).*  
* ***Logic:***  
  * *Engine calls CollectorRegistry.buildContext(service).*  
  * *Registry fires **ALL** registered collectors in parallel (Promise.allSettled).*  
  * ***Provider Registry Pattern:** Collectors like GitCollector use a GitProviderRegistry to select the appropriate provider (GitLab, GitHub) based on repository URL. Providers are registered at startup and resolved at collection time.*  
* ***Output:** EvaluationContext (Universal).*  
  * *If a collector fails (timeout/auth), its key is undefined and the error is logged to \_errors. The scan **continues**.*

### ***Phase B: Evaluation (The Truth)***

* ***Input:** EvaluationContext \+ RulepackParams.*  
* ***Logic:** Run `check.run()` after validating YAML params via `check.schema`*  
* ***Handling Missing Tools:***  
  * *Check asks: ctx.get('sonic').*  
  * *If missing: Returns status: ERROR.*  
  * *Note: A missing context can be **Telemetry Missing** or **Not Applicable**.*  
* *If the collector failed to fetch required data (timeout/auth/error), the check returns `ERROR` with `errorKind: COLLECTOR_FAILED` and `missingContextKey`.*

* *If the context is not relevant for this service (e.g., integration intentionally absent / service type does not support it), the check returns `ERROR` with `errorKind: NOT_APPLICABLE`. Enrichment treats this as **neutral** (excluded from mastery/scoring and produces no “Unblock Telemetry” recommendation).*

### ***Phase C: Enrichment (The Meaning)***

* ***Logic:***  
  * *Map NOT_MET results to implication text.*  
  * *Map ERROR(COLLECTOR\_FAILED) results to `missingSignalText` (Telemetry Missing).*  
  * *ERROR(NOT\_APPLICABLE) is treated as neutral and does not produce telemetry-missing narrative.*  
  * *Attach tags (e.g., "acute-stability").*  
  * ***Compute `blockingStatus`** for each check (single source of truth):*
    * *`EVALUABLE` when status is PASS/NOT_MET/WARN*
    * *`BLOCKED` when ERROR with COLLECTOR_FAILED*
    * *`NEUTRAL` when ERROR with NOT_APPLICABLE*
  * ***Note:** `blocking` is a config property resolved at view time, not stored in truth. It determines whether a rule affects badge evaluation (GATEKEEPER only) but is not part of the snapshot.*
  * ***Compute rule-level `status` (PASS/NOT_MET) — badge-type-specific:***
    * ***GATEKEEPER:** Each rule has its own threshold in YAML. The enricher applies `applyBinaryThreshold()` to produce PASS or NOT_MET. Missing threshold is a config validation bug (fail-fast).*
    * ***LADDER:** The rulepack defines tier thresholds (GOLD/SILVER/BRONZE) at the badge level. The enricher applies these tier thresholds to each rule's numeric `observedValue`. PASS = value reaches at least BRONZE (entry tier — the rulepack's own minimum bar). NOT_MET = value is below all tiers. The per-rule `tier` field (GOLD/SILVER/BRONZE/NOT_MET) is also set on `TruthCheckResult`.*
    * *Both badge types apply the rulepack's own thresholds at the rule level, ensuring `status` has a consistent meaning: "did this rule meet its rulepack's defined bar?"*
  * *Compute Badge Levels and Pillar Mastery.*  
    * ***Mastery rule under partial context:***  
* *If a Pillar’s required rulepack baseline cannot be evaluated due to `ERROR` with `errorKind: COLLECTOR_FAILED`, the Pillar Mastery State becomes **Telemetry Missing** (blocked).*

* ***Score remains neutral** for blocked items (no penalty).*

* *If the missing context is `NOT_APPLICABLE`, the rulepack is excluded from mastery/scoring for that service (neutral), and does not block mastery unless explicitly marked required for that service type.*

* ***Output:** EvaluationSnapshot (Immutable JSON).*

### ***Phase D: View Generation (The Value)***

* ***Component:** UpgradeRecommender (Stateless).*  
* ***Logic:***  
  1. ***Unblock Telemetry:** Only ERROR(COLLECTOR\_FAILED) that blocks mastery-required or high-weight rulepacks produces high-priority Unblock items.*  
  2. ***Optimize:** NOT_MET results \-\> Sorted by Impact / Effort.*

# 4\. Detailed Implementation Specifications

### ***A. Configuration Schema (The "Game Design")***

*(No changes from your input. Kept strictly as Reference.)*

***1\. config/rulepacks/safety\_net.yaml***

*YAML*

```

# Optional: only evaluate if datadog context is available
applicableWhen:
  contextKey: datadog

rules:
  - ruleId: "restarts_check"
    checkId: "k8s_restarts"
    params: { max: 5 }
    implication: "Crash loops destabilize the cluster."
    effort: "M"
    impact: "HIGH"
    missingSignalText: "Connect Kubernetes telemetry."

```

### ***A.1 Config Validation Rules***

All configuration is validated at startup (fail-fast). Invalid config prevents the server from starting.

Validation is layered: **schema-level** (Zod parse) catches structural errors, **semantic-level** (loader.ts) catches cross-field errors.

**Schema-level enforcement (Zod discriminated union on `badge.type`):**

`BadgeConfigSchema` is a discriminated union — invalid badge-type/baseline/threshold combos are rejected at parse time, not runtime:

* **GATEKEEPER badge:** `baseline` must be `PASS` or `NONE`. No `thresholds` field.
* **LADDER badge:** `baseline` must be `NONE`, `BRONZE`, `SILVER`, or `GOLD`. `thresholds` is **required** with `{ gold: number, silver: number, bronze: number }`.

This eliminates an entire class of config errors (e.g., GATEKEEPER + GOLD baseline, LADDER without thresholds) structurally — they cannot even be parsed.

**Semantic validation (loader.ts — cross-field checks the schema cannot express):**

* **LADDER:** Rules must NOT have `params.thresholds` (thresholds belong at badge level only).
* **GATEKEEPER:** All rules must have a `threshold` defined (required for PASS/NOT_MET computation).
* **GATEKEEPER:** `blocking` property is only valid on GATEKEEPER badge types (not LADDER).
* **Empty gate prevention:** GATEKEEPER with `baseline != NONE` must have at least one blocking rule.

**Uniqueness validation:**
* Rulepack IDs must be unique across all rulepacks.
* Rule IDs must be unique globally (across all rulepacks).

**Reference integrity validation:**
* Each rulepack's `pillarId` must reference an existing pillar.

### ***B. Core Interfaces (TypeScript)***

***1\. EvaluationContext (Universal & Resilient)***

*TypeScript*

```

interface EvaluationContext {
  service: ServiceIdentity;
  timestamp: Date;
  
  // The Data Registry (Strictly Typed Access)
  // Usage: ctx.get<GitData>('git')
  ctx: ContextRegistry;

  // Resilience Metadata
  _errors: Record<string, { kind: 'COLLECTOR_FAILED' | 'NOT_APPLICABLE'; message: string }>;
/* e.g.
{ "sonic": { kind:"COLLECTOR_FAILED", message:"Timeout" },
  "datadog": { kind:"NOT_APPLICABLE", message:"Integration not configured" } }
*/

}

```

***2\. DefinedCheck (Standardized - Truth vs Meaning)***

*TypeScript*

```

// Window type determines achievement type derivation
type WindowType = 'STATIC' | 'ROLLING';

// Check returns MEASUREMENT only - NO status/judgment
// Status is computed by enricher applying threshold from YAML
interface MeasurementResult {
  observedValue: unknown;     // Raw measurement data for threshold evaluation
  displayValue: string;       // Human-readable summary for UI (OCP-compliant)
  error?: {                   // Only when measurement cannot be completed
    kind: 'COLLECTOR_FAILED' | 'NOT_APPLICABLE';
    message: string;
    missingContextKey?: string;
  };
  evidence?: CheckEvidence;   // Verification links and debug context
}

interface DefinedCheck<P> {
  id: string;                 // checkId referenced from YAML
  windowType: WindowType;     // STATIC → CORE, ROLLING → CIRCUIT (derived)
  schema: ZodType<P>;         // Runtime validation of YAML params
  run(context: EvaluationContext, params: P): MeasurementResult | Promise<MeasurementResult>;
}

```

***3\. Display Formatting Principle (OCP for UI)***

Each check is responsible for providing its own `displayValue` - a human-readable summary of what was observed. This follows the **Single Responsibility Principle** and ensures **Open-Closed Principle compliance** in the frontend:

* **Backend (Check):** Owns the formatting logic. Each check knows best how to summarize its `observedValue` for humans.
* **Frontend (Transformer):** Uses `displayValue` directly. No switch/case logic per check type.

**Why this matters:**
* Adding a new check does **not** require modifying the frontend transformer.
* The check author (who understands the domain) writes the display logic, not the UI developer.
* Display consistency is enforced at the source, not scattered across layers.

**Display Value Guidelines:**

| Badge Type | Good displayValue | Bad displayValue |
|------------|-------------------|------------------|
| LADDER (speed) | `P90: 6m 30s (Gold < 8m)` | `tier=GOLD, percentile=0.9` |
| LADDER (stability) | `0 crashes in 24h (Gold)` | `tier=GOLD, crashes=0` |
| GATEKEEPER | `README.md present` | `exists=true` |

**Key principles:**
1. **Human-readable values:** Use `6m 30s` not `390` seconds
2. **Include context:** Show thresholds when relevant (e.g., "need < 8m for Gold")
3. **Counts with totals:** Show `18 of 20` not just `18`
4. **Percentages with context:** Show `90%` with what it means

**Example:**
```typescript
// DeploymentSpeedCheck returns:
{
  status: 'PASS',
  observedValue: { tier: 'GOLD', p90DurationSeconds: 390, ... },
  displayValue: 'P90: 6m 30s (Gold < 8m)',  // ← Human-readable with context
  message: 'P90 deployment duration: 6.5 minutes. Excellent! Under 8m threshold.'
}

// ChangeStabilityCheck returns:
{
  status: 'NOT_MET',
  observedValue: { cleanRate: 0.45, cleanMerges: 9, totalMerges: 20, ... },
  displayValue: '9 of 20 clean (45%, need 90%)',  // ← Counts + threshold
  message: 'CI stability below threshold. 11 of 20 merges had issues.'
}
```

### ***C. Scoring & View Logic***

#### 4.C EvaluationSnapshot (DB Entity)

**Definition:** The immutable record of one evaluation run for a service.  
**Goals:**

* **Truth preservation** (raw results \+ observed values)  
* **Fairness** (explicit telemetry/applicability semantics)  
* **Explainability** (why score/mastery are what they are)  
* **Stable evolution** (versioned policy)

**CRITICAL: Truth vs. Configuration Separation**

* **Snapshots store TRUTH only.** The EvaluationSnapshot contains only what was observed at scan time.
* **Configuration is looked up at view-time.** Fields like `implication`, `remediation`, `impact`, `effort`, `tags`, `name`, `missingSignalText` are resolved from YAML when rendering views.
* **Why?** This ensures engineers always see the current best guidance, even for historical scans. If YAML is improved, all scans benefit immediately.
* **How?** `TruthCheckResult` stores identity keys (ruleId, checkId, pillarId, rulepackId) sufficient for O(1) config lookup via `ConfigResolver` at view-time.

### **TypeScript**

```ts
type CheckStatus = 'PASS' | 'NOT_MET' | 'WARN' | 'ERROR';
type ErrorKind = 'COLLECTOR_FAILED' | 'NOT_APPLICABLE';
type BlockingStatus = 'EVALUABLE' | 'BLOCKED' | 'NEUTRAL';

type BadgeType = 'GATEKEEPER' | 'LADDER';
type BadgeLevel = 'PASS' | 'NOT_MET' | 'BRONZE' | 'SILVER' | 'GOLD' | 'NOT_EVALUATED';

type ReadinessStatus = 'FLIGHT_READY' | 'PRE_FLIGHT' | 'TELEMETRY_MISSING';

// STORED in DB snapshot (truth only)
interface TruthCheckResult {
  // Identity (required for config lookup)
  ruleId: string;
  checkId: string;
  pillarId: string;
  rulepackId: string;

  // Truth (what was observed)
  status: CheckStatus;
  observedValue: unknown;
  displayValue: string;
  message?: string;

  // Per-rule tier for LADDER badges (Step 3: Meaning)
  // Only set for LADDER badge rules where threshold produced a tier.
  // GOLD/SILVER/BRONZE: Value met the corresponding tier threshold
  // NOT_MET: Value exceeded all tier thresholds (below minimum bar)
  // Step 4 (Aggregation) uses MIN across all rule tiers to determine badge level.
  tier?: 'GOLD' | 'SILVER' | 'BRONZE' | 'NOT_MET';

  // Blocking semantics (derived from truth)
  blockingStatus: BlockingStatus;
  errorKind?: ErrorKind;
  missingContextKey?: string;

  // Collector identification
  collector?: string;

  // Evidence (verification data from check)
  evidence?: CheckEvidence;
}

// RETURNED to UI (truth + config hydrated at view-time)
interface ViewCheckResult extends TruthCheckResult {
  // Config (looked up from YAML at view-time via ConfigResolver)
  name?: string;           // Human-readable rule name
  implication?: string;    // Why this matters
  remediation?: string;    // How to fix
  impact?: ImpactLevel;    // HIGH/MEDIUM/LOW
  effort?: EffortLevel;    // SMALL/MODERATE/SIGNIFICANT
  tags?: string[];
  missingSignalText?: string;
  // Note: blocking is NOT a config field — it is truth (affectsBadge on TruthCheckResult)
}

interface RulepackBadgeResult {
  rulepackId: string;
  pillarId: string;

  badgeType: BadgeType;

  // The achieved badge level in this run
  badgeLevel: BadgeLevel;

  // Whether this rulepack is required for pillar readiness
  // True if rulepack is in pillar.readiness.requiredRulepacks, false if optional
  // UI should only show "Below baseline" for required rulepacks
  isRequired: boolean;

  // Baseline requirement configured for rulepack (from pillar readiness config)
  // Only meaningful when isRequired === true
  baselineRequired?: BadgeLevel;
  baselineMet: boolean;

  // Explainability
  rationale?: string; // e.g. "Silver requires restarts<=2; observed=5"
}

interface RulepackResult {
  rulepackId: string;
  pillarId: string;

  // Final badge outcome for this rulepack
  badge: RulepackBadgeResult;

  // Score contribution - computed by ScoringPolicyV1, NOT by Enricher
  // NOTE: This field is optional in Enricher output; ScoringPolicyV1 computes it
  // from badge facts. The Enricher produces badge facts only.
  rulepackScore?: number;

  // Check results (truth only - config hydrated at view-time)
  checks: TruthCheckResult[];
}

interface EvaluationSnapshot {
  serviceId: string;
  timestamp: string;

  // Versioning for trust + future changes
  scoringPolicyVersion: string; // e.g. "v1"

  // Coverage / fairness
  telemetryCoverage: {
    evaluatedRuleCount: number;
    totalRuleCount: number;
    blockedRuleCount: number;     // ERROR with COLLECTOR_FAILED
    notApplicableRuleCount: number; // ERROR with NOT_APPLICABLE
  };

  // Signal coverage (rulepack-level visibility)
  signalCoveragePct: number;     // 0-100: evaluable required / total required
  blockedRulepackIds: string[];  // which required rulepacks are BLOCKED

  // Pillar-level aggregation
  pillarScores: Record<string, number>;              // 0-100 per pillar
  pillarReadiness: Record<string, ReadinessStatus>; // FLIGHT_READY / PRE_FLIGHT / TELEMETRY_MISSING
  pillarReadinessReasons?: Record<string, string>;     // explainability

  // Rulepack-level details
  rulepackResults: RulepackResult[];

  // Resilience metadata (debugging/tracing)
  contextErrors: Record<string, { kind: ErrorKind; message: string }>; // from EvaluationContext._errors
}
```

**Notes:**

* `ruleId` is mandatory. It is the stable key for UI and recommendations.  
* `NOT_APPLICABLE` never harms score and never creates “Unblock Telemetry.”  
* `COLLECTOR_FAILED` is score-neutral but can block mastery.

#### 4.C.1 Trajectory Window Configuration

Trajectory windows control Momentum history visualization. They are independent from evaluation windows (which determine badge facts at scan time).

**Vitality vs. Trajectory:** Momentum are a **trajectory/compound-interest view** over sustained behavior. They must never be used as a proxy for (or substitute) **CIRCUIT truth**, which is evaluated directly via windowed checks.

**Configuration:**

```typescript
const TRAJECTORY_CONFIG = {
  '7d':  { days: 7,  granularity: 'daily',  dataPoints: 7  },
  '30d': { days: 30, granularity: 'daily',  dataPoints: 30 },
  '90d': { days: 90, granularity: 'weekly', dataPoints: 12 },
} as const;
```

**API Parameter:**

The `timeWindow` query parameter controls trajectory window selection:

* Endpoint: `GET /mission-map/teams?timeWindow=30d`
* Valid values: `7d`, `30d`, `90d`
* Default: `30d`

**Data Exclusion:**

Today's data is always excluded from trajectory calculations because:

1. The day is incomplete (partial data would skew trends)
2. Evaluations may still be running
3. Consistency: "last 7 days" means 7 complete days

**Granularity (Hybrid Daily Rate Model):**

Momentum are computed at view-time from badge states:

* **Earning Model:**
  * Core burst: +100 when a Core is first earned (one-time)
  * Circuit daily: +10/day per nominal Circuit

* **Daily (7d, 30d):** Each data point = daily momentum computed from `circuitCount × 10`
* **Weekly (90d):** Each data point = sum of daily momentum for that week
  * Provides ~12 data points for readable sparklines

The series values represent **daily earnings**, not cumulative totals. This ensures scan-cadence independence: a service scanned hourly earns the same daily momentum as one scanned once per day.

**Trend Calculation (Simple Direction):**

Trajectory uses a simple direction indicator comparing recent vs. earlier averages:

* **Trend:**
  * `up` — recent average > 10% higher than earlier average
  * `down` — recent average < 90% of earlier average
  * `flat` — within 10% threshold

**Scan-Cadence Independence:** Momentum are computed from calendar days and badge states, not scan frequency. This ensures the same operational state produces the same trajectory regardless of how often evaluations run.

**Altitude-Aware Display Metrics:**

FlightHoursData includes optional altitude-specific fields computed at aggregation time:

* `deltaPerService` (team): `delta / serviceCount` — normalized for fair comparison across team sizes
* `gainingFleetPct` (org): percentage of services with `delta > 0` — participation/adoption signal
* `gainingServiceCount` (org): count of services gaining momentum (delta > 0)
* `serviceCount` (team/org): number of services in the calculation

These fields are computed by altitude-aware builder functions (`buildTeamMomentumData`, `buildOrgMomentumData`) and passed to the UI for rendering. The backend provides pre-computed display labels (`headlineLabel`, `subtextLabel`, `tooltipText`, `directionLabel`) — the frontend `MomentumDisplay` component is a pure renderer with no domain logic.

**Fleet Scope Contract:** All momentum builder functions receive pre-scoped data. The fleet boundary is defined by `ServiceDirectory` and passed through the view layer. Momentum queries never define their own scope. `buildOrgMomentumData` accepts a `FleetDeltas` object (`{ deltas: ReadonlyMap<string, number>, fleetSize: number }`) — a single input where numerator (gaining count from `deltas`) and denominator (`fleetSize`) come from the same universe. Separate denominator parameters are forbidden because they allow universe mismatch (e.g., "1243 of 1240 services").

**Concept separation:** Momentum tooltips explain only momentum (gaining count over timeframe). Fleet telemetry coverage is a separate concept computed by `computeFleetCoveragePct()` in `domain/readiness.ts` and displayed via the Coverage stat card and fleet connection indicator. The two must never be conflated in tooltip text.

#### 4.D ScoringPolicyV1 (Fair, Stable, Versioned)

**Goal:** translate results into **pillarScores**, **rulepackScore**, and **mastery states** in a way that is:

* fair (no penalties for missing telemetry),  
* consistent (policy versioned),  
* aligned with intrinsic motivation (rewards progress, not shame).

> **CRITICAL BOUNDARY:** ScoringPolicyV1 owns ALL score computation.
> The Enricher produces **badge facts only** (badge level, achievement type, baselineMet).
> ScoringPolicyV1 takes these badge facts and computes scores.
> This separation ensures scoring logic is versioned and isolated from enrichment logic.

### **Scoring principles (v1)**

1. **ERROR with `COLLECTOR_FAILED` is score-neutral.**  
   It does not reduce scores. It can block mastery by producing `TELEMETRY_MISSING`.  
2. **ERROR with `NOT_APPLICABLE` is excluded.**  
   It does not affect score or mastery and produces no unblock recommendation.  
3. **WARN is partial impact.**  
   It reduces points less than NOT_MET and never blocks mastery unless a rule is explicitly marked “must” in a future extension.  
4. **Rulepack score is computed from badge \+ check statuses.**  
   Badge type determines scoring method (Gatekeeper/Ladder).  
5. **Pillar score is a weighted aggregate of rulepack scores.**  
   Uses rulepack `weight` (from YAML) normalized per pillar.

### **Minimal interface**

```ts
interface ScoringOutput {
  scoringPolicyVersion: 'v1';
  rulepackScores: Record<string, number>; // rulepackId -> 0-100
  pillarScores: Record<string, number>;   // pillarId -> 0-100
  pillarReadiness: Record<string, ReadinessStatus>;
  pillarReadinessReasons: Record<string, string>;
  telemetryCoverage: EvaluationSnapshot['telemetryCoverage'];
  // Signal coverage
  signalCoveragePct: number;     // 0-100: evaluable required rulepacks / total required
  blockedRulepackIds: string[];  // which required rulepacks are BLOCKED
}

function applyScoringPolicyV1(snapshot: Omit<EvaluationSnapshot, 'scoringPolicyVersion' | 'pillarScores' | 'pillarReadiness' | 'pillarReadinessReasons' | 'telemetryCoverage'>): ScoringOutput;
```

### **Rulepack scoring (v1 defaults)**

These defaults are intentionally simple and can evolve later.

* **Gatekeeper**  
  * If any required rule is `NOT_MET` → rulepackScore \= 0  
  * If any required rule is `ERROR(COLLECTOR_FAILED)` → rulepackScore \= neutral default (e.g. 0 but flagged as blocked) and does not impact pillar score (excluded from aggregation). When a rulepack is excluded due to telemetry blocking, it must be explicitly marked `excludedFromScoring: true` (or equivalent) so UI/aggregators never interpret its score as failure.  
  * If all required rules are `PASS` (and WARN allowed) → rulepackScore \= 100  
* **Ladder**  
  * Gold \= 100, Silver \= 80, Bronze \= 60, Below baseline \= 0  
  * If badge evaluation is blocked by `ERROR(COLLECTOR_FAILED)` → excluded from pillar aggregation and marks mastery blocked if required

### **Pillar readiness (explicit rule)**

Pillar readiness is evaluated from `pillars.yaml readiness.requiredRulepacks`:

* If any required rulepack is **blocked by COLLECTOR\_FAILED** → `TELEMETRY_MISSING`  
* Else if all required rulepacks meet `minBaseline` → `FLIGHT_READY`  
* Else → `PRE_FLIGHT`

This produces clean, explainable states.

### **Baseline Resolution Flow**

The baseline for each rulepack is declared on the rulepack's `badge.baseline` field in the rulepack YAML files, NOT in `pillars.yaml`:

```
┌─────────────────────────────────────────────────────────────────┐
│  rulepacks/ci_performance.yaml                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ badge:                                                    │  │
│  │   type: LADDER                                            │  │
│  │   baseline: BRONZE  ← Declared on rulepack                │  │
│  │   thresholds:                                             │  │
│  │     gold: 5                                               │  │
│  │     silver: 10                                            │  │
│  │     bronze: 15                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  rulepacks/safety_net.yaml                                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ badge:                                                    │  │
│  │   type: GATEKEEPER                                        │  │
│  │   baseline: PASS  ← Declared on rulepack                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Valid baseline values (enforced by schema — Zod discriminated union on `badge.type`):**
* **GATEKEEPER:** `PASS` (required for readiness) or `NONE` (informational only). No `thresholds`.
* **LADDER:** `BRONZE`, `SILVER`, `GOLD` (required at tier) or `NONE` (informational only). `thresholds` required.

**Key behaviors:**
* Baseline is declared per rulepack, enabling flexible standards per capability.
* `baselineMet` in `RulepackBadgeResult` reflects whether the achieved badge level meets the rulepack's declared baseline.
* Rulepacks with `baseline: NONE` are informational only and do not block readiness.

#### 4.E UpgradeRecommender (Curated Recommendations, Not Managed Quests)

**Goal:** Produce a **small, curated list** of the highest-leverage next actions for the service/team UX.  
No lifecycle management. No assignments. No time estimates beyond effort SMALL/MODERATE/SIGNIFICANT.

> **Dual-Path Architecture:** Two recommendation engines coexist, each serving a different purpose:
> - **`recommendUpgrades()`** (`view-layer/upgrade-recommender.ts`) — Produces a flat, category-grouped list of all applicable upgrades. Used by team/org endpoints for the full recommendation view.
> - **`selectNextBestUpgrade()`** (`view-layer/next-best-upgrade/`) — Leverage-scored selection of a single primary upgrade with 2-3 alternatives. Used by the Flight Deck NBU endpoint for focused coaching.
>
> Both share the same `UpgradeRecommendation` output type and recommendation key format. The NBU selector adds scoring, explainability, and multiplier logic on top.

### **Output contract (V2 - backward compatible)**

```ts
type UpgradeType = 'UNBLOCK_TELEMETRY' | 'REACH_BASELINE' | 'PROTECT_VITALITY' | 'REDUCE_FRICTION' | 'OPTIMIZE';

interface UpgradeRecommendation {
  type: UpgradeType;
  recommendationKey: string; // Unique key for API routing and drawer navigation
  title: string;
  action: string;
  priority: number; // higher first

  // Links for explainability and navigation
  pillarId?: string;
  rulepackId?: string;
  ruleId?: string;
  checkId?: string;

  // Context for trust
  evidence?: string; // e.g. "observed=12m, target<=8m"
  impact?: 'LOW' | 'MEDIUM' | 'HIGH';
  effort?: 'SMALL' | 'MODERATE' | 'SIGNIFICANT';
  tags?: string[];
}
```

### **Recommendation Key Format Contract**

The `recommendationKey` uniquely identifies a recommendation for API routing and drawer navigation.
All aggregators **must** use `buildRecommendationKey(type, identifiers)` to ensure consistent format.

| Type | Format | Example |
|------|--------|---------|
| UNBLOCK_TELEMETRY | `telemetry:<missingContextKey>` | `telemetry:ci:gitlab` |
| REACH_BASELINE | `reach_baseline:<rulepackId>` | `reach_baseline:safety_net` |
| PROTECT_VITALITY | `protect_vitality:<rulepackId>` | `protect_vitality:ci_stability` |
| REDUCE_FRICTION | `reduce_friction:<rulepackId>:<ruleId>` | `reduce_friction:safety_net:probes_configured` |
| OPTIMIZE | `optimize:<rulepackId>:<ruleId>` | `optimize:safety_net:hpa_configured` |

**Design principles:**
* **DRY:** Single builder function (`buildRecommendationKey`) for all key generation
* **OCP:** New types require only a new case in the switch
* **Parser compatibility:** Key format must match `parseRecommendationKey` in readiness-aggregator

### **UNBLOCK_TELEMETRY Title Convention**

For `UNBLOCK_TELEMETRY` recommendations, the title **must** follow the signal-centric pattern:

```
title: "Connect telemetry: {missingContextKey}"
```

**Why signal-centric, not rulepack-centric:**
* The `recommendationKey` is `telemetry:{missingContextKey}` — identity is the signal
* One missing signal (e.g., `git`) can block multiple rulepacks (e.g., "AI Development Readiness" + "Sonic Onboarding")
* The action to fix is the same regardless of which rulepack you clicked from
* The brief API drawer shows all unblocked rulepacks in `unblocks.rulepacks[]`

This ensures title consistency across all views (Mission Map, Team Console, Flight Deck) and with the drawer content.

**Use `buildTelemetryTitle(missingContextKey)` for all UNBLOCK_TELEMETRY title generation.**

### **Aggregation Key Contract**

When aggregating issues across services (in `aggregateIssuesAcrossServices`), the internal aggregation key determines deduplication. The aggregation key **must** align with the recommendation's identity:

| Type | Identity | Aggregation Key |
|------|----------|-----------------|
| UNBLOCK_TELEMETRY | `missingContextKey` | `buildTelemetryKey(missingContextKey)` |
| REACH_BASELINE | `rulepackId` | `${rulepackId}:baseline` |
| PROTECT_VITALITY | `rulepackId` | `${rulepackId}:circuit` |
| REDUCE_FRICTION | `rulepackId:ruleId` | `${rulepackId}:${ruleId}:qw` |
| OPTIMIZE | `rulepackId:ruleId` | `${rulepackId}:${ruleId}:opt` |

**Why UNBLOCK_TELEMETRY aggregation is special:**
* A single missing signal (e.g., `ci`) can block **multiple rulepacks** (e.g., "Need for Speed" + "Reliable CI")
* Incorrect aggregation by `rulepackId:missingContextKey` produces duplicate recommendations
* Duplicates crowd out other recommendation types (REACH_BASELINE, etc.)
* **Use `buildTelemetryKey(missingContextKey)` for telemetry aggregation**

### **V3: Next Best Upgrade Selection Algorithm**

V3 introduces leverage-based scoring with a primary + alternatives output structure.

#### **V3 Output Contract**

```ts
interface NextBestUpgradeResult {
  primary: CandidateUpgrade | null;  // Highest-leverage upgrade
  alternatives: CandidateUpgrade[];  // 2-3 alternatives
  computedAtTs: string;              // ISO timestamp
}

interface CandidateUpgrade extends UpgradeRecommendation {
  score: number;              // Computed leverage score
  isClosestWin: boolean;      // Only remaining blocker for pillar
  completesGoNoGo: boolean;   // Completes Go/No-Go → Flight Ready
  moves: UpgradeMove;         // Move metadata
  explainWhyChosen: string[]; // Human-readable explanations
  confidence: number;         // 0.3 - 1.0
  spotlight?: {               // Present when candidate matches active spotlight
    id: string;
    title: string;
    kind: 'org_focus';
  };
}

interface UpgradeMove {
  kind: 'telemetry' | 'baseline' | 'vitality' | 'friction' | 'optimize';
  description: string;
  delta?: number; // 1 if completes final baseline item
}
```

#### **V3 Leverage Scoring Formula**

```
leverage = (impactWeight / effortWeight) * confidenceWeight * multipliers
```

**Weights:**
| Impact | Weight | Effort | Weight |
|--------|--------|--------|--------|
| HIGH   | 3      | SMALL       | 1.0    |
| MEDIUM | 2      | MODERATE    | 1.6    |
| LOW    | 1      | SIGNIFICANT | 2.4    |

**Confidence:** `max(0.3, evaluatedRuleCount / totalRuleCount)`

**Multipliers (applied sequentially):**
| Condition | Multiplier | Description |
|-----------|------------|-------------|
| Vitality damper (any Circuit DEGRADING or DOWN) | 0.1x | Applied to ALL non-PROTECT_VITALITY candidates. Ensures vitality always wins. |
| Telemetry blocking baseline (when TELEMETRY_MISSING) | 10x | Only for UNBLOCK_TELEMETRY candidates that block a required baseline |
| Closest win | 2.5x | Only remaining blocker for pillar |
| Completes Go/No-Go | 2.0x | Makes pillar Flight Ready |
| Vitality protection boost | 2.2x | Applied to PROTECT_VITALITY candidates when any Circuit needs protection |
| Focus mode (varies) | 0.8x - 2.2x | Based on mode and category |
| Spotlight (when active and matched) | 1.0x - 2.0x | Org-level focus overlay |

> **Note:** The 0.1x vitality damper uses the same logic as vitality candidate generation (`deriveCircuitState`). A Circuit within the grace period (NOMINAL) does not trigger the damper.

#### **V3 Tie-Breaker Order**

When scores are within 0.01, break ties by:
1. `completesGoNoGo` (true first)
2. `isClosestWin` (true first)
3. Impact (HIGH > MEDIUM > LOW)
4. Effort (S < M < L)
5. Confidence (higher first)
6. Lexicographic: `(categoryPriority, pillarId, rulepackId, ruleId)`

#### **V3 Candidate Sources**

1. **Telemetry candidates**: `COLLECTOR_FAILED` errors blocking required baselines
2. **Baseline candidates**: Required rulepacks with `baselineMet=false` (excludes CIRCUIT achievement types -- handled by vitality)
3. **Vitality candidates**: DEGRADING or DOWN Circuits (see Vitality Candidate Logic below)
4. **Friction candidates**: High-impact/low-effort items NOT blocking baseline
5. **Optimize candidates**: Remaining NOT_MET/WARN checks

> **CIRCUIT Exclusion:** CIRCUIT achievement types are excluded from baseline candidates. Their issues are handled exclusively by vitality candidates (PROTECT_VITALITY). CIRCUITs within the grace period (NOMINAL but recently failing) may appear as REDUCE_FRICTION or OPTIMIZE candidates.

#### **V3 Vitality Candidate Logic (Updated)**

For each rulepack with `achievementType === 'CIRCUIT'`:

```typescript
// 1. Get per-rulepack thresholds (or global defaults)
const thresholds = getVitalityThresholdsForRulepack(rulepackConfig);

// 2. Derive CircuitState from stored truth (view-time meaning)
const circuitState = deriveCircuitState(lastPassedAt, thresholds);

// 3. Generate PROTECT_VITALITY candidate if DEGRADING or DOWN
if (circuitState === 'DEGRADING' || circuitState === 'DOWN') {
  candidates.push({ 
    type: 'PROTECT_VITALITY',
    moveDescription: `Protects vitality: Circuit is ${circuitState}`,
    evidence: circuitState === 'DEGRADING'
      ? `Circuit is degrading - ${hours}h since last pass, will be down in ${remaining}h`
      : `Circuit is down - ${hours}h since last pass`,
    // ... impact, effort, action from breaking check
  });
}
```

**Key Properties:**
- **Truth vs Meaning**: Calls `deriveCircuitState()` (pure function from enrichment layer)
- **View-Time Computation**: Same pattern as `FlightDeckAggregator.buildCircuits()`
- **Per-Rulepack Thresholds**: Different Circuits have different urgency profiles
- **Scan-Cadence Independent**: Time-based (hours since `lastPassedAt`), not scan-count based

**Design Principle:** "Keeping the baby alive" - operational health (Circuits) takes priority over building new foundations (baseline completion). A DEGRADING Circuit at 13h old takes priority over a CORE baseline gap.

### **Recommendation rules (v1)**

1. **Unblock Telemetry only when it matters**  
   Generate UNBLOCK only if:  
* the missing context is `COLLECTOR_FAILED` **and**  
* it blocks either:  
  * a pillar mastery-required rulepack, or  
  * a high-weight rulepack (configurable threshold, e.g. weight \>= 80\)
* **Volume threshold (org/team level):** At Mission Map and Squad Console, UNBLOCK_TELEMETRY recommendations are only surfaced when >= 30% of the evaluated fleet is affected (or >= 3 absolute services at team level). This prevents isolated noise from cluttering coaching views.
* **Per-service (Flight Deck):** NBU always shows the honest truth. No volume threshold.

Never generate UNBLOCK for `NOT_APPLICABLE`.

2. **Reach Baseline is the default "next step"**  
   If a rulepack baseline is not met and evaluation is not blocked:  
* create one `REACH_BASELINE` recommendation per rulepack  
* explain the minimal upgrade needed to meet baseline (rationale)  
3. **Optimize after baseline**  
   Only after baseline items are generated (or if baseline is already met), include OPTIMIZE items:  
* derive priority from Impact/Effort (+ optional weight)  
* cap results (e.g. top 5\) to remain curated

### **V3 Implementation**

```ts
function selectNextBestUpgrade(
  snapshot: EvaluationSnapshot,
  readinessConfig: ReadinessConfig,
  options: SelectOptions = {},
): NextBestUpgradeResult {
  // 1. Generate candidates from all sources
  const rawCandidates = generateCandidates(snapshot, readinessConfig);
  
  if (rawCandidates.length === 0) {
    return { primary: null, alternatives: [], computedAtTs: new Date().toISOString() };
  }
  
  // 2. Score candidates using leverage formula
  const scoredCandidates = scoreCandidates(rawCandidates, snapshot, options);
  
  // 3. Sort with deterministic tie-breakers
  const sortedCandidates = sortWithTieBreakers(scoredCandidates);
  
  // 4. Build explainability output
  const candidateUpgrades = buildCandidateUpgrades(sortedCandidates, snapshot);
  
  // 5. Return primary + alternatives
  return {
    primary: candidateUpgrades[0] ?? null,
    alternatives: candidateUpgrades.slice(1, 4),
    computedAtTs: new Date().toISOString(),
  };
}
```

***5\. Directory Structure***

*Plaintext*

```

/src
  /config
    pillars.yaml
    rulepacks.yaml
    spotlights.yaml
    focus-modes.yaml
    spotlight-schemas.ts
    focus-mode-schemas.ts
  /domain
    /entities
    /interfaces
    /momentum         # Momentum subsystem
      calculator.ts
      history-service.ts
      trajectory.ts
      constants.ts
  /engine
    /core             # Orchestrator, Enricher, ScoringPolicyV1
    /collectors
      /git
        git-collector.ts
        types.ts
        /registry           # GitProviderRegistry
        /providers
          /gitlab           # GitLabProvider, GitLabClient
          /github           # (future) GitHubProvider
      /ci               # CiCollector
      /datadog          # DatadogCollector, DatadogClient
      /gitops           # GitOpsCollector, K8sManifestParser
      /sonic            # SonicCollector
    /checks
      /common           # FileExistenceCheck, GenericThresholdCheck, etc.
      /k8s              # K8sChecks
      /ci               # ChangeStabilityCheck, DeploymentSpeedCheck
      /runtime          # RuntimeStabilityCheck
    /contexts           # ContextRegistry, Universal Builder
  /enrichment
    core-evaluator.ts
    vitality-evaluator.ts
    vitality-history-service.ts
  /view-layer
    readiness-aggregator.ts       # Org-level (Mission Map)
    flight-deck-aggregator.ts     # Service-level (Flight Deck)
    team-console-aggregator.ts    # Team-level (Squad Console)
    upgrade-recommender.ts        # Flat upgrade list
    /next-best-upgrade            # Leverage-scored NBU
    vocabulary-provider.ts
    spotlight-decorator.ts
    config-resolver.ts
  /db
    /queries
      scan-query-service.ts
      momentum-query-service.ts
  /queue
    scheduler.ts
  /api
    /controllers

apps/web/src/components/
  manual/
    ManualOverlay.tsx       # Centered HUD overlay container
    ManualNav.tsx          # Left pane: search, protocol tree, legend tab
    ManualContent.tsx      # Right pane content router
    renderers/
      PillarDefinition.tsx  # Pillar ambition + rulepack list
      RulepackDefinition.tsx # Core view: Ambition/Standard/Evaluation/Handshake
      RuleDefinition.tsx    # Single rule with parent context
      LegendDefinition.tsx  # Vocabulary reference

```

# 6\. Implementation Guide

### Phase 1: Foundation

* *Create ContextRegistry.*  
* *Implement CollectorRegistry with Promise.allSettled logic (Universal Collection).*

### Phase 2: The Logic Library

* *Implement GitCollector using the Adapter Pattern (detect GitHub vs GitLab).*  
* *Implement FileExistenceCheck (Generic) using ctx.get('git').*

### Phase 3: The Engine

* *Implement Orchestrator to load YAMLs.*  
* *Ensure ruleId is used as the key for results in the snapshot.*

### Phase 4: Enrichment & View

* *Merge results with YAML narrative.*  
* *Implement UpgradeRecommender handling ERROR as "Unblock" priority.*

# 7\. Check Authoring Model (Scalability)

* ***Strategy:** Parameterized Generic Checks (80%) \+ Specific Checks (20%).*  
* ***Identity:** ruleId in YAML allows reusing the same checkId multiple times.*  
* ***Guardrail:** No DSL in YAML. Logic stays in TS.*

# 8\. Technology Stack

* ***Language:** TypeScript (Node.js).*  
* ***Queue:** BullMQ (Redis) for Producer/Consumer architecture.*  
* ***Validation:** Zod for YAML runtime safety.*  
* ***Reasoning:** Shared contracts with React frontend, Backstage synergy, and native dynamic dispatch for the plugin system.*

### **Zod Schema Organization**

Domain enum schemas live in `apps/api/src/domain/schemas/enums.ts`. Place schemas here if:
- Values represent **domain states** (e.g., `ReadinessStatus`, `CheckStatus`, `BadgeType`)
- Code **branches semantically** on these values
- They exist **independent of configuration**

Configuration-specific schemas (e.g., `FocusModeIdSchema`) stay in their respective config modules. These are validation constraints for config files, not domain concepts.

### **Logging Principles**

Ground Control uses structured logging with two levels: **INFO** for production observability and **DEBUG** for investigation.

**Core Principle: Logs observe, they don't compute.**

Logging must never change the code's structure or add computational overhead. If you need to filter/map/reduce to get a value, it belongs at DEBUG level in the function that naturally computes it—not in the log statement.

**INFO Level — "What happened?"**
- Entry/exit points of major operations
- Final decisions (what won, how long)
- Values that are **already computed** — no inline `.filter()`, `.map()`, or aggregations
- 3-5 lines per operation maximum

```typescript
// ✅ Good: Log values that exist
logger.info({ serviceId, total: candidates.length }, 'NBU: enumerated');

// ❌ Bad: Computing in the log
logger.info({ byType: candidates.filter(c => c.type === 'X').length }, '...');
```

**DEBUG Level — "Why did it happen?"**
- Logged **where work happens**, not after the fact
- Per-item details during iteration (scores, decisions)
- Each generator/processor logs its own output

```typescript
// In the scoring function, where computation naturally occurs:
logger.debug({ c: `${type}:${rulepackId}`, score, mult }, 'NBU: scored');
```

**Prefix Convention:** Use a consistent prefix (e.g., `NBU:`, `SCAN:`) for grep-ability across log aggregators.

**Anti-Patterns:**
- Creating interfaces or types solely for logging
- Adding helper functions that exist only to support logging
- Changing function return types to accommodate log data
- Logging inside tight loops without aggregation
- Logging without `serviceId` or correlation key

**The Test:** If removing a log statement would simplify the code, the logging was too invasive.

# 9\. Frontend Stack

Ground Control v1 will use **React** as the UI framework, with a design system optimized for a narrative, highly-customizable “Mission Control” experience. We will adopt **Tailwind CSS** for styling tokens and rapid iteration, and **shadcn/ui (Radix-based components)** as the primary component foundation. This choice intentionally avoids a rigid, pre-themed enterprise UI look and enables full ownership of UI primitives (cards, tabs, dialogs, tooltips, command palette, badges, etc.) so the product can express the “Ground Control” tone consistently and evolve without fighting upstream constraints. For data-dense surfaces (readiness tables, service tables, sortable/filterable grids), we will use **TanStack Table** to provide robust table mechanics while keeping full control over visual presentation and theming.

The UI is **desktop-first** (no mobile support required) and will include a dedicated **TV/Kiosk mode** as a first-class layout variant: larger typography, simplified navigation, auto-rotating “views” (e.g., readiness → pillar readiness → top movers), and keyboard-only interaction. The frontend will treat the backend as the source of truth: it renders **pre-enriched snapshots** and **curated recommendations** from the API without re-implementing scoring or rule logic client-side, ensuring consistency, performance, and agent-friendly development.

**Vocabulary rendering contract:** The UI renders backend-provided vocabulary and does not define domain text. All labels, descriptions, and tooltips for domain concepts (readiness states, upgrade types, coaching metrics, etc.) come from the API — either embedded directly in the response payload or fetched from `/config/vocabulary`. Frontend components MUST NOT hardcode domain-specific strings. See Architectural Principle 7 (Topic-Based Vocabulary Contract) for the full specification.

# 10\. API Specification (REST Interface)

### **GET /config/rulepacks (Extended Response)**

Returns full protocol definitions for the Flight Manual and standards transparency. The response includes:
- **Evaluation metadata:** Tier thresholds, baseline config, applicableWhen
- **Badge semantics:** Badge type (GATEKEEPER/LADDER), tier definitions
- **Signal sources:** Context keys required for evaluation
- **editUrl:** Link to edit the rulepack source (when configured)
- **Per-rule criteria:** Threshold config, criteriaText, implication, effort, impact for each rule

### **Design Principles**

1. **Resource-Oriented:** URLs represent resources (Services, Scans, Teams), not actions.  
2. **Async by Default:** Heavy calculations (Evaluations) are triggered via POST /scans (fire-and-forget) and return 202 Accepted.  
3. **Read-Optimized:** GET endpoints return pre-calculated Snapshots from the DB, ensuring sub-100ms response times for the UI.

---

### **Implementation Hints for the AI Agent**

1. **Filtering:** Implement standard query params for /services and /teams:  
   * ?teamId=...  
   * ?tier=TIER\_1  
   * ?limit=20\&offset=0  
2. **Snapshot Expansion:** The GET /services/:id/snapshot endpoint should return the *enriched* snapshot. Do not require the UI to fetch checking logic. The Backend does the heavy lifting.  
3. **Upgrade Logic:** The /services/:id/upgrades endpoint does not access the DB directly. It fetches the latest Snapshot and passes it through the recommendUpgrades() pure function defined in Section 4.E.  
4. **Error Handling:** Use standard HTTP codes.  
   * 404: Service not found.  
   * 422: Config validation failed (Zod).  
   * 503: Redis/Queue unavailable.

# 11\. Database Strategy & Design Specification

### **11.1 Core Principles**

1. **Backstage is the Source of Truth** for catalog metadata (services, teams).  
2. **Ground Control DB is read-optimized**: everything the UI needs must be queryable via fast SQL (no “join in Node”).  
3. **Store only what we query** (sort/filter/join) as columns; store deep details as **JSONB**.  
4. **Immutable history**: evaluation results are append-only; retention is a policy, not a requirement for correctness.

**11.2 Service Catalog Strategy: “Mirror with Soft Deletes”**

**Decision:** Maintain a **local mirror** of Services (and Teams) in PostgreSQL. Backstage remains master.

**Why we mirror (non-negotiable for team/org aggregation views)**

* Readiness aggregation and team views require **SQL joins** across: Service → Team → latest Scan scores.  
* If catalog lives only in Backstage API, you’d fetch thousands of scans and thousands of services and join in memory (slow, non-paginatable, fragile).

**Sync workflow**

* A **catalog sync job** runs:  
  * on startup  
  * then every **60 minutes** (configurable)  
* Behavior:  
  * **Upsert** services \+ teams by stable IDs  
  * **Soft delete**: if a service is missing from Backstage, mark it inactive (`isActive=false`, set `deletedAt`)  
  * Do **not** hard-delete immediately (keeps history and prevents accidental data loss)

**Handling service moves between teams**  
Services may move ownership. We support two valid behaviors; pick one and lock it. Team/org views reflect today’s team ownership even for historical scans (simpler but retroactively changes history).

**11.3 Schema Management & Upgrades**

**Decision:** Use **Prisma \+ Prisma Migrate**.

**Why Prisma fits this project**

* **Migration workflow** maps cleanly to Flyway’s mental model: schema evolves via versioned migrations committed to git.  
* Prisma generates a **type-safe client** for TypeScript, which reduces query drift and supports AI-agent development (fewer runtime errors, more compile-time feedback).  
* We still keep full control: raw SQL is available when needed.

**Operational rules**

* Migrations are **forward-only**.  
* Migrations run **before** new app instances serve traffic (K8s init container).  
* App code must tolerate brief rolling overlap during deploy.

**11.4 What We Store (and why)**

#### **A) Structured relational data (for JOIN/SORT/FILTER)**

1. **Team \-** Needed for team/org aggregation, labels, UI headers.  
2. **Service \-** Needed for join integrity, filtering by team/group, and “latest snapshot per service”.  
3. **Scan (evaluation run) \-** Needed for historical charts, deltas, and “latest score”.

#### **B) Unstructured JSONB (for explainability \+ evolution)**

* Store the full **EvaluationSnapshot** as `data: Json`.  
* This avoids schema churn as rulepacks/checks evolve.  
* We only extract a small set of “query fields” into columns for speed.

**Explicitly NOT stored in v1**

* raw collector payloads (git trees, k8s objects, raw datadog responses) **as a dedicated data store**  
* normalized per-check SQL tables (no `check_results` table)  
* managed quest lifecycle state

**Note (Evidence V2):** We may store check-scoped evidence/provenance as part of check results within the EvaluationSnapshot. This includes:
* verification links (pipelines, dashboards, docs)
* targeted debug context (query details, sampled IDs, window bounds)

This is conceptually "evidence for the judgement," not "persist everything the collector fetched." Check authors own the evidence structure and decide what helps answer "why do you believe this?" and "how do I debug it?"

**11.5 Minimal Prisma Models (v1)**

```
model Team {
  id        String   @id   // Backstage team ref or stable slug
  name      String
  updatedAt DateTime @updatedAt

  services  Service[]
}

model Service {
  id          String   @id  // Backstage entity ref (e.g. "component:default/payment-service")
  name        String
  teamId      String
  team        Team     @relation(fields: [teamId], references: [id])

  repoUrl     String?
  isActive    Boolean  @default(true)
  deletedAt   DateTime?
  lastSeenAt  DateTime?

  scans       Scan[]

  @@index([teamId])
  @@index([isActive])
}

model Scan {
  id          String   @id @default(uuid())
  serviceId   String
  service     Service  @relation(fields: [serviceId], references: [id])

  createdAt   DateTime @default(now())

  // (Recommended) Preserve historical ownership attribution:
  teamIdAtScan String?

  // Query fields for aggregation/sorting
  globalScore  Int
  pillarScores Json
  readinessStatus String   // e.g. "FLIGHT_READY" | "PRE_FLIGHT" | "TELEMETRY_MISSING"

  // Full immutable snapshot (EvaluationSnapshot)
  data         Json

  @@index([serviceId, createdAt(sort: Desc)])    // per-service: history, vitality, latest
  @@index([teamIdAtScan, createdAt(sort: Desc)]) // per-team: team momentum queries
  @@index([createdAt(sort: Desc)])                // org-wide: momentum, deltas
}
```

**Notes**

* `pillarScores` is stored as JSON for flexibility; UI queries typically fetch the latest scan per service, not aggregate across arbitrary pillar keys.  
* `data` stores the full EvaluationSnapshot (source of truth for explanations, evidence, rulepack results).

**11.5.1 ServiceState — Current Truth Table**

ServiceState is a single row per service, upserted by the worker at the end of each scan. Aggregators read from this table instead of running expensive DISTINCT ON queries against Scan.

"""
model ServiceState {
  serviceId            String   @id
  readinessStatus      String   // FLIGHT_READY | PRE_FLIGHT | TELEMETRY_MISSING
  pillarReadiness      Json     // Record<pillarId, ReadinessStatus>
  pillarScores         Json     // Record<pillarId, number>
  globalScore          Int      @default(0)
  signalCoveragePct    Int      @default(0) // 0-100: evaluable required / total required
  blockedRulepackIds   Json     @default("[]")
  coreCount            Int      @default(0)
  circuitCount         Int      @default(0)
  riskLevel            String   @default("NOMINAL")
  isClosestWin         Boolean  @default(false)
  failingRequiredCount Int      @default(0)
  lastEvaluatedAt      DateTime
  latestScanId         String   // FK to Scan for drilldown
  teamId               String
  groupId              String?
  updatedAt            DateTime @updatedAt
}
"""

**Write flow**: Scan Worker saves Scan (immutable history) then upserts ServiceState (current truth). Writes are batched per scan job using `$transaction` to minimize DB round-trips.

**Freshness guard**: ScanQueryService filters `WHERE lastEvaluatedAt > NOW() - INTERVAL '3 days'`, ensuring stale services are excluded from active counts.

**11.5.2 Index Strategy**

**Design principle:** One index per dominant query pattern. Every index must map to a real query in the codebase; speculative indexes waste write I/O and are removed.

**Scan indexes** (append-only history table — indexes optimized for read patterns):

| Index | Query pattern it serves |
|-------|------------------------|
| `[serviceId, createdAt DESC]` | Per-service: latest scan, scan history, vitality lastPassedAt lookup |
| `[teamIdAtScan, createdAt DESC]` | Per-team: team momentum history |
| `[createdAt DESC]` | Org-wide: momentum aggregation, service deltas |

All momentum queries also filter `isActiveAtScan = true`. Since this is a boolean with ~99%+ true selectivity, it is not included in the index — Postgres evaluates it as a cheap inline filter with near-zero rejection.

**ServiceState indexes** (current-truth table — one row per service):

| Index | Query pattern it serves |
|-------|------------------------|
| `[teamId, lastEvaluatedAt DESC]` | Team Console: team services with freshness guard |
| `[teamId, readinessStatus]` | Team readiness aggregation and pillar readiness |
| `[readinessStatus]` | Org summary: readiness distribution counts |
| `[riskLevel]` | Risk-based filtering |
| `[groupId]` | Group-level aggregation |
| `[lastEvaluatedAt]` | Freshness-only queries (latest eval timestamp, count evaluated) |

**Scaling path (Scan table partitioning):**

The Scan table grows linearly (~N rows per scan cycle, where N = number of services). With the correct indexes above, Postgres handles this efficiently up to low millions of rows. When Scan exceeds ~5M rows or momentum query p95 exceeds the SLA, partition by `createdAt` using `pg_partman`:

1. Convert to declarative range partitioning: `PARTITION BY RANGE ("createdAt")`
2. Register with `pg_partman` for automatic monthly partition creation (`p_premake = 3`)
3. Application code and Prisma see a single `"Scan"` table — fully transparent

This is a future scaling operation, documented here as the planned path. No partitioning is needed at launch.

**Separation of concerns**:
- `Service` = What Backstage tells us (catalog mirror)
- `ServiceState` = What Ground Control knows now (evaluation truth)
- `Scan` = Immutable evaluation history

**11.6 Scan Trigger (API /scans)**

The scan endpoint is a pure **fire-and-forget fan-out**. There is no status tracking or polling.

The database sources of truth are the `Scan` table (immutable per-service evaluation records) and `ServiceState` (latest materialized view per service).

`POST /scans` accepts optional `{ serviceId, teamId }`, enqueues a BullMQ job, and returns `202 Accepted` immediately. No intermediate state table is needed.

**Team resolution:** Team is always resolved at scan time via the catalog mirror.

**11.7 Retention & Cleanup**

**V1:** Keep everything (simple). The volume is manageable for Postgres.

**11.8 Why This Design Wins (Simplicity \+ Trust)**

* **Fast UI** (SQL joins locally, no runtime joins in Node)  
* **Fairness preserved** (snapshot retains evidence and mastery reasons)  
* **Low maintenance** (soft deletes; Backstage remains the master)  
* **Schema evolution without pain** (JSONB snapshot \+ Prisma migrations)

**11.9 Fleet Scope Contract**

`activeServiceIds` from `ServiceDirectory` is the **sole authority** for which services participate in any aggregation. This is a first-class architectural concept:

* **Controllers** resolve `activeServiceIds` from `ServiceDirectory.getAllServiceIds()` and pass it as a required parameter to all aggregators.
* **Aggregators** (ReadinessAggregator, TeamConsoleAggregator) forward `activeServiceIds` to every query that computes per-service metrics — including `MomentumQueryService` calls.
* **Query services** that compute per-service metrics (`MomentumQueryService`) accept `activeServiceIds` as a required parameter and filter SQL queries to the active fleet. They never define their own fleet boundary.
* **Domain functions** that produce ratios (`buildOrgMomentumData`) derive both numerator and denominator from the same scoped input (`FleetDeltas`). Separate denominator parameters are forbidden because they allow universe mismatch.

This design prevents the "1243 of 1240 services" class of bugs where numerator and denominator come from different fleet definitions.

# 12. V3 Vitality Layer Architecture

The Vitality Layer implements **scan-cadence independent** state derivation for CIRCUIT achievements, ensuring the same real-world situation produces the same CircuitState regardless of how often we scan.

## 12.1 The Core Truth

> **"Is the problem still there?"** — Only reality matters.

DEGRADING/DOWN answers a single question: "How long has this problem existed?" This is fundamentally a **time-based** question, not a scan-count question. A CI stability issue that has persisted for 48 hours is equally problematic whether we've scanned 2 times or 20 times.

## 12.2 The Fairness Argument

Same problem duration = same CircuitState, regardless of activity level.

Consider two services with the same CI stability issue:
- **Service A:** Scanned 10 times in 48 hours (every ~5 hours)
- **Service B:** Scanned 2 times in 48 hours (every ~24 hours)

With scan-count based vitality, Service A would appear worse (more consecutive failures) despite having the *exact same problem* for the *exact same duration*. This violates fairness.

**Time-based vitality ensures fairness:** Both services show the same CircuitState because the underlying reality (48 hours without passing) is identical.

## 12.3 Two Dimensions (Conceptually Separate)

| Dimension | Purpose | Example | Where Configured |
|-----------|---------|---------|------------------|
| **Check Window** | Measurement scope (truth) | "Last 24h of crashes", "Last 20 MRs" | Check params in rulepack YAML |
| **Vitality Thresholds** | Coaching urgency (meaning) | "DEGRADING after 12h", "DOWN after 36h" | VitalityThresholds (per-rulepack or global) |

These are independent concerns:
- A check might measure "crash rate in last 24 hours" (window = 24h for data collection)
- Vitality might be "DOWN after 36 hours without passing" (urgency = 36h for state transition)

## 12.4 Truth vs Meaning for Vitality

| Layer | Name | When Computed | Stored? | Description |
|-------|------|---------------|---------|-------------|
| **Truth** | `lastPassedAt` | Scan-time | Yes (in `RulepackBadgeResult`) | ISO timestamp when `baselineMet=true` last occurred |
| **Meaning** | `CircuitState` | View-time | No | NOMINAL/DEGRADING/DOWN derived from `(now - lastPassedAt)` + hour-based thresholds |

**Why timestamp instead of count?**
- Timestamps are **scan-cadence independent** — the truth doesn't change based on scan frequency
- Hours since last pass is a **real-world duration** that reflects how long the problem has existed
- Policy changes (adjusting thresholds) immediately affect ALL historical views

## 12.5 Configuration Separation

**Engine Config** (controls observation scope):
- `vitalityLookbackScans`: How many historical scans to search when finding `lastPassedAt`
- Similar to check params — defines the scope of historical observation
- Loaded from environment variable: `VITALITY_LOOKBACK_SCANS` (default: 10)

**Meaning Config** (controls interpretation - VitalityThresholds):
- `degradingAfterHours`: Hours since lastPassedAt before DEGRADING state (default: 12)
- `downAfterHours`: Hours since lastPassedAt before DOWN state (default: 36)
- **Per-rulepack configurable:** Different Circuits can have different urgency profiles

```yaml
# Example: High-urgency Circuit (runtime stability)
rulepacks:
  - id: runtime_stability
    vitalityThresholds:
      degradingAfterHours: 6    # More urgent
      downAfterHours: 24

# Example: Lower-urgency Circuit (CI flakiness)
  - id: ci_stability
    vitalityThresholds:
      degradingAfterHours: 24   # Less urgent
      downAfterHours: 72
```

## 12.6 Data Flow

```
Scan-Time:
1. Orchestrator evaluates service → produces RulepackBadgeResult with baselineMet
2. If baselineMet=true: no lastPassedAt needed (currently NOMINAL)
3. If baselineMet=false: VitalityHistoryService finds lastPassedAt from history
4. Stores lastPassedAt (timestamp) in snapshot

View-Time:
1. FlightDeckAggregator reads lastPassedAt from snapshot (truth)
2. Calculates hoursSinceLastPass = (now - lastPassedAt)
3. Calls deriveCircuitState(lastPassedAt, VitalityThresholds)
4. Returns CircuitState to UI (meaning)
```

## 12.7 CircuitState Derivation

```typescript
function deriveCircuitState(
  lastPassedAt: string | undefined,
  thresholds: VitalityThresholds,
  now: Date = new Date()
): CircuitState {
  // If no lastPassedAt recorded, the check is either:
  // 1. Currently passing (so we didn't need to record when it last passed), or
  // 2. First scan ever (no history yet)
  // Both cases are NOMINAL
  if (!lastPassedAt) return 'NOMINAL';
  
  const hoursSinceLastPass = (now - new Date(lastPassedAt)) / (1000 * 60 * 60);
  
  if (hoursSinceLastPass >= thresholds.downAfterHours) return 'DOWN';
  if (hoursSinceLastPass >= thresholds.degradingAfterHours) return 'DEGRADING';
  return 'NOMINAL';
}
```

## 12.8 Key Components

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `VitalityHistoryService` | `enrichment/vitality-history.ts` | Query historical scans, find `lastPassedAt` timestamp |
| `deriveCircuitState()` | `enrichment/vitality-evaluator.ts` | Pure function: lastPassedAt + thresholds → CircuitState |
| `calculateFadeProgress()` | `enrichment/vitality-evaluator.ts` | Pure function: compute UI fade animation progress (0-1) |
| `VitalityThresholds` | `domain/interfaces/rulepack.ts` | Interface defining `degradingAfterHours` and `downAfterHours` |
| `getVitalityThresholdsForRulepack()` | `config/loader.ts` | Get threshold config for specific rulepack (with fallback to global) |

## 12.9 Backward Compatibility

- Existing snapshots without `lastPassedAt` field are treated as NOMINAL
- Per-rulepack thresholds fall back to global defaults if not configured
- No need for backward compatibility as this product was never released yet!!

# 13. Navigation Context Architecture

Ground Control's three views form an altitude hierarchy: **Mission Map** (org) → **Squad Console** (team) → **Flight Deck** (service). Users navigate up and down this hierarchy constantly. The Navigation Context architecture ensures the user's "viewing lens" — their Command Bar selections — persists across view transitions without leaking view-local state.

## 13.1 The Problem

All UI state is stored in URL search params (see Section 9, Frontend Stack). When navigating between views, the URL changes entirely (e.g., `/team/X?pillar=prod_health` → `/service/Y`). Without explicit context propagation, the user's pillar filter, focus mode, and timeframe selections are lost on every hop.

## 13.2 Two Categories of URL State

| Category | Params | Behavior on Navigation |
|----------|--------|------------------------|
| **Navigation Context** (shared lens) | `pillar`, `focus`, `timeframe` | Auto-carried to the target view |
| **View-Local State** | `segment`, `protocolFilter`, `protocolSort`, `drawer`, `sortBy`, `sortDir`, `search`, `protocol`, `q`, `section`, `showVitality`, `showLeverage`, `manual`, `manualTab`, `manualQ` | Dropped — stays behind in the source view |

**Manual URL params** (view-local):
- `manual`: Manual target (e.g., `rulepack:ci_performance`)
- `manualTab`: Active tab (`standard` | `legend`)
- `manualQ`: Search query within manual

**Design principle:** Navigation Context params are exactly the params the Context Strip (FMA) annunciates. If the FMA shows it, it travels. If the FMA doesn't show it, it's view-local.

## 13.3 NavigationContext Interface

```typescript
interface NavigationContext {
  pillar?: string;
  focus?: FocusMode;
  timeframe?: Timeframe;
}
```

`extractNavigationContext(state: ViewState): NavigationContext` is a pure function that picks only shared params from the current ViewState. It is the **single source of truth** for what is shareable.

## 13.4 Auto-Carry Contract

All cross-view navigation functions (`navigateToService`, `navigateToTeam`, `navigateToMissionMap`) follow the same contract:

1. Read the current URL's `searchParams`
2. Call `extractNavigationContext()` to get the shared lens
3. Merge with any explicit overrides from the caller
4. Serialize and navigate to the target path

```
navigateToService(serviceId, overrides?)
  → context = extractNavigationContext(currentURL)
  → merged  = { ...context, ...overrides }
  → navigate("/service/{serviceId}?{merged}")
```

**Callers are unchanged.** `navigateToService(id)` still works with no additional arguments. The context propagation is invisible to consumers — separation of concerns.

## 13.5 Breadcrumb Hierarchy

Each view renders a breadcrumb header that reflects the altitude hierarchy:

| View | Breadcrumb | Back Target |
|------|-----------|-------------|
| **Mission Map** | *(none — top level)* | — |
| **Squad Console** | `← Mission Map \| Ground Control · Squad Console` | Mission Map (via `navigateToMissionMap`) |
| **Flight Deck** | `← {Team Name} \| Ground Control · Flight Deck` | Owning team's Squad Console (via `navigateToTeam`) |

The Flight Deck breadcrumb uses `ServiceCockpit.ownerTeamId` and `ownerTeamName` from the API response. When team context is unavailable (error/loading states), it falls back to navigating to the Mission Map.

All breadcrumb back-links use the navigation functions (not static `<Link>` elements) so NavigationContext is preserved.

## 13.6 Relationship to Context Strip (FMA)

The Context Strip reads NavigationContext params directly from the URL via `useViewState`. Because the navigation functions carry these params forward, the FMA automatically reflects the correct lens on every view — no additional wiring needed. The FMA annunciates what was carried; the navigation functions carry what the FMA annunciates. This is a closed loop.

## 13.7 Key Components

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `NavigationContext` | `hooks/useViewState.ts` | Interface defining the shared lens |
| `extractNavigationContext()` | `hooks/useViewState.ts` | Pure function: ViewState → NavigationContext |
| `navigateToService()` | `hooks/useViewState.ts` | Navigate to Flight Deck with auto-carried context |
| `navigateToTeam()` | `hooks/useViewState.ts` | Navigate to Squad Console with auto-carried context |
| `navigateToMissionMap()` | `hooks/useViewState.ts` | Navigate to Mission Map with auto-carried context |
| `FlightDeckBreadcrumb` | `views/FlightDeckView.tsx` | Breadcrumb with team-aware back navigation |
| `SquadConsoleBreadcrumb` | `views/SquadConsoleView.tsx` | Breadcrumb with Mission Map back navigation |
| `ContextStrip` | `components/shared/ContextStrip.tsx` | FMA — reads same params, no changes needed |

# 14. Queue & Scheduling Architecture

Ground Control uses **BullMQ** (backed by Redis) for asynchronous scan processing. The queue decouples scan triggering (API or cron) from scan execution (worker), enabling retries, concurrency control, and multi-instance safety without leader election.

## 14.1 Architecture Overview

A single Node.js process runs three roles:

| Role | Responsibility | Controlled By |
|------|---------------|---------------|
| **HTTP Server** (Fastify) | Serves REST API, accepts scan triggers | Always on |
| **Worker** (BullMQ) | Consumes and processes scan jobs from the queue | `SKIP_WORKER` env var |
| **Scheduler** (BullMQ) | Registers repeatable cron jobs into Redis | `ENABLE_SCHEDULER` env var |

All three share the same process, Redis connection, and Postgres connection. The startup order is: Database → Catalog Sync → Worker → Scheduler → HTTP Server.

## 14.2 Queue Design

**Queue name:** `ground-control-scans` (single queue for all scan types).

**Design principle:** The queue is a **trigger mechanism only** — the database is the source of truth for scan results. Completed and failed jobs are removed from Redis immediately (`removeOnComplete: true`, `removeOnFail: true`). If Redis loses all data, the only impact is losing pending jobs and the repeatable schedule (both are re-established on next pod start).

**Job data contract:**

```typescript
interface ScanJobData {
  serviceId?: string;    // Present for evaluation, absent for scheduler fan-out trigger
  createdAt: string;     // ISO timestamp
  scheduled?: boolean;   // True if created by scheduler (not API)
}
```

Every evaluation job carries exactly one `serviceId`. The only case where `serviceId` is absent is the scheduler fan-out trigger (see 14.3).

**Retry configuration:**

```
Attempts: 4 (1 initial + 3 retries)
Backoff: exponential, 60s base delay
Timing: 60s → 120s → 240s
```

This is tuned for **GitLab API 429 rate limit recovery**. The exponential backoff with a 60-second base gives the rate limiter time to reset between attempts.

## 14.3 Two Entry Points Into the Queue

### API-Triggered Scans (Manual)

`POST /scans` with optional `{ serviceId, teamId }`:

1. The controller resolves services from the catalog (single service / team / all org)
2. Enqueues **one BullMQ job per service** via `addBulk` (single Redis round-trip)
3. Returns `202 Accepted` immediately (fire-and-forget, no status polling)

With `WORKER_CONCURRENCY=5`, five services evaluate in parallel.

### Scheduled Scans (Automatic via Cron)

Uses BullMQ's **repeatable jobs** feature. On startup, the scheduler:

1. Removes any existing repeatable job named `scheduled-org-scan` (clean slate)
2. Re-registers with the current `SCAN_CRON` expression
3. BullMQ stores the schedule in Redis and creates a delayed job for the next cron tick

The scheduled job is a **fan-out trigger** (no `serviceId`). When the worker picks it up, it resolves all org services and enqueues individual per-service jobs via `addBulk`. This two-phase approach is needed because the service list changes over time and cannot be baked into the repeatable job at registration.

**Why repeatable jobs instead of an in-process timer (e.g., `node-cron`):**

- The schedule is persisted in **Redis**, not process memory. If the pod crashes 1 minute before a cron tick, the delayed job is already queued in Redis and will be picked up on restart. An in-process timer would miss that tick entirely.
- The `jobId` acts as a **natural deduplication key** across multiple instances. Even without leader election, only one delayed job instance is created per cron tick.

## 14.4 Worker Execution Pipeline

Each BullMQ job evaluates exactly **one service**. The worker has two paths:

**Path A -- Scheduled fan-out trigger** (no `serviceId`):
```
1. Resolve all org services from ServiceDirectory
2. Enqueue individual per-service jobs via addBulk
3. Return (no evaluation, no metrics)
```

**Path B -- Single-service evaluation** (`serviceId` present):
```
1. Fetch service from ServiceDirectory
2. Collection  → Run collectors in parallel (Git, GitOps, Sonic, Datadog, CI)
3. Evaluation  → Run checks against collected context per rulepack
4. Enrichment  → Add lastPassedAt timestamps for CIRCUIT rulepacks
5. Scoring     → Compute pillar scores, readiness, achievement metrics
6. Persist     → Save immutable Scan record + upsert ServiceState
7. Record OTel metrics (per-service duration, outcome)
```

**Fail-safe pattern:** Each service is an independent BullMQ job. If 1 of 200 services fails, BullMQ retries that specific job without affecting the other 199. Fatal errors (e.g., DB unreachable) re-throw for BullMQ retry.

## 14.5 Restart & Crash Behavior

| Scenario | What Happens |
|----------|-------------|
| **Pod restarts quickly** | Repeatable job definition stayed in Redis. BullMQ fires the next delayed job at the scheduled time. Worker picks it up. No missed schedules. |
| **Pod is down for a while** | Repeatable job persists in Redis. BullMQ enqueues the **next** occurrence after the worker reconnects. It does NOT retroactively fire missed occurrences. |
| **Rolling deploy** | Old pod shuts down (does NOT remove repeatable jobs from Redis — intentional). New pod starts, removes old repeatable job, registers with current config. Seamless. |
| **Crash (no graceful shutdown)** | Same as "pod is down." Repeatable job persists. Any in-flight job stays in `active` state in Redis; BullMQ's stalled-job checker re-queues it after timeout. The job retries when the worker comes back. |

**Why repeatable jobs are NOT removed on shutdown:** The repeatable job is a shared resource. Removing it during a rolling deploy would break scheduling until another instance registers it. The registration is idempotent (same `jobId` = same job), so new instances with updated cron config naturally replace the old definition.

## 14.6 Graceful Shutdown

On `SIGTERM` or `SIGINT`, the process shuts down in order:

1. **Fastify** — stop accepting new HTTP requests
2. **Scheduler** — no-op (repeatable jobs intentionally remain in Redis)
3. **Worker** — `worker.close()` waits for currently-processing jobs to finish, then stops accepting new ones
4. **Catalog Sync** — stop periodic sync timer
5. **Database** — disconnect Prisma client
6. **Metrics** — flush OpenTelemetry

## 14.7 Multi-Instance Safety (Without Leader Election)

All components are safe to run across multiple pod replicas:

| Component | Multi-Instance Safe? | Mechanism |
|-----------|---------------------|-----------|
| **Worker** | Yes | BullMQ atomic job assignment (Redis BRPOPLPUSH). Each job delivered to exactly one worker. |
| **Scheduler** | Yes | `jobId` deduplication. Multiple pods registering the same repeatable job produce one delayed job per cron tick. |
| **HTTP API** | Yes | Stateless. Reads/writes go through Postgres. |
| **Catalog Sync** | Yes (redundant) | Multiple pods run sync in parallel — redundant work but harmless (upserts are idempotent). |

**When leader election becomes valuable:** Leader election is not needed for correctness, but improves efficiency at scale. It would prevent N pods from all running catalog sync, and consolidate scheduler registration to a single instance. This is a future optimization, not a prerequisite for scaling.

## 14.8 Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `REDIS_URL` | `redis://localhost:6379` | Redis connection for BullMQ |
| `WORKER_CONCURRENCY` | `5` | How many jobs the worker processes in parallel |
| `ENABLE_SCHEDULER` | `false` (disabled) | Must be `"true"` to activate cron scheduling |
| `SCAN_CRON` | `0 6,18 * * *` | Cron expression for scheduled scans (default: 6 AM and 6 PM UTC) |
| `SKIP_WORKER` | `false` | Set to `"true"` to disable the worker (API-only mode) |

## 14.9 Key Components

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `getRedisConnectionOptions()` | `queue/connection.ts` | Redis connection singleton |
| `getScanQueue()` | `queue/producer.ts` | Queue creation with retry/cleanup config |
| `enqueueScan()` | `queue/producer.ts` | Enqueue a single scan job |
| `enqueueBulkServiceScans()` | `queue/producer.ts` | Enqueue one job per service via addBulk |
| `startScheduler()` | `queue/scheduler.ts` | Register repeatable cron job in Redis |
| `stopScheduler()` | `queue/scheduler.ts` | No-op (repeatable jobs persist intentionally) |
| `startScanWorker()` | `queue/worker.ts` | Start BullMQ worker with concurrency config |
| `stopScanWorker()` | `queue/worker.ts` | Graceful worker shutdown |
| `processScanJob()` | `queue/worker.ts` | Job processor — fan-out trigger or single-service evaluation |

# 15. Recommendation Engine Architecture

Ground Control uses a two-engine model with a shared kernel to produce upgrade recommendations at every altitude (service, team, organization).

## 15.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                  Recommendation Kernel                   │
│  recommendation-kernel.ts                                │
│                                                          │
│  CANONICAL_PRIORITY        Authoritative priority order  │
│  needsVitalityProtection() Time-based vitality predicate │
│  isHighImpactLowEffort()   Friction threshold predicate  │
│  applyFocusModeBoost()     Focus mode priority boost     │
└─────────────────┬───────────────────────┬───────────────┘
                  │                       │
    ┌─────────────▼──────────┐  ┌────────▼────────────────┐
    │   NBU Engine           │  │   Fleet Engines          │
    │   (Service Level)      │  │   (Team + Org Level)     │
    │                        │  │                          │
    │   algorithm.ts         │  │   team-console-           │
    │   pure-functions.ts    │  │     aggregator.ts        │
    │                        │  │   readiness-              │
    │   Candidate enumeration│  │     aggregator.ts        │
    │   Leverage scoring     │  │                          │
    │   Deterministic select │  │   Cross-service aggregate │
    │   Explainability       │  │   Volume thresholds      │
    └────────────────────────┘  └──────────────────────────┘
```

**The kernel owns "what" and "order".** It defines the shared predicates (what qualifies as each recommendation type) and the canonical priority order. Engines own "how" — the scoring math, aggregation strategy, and output format.

## 15.2 Recommendation Kernel

Location: `apps/api/src/view-layer/recommendation-kernel.ts`

The kernel is the single source of truth for cross-engine concerns:

| Export | Purpose |
|--------|---------|
| `CANONICAL_PRIORITY` | Priority values for each `UpgradeType`. PROTECT_VITALITY (100) > UNBLOCK_TELEMETRY (90) > REACH_BASELINE (80) > REDUCE_FRICTION (60) > OPTIMIZE (40). |
| `needsVitalityProtection()` | Time-based check using `deriveCircuitState()`. Returns true for DEGRADING or DOWN Circuits. Scan-cadence independent. |
| `isBaselineGapCandidate()` | Returns true for CORE rulepacks with required baseline not met. Explicitly excludes CIRCUITs (handled by PROTECT_VITALITY). |
| `isHighImpactLowEffort()` | Returns true for (HIGH or MEDIUM impact) AND (SMALL or MODERATE effort). |
| `classifyRulepack()` | **Mutually exclusive** rulepack classifier for fleet engines. Returns `BASELINE`, `VITALITY`, or `RULE_LEVEL`. Single `if/else if` chain prevents a rulepack from appearing in multiple recommendation types. |
| `applyFocusModeBoost()` | Priority boost for focus modes: BASELINE_READY +50 for telemetry/baseline, others +30 for tag matches. |

## 15.3 NBU Engine (Service Level)

Location: `apps/api/src/view-layer/next-best-upgrade/`

Produces a single "next best upgrade" with ranked alternatives for one service. Three-phase algorithm:

1. **Enumeration**: Generates `RawCandidate` objects from 5 sources (telemetry, baseline, vitality, friction, optimize). Uses kernel predicates for vitality and friction detection.
2. **Scoring**: Computes leverage = `(impact / effort) * confidence * multipliers`. Multipliers include the "Save the Baby" damper (0.1x on non-vitality when any Circuit needs protection), focus mode, and spotlight boosts.
3. **Selection**: Deterministic sort with 7-level tie-breaking. Returns primary + alternatives with explainability.

## 15.4 Fleet Engines (Team and Org Level)

Location:
- `apps/api/src/view-layer/team-console-aggregator.ts`
- `apps/api/src/view-layer/readiness-aggregator.ts`

Aggregate recommendations across multiple services. Key differences from NBU:

- **Volume thresholds**: Telemetry recommendations require 30% or more of fleet or 3+ absolute services.
- **Aggregation**: Issues are grouped by rulepack across services (affected service counts).
- **Priority**: Uses `CANONICAL_PRIORITY` from kernel as base priority, with optional focus mode boost.
- **Output**: Flat list of `UpgradeRecommendation` (not primary + alternatives).

### Mutual-Exclusion Classification

Fleet engines classify each rulepack via `classifyRulepack()` from the kernel. The classification is a single `if/else if` chain that guarantees mutual exclusion:

1. **Telemetry** (per-check, always processed): BLOCKED checks produce UNBLOCK_TELEMETRY regardless of rulepack classification.
2. **BASELINE** (per-rulepack): If `isBaselineGapCandidate(pack)` is true, the entire rulepack enters `baselineGaps`. No individual rules from this rulepack appear in REDUCE_FRICTION or OPTIMIZE.
3. **VITALITY** (per-rulepack): If `needsVitalityProtection(pack, config)` is true, the rulepack enters `downCircuits`. No rule-level leakage.
4. **RULE_LEVEL** (per-check): Only reached when the rulepack is neither a baseline gap nor a vitality concern. Individual failing checks are classified as REDUCE_FRICTION (high impact, low effort) or OPTIMIZE (everything else).

This design eliminates the need for post-hoc deduplication (`buildExclusionSet`/`shouldExclude`) in fleet engines. Mutual exclusion is enforced structurally by the `if/else if` chain, not by cleanup passes.

## 15.5 Canonical Priority Order

The priority order follows the "Keep the Baby Alive" principle:

| Priority | Type | Rationale |
|----------|------|-----------|
| 100 | PROTECT_VITALITY | Save the baby — non-negotiable |
| 90 | UNBLOCK_TELEMETRY | Connect instruments — can't fix what you can't see |
| 80 | REACH_BASELINE | Build the foundation |
| 60 | REDUCE_FRICTION | Quick wins (high impact, low effort) |
| 40 | OPTIMIZE | Remaining improvements |

This order is defined once in `CANONICAL_PRIORITY` and consumed by all engines.

## 15.6 Vitality Detection

Vitality is detected via `needsVitalityProtection()` which delegates to `deriveCircuitState()` in the enrichment layer. The detection is time-based and scan-cadence independent:

- **NOMINAL**: Currently passing or within grace period
- **DEGRADING**: Failing beyond the degrading threshold (default 24h)
- **DOWN**: Failing beyond the down threshold (default 72h)

All engines (NBU, Team, Org) use the same kernel function, ensuring consistent behavior.

## 15.7 Adding a New Recommendation Type

Checklist:

1. Add the type to `UpgradeType` in `@ground-control/shared` — **compiler enforces** via exhaustive switch in vocabulary provider
2. Add vocabulary entry in `vocabulary-provider.ts` — **compiler enforces** (exhaustive switch)
3. Add entry to `CANONICAL_PRIORITY` in `recommendation-kernel.ts` — **kernel enforces** (TypeScript `Record<UpgradeType, number>`)
4. Add enumeration predicate in kernel (if the type has a shared detection predicate)
5. Add enumeration logic in NBU `algorithm.ts`
6. Add aggregation bucket in fleet engines
7. Add `RECOMMENDATION_TYPE_DEFAULTS` entry in `recommendation-constants.ts`

Steps 1-3 are compiler-enforced. Steps 4-7 are documented here as the authoritative checklist.

## 15.8 Fleet-Level Type Diversity

### Mental Model

At **service level (NBU)**, recommendations use strict canonical priority — the service owner sees the full, honest truth with no filtering or capping.

At **fleet level (org and team)**, the audience is an engineering leader reviewing a coaching summary. The summary must represent the *landscape of issue types*, not just the loudest category. Priority defines display **order**, not **exclusion**.

Without diversity enforcement, a fleet with many CIRCUIT rulepacks may produce enough `PROTECT_VITALITY` candidates to consume all recommendation slots, crowding out `REACH_BASELINE`, `REDUCE_FRICTION`, and other types entirely. This reduces the coaching summary's value as a strategic overview.

### Mechanism: Per-Type Slot Cap

The kernel exports a pure function `applyTypeDiversityCap` that both fleet engines call after sorting and before returning results:

"""typescript
applyTypeDiversityCap<T extends { type: UpgradeType }>(sorted: T[], max: number): T[]
"""

**Algorithm:**

1. Count how many distinct `UpgradeType`s have at least one candidate in the sorted list.
2. Compute `maxPerType = ceil(max / typesWithCandidates)`.
3. Walk the pre-sorted list, keeping at most `maxPerType` items per type.
4. Return the first `max` items.

The formula is elastic and config-agnostic:

| Types with candidates | max | maxPerType |
|---|---|---|
| 1 | 6 | 6 (no suppression) |
| 2 | 6 | 3 |
| 3 | 6 | 2 |
| 5 | 6 | 2 |

### Properties

- **Config-agnostic**: Works regardless of how many rulepacks, circuits, or services are configured.
- **Scale-independent**: Correct for 40 services or 40,000.
- **Priority-preserving**: Within each type, the original sort order (by priority, then by affected services count) is maintained.
- **No artificial suppression**: When only one type has candidates, it receives all slots.
- **NBU unaffected**: Service-level recommendations remain strict priority with no cap.

### Consumers

- `readiness-aggregator.ts` → `getOrgUpgrades()` calls `applyTypeDiversityCap(sorted, max)` after sorting.
- `team-console-aggregator.ts` → `getUpgrades()` calls `applyTypeDiversityCap(sorted, max)` after sorting.