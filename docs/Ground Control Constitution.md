# Ground Control Constitution

**The Immutable Foundation**

This document defines the core concepts, terminology, and relationships that form the foundation of Ground Control. These definitions are permanent and will not change—they define the essence of what Ground Control is and how it thinks about engineering excellence.

> **Implementation guidance** (UX specs, guardrails, configuration details) lives in the [Product Definition](./Product%20Definition_%20Ground%20Control%20-%20V4.md).

---

## 1. Mission Statement

**Ground Control** is an internal Mission Control for engineering excellence. It transforms abstract goals into a trusted coaching cockpit: clear standards, verified evidence, and guided upgrade paths—so teams move from reactive patching to proactive excellence.

Ground Control is intentionally designed to feel like:

- **Coaching**, not surveillance
- **Progress**, not punishment
- **Evidence-backed**, not opinion-based
- **Autonomy-preserving**, not centrally dictated behavior

---

## 2. The Service Health Model

Every service in Ground Control is understood through four dimensions:

```
SERVICE HEALTH MODEL
════════════════════

┌─────────────────────────────────────────────────────────────────────────────┐
│                              SERVICE                                        │
│                                                                             │
│   FOUNDATION (What you've built)                                            │
│   ├── Cores = Permanent achievements from STATIC checks                     │
│   └── "Upgrades you install and keep forever"                               │
│                                                                             │
│   VITALITY (How healthy you are now)                                        │
│   ├── Circuits = Health indicators from ROLLING checks                      │
│   └── "Live systems you must monitor—glitches when degrading"                                        │
│                                                                             │
│   READINESS (Can you fly?)                                                  │
│   ├── Per-pillar gate check based on current baselines                      │
│   └── "Are all required systems currently operational?"                     │
│                                                                             │
│   TRAJECTORY (How are you trending?)                                        │
│   ├── Momentum = Daily rate + window delta                              │
│   └── "Your journey through continuous improvement"                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. The Hierarchy

Ground Control uses a strict hierarchy to keep the system explainable and evolvable:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PILLAR (Strategy Layer)                           │
│  High-level outcomes: Operational Excellence, Production Health, etc.       │
│  Owns: readiness definition, requiredRulepacks, minBaseline per rulepack    │
├─────────────────────────────────────────────────────────────────────────────┤
│                         RULEPACK (Tactics Layer)                            │
│  A capability a service can master (e.g., "Safety Net", "Need for Speed")   │
│  Belongs to ONE pillar. Defines: badgeType, thresholds, applicableWhen      │
├─────────────────────────────────────────────────────────────────────────────┤
│                           RULE (Action Layer)                               │
│  User-facing unit of "what's wrong / what to do"                            │
│  Carries: implication, remediation, impact, effort, tags, threshold,        │
│           blocking (default: true)                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                           CHECK (Unit Layer)                                │
│  Reusable evaluator — measures WITHOUT judging (TRUTH only)                 │
│  Returns: observedValue, displayValue. Has: windowType (STATIC/ROLLING)     │
├─────────────────────────────────────────────────────────────────────────────┤
│                          SIGNAL (Evidence Layer)                            │
│  Raw facts from systems of record (Backstage, Git, K8s, Datadog, CI)        │
│  Versioned, timestamped, traceable                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Rule Blocking Behavior

Rules within a **GATEKEEPER** rulepack can be marked as blocking or non-blocking:

| Property | Default | Effect on Badge |
|----------|---------|-----------------|
| `blocking: true` | Yes | Must pass for GATEKEEPER badge to PASS |
| `blocking: false` | No | Evaluated and shown in UI, but doesn't affect badge outcome |

**Configuration Constraint:** The `blocking` property is validated at startup and is ONLY valid for GATEKEEPER badge types. Using it on LADDER rulepacks will fail config loading with a clear error message.

**Use cases for non-blocking rules:**

- **Optional recommendations:** Rules like HPA (autoscaling) that don't apply to all services
- **Aspirational goals:** Org-wide standards being rolled out gradually without blocking readiness
- **Progressive adoption:** Allow teams to see guidance while working toward compliance

Non-blocking rules:
- Are still evaluated and displayed in the Flight Deck
- Appear in Suggested Upgrades under the REDUCE_FRICTION or OPTIMIZE category
- Do not affect the GATEKEEPER badge PASS/NOT_MET outcome
- Do not block a pillar from reaching FLIGHT_READY status

**Note:** LADDER badges evaluate a single numeric metric mapped to tiers (GOLD/SILVER/BRONZE). Each LADDER rulepack must have exactly one rule. The blocking concept does not apply to LADDER.

---

## 4. Achievement Types

Achievements are the user-facing representation of rulepack evaluation results. The type is **derived automatically** from the check's `windowType`.

### 4.1 Cores (Foundation)

Permanent achievements earned from STATIC checks. Once earned, they remain visible as part of the service's foundation.

| Property | Value |
|----------|-------|
| Derived from | `check.windowType: STATIC` |
| Permanence | Permanent — stays earned once built |
| Use case | Config/hygiene checks that stay true once configured |
| Visual | Trophy/badge in gallery |

**Core States:**

| State | Meaning |
|-------|---------|
| `earned` | Badge criteria met |
| `not_earned` | Badge criteria not yet met |
| `telemetry_missing` | Cannot evaluate (collector failed) |

### 4.2 Circuits (Vitality)

Temporary achievements that must be maintained. They represent current operational vitality over a rolling time/event window.

| Property | Value |
|----------|-------|
| Derived from | `check.windowType: ROLLING` |
| Permanence | Temporary — must be continuously maintained |
| Use case | Rolling metrics (24h crash rate, CI stability over N merges) |
| Visual | Pulsing/glowing indicator with health state |

**Circuit States** (derived at view-time from hours since last pass):

| State | Meaning |
|-------|---------|
| `nominal` | Currently passing or within grace period |
| `degrading` | Problem exists but within grace threshold |
| `down` | Problem persisted beyond threshold |

**Circuit State Derivation (Scan-Cadence Independent):**

```
if check is currently passing       → NOMINAL
if lastPassedAt is undefined        → NOMINAL
if hours since lastPassedAt >= downAfterHours       → DOWN
if hours since lastPassedAt >= degradingAfterHours  → DEGRADING
else → NOMINAL
```

This ensures the same real-world problem duration produces the same state regardless of scan frequency.

---

## 5. Badge Types

Badge types define HOW a rulepack is evaluated. This is orthogonal to achievement type.

### 5.1 GATEKEEPER

Binary pass/fail evaluation based on threshold.

| Outcome | Meaning |
|---------|---------|
| `PASS` | Met threshold |
| `NOT_MET` | Below threshold |
| `NOT_EVALUATED` | Cannot evaluate (blocked) |

### 5.2 LADDER

Tiered evaluation with progressive levels.

| Outcome | Meaning |
|---------|---------|
| `GOLD` | Top tier |
| `SILVER` | Middle tier |
| `BRONZE` | Entry tier |
| `NOT_EVALUATED` | Cannot evaluate (blocked) |

---

## 6. Badge Levels

The complete set of possible badge evaluation outcomes:

| Level | Badge Type | Meaning |
|-------|------------|---------|
| `PASS` | GATEKEEPER | Met threshold |
| `NOT_MET` | GATEKEEPER | Below threshold |
| `GOLD` | LADDER | Top tier |
| `SILVER` | LADDER | Middle tier |
| `BRONZE` | LADDER | Entry tier |
| `NOT_EVALUATED` | All | Cannot evaluate |

---

## 7. Readiness Status

Readiness is a **current-state computation** (not a permanent achievement) that answers: "Based on current evaluation, is this pillar ready?"

### 7.1 Pillar Readiness

Each pillar has an independent readiness status:

| Status | Condition | Meaning |
|--------|-----------|---------|
| `FLIGHT_READY` | All evaluable required rulepacks meet baseline | All systems go |
| `PRE_FLIGHT` | Some evaluable required rulepacks below baseline, or partial blocking | Preparing for launch |
| `TELEMETRY_MISSING` | ALL required rulepacks blocked (truly blind) | Instruments offline |

### 7.2 Readiness Computation

"""
For each PILLAR:
|-- Get required rulepacks for this pillar
|-- Partition required rulepacks into:
|   |-- evaluable: badge produced (badgeLevel != NOT_EVALUATED)
|   |-- blocked: badge = NOT_EVALUATED (all affectsBadge checks are BLOCKED)
|
|-- ALL required rulepacks BLOCKED? --> TELEMETRY_MISSING (truly blind)
|-- ALL evaluable required baselines met? --> FLIGHT_READY
|-- else --> PRE_FLIGHT

For SERVICE (overall):
|-- ALL pillars TELEMETRY_MISSING? --> TELEMETRY_MISSING (truly blind)
|-- ALL pillars FLIGHT_READY? --> FLIGHT_READY
|-- else --> PRE_FLIGHT
"""

**Signal Coverage**: `signalCoveragePct = evaluableRequired / totalRequired * 100`.
A pillar with 5 required rulepacks where 4 produce badges and 1 is blocked is PRE_FLIGHT (not TELEMETRY_MISSING) with 80% signal coverage.

### 7.2.1 ServiceState (Current Truth)

ServiceState is a single row per service, upserted at the end of every scan. It stores the current readiness, pillar states, signal coverage, and a pointer to the latest Scan for full snapshot access.

Aggregators read from ServiceState (not DISTINCT ON Scan). A freshness guard of 3 days ensures stale services are excluded from active counts.

### 7.2.2 Recommendation Volume Threshold

At org and team levels, UNBLOCK_TELEMETRY recommendations are only surfaced when >= 30% of the evaluated fleet is affected. This prevents isolated noise from cluttering coaching views. Per-service Next Best Upgrade (Flight Deck) always shows the honest, unfiltered truth.

### 7.3 Go/No-Go Pending

A pillar is "Go/No-Go pending" when exactly 1–2 required rulepacks remain to reach FLIGHT_READY. This highlights services that are close to completion.

---

## 8. Blocking Status

When a check cannot be evaluated, it has a blocking status that determines its impact:

| Status | Error Kind | Effect on Readiness | Recommendation |
|--------|------------|---------------------|----------------|
| `EVALUABLE` | None | Normal evaluation | Normal upgrades |
| `BLOCKED` | `COLLECTOR_FAILED` | Blocks readiness | "Connect telemetry" |
| `NEUTRAL` | `NOT_APPLICABLE` | Excluded entirely | None (not relevant) |

---

## 9. Momentum (Trajectory)

Momentum are trajectory feedback showing improvement momentum over time. They are **scan-cadence independent** and computed at view-time from badge states.

### 9.1 Hybrid Earning Model

| Event | Momentum Earned |
|-------|---------------------|
| Core first earned | +100 (one-time burst) |
| Circuit nominal per day | +10/day |

**Key Design Principles:**
- **Core Burst:** Awarded once when a Core is first earned, not repeatedly per scan
- **Circuit Daily:** Accrues based on calendar days, not scan frequency
- **Scan-Cadence Independence:** Same operational state produces same Momentum regardless of scan frequency

### 9.2 Trajectory Display

| Field | Meaning |
|-------|---------|
| `dailyRate` | Current momentum/day from nominal Circuits |
| `delta` | Sum of daily momentum over the window |
| `trend` | Direction indicator: up/down/flat |

**Trend Calculation:**
- `up`: Recent average > 10% higher than earlier average
- `down`: Recent average < 90% of earlier average  
- `flat`: Within 10% threshold

### 9.3 Guardrails

- No org-wide Momentum leaderboards
- No "top teams by Momentum"
- Momentum are computed at view-time, not stored
- Momentum must remain secondary to readiness and upgrades

### 9.4 Vitality vs Momentum (Conceptual Separation)

| Concept | Purpose | Timeframe | Key Question |
|---------|---------|-----------|--------------|
| **Vitality (Circuits)** | Operational health | NOW (current state) | "Is it healthy right now?" |
| **Momentum** | Trajectory feedback | Over time (7d/30d/90d) | "How is it trending?" |

**Critical distinction:**
- **Vitality** answers: "What is the current operational state?" (NOMINAL/DEGRADING/DOWN)
- **Momentum** answers: "What is the trajectory over time?" (up/down/flat trend)

These are separate dimensions:
- Vitality has **state** (current health), not trend
- Momentum has **trend** (direction over time)

Vitality does not have a trend — it has a current state derived from CircuitState. For trajectory information, use Momentum.

---

## 10. Upgrade Recommendation Types

When recommending next actions, Ground Control prioritizes in this order:

| Priority | Type | Condition | Multiplier |
|----------|------|-----------|------------|
| 1 | `PROTECT_VITALITY` | Any Circuit is DEGRADING or DOWN | 2.2x + 0.1x damper on others |
| 2 | `UNBLOCK_TELEMETRY` | Readiness is TELEMETRY_MISSING | 10x |
| 3 | `REACH_BASELINE` | All Circuits NOMINAL, baseline not met | 2.5x (closest win) |
| 4 | `REDUCE_FRICTION` | High-impact/low-effort improvements | 1.0x |
| 5 | `OPTIMIZE` | Remaining improvements | 1.0x |

**"Save the Baby" Philosophy:** When any Circuit is DEGRADING or DOWN, all non-vitality candidates receive a 0.1x penalty (vitality damper). This ensures PROTECT_VITALITY always wins regardless of other multipliers. Rationale: when the baby is dying, you save the baby before connecting more instruments.

**CIRCUIT Exclusion from Baseline:** CIRCUIT achievement types are handled exclusively by PROTECT_VITALITY, never REACH_BASELINE. CIRCUITs within the grace period (NOMINAL but recently failing) appear as REDUCE_FRICTION or OPTIMIZE candidates.

**Grace Period:** A CIRCUIT with `lastPassedAt` set but within the `degradingAfterHours` threshold is NOMINAL -- the damper does not fire. Only genuinely DEGRADING or DOWN Circuits trigger the damper.

**Fleet-Level Type Diversity:** The priority order above applies at all altitudes. At fleet level (Mission Map and Squad Console), no single type may consume the entire recommendation list. A per-type slot cap guarantees that each type with candidates receives fair representation in the coaching summary. This does not weaken the "Save the Baby" philosophy: PROTECT_VITALITY still appears first and retains the highest priority — it simply cannot crowd out all other types when the audience needs to see the full issue landscape. Per-service (Flight Deck) remains strict priority with no diversity cap.

---

## 11. Risk Levels

Services are assessed for operational risk:

| Level | Condition | Meaning |
|-------|-----------|---------|
| `NOMINAL` | Default healthy state | All good |
| `WATCHLIST` | Some Circuits down OR few baselines not met | Needs attention |
| `HOLD` | Critical telemetry missing OR multiple required rulepacks failing | Requires immediate action |

---

## 12. The Four Views

Ground Control provides four views:

| View | Altitude | Purpose |
|------|----------|---------|
| **Mission Map** | Organization | Fleet overview, pillar readiness, top movers |
| **Squad Console** | Team | Coordinate upgrades across services |
| **Flight Deck** | Service | Coaching cockpit with evidence |
| **Flight Manual** | Overlay | Read-only, centered HUD overlay that surfaces the system's evaluation standards, signal sources, and vocabulary. Not a new route — it overlays the current view via URL params (?manual=). Provides transparency into "What is measured, How, and Why" for every protocol. |

---

## 13. Truth vs Meaning Separation

Ground Control maintains strict separation between observed facts and their interpretation:

| Layer | What | When Computed | Stored? |
|-------|------|---------------|---------|
| **Check** | observedValue, displayValue | Scan-time | Yes |
| **Threshold** | status (PASS/NOT_MET), tier | Scan-time (enricher) | Yes |
| **Config** | implication, remediation, tags | View-time | No (from YAML) |
| **CircuitState** | NOMINAL/DEGRADING/DOWN | View-time | No (derived from timestamp) |
| **FlightHours** | daily rate, delta, trend | View-time | No (derived from badge states) |

**Why?** Engineers always see current guidance. If YAML improves, all historical scans benefit immediately. Momentum computed at view-time ensures scan-cadence independence.

---

## 14. Complete State Matrix

### Achievement × Badge Type Combinations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ACHIEVEMENT TYPE: CORE                               │
│                     (from check.windowType = STATIC)                        │
├──────────────────┬──────────────────┬──────────────────────────────────────┤
│ Badge Type       │ Badge Level      │ Core State (UI)                      │
├──────────────────┼──────────────────┼──────────────────────────────────────┤
│ GATEKEEPER       │ PASS             │ earned                               │
│ GATEKEEPER       │ NOT_MET          │ not_earned                           │
│ GATEKEEPER       │ NOT_EVALUATED    │ telemetry_missing                    │
├──────────────────┼──────────────────┼──────────────────────────────────────┤
│ LADDER           │ GOLD             │ earned (if baseline ≤ GOLD)          │
│ LADDER           │ SILVER           │ earned (if baseline ≤ SILVER)        │
│ LADDER           │ BRONZE           │ earned (if baseline ≤ BRONZE)        │
│ LADDER           │ NOT_EVALUATED    │ telemetry_missing                    │
└──────────────────┴──────────────────┴──────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        ACHIEVEMENT TYPE: CIRCUIT                            │
│                    (from check.windowType = ROLLING)                        │
├──────────────────┬──────────────────┬──────────────────────────────────────┤
│ Badge Type       │ Badge Level      │ Circuit State (UI, time-derived)     │
├──────────────────┼──────────────────┼──────────────────────────────────────┤
│ GATEKEEPER       │ PASS             │ nominal (baselineMet=true)           │
│ GATEKEEPER       │ NOT_MET          │ degrading → down (based on time)     │
│ GATEKEEPER       │ NOT_EVALUATED    │ (not shown as Circuit)               │
├──────────────────┼──────────────────┼──────────────────────────────────────┤
│ LADDER           │ GOLD             │ nominal (if baseline ≤ GOLD)         │
│ LADDER           │ SILVER           │ varies by baseline                   │
│ LADDER           │ BRONZE           │ varies by baseline                   │
│ LADDER           │ NOT_EVALUATED    │ (not shown as Circuit)               │
└──────────────────┴──────────────────┴──────────────────────────────────────┘
```

