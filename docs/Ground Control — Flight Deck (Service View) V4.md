# Ground Control — Flight Deck (Service View) V4

**Version:** 4.0
**Status:** Canonical — supersedes V3, V3.1, Cores Section Redesign spec, and Lower Section Redesign spec
**Changelog:** Consolidates all sub-design specs into a single source of truth

---

## 1. Purpose & Posture

### Purpose

The Flight Deck is the "deep work" service cockpit: **Orient → Choose the next best move → Verify evidence → Act → Re-check later**.

### Core Posture (Non-Negotiable)

The Flight Deck must feel like:

- **Coaching, not surveillance** — every judgment is explainable and evidence-linked.
- **Progress, not punishment** — trajectory and near-wins dominate; no ranking defaults.
- **Truth over controls** — no UI action may "change" evaluation truth; only how we *attend* to it.
- **No-auth reality** — all state is URL/shareable; personalization is anonymous.

---

## 2. Terminology (V4 Canonical)

| Retired Term | Canonical Term | Context |
|--------------|----------------|---------|
| `FOUNDATION` (section header) | `CORES` | Service Identity Card section label |
| `VITALITY` (section header) | `CIRCUITS` | Service Identity Card section label |
| Foundation Shelf | Cores Gallery | Zone F reference |
| Mastery Ladder (as panel) | Flight Readiness Strip | Replaced by compact strip |
| Protocols Workbench (half-width) | Protocols Workbench (full-width) | Same concept, new layout |

The unit name *is* the section name. No parent category headers wrapping them.

---

## 3. Key Concepts: Vitality vs Momentum

### Circuits (Vitality) = NOW

Rolling operational health indicators derived from checks with ROLLING windowType.

**States:**
- **NOMINAL** — Currently healthy (baseline met)
- **DEGRADING** — Problem persisting past warning threshold (grace period)
- **DOWN** — Problem persisting past critical threshold (action required)

**Key properties:**
- Time-based state derivation (scan-cadence independent)
- `lastPassedAt` timestamp stored at scan-time
- `CircuitState` derived at view-time from `(now - lastPassedAt)`
- Per-rulepack urgency thresholds

**Vitality answers:** "Is this service operationally healthy right now?"

### Momentum = TRAJECTORY

Daily contribution reflecting active investment in excellence.

**Accrual model:**
- **Circuits:** +10/day per NOMINAL Circuit (0 when DEGRADING or DOWN)
- **Cores:** +100 one-time burst when first earned

**Key properties:**
- Computed at view-time, not stored
- Scan-cadence independent (daily rate, not per-scan)
- Can fluctuate (losing Circuits reduces daily rate)
- Shows trajectory, not cumulative total

**Momentum answers:** "Is this service actively maintaining/building excellence?"

### The Separation

| Concept | Timeframe | Can Decrease? | Purpose |
|---------|-----------|---------------|---------|
| Vitality (Circuits) | NOW | State can worsen | Operational health pulse |
| Momentum | TRAJECTORY | Daily rate can drop | Investment in excellence |

---

## 4. Page Structure — Zone Map

