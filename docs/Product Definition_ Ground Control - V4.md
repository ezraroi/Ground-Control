# Product Definition: Ground Control — V4

> **Foundation Reference:** This document builds on the [Ground Control Constitution](./Ground_Control_Constitution.md), which defines immutable core concepts, terminology, and relationships. For definitions of Pillars, Rulepacks, Rules, Checks, Signals, Cores, Circuits, Readiness, and Momentum, consult the Constitution.

---

## 1. Product Vision

**Ground Control** is an internal Mission Control for engineering excellence. It turns abstract goals (e.g., Operational Excellence, Production Health) into a **trusted coaching cockpit**: clear standards, verified evidence, and guided upgrade paths—so teams move from reactive patching to proactive mastery.

It is intentionally designed to feel like:

- **Coaching**, not surveillance
- **Progress**, not punishment
- **Evidence-backed**, not opinion-based
- **Autonomy-preserving**, not centrally dictated behavior

Ground Control motivates through:

- **Fair measurement** (deterministic, window-based truth)
- **Purposeful guidance** (why it matters + how to fix)
- **Recognition** (celebrating foundation and vitality without encouraging ranking culture)

---

## 2. Cultural Intent and Behavioral North Star

Ground Control is not only a measurement system. It is a **cultural instrument** designed to reinforce a specific engineering posture across AppsFlyer:

- **Engineers are allowed to be great.**  
  Excellence is not perfectionism or ego; it is responsibility, care, and craftsmanship.

- **Engineers are allowed to slow down to understand.**  
  Ground Control should normalize taking time to think, verify, and ask questions—especially when systems feel complex or ambiguous.

- **Excellence is a habit built from small upgrades.**  
  The product must make "the next best improvement" visible and achievable, so teams build compounding quality over time.

- **Readiness is a shared language, not a weapon.**  
  "Flight Ready / Pre-flight / Telemetry Missing" are coaching states that guide action. They must never become labels used for blame, performance evaluation, or surveillance.
  - `TELEMETRY_MISSING` means "truly blind" — ALL required rulepacks for a service are blocked. A single missing signal does not invalidate what we can already see.
  - `signalCoveragePct` expresses how much of the required surface is evaluable (0-100%). Even partial coverage produces actionable coaching.
  - At org/team levels, telemetry recommendations are only surfaced when >= 30% of the fleet is affected — isolated noise does not clutter coaching views.

### Product Implications (Non-Negotiable)

To preserve this intent, Ground Control must:

1. Default to **coaching and action** (Suggested Upgrades, closest wins, impact/effort) rather than reporting.
2. Always provide **purpose at the moment of action** ("why it matters" + "how to fix") and **evidence lineage** (signals, source, time, contract version).
3. Reinforce **trajectory through Momentum** (daily rate from active Circuits, sparkline, trend indicator) without introducing competitive ranking dynamics. Note: Vitality (Circuits) shows current state; Momentum shows trajectory over time.
4. Remain **engineer-first** in tone and density: calm, factual, respectful, and optimized for daily decision-making.

---

## 3. Product Posture & Non-Negotiable Guardrails

These are **hard constraints** that shape every UX and every future feature decision.

### A) Trust & Anti-Surveillance

- Ground Control must **never** feel like a compliance audit tool.
- Every status must be **explainable** and **clickable to evidence lineage** (signals + source + timestamps).
- "Red" must translate to **actionable truth** (what failed, why it matters, how to remediate), not shame.

### B) Progress Over Percentile

- Default experiences emphasize **trajectory and near-wins**, not "best vs worst."
- If any comparison exists, it must be **explicitly user-initiated**, scoped, and framed as learning ("how others achieved this"), not ranking.

### C) Momentum Guardrails

Momentum exist as persistent growth feedback, not as a public competitive currency.

- No org-wide Momentum leaderboards.
- No "top teams/services by Momentum."
- No negative Momentum. Circuits degrading is the negative feedback loop.
- Momentum must remain **secondary to mastery and upgrades**.
- **API exposure of Momentum at team/org level is for trajectory visualization only. Raw totals must not be extracted for external ranking dashboards.**

### D) Language & Tone

- Avoid fantasy/RPG lore.
- Avoid "quests" as a primary framing.
- Preferred tone: **Mission Control** and **engineering clarity**:
  - "Suggested Upgrades", "Pre-flight Checklist", "Flight Plan", "Evidence", "Baselines", "Readiness".

### D.1) Celebration Model (Four-State, Charter-Aligned)

Celebration is a **derived display state** that combines Circuit health (Vitality, NOW), Readiness (Foundation, PERMANENT), and Core mastery. It determines the tone, content, and theme usage of celebration UI across all views.

**States:**

| State | Condition | Tone Tier | Theme |
|-------|-----------|-----------|-------|
| No celebration | Any Circuit DEGRADING or DOWN | Tier A — Mission Control | No theme. Pure coaching mode. |
| `crew_encouragement` | All Circuits NOMINAL, not FLIGHT_READY | Tier B — Crew Encouragement | No Bowie. Warm acknowledgment + redirect to next goal. |
| `earned_celebration` | All Circuits NOMINAL AND FLIGHT_READY | Tier C — Celebration (Earned Only) | David Bowie accent: "Ground Control to Major Tom: ..." |
| `full_mastery` | All Circuits NOMINAL AND FLIGHT_READY AND all Cores earned | Tier C — Celebration (Peak) | David Bowie accent. Rarest, warmest celebration. |

