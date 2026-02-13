# Ground Control — Drawer System Architecture

## Philosophy: Truth vs Meaning

Ground Control's drawer system is built on the foundational principle from the Constitution: **Truth vs Meaning**.

| Drawer Type | Purpose | What It Shows |
|-------------|---------|---------------|
| **TelemetryDrawer** | Truth | Raw signals, data, evidence — "Prove it to me" |
| **BriefDrawer** | Meaning | Interpreted guidance, recommendations — "What should I do?" |

This maps directly to Ground Control's coaching posture: we show the raw telemetry (truth), then provide actionable guidance (meaning).

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DRAWER SYSTEM                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────┐     ┌─────────────────────────────────────────┐   │
│   │   TelemetryDrawer   │     │            BriefDrawer                  │   │
│   │      (Truth)        │     │           (Meaning)                     │   │
│   │                     │     │                                         │   │
│   │  • Rule telemetry   │     │  ┌─────────────┐  ┌─────────────────┐  │   │
│   │  • Protocol telem.  │     │  │ UpgradeBrief│  │ SpotlightBrief  │  │   │
│   │  • Service telem.   │     │  └─────────────┘  └─────────────────┘  │   │
│   │                     │     │  ┌─────────────┐  ┌─────────────────┐  │   │
│   │                     │     │  │VitalityBrief│  │ClosestWinsBrief │  │   │
│   │                     │     │  └─────────────┘  └─────────────────┘  │   │
│   └──────────┬──────────┘     └──────────────────┬──────────────────┘   │
│              │                                    │                      │
│              └────────────────┬───────────────────┘                      │
│                               ▼                                          │
│                    ┌─────────────────────┐                               │
│                    │     DrawerShell     │                               │
│                    │   (DRY - Shared)    │                               │
│                    └──────────┬──────────┘                               │
│                               ▼                                          │
│                    ┌─────────────────────┐                               │
│                    │  drawer-primitives  │                               │
│                    │ (Section, States)   │                               │
│                    └─────────────────────┘                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Drawers vs Manual Overlay

| Aspect | Drawers | Manual Overlay |
|--------|---------|----------------|
| **Purpose** | Contextual evidence and briefs | Global standards transparency |
| **Layout** | Right-panel sliding drawer | Centered HUD overlay |
| **Trigger** | `?drawer=` URL param | `?manual=` URL param |
| **Use case** | Drill-down on evidence, upgrade advisory, briefs | Browse protocol definitions, vocabulary, legend |

**Mutual exclusion:** When the manual overlay is open, it visually supersedes drawers (higher z-index). Both params can coexist in the URL, but the view renders only one at a time — manual takes priority.

---

## URL Routing Contract

All drawer state is URL-driven via the `?drawer=` query parameter. This enables:
- Deep linking (share a URL, recipient sees same drawer)
- Browser history navigation (back/forward)
- Bookmarking specific evidence or recommendations

### URL Patterns

| Action | URL Pattern | Drawer | Title |
|--------|-------------|--------|-------|
| View rule telemetry | `telemetry:rule:{ruleId}` | TelemetryDrawer | "Telemetry" |
| View protocol telemetry | `telemetry:protocol:{rulepackId}` | TelemetryDrawer | "Telemetry" |
| View service telemetry | `telemetry:service:{serviceId}` | TelemetryDrawer | "Telemetry" |
| View upgrade advisory | `brief:upgrade:{type}:{recommendationKey}` | BriefDrawer | "Upgrade Advisory" |
| View mission focus | `brief:spotlight:{spotlightId}` | BriefDrawer | "Mission Focus" |
| View vitality status | `brief:vitality:{state}` | BriefDrawer | "Vitality Status" |
| View quick wins | `brief:closest-wins:{scope}` | BriefDrawer | "Quick Wins" |

### Pattern Structure

```
{drawerType}:{target}:{id}[:{additionalParts}]
```

- **drawerType**: `telemetry` or `brief`
- **target**: Sub-type (e.g., `rule`, `protocol`, `upgrade`, `vitality`)
- **id**: The specific resource identifier
- **additionalParts**: Optional, joined with `:` (e.g., upgrade type + key)

---

## Component Structure

### File Organization

```
apps/web/src/components/
├── drawers/
│   ├── index.ts                    # Single export point
│   ├── DrawerShell.tsx             # Shared shell (DRY)
│   ├── TelemetryDrawer.tsx         # Truth drawer
│   ├── BriefDrawer.tsx             # Meaning drawer (polymorphic router)
│   └── briefs/
│       ├── index.ts
│       ├── UpgradeBrief.tsx        # Upgrade Advisory content
│       ├── SpotlightBrief.tsx      # Mission Focus content
│       ├── VitalityBrief.tsx       # Vitality Status content
│       └── ClosestWinsBrief.tsx    # Quick Wins content
└── shared/
    ├── drawer-primitives.tsx       # DrawerSection, LoadingState, ErrorState, EmptyState
    ├── DrawerServicesList.tsx      # Reusable paginated services list
    └── index.ts
```