```
┌─────────────────────────────────────────────────────┐
│  ZONE A — Command Bar                               │
├─────────────────────────────────────────────────────┤
│  ZONE B — Service Identity Card (Cockpit Header)    │
│  ┌──────────┬──────────┬──────────┐                 │
│  │ CIRCUITS │  CORES   │TRAJECTORY│                 │
│  └──────────┴──────────┴──────────┘                 │
├─────────────────────────────────────────────────────┤
│  ZONE C — Next Best Upgrade (the Brain)             │
├─────────────────────────────────────────────────────┤
│  ZONE D — Flight Readiness Strip                    │
├─────────────────────────────────────────────────────┤
│  ZONE E — Protocols Workbench (full width)          │
│  ┌─────────────────────────────────────────────┐    │
│  │  Filter Bar + Sort                          │    │
│  │  Protocol Cards (expandable, full width)    │    │
│  │  "At Baseline" compact section              │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

---

## 5. Zone A — Command Bar (URL-driven, no-auth)

### Controls

- Search protocols/rules (service-local)
- Pillar selector (All / specific pillar)
- Focus mode selector (Baseline Ready / Break Less / Ship Faster / Reduce Friction)
- Trajectory window selector (7d / 30d / 90d)
- Share link

**No Vitality toggle.** Vitality is always visible — calm comes from design + acknowledgment mechanics, not hiding truth.

---

## 6. Zone B — Service Identity Card (Cockpit Header)

The always-visible truth row. Three-column layout.

### Top Row: Identity & Trust Anchors

- Service name
- Owner/team
- Repo link + runtime language/stack
- **Readiness label:** Flight Ready / Pre-flight / Telemetry Missing
- Coverage % (evaluated coverage) with drilldown ("what's missing?")
- Updated timestamp

### Lower Row: Three Columns

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CIRCUITS  1 down · 3 nominal  │  CORES  2 of 6    │  TRAJECTORY       │
│                                │                    │                   │
│  ○ Reliable CI                 │  [icon] [icon]     │  ╱ sparkline      │
│  ● Need for Speed              │  [icon] [icon]     │  30/day           │
│  ● Right-Sizing                │  [icon] [icon]     │  +30 · 7d ↑       │
│  ● Runtime Stability           │                    │                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Column 1: CIRCUITS (Vitality HUD — NOW)

**Header:** `CIRCUITS` with inline summary: `1 down · 3 nominal`

A row of Circuit indicators (lights), always visible. Each Circuit has a state:
- **NOMINAL** — green, steady
- **DEGRADING** — amber, subtle pulse
- **DOWN** — red, static (not alarming)

Each Circuit is clickable → opens a lightweight detail popover with current state explanation, time in current state ("degrading for 28h"), window definition (time/event), and evidence link.

**Display rule:** Show up to 5 Circuits directly. If more, show "+N" overflow that opens a Vitality Panel. Current rulepacks produce 4 Circuits — all visible without overflow.

#### CircuitState Computation (Scan-Cadence Independent)

CircuitState is computed at view-time, NOT stored:

1. At scan-time: `lastPassedAt` (timestamp when `baselineMet=true` last occurred) is stored
2. At view-time: `deriveCircuitState(lastPassedAt, VitalityThresholds)` calculates state

#### Per-Rulepack Vitality Thresholds (Coaching Cadence)

| Rulepack | DEGRADING | DOWN | Rationale |
|----------|-----------|------|-----------|
| safety_net | 24h | 72h | Production crashes — high urgency |
| ci_stability | 24h | 72h | Flaky CI — blocks delivery |
| ci_performance | 48h | 120h | Slow pipelines — friction, not emergency |
| resource_efficiency | 72h | 168h | Cost/sizing — needs planned work |

**Why these values:** Minimum 24h DEGRADING gives a full day to notice and investigate. DOWN thresholds give meaningful grace period before "critical." Different urgency profiles reflect real-world fix cycles. Twice-daily scan cadence means ~2 scans before DEGRADING, ~6 before DOWN.

---

### Column 2: CORES (Iconic Badges — PERMANENT)

**Header:** `CORES` with inline summary: `2 of 6` (secondary text for "of 6")

Replaces the previous anonymous dots with **iconic Core badges**. Each Core gets a distinct icon that communicates its capability at a glance. The result feels like a row of cockpit instrument indicators: glance-readable, information-dense, and satisfying to "light up."

#### Badge Grid Layout

The grid adapts to the number of Cores:
- **1–3 Cores:** Single row
- **4–6 Cores:** 2-column grid, 2–3 rows (current state: 6 Cores = 3 rows × 2 columns)
- **7–9 Cores:** 3-column grid (future-proofing)

Each badge occupies a consistent cell size. Badges are vertically and horizontally centered within the Cores column.

#### Core Badge Anatomy

```
┌──────────────────────┐
│      [ ICON ]        │   ← Distinct icon per Core capability
│    Core Name         │   ← Short label, always visible
└──────────────────────┘
```

**Dimensions:**
- Badge size: ~64px × 64px total (icon + label combined)
- Icon size: ~28–32px
- Label: Truncate long names with ellipsis if needed; prefer short display names

#### Visual States

**Earned State:**
- Icon is filled and lit using teal/cyan accent (`#00D4AA`)
- Subtle glow effect: soft 2–4px teal box-shadow at ~15–20% opacity
- Label text at normal opacity (full white/primary)
- Overall: "powered on"