Both `earned_celebration` and `full_mastery` are Tier C with Bowie accent. `full_mastery` is the peak state — it fires only when every baseline is met AND every available Core has been earned. With `baseline: NONE` support, a service can be FLIGHT_READY without earning all Cores, which is why these are distinct states.

**Rules:**

1. **Tier C is the only state that permits the Bowie/space theme accent.** Delight is earned and rare.
2. **Tier B acknowledges operational health but redirects.** The team is keeping the baby alive — that deserves a warm nod, but the mission isn't complete.
3. **State 1 (no celebration) shows nothing celebratory.** When vitality is degrading, the system is in coaching mode. No theme, no warmth, just truth + action.
4. **Text is backend-owned.** The frontend renders celebration text provided by the API; no hardcoded copy or fallback values in the UI.
5. **Logic is altitude-agnostic.** The same derivation function works at service, team, and org level. Only the display text varies by altitude.
6. **`full_mastery` gets elevated visual treatment.** While both Tier C states share the Bowie accent and Sparkles icon, `full_mastery` adds the `.core-light-up` glow animation to mark the peak achievement.

**Copy guidelines by altitude:**

| Altitude | Tier B (crew_encouragement) | Tier C (earned_celebration) | Tier C peak (full_mastery) |
|----------|----------------------------|----------------------------|----------------------------|
| Service (Flight Deck) | Acknowledge circuits, redirect to remaining cores | "Ground Control to Major Tom" — all systems verified | "Ground Control to Major Tom" — commencing countdown, engines on |
| Team (Squad Console) | Acknowledge team vitality, redirect to pre-flight services | "Ground Control to Major Tom" — full orbit | "Ground Control to Major Tom" — the stars look very different today |
| Org (Mission Map) | Acknowledge fleet vitality, redirect to readiness % | "Ground Control to Major Tom" — the fleet is flying | "Ground Control to Major Tom" — planet Earth is blue |

### D.2) Console Boot Sequence (The Engineering Channel)

Ground Control's primary audience is engineers. The kind of engineer who opens browser DevTools is not debugging — they are exploring. They are demonstrating exactly the posture this product wants to reinforce: *"Engineers are allowed to slow down to understand."*

The Console Boot Sequence rewards that curiosity. When the app loads, a styled message block prints to the browser console — a Mission Control identity stamp, a moment of recognition, and a direct link to the source.

**Purpose:** Speak to the person, not the service. This is the one place Ground Control acknowledges the human behind the screen. It says: *we built this for people like you.*

**Content (three layers):**

1. **Identity** — A Mission Control header announcing the system (`GROUND CONTROL // SYSTEM CODEX v4.0`)
2. **Detection** — A simulated boot sequence that builds to `ENGINEER DETECTED` — the fourth-wall break. The system "sees" the curious engineer and names them.
3. **The Payload** — A rotating David Bowie lyric (from *Space Oddity*), and a direct link to the Ground Control GitLab repository.

**Bowie rotation** (one random per session):

- "Commencing countdown, engines on..."
- "The stars look very different today."
- "Planet Earth is blue, and there's nothing I can do."
- "Check ignition and may God's love be with you."
- "Can you hear me, Major Tom?"

**Relationship to the Celebration Model (D.1):**

The Celebration Tier model governs the **product UI**. It gates Bowie accents on system state — only services (or teams, or the org) that have earned Flight Ready status see Tier C celebration. That constraint exists to protect trust on the product's decision-making surfaces.

The console is not a product surface. It is a hidden channel, discovered only through intentional curiosity. The Bowie lyric here does not celebrate the system's state — it rewards the **engineer's behavior**. This distinction is deliberate: the UI remains pure coaching, evidence, and truth. The console is the one place we allow a human wink, unconditionally.

**Constraints:**

- Fires once per session reload (module-level guard prevents noise)
- No domain state — the console does not reflect readiness, vitality, or any evaluation data
- No backend dependency — static creative text, frontend-only
- Not subject to the Vocabulary Contract (Architectural Principle 7) — this is not domain terminology that evolves with rulepacks or checks; it is a fixed creative piece, closer in nature to the product name "Ground Control" than to a coaching label

**Implementation:** `apps/web/src/lib/console-boot.ts` — self-contained module, zero project dependencies. Called once from the root `App` component on mount.

### E) Internal Tool, No Auth (for now)

- Ground Control will remain **no-auth** for the foreseeable future.
- It is an internal tool; identity and access control are handled by the surrounding ecosystem (Backstage, GitLab, observability tools, network boundaries).

### F) Configuration as Code (YAML)