### Celebration Tier (Derived State)

Celebration tier is a **view-time derivation** combining Circuit health (Vitality), Readiness state (Foundation), and Core mastery. It determines the tone and content of celebration UI across all views, aligned with the Charter v4.0 Tone Model.

| Tier | Condition | Charter Tone | Theme Accent |
|------|-----------|--------------|--------------|
| *(none)* | Any Circuit DEGRADING or DOWN | Tier A — Mission Control | None. Pure coaching. |
| `crew_encouragement` | All Circuits NOMINAL, but not FLIGHT_READY | Tier B — Crew Encouragement | None. Warm acknowledgment + redirect. |
| `earned_celebration` | All Circuits NOMINAL AND FLIGHT_READY | Tier C — Celebration (Earned Only) | Bowie accent permitted. Rare, warm, brief. |
| `full_mastery` | All Circuits NOMINAL AND FLIGHT_READY AND all Cores earned | Tier C — Celebration (Peak) | Bowie accent permitted. Rarest, warmest. |

Both `earned_celebration` and `full_mastery` are Tier C (Bowie accent). `full_mastery` is the peak celebration — it fires only when a service (or team/org) has met all baselines AND earned every available Core. With `baseline: NONE` support, a service can be FLIGHT_READY without earning all Cores, which is why these are distinct states.