**Not Earned State:**
- Icon rendered as dim outline — stroke-only at ~25–30% opacity
- No glow
- Label text at ~40–50% opacity
- Overall: "dark instrument" — visible, identifiable, clearly not active
- Acts as a natural nudge toward completion

**Telemetry Missing State:**
- Icon as outline (same as Not Earned)
- Small indicator (tiny `?` badge or dashed border) to distinguish from "evaluated and not earned"
- Label text at reduced opacity with same treatment
- Rare state — keep treatment minimal

#### Icon Mapping

Use a consistent icon library (Lucide recommended). All icons from the same family. Icons must be distinct in silhouette at 28px.

| Core Display Name | Suggested Icon | Rationale |
|-------------------|----------------|-----------|
| **Cloud Native** | `Cloud` / `CloudCog` | Cloud infrastructure maturity |
| **Safety Net** | `ShieldCheck` / `Shield` | Protection, guardrails |
| **Observability** | `Eye` / `Radar` / `Activity` | Monitoring, visibility |
| **CI Performance** | `Gauge` / `Zap` | Speed, performance |
| **Docs & Ownership** | `FileText` / `BookOpen` | Documentation, ownership |
| **AI Dev Readiness** | `Cpu` / `BrainCircuit` | AI infrastructure readiness |

**Implementation:** Store icon mapping in a configuration object (not hardcoded per-component). Fallback to `Hexagon` if no mapping exists — treat as config gap.

#### Interaction

**Hover/Click on a Core Badge → Popover** (same component as Circuit popovers):

- **Earned:** Core name, state `Earned` (teal), rulepack description, "View details ›" → Protocols Workbench
- **Not Earned:** Core name, state `Not Earned` (subdued), top blocker summary, "View upgrade path ›"
- **Telemetry Missing:** Core name, state `Telemetry Missing` (amber), missing signal, "Connect telemetry ›"

Popover appears on hover (~200ms delay), also accessible via click. Only one popover visible at a time.

Core badges are static in position. Order determined by configuration.

#### Animation

- **On page load:** Earned Cores appear with glow immediately. No entrance animation.
- **On state change (newly earned during session refresh):** Brief "light-up" transition: outline/dim → filled/glowing over ~400ms with slight scale pulse (1.0 → 1.05 → 1.0). Tier C celebration — rare, tasteful.
- **Not Earned:** No animation ever. Static, dim, waiting.
- **Hover:** Subtle brightness increase (~10%) matching Circuit indicator hover treatment.

---

### Column 3: TRAJECTORY (Momentum — HISTORY)

**Header:** `TRAJECTORY`

**Display format:**
```
Momentum: 40/day
+280 · 7d ↑
```

**Components:**
- **Daily rate:** Current Momentum earning rate (NOMINAL Circuits × 10)
- **Window delta:** Sum of daily rates over selected window + any Core bursts
- **Trend arrow:** ↑ gaining / → steady / ↓ losing
- **Sparkline:** Daily Momentum values over the selected window. Fluctuations are normal. Core bursts appear as spikes.

---

## 7. Zone C — Next Best Upgrade (the Brain)

Single best next step with **locked prioritization order** matching the "Keep the Baby Alive" philosophy.

### Selection Logic (Locked)

Order of precedence:

1. **Unblock Telemetry** — If readiness is Telemetry Missing (overall or for selected pillar), the card must be a telemetry action.
2. **Protect Vitality (Circuits first)** — If any displayed Circuit is DEGRADING or DOWN, target protection of that Circuit. This is "Keep the baby alive" — operational health before new construction.
3. **Reach Baseline (Mastery blockers)** — If all Circuits are NOMINAL, prioritize the closest baseline blocker.
4. **Reduce Friction** — High-impact, low-effort improvements NOT blocking baseline. Quick wins for developer experience.
5. **Optimize** — Remaining improvements, prioritized by impact/effort ratio. Excellence beyond baseline.