### DrawerShell (DRY Infrastructure)

Single source of truth for drawer structure. All drawers use this shell.

```typescript
interface DrawerShellProps {
  title: string;
  icon: LucideIcon;
  subtitle?: string;
  children: React.ReactNode;
  onClose: () => void;
}
```

**Responsibilities:**
- Backdrop overlay with click-to-close
- Slide-in panel from right
- Header with icon, title, subtitle, close button
- Scrollable content area
- Keyboard handling (Escape to close)
- Body scroll lock when open

### TelemetryDrawer

Shows raw signal data — the **Truth**.

**Thematic Naming:**
- Title: "Telemetry"
- Icon: `Radio`
- Subtitle: "Signal readout · {Target Name}"

**Content Sections (protocol-level):**
- Target information with BadgePill
- Conclusion with semantic state indicator
- Rules summary (three groups: Badge-Affecting, Advisory, Passing)
- Signals with timestamps and values
- Coaching (collapsible)
- Links and debug data

**Content Sections (rule-level):**
- Target information with impact badge
- Conclusion with semantic state indicator
- Observed value (suppressed when identical to conclusion summary)
- Signals with timestamps and values
- Coaching (collapsible)
- Links and debug data

#### Evidence Uniformity Principle

Every `SignalEvidence` in the drawer represents truth from a moment in time -- whether the observation succeeded (evaluable) or failed (blocked). Blocked signals carry their collector error details in the `raw` field, ensuring the drawer always shows complete truth without special-casing by signal state.

| Signal State | `sample` | `raw` |
|-------------|----------|-------|
| **Evaluable** | Observed Value (from check) | Query/sampling provenance (from `check.evidence.raw`) |
| **Blocked** | "Collector Failed" (from vocabulary) | Collector error context: `errorKind`, `message`, `missingContextKey`, `blockingStatus` |
| **Neutral** | *(no SignalEvidence produced)* | *(excluded from evidence -- NOT_APPLICABLE is not truth)* |

This ensures `SignalCard` renders uniformly for all signal types. The `DebugDataSection` component (collapsible JSON viewer) displays the `raw` field for both evaluable and blocked signals, giving engineers full debug context regardless of signal state.

#### Rules Summary (Protocol-Level)

The rules summary provides the full denominator for protocol evaluation. All applicable rules are shown (NOT_APPLICABLE excluded), grouped by role and state:

1. **Badge-Affecting Rules** — Non-passing rules that affect the badge outcome (`affectsBadge: true`, `state !== 'pass'`)
2. **Advisory Rules** — Non-passing rules that don't affect badge outcome (`affectsBadge: false`, `state !== 'pass'`)
3. **Passing** — All rules with `state === 'pass'` (collapsed by default, expandable)

Each rule row renders a state icon from `RULE_STATE_CONFIG` (shared single-source-of-truth in `apps/web/src/lib/rule-state-visuals.ts`), ensuring visual continuity with ProtocolDetail in the main Flight Deck view.

| State | Icon | Color Variable | Meaning |
|-------|------|---------------|---------|
| `pass` | CheckCircle2 | `--gc-flight-ready` (teal) | Criteria satisfied |
| `not_met` | Circle | `--gc-preflight` (amber) | Criteria not yet met |
| `warn` | AlertTriangle | yellow-500 | Met with caveats |
| `missing_signal` | CircleSlash | `--gc-telemetry-missing` (gray) | Cannot evaluate |

**Data contract:** `RuleSummary.state` uses lowercase `RuleState` values (`'pass' | 'not_met' | 'warn' | 'missing_signal'`), aligned with `ProtocolRule.state` from the Flight Deck API.

#### Conclusion State Colors

The `StatusIndicator` uses semantic `--gc-*` CSS variables aligned with ProtocolList and ProtocolDetail:

| Conclusion State | CSS Variable | Color |
|-----------------|-------------|-------|
| `passed` | `--gc-flight-ready` | Teal |
| `not_met` | `--gc-preflight` | Amber |
| `warning` | yellow-500 | Yellow |
| `blocked` | `--gc-telemetry-missing` | Gray |
| `not_applicable` | muted-foreground | Muted |

#### Observed Section Deduplication