- Rulepacks remain **YAML-defined** and curated centrally.
- No in-product rule builder / drafts UX in the current roadmap.
- All teams start with the **same pillars and standards**; future evolution may add scoped overlays (team/pillar variants) without changing the authoring model.
- The **Flight Manual** provides the human-readable lens on YAML configuration (protocol definitions, evaluation physics, vocabulary).

### G) Truth vs. Configuration Separation

- **Snapshots store truth, not guidance.** EvaluationSnapshots contain only what was observed at scan time (status, observedValue, displayValue, blockingStatus, evidence).
- **Configuration is looked up at view-time.** Narrative fields (implication, remediation, impact, effort, tags, name, missingSignalText) are resolved from YAML at render time.
- **Engineers always see current guidance.** If YAML config is improved (better remediation text, updated impact scores), historical scans immediately benefit—no data migration needed.
- **Identity keys enable lookup.** Each check result includes ruleId, checkId, pillarId, rulepackId—sufficient for O(1) config lookup at view-time.

---

## 4. Hierarchy Implementation Notes

> **See Constitution Section 3** for the canonical hierarchy definition (Pillar → Rulepack → Rule → Check → Signal).

### Pillar Configuration

Each pillar must have:

- A clear **description** (purpose at altitude)
- A **mastery definition** (what "complete" means)
- `requiredRulepacks` list

### Rulepack Configuration

Each rulepack must include:

- `pillarId` — exactly one pillar ownership
- `badgeType` — GATEKEEPER or LADDER
- `description` — why the rulepack exists
- `baseline` — required baseline for Flight Ready status:
  - **GATEKEEPER:** `PASS` (required for readiness) or `NONE` (informational)
  - **LADDER:** `BRONZE` / `SILVER` / `GOLD` (required at tier) or `NONE` (informational)
- `applicableWhen` (optional) — gates for non-applicable services
- Urgency profile for Circuits (if windowType is ROLLING):
  - `degradingAfterHours` and `downAfterHours` thresholds

**Note:** Achievement type (Core vs Circuit) is **derived from the check's `windowType`** (STATIC → Core, ROLLING → Circuit). Do not declare it explicitly on the rulepack.

### Config Validation

Validation is layered: invalid states are rejected as early as possible.

**Schema-level (Zod discriminated union on `badge.type`):**

The badge config uses a discriminated union — invalid badge-type/baseline/threshold combos are structurally unrepresentable and rejected at YAML parse time:

- **GATEKEEPER:** `baseline` can only be `PASS` or `NONE`. No `thresholds` field.
- **LADDER:** `baseline` can only be `NONE`, `BRONZE`, `SILVER`, or `GOLD`. `thresholds` (`{ gold: number, silver: number, bronze: number }`) is **required**.

**Semantic-level (cross-field checks at startup):**

- **Empty gate prevention:** GATEKEEPER rulepacks with `baseline != NONE` must have at least one blocking rule (prevents badges that can never be achieved)
- **GATEKEEPER threshold requirement:** All GATEKEEPER rules must have `threshold` defined
- **Blocking field validity:** The `blocking` field is only valid on GATEKEEPER rules (LADDER rules do not support blocking semantics)
- **LADDER params prohibition:** LADDER rules must not have `params.thresholds` (thresholds belong at badge level)
- **Unique identifiers:** Rulepack IDs must be unique across all rulepacks; rule IDs must be globally unique
- **Valid pillar references:** All rulepacks must reference valid pillar IDs from the pillar configuration

### Rule Configuration

Each rule must carry purpose and coaching:

- `ruleId` — unique identifier
- `checkId` — reference to the evaluator
- `implication` — why the current state is harmful (risk/cost)
- `remediation` — how to fix in practical terms
- `impact` — HIGH / MEDIUM / LOW
- `effort` — SMALL / MODERATE / SIGNIFICANT
- `missingSignalText` — what's needed when telemetry is absent
- `tags` — 1-2 outcome tags (max 3)
- `blocking` — (optional, default: true) whether this rule blocks badge evaluation

**Blocking behavior:**

| `blocking` | Effect |
|------------|--------|
| `true` (default) | Rule must pass for GATEKEEPER badge to PASS |
| `false` | Rule is evaluated and shown, but doesn't affect badge outcome |

Use `blocking: false` for:
- Optional recommendations (e.g., HPA for services with constant traffic)
- Aspirational org-wide standards being rolled out gradually
- Rules that provide value as guidance but shouldn't block readiness

Example:

```yaml
- ruleId: canary_deployment_enabled
  checkId: gitops_canary_deployment
  params:
    contextKey: gitops
  implication: |
    All-or-nothing deployments expose 100% of traffic to risk...
  remediation: |
    Replace the Deployment manifest with an Argo Rollout...
  impact: HIGH
  effort: MODERATE
  missingSignalText: Connect GitOps manifests to analyze deployment configuration.
  tags: [safer-deploys]

# Non-blocking rule example
- ruleId: hpa_configured
  checkId: gitops_hpa_configured
  blocking: false  # Optional - doesn't block badge
  implication: |
    Without autoscaling, capacity decisions are manual...
  remediation: |
    Add an HPA targeting the Deployment/Rollout...
  impact: MEDIUM
  effort: SIGNIFICANT
  tags: [cost-efficiency]
```