### Card Anatomy

- Title (action verb)
- Why it matters (implication)
- How to do it (remediation)
- Impact + effort chips
- **"Moves" label:** Unblocks Telemetry / Protects Vitality / Reaches Baseline / Reduces Friction / Optimizes
- Evidence entry point (always ≤2 clicks to raw truth)
- "Open fix steps →" action button
- "View Evidence" secondary button
- "View alternatives" expandable link

---

## 8. Zone D — Flight Readiness Strip

This compact strip replaces the old Mastery Ladder panel. It answers "how close am I?" in one glance.

### Anatomy

```
FLIGHT READINESS
████████████████████░░░░░░░░  6 of 10 protocols met · 4 below baseline
```

### Specifications

**Section header:** `FLIGHT READINESS` — same caps/small/subdued style as `CORES`, `CIRCUITS`, `TRAJECTORY` headers.

**Progress bar:**
- Full width of the content area
- Height: 6–8px, rounded ends
- **Filled segments:** Teal accent color (same as earned Cores, nominal Circuits)
- **Unfilled segments:** Dark surface color at ~15% opacity
- **Segmented, not continuous.** Each protocol is a discrete segment with 1–2px gap. Communicates "countable things" not an abstract score.
- Proportional: with 10 protocols, each segment is 10% of bar width.

**Summary text:**
- Format: `6 of 10 protocols met · 4 below baseline`
- "6 of 10 protocols met" in primary text color
- "4 below baseline" in amber/warning color
- If all met (Flight Ready): `10 of 10 protocols met · Flight Ready` with "Flight Ready" in teal accent

**Interaction:** The progress bar is NOT interactive. Pure status display.

### Relationship to Service Identity Card

The Identity Card shows `Pre-flight` / `Flight Ready` / `Telemetry Missing` badge. The Flight Readiness Strip provides the **quantified detail** behind that badge. Complementary, not redundant.

---

## 9. Zone E — Protocols Workbench (Full Width)

A single full-width column. No side-by-side panels. Contains the Filter Bar, Protocol Cards, and At Baseline compact section.

### 9.1 Filter Bar

```
[Below Baseline]  [At Baseline]  [All]              [Sort: Priority ▾]
```

#### Filter Tabs

Three mutually exclusive states:

| Tab | Shows | Default? |
|-----|-------|----------|
| **Below Baseline** | Only protocols that haven't met baseline | **Yes** |
| **At Baseline** | Only protocols that have met baseline | No |
| **All** | Every protocol, grouped by status | No |

**Why "Below Baseline" is the default:** The engineer came here to understand what needs work. Starting with completed items wastes the first impression.

#### Sort Control

| Sort Option | Behavior | Default? |
|-------------|----------|----------|
| **Priority** | Locked order: Unblock Telemetry → Protect Vitality → Reach Baseline → Reduce Friction → Optimize | **Yes** |
| **Pillar** | Group by pillar, alphabetical within pillar | No |
| **Effort** | Smallest effort first (S → M → L) | No |
| **Impact** | Highest impact first (HIGH → MEDIUM → LOW) | No |

#### Visual Treatment

- Filter tabs: Pill-style buttons or underlined tab pattern. Active tab uses teal accent.
- Sort control: Right-aligned compact dropdown. Shows current sort as label.
- Sits directly below the Flight Readiness Strip.

---

### 9.2 Protocol Cards (Full Width)

The core working surface. Each protocol (rulepack) rendered as an expandable card at full width.

#### Collapsed Card

```
┌─────────────────────────────────────────────────────────────────┐
│  ○  AI Development Readiness                    🔷 Core    ▸   │
│     Top blocker: AI coding guidance is defined                  │
└─────────────────────────────────────────────────────────────────┘
```