**Derivation (altitude-agnostic):**

"""
if any Circuit is DEGRADING or DOWN             → no celebration
if all Circuits NOMINAL but not FLIGHT_READY     → crew_encouragement
if all Circuits NOMINAL AND FLIGHT_READY         → earned_celebration
if all Circuits NOMINAL AND FLIGHT_READY AND all Cores earned → full_mastery
"""

**Key properties:**
- Computed at view-time, not stored
- Altitude-agnostic logic: same function at service, team, and org level
- At team/org level, "all Circuits NOMINAL" means all services have all Circuits nominal; "FLIGHT_READY" means all services are Flight Ready; "all Cores earned" means every service has all evaluable Cores earned

---

## 15. Key Relationships

```
                                 ┌─────────────┐
                                 │   PILLAR    │
                                 │ (Strategy)  │
                                 │             │
                                 │ • readiness │
                                 │ • baseline  │
                                 └──────┬──────┘
                                        │ 1:N
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
             ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
             │  RULEPACK   │     │  RULEPACK   │     │  RULEPACK   │
             │  (Tactics)  │     │  (Tactics)  │     │  (Tactics)  │
             │             │     │             │     │             │
             │ • badgeType │     │ • badgeType │     │ • badgeType │
             │ • threshold │     │ • threshold │     │ • threshold │
             └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
                    │ 1:N              │                    │
                    ▼                  ▼                    ▼
             ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
             │    RULE     │     │    RULE     │     │    RULE     │
             │  (Action)   │     │  (Action)   │     │  (Action)   │
             │             │     │             │     │             │
             │ • coaching  │     │ • coaching  │     │ • coaching  │
             │ • threshold │     │ • threshold │     │ • threshold │
             └──────┬──────┘     └─────────────┘     └─────────────┘
                    │ 1:1
                    ▼
             ┌─────────────┐          ┌─────────────────────┐
             │   CHECK     │◀─────────│ windowType: STATIC  │→ CORE
             │   (Unit)    │          │ windowType: ROLLING │→ CIRCUIT
             │             │          └─────────────────────┘
             │ • measure   │
             │ • NO judge  │
             └──────┬──────┘
                    │ reads
                    ▼
             ┌─────────────┐
             │   SIGNAL    │
             │  (Evidence) │
             └─────────────┘
```