The Observed section is suppressed when `displayValue === conclusion.summary`. For simple checks (the majority), these values are identical and showing both creates visual noise. When they differ (complex evaluations), both render normally. This is pure view deduplication — no domain logic.

### BriefDrawer (Polymorphic)

Shows interpreted guidance — the **Meaning**.

Routes to the appropriate content component based on brief type:

| Brief Type | Component | Title | Icon |
|------------|-----------|-------|------|
| `upgrade` | UpgradeBrief | "Upgrade Advisory" | Rocket |
| `spotlight` | SpotlightBrief | "Mission Focus" | Target |
| `vitality` | VitalityBrief | "Vitality Status" | HeartPulse |
| `closest-wins` | ClosestWinsBrief | "Quick Wins" | Zap |

---

## Thematic Naming

Ground Control uses a **NASA Mission Control** theme — professional, serious, supportive. Not gamified.

| Brief Type | Drawer Title | Subtitle Pattern |
|------------|--------------|------------------|
| upgrade | "Upgrade Advisory" | "Recommended by Ground Control" |
| spotlight | "Mission Focus" | "Org-wide priority" |
| vitality | "Vitality Status" | "{N} circuits need attention" |
| closest-wins | "Quick Wins" | "{N} services · {currentPct}% → {projectedPct}% Flight Ready" |

### Closest Wins Brief — Enriched Contract

The **closest-wins** brief follows the coaching pattern: backend owns all coaching text, frontend renders.

**Data contract** (`ClosestWinsResponse`):

| Field | Source | Purpose |
|-------|--------|---------|
| `coaching.headline` | `buildClosestWinsCoaching()` in `vocabulary-provider.ts` | Impact headline (1 or N services) |
| `coaching.guidance` | `buildClosestWinsCoaching()` | Team-specific coaching text |
| `readinessImpact` | Aggregator computation | Current vs projected Flight Ready % |
| `commonBlockers[]` | Aggregator grouping | Top blocking patterns across services |
| `services[].progress` | Aggregator per-scan | Baselines met vs required |
| `services[].blockingItem.pillarId/Name` | Config lookup | Pillar context for the blocker |
| `services[].blockingItem.impact/effort` | `ConfigResolver.hydrateCheck()` | Coaching prioritization |
| `services[].blockingItem.implication` | `ConfigResolver.hydrateCheck()` | Why this matters |
| `lastEvaluatedAt` | Most recent scan timestamp | Trust/freshness indicator |

**Text ownership**: All coaching text is generated by `buildClosestWinsCoaching()` in `vocabulary-provider.ts`. The frontend receives pre-computed strings and renders them without modification or fallback.

---

## Implementation Contracts

### NO FALLBACK VALUES

Ground Control's frontend philosophy prohibits default/fallback values. This ensures:
- Data integrity (no false positives)
- User trust (what you see is what exists)
- Clear error states (missing data is explicit, not hidden)

**Rules:**
1. If required data is missing, return `null` from component
2. Never use `|| defaultValue` patterns for display data
3. Show explicit error states when data fetch fails
4. Empty states must be intentional, not fallback

### Type Definitions

```typescript
// Drawer types - only two
export type DrawerType = 'telemetry' | 'brief';

// Telemetry targets
export type TelemetryTarget = 'rule' | 'protocol' | 'service';

// Brief types
export type BriefType = 'upgrade' | 'spotlight' | 'vitality' | 'closest-wins';

// Parsed drawer content
export interface ParsedDrawer {
  type: DrawerType;
  target: TelemetryTarget | BriefType;
  id: string;
  upgradeType?: string; // For upgrade briefs only
}
```

### Parsing Contract

```typescript
function parseDrawerContent(content: string | undefined): ParsedDrawer | null {
  // Returns null for:
  // - undefined/empty content
  // - Invalid drawer type (not telemetry or brief)
  // - Invalid target for drawer type
  // - Missing required parts
  
  // NO FALLBACK - invalid input = null output
}
```

---

## View Integration

Each view that renders drawers follows this pattern:

```typescript
function SomeView() {
  const { isDrawerOpen, drawerContent } = useViewState();
  const parsed = parseDrawerContent(drawerContent);
  
  return (
    <>
      {/* View content */}
      
      {/* Drawer rendering - explicit type checking, no fallback */}
      {isDrawerOpen && parsed?.type === 'telemetry' && (
        <TelemetryDrawer 
          target={parsed.target as TelemetryTarget}
          targetId={parsed.id}
        />
      )}
      {isDrawerOpen && parsed?.type === 'brief' && (
        <BriefDrawer 
          briefType={parsed.target as BriefType}
          briefId={parsed.id}
          upgradeType={parsed.upgradeType}
        />
      )}
    </>
  );
}
```