**Elements:**
- **Status indicator:** Empty circle = below baseline, filled check = at baseline
- **Protocol name:** Bold, primary text
- **Achievement type badge:** Compact indicator — Core (uses icon from icon mapping at ~16px + label) or Circuit (Circuit indicator icon + label). Reinforces the Vitality vs. Foundation distinction.
- **Top blocker summary:** One line showing the blocking rule name
- **Expand chevron:** Right-aligned, click to expand
- **Drawer entry:** Protocol name or dedicated telemetry icon opens the Telemetry Drawer. Chevron expands inline; name opens drawer.

#### Expanded Card

```
┌─────────────────────────────────────────────────────────────────┐
│  ○  AI Development Readiness                    🔷 Core    ▾   │
│                                                                 │
│  Checks that the repository provides the minimum context        │
│  needed for safe, consistent AI-assisted development—so         │
│  tools can navigate the service, follow conventions, and        │
│  avoid incorrect assumptions.                                   │
│                                                                 │
│  Blockers to baseline (1)                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  ●  AI coding guidance is defined    HIGH   Small    ▸  │    │
│  │     Observed value: Not found                           │    │
│  │     Without repo-level AI guidance, AI-assisted         │    │
│  │     changes tend to be generic and inconsistent...      │    │
│  │     [better-dx]  [change-confidence]                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ▸ Show passing (2)                                             │
└─────────────────────────────────────────────────────────────────┘
```

**Elements in expanded state:**
- **Description:** Rulepack purpose, 1–3 lines
- **Blockers to baseline (N):** Each blocking rule shows: rule name, impact chip (HIGH/MEDIUM/LOW), effort chip (Small/Medium/Large), observed value in monospace, implication text (truncated, full in drawer), tags, chevron → Telemetry Drawer
- **Show passing (N):** Collapsed by default. Expandable. Not actionable — keep out of the way.

**Full width advantage:** Impact, effort, tags, and observed value fit on a single row without wrapping. Implication text gets 2–3 readable lines.

#### Cards for "At Baseline" Protocols

```
┌─────────────────────────────────────────────────────────────────┐
│  ✓  Cloud Native                                🔷 Core    ▸   │
└─────────────────────────────────────────────────────────────────┘
```

Compact single-line card. Green check. No "top blocker" line. Expandable to show detail (all rules passing). Drawer still accessible.

---

### 9.3 "At Baseline" Compact Section

When the "Below Baseline" filter is active, met protocols appear as a compact secondary section below the working cards.

```
AT BASELINE  (5)                                          [Show all]

✓ Need for Speed  ·  ✓ Cloud Native  ·  ✓ Right-Sizing
✓ Runtime Stability  ·  ✓ Service Coupling
```

- **Header:** `AT BASELINE` in subdued caps with count
- **Layout:** Compact wrapping inline list, dot-separated. Green check prefix per name.
- **Density:** Summary footnote feel. 2–3 lines max.
- **"Show all":** Switches filter to "At Baseline"
- **Click protocol name:** Opens Telemetry Drawer directly

---

### 9.4 Priority Ordering (Vitality-First)

When "Below Baseline" filter + "Priority" sort (defaults) are active:

1. **Telemetry Missing** — Protocols that can't be evaluated
2. **Circuits (Vitality) first** — Below-baseline protocols producing a Circuit. "Keep the Baby Alive" — operational health before new construction.
3. **Cores (Foundation) second** — Below-baseline protocols producing Cores, ordered by closest-to-completion (fewest remaining blockers)

Within each group, sort by closest-to-completion (fewest blockers, then highest impact).

The achievement type badge on each card makes this ordering self-explanatory.

---

## 10. Telemetry Drawer (Locked Pattern — Unchanged)

The Telemetry Drawer is a proven, locked pattern. V4 does NOT change the drawer.

### Entry Points

1. **Clicking a protocol name or telemetry icon** on any protocol card → Opens at protocol level (Conclusion → Blocking Rules → Signals → Coaching)
2. **Clicking a rule chevron** inside an expanded card → Opens at rule level
3. **Clicking a protocol name** in the "At Baseline" compact section → Opens at protocol level

### Behavior

- Opens from the right side, overlaying Workbench content
- Same component, structure, and content at every entry point
- Close via X button or clicking outside
- Only one drawer open at a time