---

## 4.5 Badge Evaluation Model (Unified 4-Step)

Ground Control uses a unified evaluation model for both badge types (GATEKEEPER and LADDER). This model ensures consistent behavior and clear separation of concerns.

### The Four Steps

| Step | Name | Purpose | Shared/Unique |
|------|------|---------|---------------|
| 1 | **Collection** | Can we get data? | Shared |
| 2 | **Truth** | What's the raw value? | Shared |
| 3 | **Meaning** | Does it meet threshold? | Unique per badge type |
| 4 | **Aggregation** | Combine rule outcomes | Unique per badge type |

### Step 1: Collection (Blocking Status)

Before evaluation, each rule check is classified by its blocking status:

| Status | Meaning | Rulepack Effect |
|--------|---------|-----------------|
| `EVALUABLE` | Check ran successfully (PASS/NOT_MET/WARN) | Included in evaluation |
| `BLOCKED` | Collector failed (telemetry missing) | Badge = NOT_EVALUATED |
| `NEUTRAL` | Not applicable to this service | Excluded from evaluation |

**Key principle:** If ANY check is BLOCKED, the entire rulepack returns `NOT_EVALUATED`. This prevents partial evaluations from producing misleading badge levels.

### Step 2: Truth (Observed Value)

Each check measures and returns a raw fact (observedValue). The check itself does NOT apply meaning or judgment. It simply reports what was observed.

Example: A runtime stability check might return `{ value: 3, totalCrashes: 3, window: "24h" }`.

### Step 3: Meaning (Per-Rule Evaluation)

This step applies thresholds to determine the outcome for each individual rule.

| Badge Type | Per-Rule Outcome | Threshold Source |
|------------|------------------|------------------|
| **GATEKEEPER** | PASS or NOT_MET | Rule's `params.threshold` |
| **LADDER** | GOLD, SILVER, BRONZE, or NOT_MET | Badge's `thresholds` config |

For LADDER badges, the tiered thresholds determine the per-rule tier:
- value ≤ gold threshold → GOLD
- value ≤ silver threshold → SILVER
- value ≤ bronze threshold → BRONZE
- value > bronze threshold → NOT_MET

**Important:** NOT_MET is a distinct outcome, not mapped to BRONZE. A failing rule explicitly indicates the standard was not met.

### Step 4: Aggregation (Combine Results)

This step combines all rule outcomes into a final badge level.

| Badge Type | Aggregation Logic | Badge Level |
|------------|-------------------|-------------|
| **GATEKEEPER** | AND (all blocking rules must pass) | PASS or NOT_MET |
| **LADDER** | MIN (lowest tier wins) | GOLD, SILVER, BRONZE, or NOT_MET |

**GATEKEEPER AND semantics:** All blocking rules must pass for the badge to be PASS. Non-blocking rules are evaluated and displayed but don't affect the badge outcome.

**LADDER MIN semantics:** The final badge level equals the lowest tier achieved across all rules. This ensures all rules must meet the standard for the badge level to be awarded.

### Badge Levels Summary

| Badge Type | Possible Levels |
|------------|-----------------|
| **GATEKEEPER** | PASS, NOT_MET, NOT_EVALUATED |
| **LADDER** | GOLD, SILVER, BRONZE, NOT_MET, NOT_EVALUATED |

### Baseline Evaluation

Each rulepack's badge configuration defines a `baseline` requirement (not the pillar). The `baselineMet` flag indicates whether the achieved badge level meets or exceeds the required baseline.

| Badge Type | Baseline Value | Meaning | Baseline Met When |
|------------|----------------|---------|-------------------|
| **GATEKEEPER** | `PASS` | Binary gate, required for Flight Ready | badgeLevel = PASS |
| **GATEKEEPER** | `NONE` | Informational only | Always true (does not block readiness) |
| **LADDER** | `BRONZE` / `SILVER` / `GOLD` | Minimum tier required for Flight Ready | achievedTier >= requiredBaseline (tier rank comparison) |
| **LADDER** | `NONE` | Informational only | Always true (does not block readiness) |

Tier ranking: GOLD (3) > SILVER (2) > BRONZE (1) > NOT_MET (0)

### Multi-Rule Support