- **Flight Manual** is a read-only lens over the Pillar > Rulepack > Rule hierarchy. It renders evaluation metadata resolved at bootstrap by ProtocolMetadataResolver, not runtime state.

---

## 16. Glossary

| Term | Definition |
|------|------------|
| **Pillar** | High-level strategic outcome (e.g., "Production Health") |
| **Rulepack** | A capability a service can achieve, belonging to one pillar |
| **Rule** | User-facing unit with coaching (implication, remediation) |
| **Check** | Reusable measurement unit that returns facts, not judgments |
| **Signal** | Raw data from external systems (Git, K8s, Datadog, CI) |
| **Core** | Permanent achievement from STATIC check |
| **Circuit** | Temporary health indicator from ROLLING check |
| **Readiness** | Current pillar status (FLIGHT_READY / PRE_FLIGHT / TELEMETRY_MISSING) |
| **Momentum** | Trajectory feedback computed from badge states (not a ranking metric) |
| **Baseline** | Minimum badge level required for a rulepack to contribute to readiness |
| **Badge** | Evaluation result for a rulepack (GATEKEEPER or LADDER type) |
| **Badge Level** | Specific outcome (PASS, NOT_MET, GOLD, SILVER, BRONZE, NOT_EVALUATED) |
| **Closest Win** | A service that is exactly 1 required rulepack away from FLIGHT_READY globally. Determined at scan-time and stored as `isClosestWin` on ServiceState. Scoped views (spotlights, teams) filter closest wins to their context but never redefine the term. |
| **Celebration Tier** | Derived display state combining Circuit health + Readiness + Core mastery (crew_encouragement / earned_celebration / full_mastery) |
| **Flight Manual** | The system's standards transparency layer. A read-only HUD overlay accessible from any view that surfaces protocol definitions (ambition, standard, evaluation physics, and Git handshake), organized by pillar > rulepack > rule hierarchy. URL-driven via ?manual= param. |

---

## 17. Architectural Principles

1. **Open-Closed Principle (OCP):** The Engine is immutable. Extensions (Checks, Collectors, Rulepacks) are added as plugins.

2. **Truth vs Meaning Separation:** Checks measure. Config judges. These are separate concerns.

3. **Signal Integrity:** We distinguish between Service Quality (NOT_MET) and Signal Quality (BLOCKED).

4. **Scan-Cadence Independence:** The same real-world situation must produce the same result regardless of scan frequency.

5. **Fair Scoring:** Missing telemetry never penalizes scores. BLOCKED is score-neutral.

6. **Evidence Lineage:** Every status must be explainable and traceable to source signals.

---

## 18. The Complete Picture

> **Ground Control** transforms abstract engineering goals (Pillars) into specific capabilities (Rulepacks) that are evaluated through deterministic checks returning measured facts (Signals).
>
> Permanent achievements (Cores) and ongoing health indicators (Circuits) guide teams toward readiness (Flight Ready) through evidence-backed, prioritized upgrade recommendations.
>
> Momentum track trajectory without ranking, ensuring Ground Control remains a coaching tool that celebrates progress over punishment.

---

*This document is the immutable foundation of Ground Control. All implementation must conform to these definitions.*