---

## 11. Evidence Access (Unchanged Principle)

Every judgment has evidence reachable in ≤2 interactions. This principle applies across all zones — Circuit popovers, Core popovers, Next Best Upgrade card, Protocol Cards, and the Telemetry Drawer.

---

## 12. Visual Design System

### Color Palette (Dark Theme Only)

No light theme variant. All colors match the existing dark theme:

| Element | Color |
|---------|-------|
| Earned Core icon fill / Nominal Circuit / Progress bar filled | Teal accent (`#00D4AA`) |
| Earned Core glow | Teal at ~15–20% opacity, 3–4px spread |
| Not Earned Core outline | Primary text at ~25% opacity |
| DEGRADING Circuit | Amber |
| DOWN Circuit | Red, static |
| "Below baseline" count text | Amber/warning |
| Filter tab active | Teal accent underline/fill |
| Impact HIGH chip | Red/coral |
| Impact MEDIUM chip | Amber |
| Effort chips | Neutral/subdued |
| Progress bar unfilled | Dark surface at ~15% opacity |

### Typography

- **Section headers** (`CIRCUITS`, `CORES`, `TRAJECTORY`, `FLIGHT READINESS`, `BELOW BASELINE`, `AT BASELINE`): Caps, small, subdued
- **Labels:** Same font family and weight as Circuit labels
- **Protocol names:** Same weight/size as existing workbench names
- **Rule names inside expanded cards:** Same as current workbench rule names

### Spacing

- Between Flight Readiness Strip and Filter Bar: 16–20px
- Between Filter Bar and first protocol card: 16–20px
- Between protocol cards: 12–16px
- Between last below-baseline card and "At Baseline" section: 24–32px (clear visual break)
- Core badge gutters: 8–12px
- Card internal padding: Match existing workbench cards

---

## 13. Interaction Summary

| Action | Result |
|--------|--------|
| Click Circuit indicator in Identity Card | Popover with state, time, evidence link |
| Click/hover Core badge in Identity Card | Popover with status, blockers or details, navigation link |
| Click filter tab (Below Baseline / At Baseline / All) | Filters protocol card list |
| Click sort dropdown option | Re-sorts protocol card list |
| Click chevron on collapsed protocol card | Expands card inline |
| Click chevron on expanded protocol card | Collapses card |
| Click protocol name or telemetry icon on card | Opens Telemetry Drawer at protocol level |
| Click rule chevron inside expanded card | Opens Telemetry Drawer at rule level |
| Click protocol name in "At Baseline" compact section | Opens Telemetry Drawer at protocol level |
| Click "Show all" in "At Baseline" compact section | Switches filter to "At Baseline" |
| Click "Open fix steps →" on Next Best Upgrade | Opens remediation flow |
| Click "View Evidence" on Next Best Upgrade | Opens evidence for recommended action |

---

## 14. Edge Cases

### All Protocols Met (Flight Ready) — Tier C Celebration

- Flight Readiness Strip: full teal bar + "10 of 10 protocols met · Flight Ready"
- Filter defaults to "All" (since "Below Baseline" is empty)
- All cards in compact at-baseline format
- **If all Circuits NOMINAL:** Tier C earned celebration in Cockpit Header — Bowie accent text ("Ground Control to Major Tom: ..."), sparkles, teal glow. Rare and warm.
- **If any Circuit DEGRADING/DOWN:** No celebration. Vitality protection takes priority over readiness celebration.

### All Circuits Nominal, Cores Incomplete (Pre-flight) — Tier B Acknowledgment

- All Circuit indicators green/nominal in Circuits Cluster
- Some Cores still dim/not earned in Cores Cluster
- **Cockpit Header shows Tier B acknowledgment:** warm, factual — acknowledges operational health, redirects to remaining cores. No Bowie theme.
- Next Best Upgrade targets the closest Core (REACH_BASELINE priority)

### No Protocols Met

- Flight Readiness Strip: empty bar + "0 of 10 protocols met · 10 below baseline"
- No "At Baseline" compact section
- All cards in working/expanded format
- Default priority sort applies — Circuits first, then Cores