Both badge types support multiple rules per rulepack:
- **GATEKEEPER:** All blocking rules must pass (checklist model)
- **LADDER:** All rules contribute to the MIN aggregation (each rule's value is mapped to a tier)

---

## 5. Circuit Urgency Profiles

Different Circuits can have different urgency levels based on operational criticality:

| Circuit Type | degradingAfterHours | downAfterHours | Rationale |
|--------------|---------------------|----------------|-----------|
| Runtime stability (crashes) | 6 | 24 | High urgency — user impact |
| CI stability (flaky tests) | 24 | 72 | Moderate urgency — dev friction |

This ensures scan-cadence independence: the same problem duration produces the same CircuitState regardless of how often we scan.

---

## 6. Momentum Implementation

> **See Constitution Section 9** for core Momentum definitions and earning model.

### Trajectory Window Options

Users can select how far back to view Momentum trends:

| Window | Granularity | Data Points | Use Case |
|--------|-------------|-------------|----------|
| **7d** | Daily | 7 | Recent week trend |
| **30d** | Daily | 30 | Monthly pattern (default) |
| **90d** | Weekly aggregation | ~12 | Quarterly trend |

Today's (incomplete) data is always excluded from trajectory calculations.

### Trajectory Interpretation

- **dailyRate**: Current momentum/day from active Circuits
- **delta**: Sum of daily momentum over the selected window
- **trend**: Direction indicator (up/down/flat) based on delta comparison

### Scoped Momentum Display

Momentum meaning changes by altitude. All altitudes share the same visible shape: **direction word + timeframe arrow**. Altitude-specific detail is available via tooltip (P-M3 inspectability).

| View | Visible Display | Tooltip Detail |
|------|--------|---------|
| **Service** | Direction word (`Gaining · 7d ↑`) | Circuit breakdown + delta |
| **Team** | Direction word (`Gaining · 7d ↑`) | Normalized /svc delta + service count |
| **Org** | Direction word (`Gaining · 7d ↑`) | Fleet percentage + service counts |

### Momentum Display Principles

These principles govern how Momentum is presented across all views. They derive from the Constitution (Section 9) and Charter (P8, Principle 10) and must be preserved in all future feature work.

| Principle | Rule |
|-----------|------|
| **P-M1 Always Secondary** | Momentum is visually subordinate to Readiness, Vitality, and Upgrades at every altitude. If you removed Momentum from any view, the view must still be complete and useful. |
| **P-M2 Non-Competitive** | No sorting, ranking, or comparison by Momentum at any level — not in the default view, not as an option. |
| **P-M3 Inspectable** | Every Momentum cue must be explainable in one interaction (hover or click), traceable to badge states (active Circuits, earned Cores). |
| **P-M4 Direction Over Magnitude** | Sparkline shape and trend arrow are the primary communication. Numbers are available via tooltip inspection (P-M3) but never the primary display. |
| **P-M5 One Question Per Altitude** | Each view's Momentum display answers exactly one question — see table below. |
| **P-M6 No Alarm State** | Declining trend uses neutral color, not warning color. Corrective action comes from Vitality and Readiness instruments, not from Momentum. |
| **P-M7 No Rank in Data Model** | The API contract must not expose ranking or score fields. Momentum totals must not be extractable for external ranking dashboards. |
| **P-M8 Shape Over Numbers** | At higher altitudes, trajectory communicates through visual shape (sparkline + direction), not numeric values. Numbers are available via inspection (P-M3) but never the primary display. The higher the altitude, the more momentum relies on shape alone. |
| **P-M9 Single Visual Source** | All momentum visual decisions (icon, sparkline sizes, trend colors) are defined in `momentum-visuals.ts`. No local overrides — same discipline as `recommendation-visuals.ts`. |

### Per-View Momentum Behavior

| View | Question Answered | Placement | Variant | Primary Cue | Sortable? |
|------|-------------------|-----------|---------|-------------|-----------|
| **Flight Deck** (Service) | "Is my work compounding?" | Rightmost instrument cluster (smallest) | `full` — sparkline + direction + timeframe | Sparkline shape + direction word | N/A |
| **Squad Console** — Huddle (Team) | "Is the team improving?" | Trailing indicator below primary metrics | `full` — sparkline + direction + timeframe | Sparkline shape + direction word | N/A |
| **Squad Console** — Table (Team) | — | Non-interactive column | `spark` — sparkline + arrow only | Visual shape only | **No** |
| **Mission Map** — Summary (Org) | "Is the culture spreading?" | Smaller trailing indicator | `visual` — sparkline + direction word | Sparkline shape + "gaining"/"steady"/"easing" | N/A |
| **Mission Map** — TeamRow (Org) | — | Compact trailing cell | `spark` — sparkline + arrow only | Visual shape only | **No** |

---

## 7. Spotlights (Org-Level Focus Overlay)

Spotlights provide an **org-level focus overlay** that biases recommender scoring without introducing a new "campaign" domain entity.

### Spotlight Rules

- A spotlight is a **nudge**, not enforcement—it reorders recommendations via a multiplier (max 2.0x), never hides baseline truth.
- Only **0 or 1** spotlight may be active at any time.
- Spotlights match candidates by pillar, rulepack, rule, or tags (matchAny semantics).

### UI Treatment

- Banner: "Spotlight: {title}"
- Badge on affected recommendations
- Explainability bullet when multiplier applied

### Spotlight Guardrails

- No lifecycle management, no assignments, no time-bound activation in v1.
- Spotlight must remain secondary to hard-priority telemetry logic (10x multiplier).
- No leaderboards or ranking tied to spotlight participation.

---

## 8. Tags Philosophy

Tags serve two purposes in Ground Control:

1. **Focus Mode Matching** — Enable user-selected focus modes to filter and boost relevant recommendations
2. **Spotlight Targeting** — Allow org-level spotlights to match recommendations by tag

### Outcome Tags (Single Dimension)

Rules should have 1–2 outcome tags (max 3) describing the benefit engineers receive:

| Tag | Benefit |
|-----|---------|
| `faster-recovery` | Reduces time-to-restore during incidents |
| `safer-deploys` | Reduces blast radius and deployment risk |
| `faster-delivery` | Speeds up the path from code to production |
| `better-dx` | Improves developer experience and reduces friction |
| `cost-efficiency` | Optimizes resource utilization |
| `change-confidence` | Increases confidence in changes landing cleanly |

### Tag Guardrails

- Every rule **SHOULD** have 1–2 outcome tags (max 3)
- Tags are **lowercase, hyphen-separated** (e.g., `faster-recovery`)
- **No technical system tags** — Derive system context from pillar/rulepack instead
- Tags must support the coaching model — they help engineers understand *why* to act

### Focus Mode Configuration

Focus modes are configured via YAML (`focus-modes.yaml`):

| Focus Mode | Primary Boost | Tag Boost |
|------------|---------------|-----------|
| **Reduce Friction** | REDUCE_FRICTION category | `better-dx`, `cost-efficiency` |
| **Ship Faster** | REDUCE_FRICTION category | `faster-delivery`, `change-confidence` |
| **Break Less** | PROTECT_VITALITY category | `faster-recovery`, `safer-deploys` |
| **Baseline Ready** | UNBLOCK_TELEMETRY, REACH_BASELINE | (no tag boost) |

---

## 9. Key User Experiences (Views)

> **See Constitution Section 12** for view definitions. Below are implementation details.

### View 1: Mission Map (Organization Overview)

**Goal:** Clarity and movement without turning into a leaderboard.

**Components:**

- Pillar readiness overview (per-pillar aggregate readiness bars)
- Telemetry coverage visibility (separate from health)
- Top movers (improvement highlights)
- Momentum indicators (trajectory, not rank)

**Guardrail:** No default ranked league table. If ranking exists, it must be user-initiated and constrained.

### View 2: Squad Console (Team View)

**Goal:** Help a team coordinate upgrades and protect vitality.

**Components:**

- **Huddle Header (Squadron Dashboard):** Overall readiness spine, per-pillar readiness overview showing team fleet distribution per pillar, closest wins, trajectory, coverage, circuits attention (always visible), main blocker
- Service list with readiness status, next step (pillar-contextualized), Cores, Circuits (when vitality toggle on), Momentum (when vitality toggle on)
- Aggregated Suggested Upgrades prioritized toward path-to-completion
- Learning references: "How similar services achieved this baseline"

### View 3: Flight Deck (Service View)

**Goal:** The service coaching cockpit—evidence-first, purpose-first.

**Components:**

- **Header instrumentation:** Core gallery, nominal Circuits, Momentum sparkline
- **Pillar readiness panel:** what blocks mastery, what's closest to completion
- **Rulepack cards:** purpose summary, drill-down to rules with implication/remediation
- **Suggested Upgrades:** phrased as upgrades, each with Action, Impact, Effort, Evidence, Why

### View 4: Flight Manual (Standards Transparency Overlay)

**Goal:** Accessible from any view, surfaces protocol definitions organized as Pillar > Rulepack > Rule.

**Components:**

- Each rulepack shows **Ambition** (why), **Standard** (what), **Evaluation** (how — signal sources, checks, badge semantics from backend), and **Handshake** (where — direct Git edit link).
- **Legend** tab for shared vocabulary.
- Client-side search.
- URL-driven deep-linking via `?manual=` params.

### Suggested Upgrade Priority Order

> **See Constitution Section 10** for the sealed priority model.

1. **Protect Vitality** (2.2x + 0.1x damper on others) — If any Circuit is DEGRADING or DOWN
2. **Unblock Telemetry** (10x) — If readiness is TELEMETRY_MISSING
3. **Reach Baseline** (2.5x closest win) — If all Circuits are NOMINAL
4. **Reduce Friction** (1.0x) — High-impact/low-effort improvements
5. **Optimize** (1.0x) — Remaining improvements

**"Save the Baby" Philosophy:** When the baby is dying, you save the baby before connecting more instruments. A DEGRADING Circuit triggers a 0.1x penalty on all non-vitality candidates, ensuring vitality always wins. Protect the now before pursuing the nice-to-have.

**Fleet-Level Type Diversity:** At org and team altitudes, the recommendation list is a coaching summary, not a service-level triage queue. A per-type slot cap ensures no single UpgradeType monopolizes the limited slots — the leader sees the full landscape of issue types. The priority order above defines display **order**, not **exclusion**. `PROTECT_VITALITY` still appears first; it simply cannot consume every slot when other types have candidates. Per-service (Flight Deck NBU) remains strict priority with no diversity cap.

---

## 9.5 Cross-View Pattern: Context Strip (Flight Mode Annunciator)

### The MCP / FMA Model

Ground Control's Command Bar is the **Mode Control Panel (MCP)** — the surface where engineers set their viewing lens (pillar, focus mode, segment). However, the MCP scrolls with the page. Once an engineer scrolls into the content zones, they lose awareness of which lens is active. The page looks the same whether they are viewing "All Pillars" or "Production Health."

The **Context Strip** solves this by acting as the **Flight Mode Annunciator (FMA)** — a compact, floating indicator that shows which modes are currently engaged. In aviation, the FMA is always visible on the Primary Flight Display so the pilot never needs to look at the MCP to know the active mode. Ground Control adopts the same principle: the MCP is where you set the lens; the FMA is where you see it.

### Purpose

Persistent awareness of the active viewing lens without duplicating controls. The Context Strip is **read-only** — it shows what is active, not controls to change it. To change the lens, the engineer returns to the Command Bar.

### Visibility Rules

The Context Strip appears when **both** conditions are met:

1. The Command Bar has scrolled out of the viewport (detected via IntersectionObserver)
2. At least one non-default filter is explicitly active in the URL

When all filters are at their default state (no pillar selected, no focus mode, default segment), the Context Strip remains hidden even if the Command Bar is scrolled away. Absence of filters = absence of strip. This ensures it only demands attention when a lens is active.

### Active Filters Shown

The Context Strip displays chips for:

| Filter | Shown When |
|--------|-----------|
| **Pillar** | A specific pillar is selected (not "All Pillars") |
| **Focus Mode** | A focus mode is active (Baseline Ready / Break Less / Ship Faster / Reduce Friction) |
| **Segment** | A segment filter is active in Team View (Flight Ready / Pre-flight / Telemetry Missing) |

**Timeframe is excluded.** Timeframe affects Momentum visualization, not the core content lens. Including it would add noise without aiding comprehension.

### Visual Treatment

- **Shape:** Floating pill (rounded-full), compact, visually distinct from page chrome
- **Position:** Fixed bottom-center of the viewport (`bottom: 24px, left: 50%, transform: translateX(-50%)`). Like a cockpit HUD — always visible, below the line of primary instruments, never competing with content zones.
- **Surface:** Semi-transparent card surface with strong backdrop blur and subtle shadow — `bg-card/80 backdrop-blur-md border border-border/60 shadow-lg`. Visually distinct from the Command Bar to reinforce that this is an *instrument*, not a *toolbar*.
- **Filter chips:** Small text, teal diamond prefix (`◆`), dot-separated active filter labels
- **Layout:** Single row — `◆ Filter · Filter · Filter ▴` — the entire pill is clickable
- **Transition:** Smooth fade + slide-up (300ms ease-out) when appearing; fade + slide-down when hiding
- **Example:** `◆ Cloud Native · Ship Faster ▴`

### Interaction

| Action | Result |
|--------|--------|
| Click the pill (anywhere) | Smooth-scrolls the viewport to the Command Bar |

The entire pill is a single clickable surface — no separate "Adjust" button. This keeps it minimal and cockpit-like: the FMA is a single instrument you tap to return to the MCP. It annunciates; it does not control.

### Cross-View Consistency

The Context Strip is **identical** across all three views (Mission Map, Squad Console, Flight Deck). It reads the same URL state that the Command Bar writes. This is implemented via a shared ViewShell layout component, ensuring the pattern is defined once and composed into every view.

### Acceptance Criteria

- [ ] Context Strip is hidden when the Command Bar is visible in the viewport
- [ ] Context Strip is hidden when no non-default filters are active (empty URL = no strip)
- [ ] Context Strip appears with smooth fade + slide-up transition when Command Bar scrolls away AND filters are active
- [ ] Filter chips accurately reflect current URL state (pillar name, focus label, segment label)
- [ ] Clicking the pill smooth-scrolls to the Command Bar
- [ ] Identical behavior across Mission Map, Squad Console, and Flight Deck
- [ ] Pill is read-only annunciation — no filter controls embedded
- [ ] Floating pill is visually distinct from page chrome (shadow, blur, rounded-full shape)
- [ ] Pill does not obscure critical content (bottom-center positioning, compact size)
- [ ] Does not conflict with existing color semantics (teal, amber, red)

---

## 10. Data Sources & Integration

### Service Catalog (Source of Truth)

**Backstage** provides service identity, ownership, team mapping, repo location.

### Collector Engine (Signals)

Plugin-based signal collection:

- **Kubernetes:** deployments, probes, restarts, rollout strategy
- **Observability:** stability/error/latency signals
- **Git provider:** repo hygiene, docs, ownership, GitOps manifests
- **CI system:** duration, stability, retries, lead time

### Evaluation Orchestrator

Runs on a schedule:

- Executes rulepacks on services using latest signals
- Produces immutable evaluation runs + a latest snapshot for UI

---

## 11. Success Metrics

| Category | Metric |
|----------|--------|
| **Adoption** | Weekly active usage; repeat visits to Flight Deck |
| **Movement** | Increased pillar mastery rates; faster time-to-mastery |
| **Vitality** | Fewer streak breaks; reduced "Circuits down" frequency |
| **Outcomes** | Correlation between Production Health mastery and reduced incidents |
| **Delivery** | Measurable CI stability and performance improvements |
| **Telemetry** | Decreasing "Telemetry Missing" coverage gaps |
| **Trust** | Reduced time-to-evidence ("why is this red?" answered quickly) |

---

## 12. Product Language Reference

Ground Control uses consistent terminology across all surfaces:

| Layer | Terms |
|-------|-------|
| **Views** | Mission Map / Squad Console / Flight Deck |
| **Hierarchy** | Pillars / Rulepacks / Rules / Checks / Signals |
| **Achievements** | Cores (Foundation) / Circuits (Vitality) |
| **Trajectory** | Momentum |
| **Readiness** | Flight Ready / Pre-Flight / Telemetry Missing |

Any future feature must preserve: evidence lineage, progress-over-ranking defaults, and non-competitive Momentum (no leaderboards, no negative scoring).

---

## 13. Visual Language Contract

Ground Control's color system follows a **three-layer semantic model**. Colors are not decorative — they communicate product meaning. Every color choice must be traceable to one of these layers.

### Layer 1: Readiness & Vitality States (System Truth)

These CSS variables represent **current system state**. They are the foundation of the visual language.

| Variable | Color | Meaning |
|----------|-------|---------|
| `--gc-flight-ready` / `--gc-nominal` | Teal | Ready / Healthy / Earned |
| `--gc-preflight` / `--gc-degrading` | Amber | In Progress / Pre-flight / Degrading |
| `--gc-down` | Muted red | Offline / Down |
| `--gc-telemetry-missing` | Cool gray | Blocked / Unknown |

### Layer 2: Achievement Types (Foundation vs Vitality)

Achievement type (Core vs Circuit) must be **visually obvious** wherever upgrades or recommendations are shown (Charter P6). This applies to: Protocol List, NextBestUpgrade alternatives, UpgradeCard, and any future recommendation surface.

| Achievement | Icon | Color | Meaning |
|------------|------|-------|---------|
| **Core** | Shield | Sky blue | Permanent foundation — built once, kept forever |
| **Circuit** | Zap | Violet | Living health indicator — must be maintained |

### Layer 3: Recommendation Types (Why Act?)

Recommendation type colors answer: **"Why is Ground Control telling me to do this?"** Where a recommendation type responds to a specific system state, its color **derives from that state's CSS variable**.

| UpgradeType | Icon | CSS Variable | Derives From | Rationale |
|---|---|---|---|---|
| `PROTECT_VITALITY` | ShieldAlert | `--gc-rec-vitality` | `--gc-degrading` (amber) | Circuit health urgency |
| `UNBLOCK_TELEMETRY` | Signal | `--gc-rec-telemetry` | `--gc-telemetry-missing` (gray) | Blocked signals |
| `REACH_BASELINE` | Target | `--gc-rec-baseline` | `--gc-preflight` (warm amber) | Pre-flight path |
| `REDUCE_FRICTION` | Wrench | `--gc-rec-friction` | Self (calm blue) | Non-urgent improvement |
| `OPTIMIZE` | Lightbulb | `--gc-rec-optimize` | Self (muted violet) | Lowest priority, optional |

### Implementation Rules

1. **Single source of truth.** Recommendation type visuals are defined in `apps/web/src/lib/recommendation-visuals.ts`. All components import from there — no local TYPE_CONFIG maps.
2. **Semantic CSS variables.** Recommendation colors must use `--gc-rec-*` variables, never raw Tailwind colors (e.g., `emerald-500`, `orange-500`).
3. **Achievement type visibility.** Wherever upgrade recommendations appear (including alternatives), the Core/Circuit badge must be shown.
4. **Impact/effort centralization.** Impact and effort display is defined in `apps/web/src/lib/impact-effort.ts`. Same pattern, same rules.
5. **No color conflicts.** Non-urgent recommendation colors (friction, optimize) must not use teal, amber, red, or gray — those are reserved for state communication.

### Layer 4: Trajectory (Momentum Visuals)

Trajectory/momentum visuals communicate improvement direction without alarm or competition. They follow the same single-source-of-truth pattern as recommendation types and impact/effort.

| Variable | Color | Meaning |
|----------|-------|---------|
| `--gc-trajectory` | Teal (from `--primary`) | Sparkline stroke, upward trend text |
| `--gc-trajectory-muted` | Muted foreground | Declining/flat trend text (P-M6: no alarm) |

**Implementation Rules:**

1. **Single source of truth.** Momentum visual config (icon, sparkline sizes, trend colors) is defined in `apps/web/src/lib/momentum-visuals.ts`. All components import from there — no local icon choices or size overrides.
2. **Canonical icon.** `TrendingUp` (from lucide) is the trajectory icon across all views.
3. **Locked sparkline dimensions.** `SPARKLINE_SIZES` defines canonical `{ width, height }` per variant (`full`, `visual`, `spark`). Consumption sites reference these constants, not hardcoded numbers.
4. **Semantic CSS variables.** Trend colors use `--gc-trajectory*` variables, not raw Tailwind classes.
5. **No alarm state (P-M6).** Declining trend uses `--gc-trajectory-muted` (neutral), never amber/red. Corrective action comes from Vitality, not Momentum.

---

*This document defines implementation guidance for Ground Control. For immutable core definitions, see the [Ground Control Constitution](./Ground_Control_Constitution.md).*