---

## Opening Drawers

Use the `openDrawer` function from `useViewState`:

```typescript
const { openDrawer } = useViewState();

// Open telemetry
openDrawer(`telemetry:rule:${ruleId}`);
openDrawer(`telemetry:protocol:${protocolId}`);
openDrawer(`telemetry:service:${serviceId}`);

// Open briefs
openDrawer(`brief:upgrade:${type}:${recommendationKey}`);
openDrawer(`brief:spotlight:${spotlightId}`);
openDrawer(`brief:vitality:${state}`);
openDrawer(`brief:closest-wins:${scope}`);
```

---

## Testing Requirements

For each drawer type:
- [ ] URL pattern parses correctly
- [ ] Drawer opens with correct title and icon
- [ ] Content renders properly with real data
- [ ] Close button works
- [ ] Escape key closes drawer
- [ ] Backdrop click closes drawer
- [ ] Browser back button works
- [ ] Deep links work (copy URL, paste in new tab)
- [ ] Invalid URL patterns return null (no render)
- [ ] No console errors or warnings

---

## Unified Evidence Conclusion Model

The TelemetryDrawer's conclusion section uses a **unified vocabulary** regardless of the evidence target type (rule, protocol, or service). This replaces the previous dual-vocabulary approach where CheckStatus and ProtocolState produced inconsistent labels.

### EvidenceConclusionState

A dedicated view-domain enum defined in `enums.ts`:

| State | Label | Description |
|-------|-------|-------------|
| `passed` | Passed | Evaluation criteria are satisfied. |
| `failed` | Not Yet Met | Evaluation criteria are not yet met. |
| `warning` | Warning | Criteria are met with caveats. |
| `blocked` | Telemetry Missing | Required signals are not connected -- cannot evaluate. |
| `not_applicable` | Not Applicable | This evaluation does not apply to this service. |

### Mapping

Backend mapping functions convert domain states to the unified conclusion state:

| Source | Input | Conclusion State |
|--------|-------|------------------|
| CheckStatus | PASS | passed |
| CheckStatus | FAIL | failed |
| CheckStatus | WARN | warning |
| CheckStatus | ERROR + COLLECTOR_FAILED | blocked |
| CheckStatus | ERROR + NOT_APPLICABLE | not_applicable |
| ProtocolState | met | passed |
| ProtocolState | below | failed |
| ProtocolState | telemetry_missing | blocked |
| ProtocolState | not_applicable | not_applicable |
| ReadinessState | flight_ready | passed |
| ReadinessState | preflight | failed |
| ReadinessState | telemetry_missing | blocked |

### Criteria Field

The `criteria` field on `EvidenceConclusion` shows the pass/fail criteria:

- **Rule-level**: Auto-formatted from `ThresholdConfig` (e.g., "exists = true") or overridden by `criteriaText` in YAML config.
- **LADDER protocol**: Tier thresholds (e.g., "Gold <= 480 . Silver <= 900 . Bronze <= 1500").
- **GATEKEEPER protocol**: Blocking rule count (e.g., "All 3 blocking rules must pass").

### Baseline Field

The `baseline` field on `EvidenceConclusion` provides pillar context for protocol evidence:

"""typescript
interface EvidenceBaseline {
  required: string;      // "BRONZE", "PASS"
  pillarName: string;    // "Operational Excellence"
  baselineMet: boolean;
}
"""

This replaces the tooltip-only display in `BadgePill`, making baseline requirements directly visible in the conclusion section.

### Separation of Concerns

| Layer | Responsibility |
|-------|---------------|
| Domain (`enums.ts`) | Defines what conclusion states exist |
| Vocabulary (`vocabulary-provider.ts`) | Defines labels/descriptions, mapping functions, criteria formatting |
| Config (`rulepack YAML`) | Defines `criteriaText` overrides |
| View-layer (`flight-deck-aggregator.ts`) | Resolves and assembles the conclusion; maps CheckStatus to RuleState |
| Shared visuals (`rule-state-visuals.ts`) | Single source of truth for rule state icons and colors |
| Frontend (`TelemetryDrawer.tsx`) | Renders what the backend provides (no domain logic, no fallbacks) |

---

**Benefits:**
- DRY: Single DrawerShell shared by all; RULE_STATE_CONFIG shared across ProtocolDetail and TelemetryDrawer
- Consistent UX: All drawers follow same patterns; state colors use semantic CSS variables throughout
- Clear separation: Truth (Telemetry) vs Meaning (Brief)
- Extensible: New brief types are just new content components
- Self-documenting URLs: Pattern tells you exactly what opens
- Visual continuity: Rule state icons are identical in the main view and the drill-down drawer