### Telemetry Missing (Some Protocols)

- Protocols with `telemetry_missing` appear at top of "Below Baseline" list (priority #1)
- Card uses amber color + additional "Telemetry Missing" indicator
- Progress bar counts them as "not met" segments

### Single Protocol Below Baseline

- Strip still shows ("9 of 10, almost there")
- Single below-baseline card expanded by default
- "At Baseline" compact section shows the other 9

### Many Protocols (Future: 15+)

- Full-width single-column layout scales naturally
- Compact "At Baseline" section wraps as needed
- Filter and sort become increasingly valuable

---

## 15. Responsive & Platform

- **Desktop only.** No mobile support needed.
- Cores column adapts to available width; badges shrink proportionally. Labels truncate with ellipsis. Icons maintain minimum 20px.
- The full-width workbench layout handles browser resize gracefully.

---

## 16. Acceptance Criteria (V4 Must-Pass)

### Identity Card

1. **Section header reads "CIRCUITS"** (not "VITALITY") with inline summary
2. **Section header reads "CORES"** (not "FOUNDATION") with "X of Y" summary
3. **Each Core has a distinct, recognizable icon** — no two Cores share an icon
4. **Earned Cores are visually "lit"** with teal fill + subtle glow
5. **Not Earned Cores show dim outline** — capability identifiable without hover
6. **Telemetry Missing Cores are distinguishable** from Not Earned
7. **Core popovers work** on hover/click with correct state info and navigation
8. **Labels are always visible** (not hover-only)
9. **Grid fits** within existing column without overflow
10. **Icon mapping is configurable** — new Core = config entry only

### Vitality & Momentum

11. **No Vitality toggle exists anywhere.** Always visible, calm by design.
12. **CircuitState uses per-rulepack thresholds** with 24h minimum DEGRADING
13. **Momentum displays as daily rate** (e.g., "40/day") not cumulative total

### Next Best Upgrade

14. **If any Circuit is DEGRADING/DOWN, Next Best Upgrade targets it** (unless telemetry missing). Language: "Protects Vitality" not "Restores Vitality."
15. **Priority order enforced:** Unblock Telemetry → Protect Vitality → Reach Baseline → Reduce Friction → Optimize

### Flight Readiness Strip

16. **Segmented progress bar** with accurate count
17. **Non-interactive** — pure status display

### Protocols Workbench

18. **Two-panel layout is replaced** with single full-width column
19. **Default filter: "Below Baseline"** — engineers land on actionable items
20. **Default sort: "Priority"** — locked order
21. **Each protocol card shows achievement type** (Core or Circuit badge)
22. **Expanded cards use full width** — blocking rules show impact/effort/tags without cramming
23. **"At Baseline" protocols appear as compact summary** below working items
24. **Drawer opens correctly** from all entry points
25. **Only one drawer open at a time**
26. **Celebration uses three-state model:** No celebration when circuits degrading/down; Tier B acknowledgment when all circuits nominal but not Flight Ready; Tier C earned celebration (Bowie accent) only when all circuits nominal AND Flight Ready
27. **Scroll position preserved** when expanding/collapsing cards and closing drawer

### General

28. **Every judgment has evidence reachable in ≤2 interactions**
29. **Language is Mission Control** (not RPG), defaults avoid ranking dynamics
30. **Language is coaching, not compliance** — "Below Baseline" not "Failing," "Blockers to baseline" not "Violations"

---

## 17. What Is NOT Changing in V4

- **Telemetry Drawer:** Structure, content, behavior — locked pattern
- **Data model:** Protocol evaluation, badge computation, circuit state derivation — all unchanged
- **Trajectory section:** Completely unchanged from V3
- **Service Identity Card top row:** Service name, team, repo, language, readiness badge, coverage, timestamp — unchanged
- **Popover component:** Reuse existing, don't build new

---

*This document is the single canonical Flight Deck design spec. It supersedes V3, V3.1, the Cores Section Redesign spec, and the Lower Section Redesign spec. For system definitions, see the Ground Control Constitution. For voice and tone, see the Charter v4.0.*