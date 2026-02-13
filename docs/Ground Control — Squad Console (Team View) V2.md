# **Ground Control — Squad Console (Team View) V3**

## **"The Huddle" Wireframe & Frontend Implementation Spec**

### **Purpose of this screen**

The Squad Console is the team's **weekly coordinating ritual**—a calm, evidence-backed cockpit that turns "readiness truth" into **ownership + action**.

It must consistently drive the loop:

**Orient → Choose → Act → Validate**

**Not a report. Not a blame tool.**
A team-level coaching interface that scales to ~1300 services overall while keeping a team's scope manageable.

---

## **0) Key UX Outcomes (what users should feel)**

1. **Clarity:** "We know where we stand on the runway."
2. **Autonomy:** "We can choose what matters this week."
3. **Achievability:** "We have 3–5 upgrades worth doing next."
4. **Responsibility (not panic):** "Vitality is fading; we can restore it deterministically."
5. **Trust:** "This is truth with evidence, not judgment."
6. **Pride:** "Our foundation is growing; our vitality is holding."

---

## **1) Page Layout (desktop-first; TV-compatible)**

### **High-level sections**

1. **Team Command Bar** (filters, focus, timeframe, share)
2. **The Huddle Header** (Team Readiness Spine + Pillar Readiness + Closest Wins + Coverage + Trajectory + Circuits Attention + Main Blocker)
3. **Suggested Upgrades (Curated)** (3–5 items, leverage-sorted)
4. **Team Vitality (Optional, calm)** (Circuits + momentum cues — toggle controls Services Table columns only)
5. **Services Table (Scannable, not ranked by worst)** (virtualized)
6. **Evidence & Trust (Right rail or expandable panel)** (purpose + measurement contract + links)

### **Wireframe (desktop)**

"""
┌──────────────────────────────────────────────────────────────────────────────┐
│ Ground Control · Squad Console · <Team Name>                 [Share] [Help]  │
│ [Search services] [Pillar: All ▼] [Focus: Baseline Ready ▼] [Timeframe: 7d▼]│
│ [Segment: All ▼] [Vitality: Off/On] [Leverage: High Impact+Low/Med Effort]  │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ THE HUDDLE                                                     N services    │
│──────────────────────────────────────────────────────────────────────────────│
│ [Overall Readiness Spine — full width bar]                                   │
│                                                                              │
│ ┌─ Pillar Readiness ──────────────┐  ┌─ Closest Wins ─────────────────────┐ │
│ │  Production Health   ████░░░░░  │  │  8 services one step away          │ │
│ │  Delivery Confidence ░░░░░░░░░  │  │  [View closest wins →]             │ │
│ │  Cloud Native        ████████░  │  │                                    │ │
│ │  AI Dev Readiness    ░░░░░░░░░  │  │  TRAJECTORY ──/‾ Gaining           │ │
│ └─────────────────────────────────┘  │  Coverage: 100%                    │ │
│                                      └────────────────────────────────────┘ │
│                                                                              │
│ ┌─ Circuits Need Attention (always visible) ─────────────────────────────┐  │
│ │  ⚠ 5 circuits degrading or down · 12 of 15 services fully nominal     │  │
│ │  [View circuits →]                                                      │  │
│ ├─ Main Blocker ─────────────────────────────────────────────────────────┤  │
│ │  ⚠ "Safe Releases baseline missing" · 9 affected                       │  │
│ │  [View blocker →]                                                       │  │
│ └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┬───────────────┐
│ SUGGESTED UPGRADES (Curated)                                  │ TRUST PANEL   │
│ ┌───────────────────────────────────────────────────────────┐│ - Why this    │
│ │ Upgrade Card                                               ││ - Coverage    │
│ ├───────────────────────────────────────────────────────────┤│ - Evidence    │
│ │ Upgrade Card                                               ││ - How to use  │
│ ├───────────────────────────────────────────────────────────┤│               │
│ │ Upgrade Card                                               ││               │
│ └───────────────────────────────────────────────────────────┘│               │
└──────────────────────────────────────────────────────────────┴───────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ SERVICES (Team scope)                                                        │
│ [Service] [Readiness] [Next Step] [Cores] [Circuits*] [Momentum*]           │
│ (virtualized rows; default sort A→Z or org-defined)                          │
└──────────────────────────────────────────────────────────────────────────────┘
* Circuits/Momentum columns appear only when Vitality toggle is On
  (Circuits Attention in Huddle is always visible regardless of toggle)
"""

---

## **2) URL-driven state (no-auth, shareable)**

Example:
`/team/<teamId>?pillar=operational_excellence&focus=baseline_ready&timeframe=7d&segment=preflight&vitality=1&leverage=hi_low&q=rollout`

### **Required params**

* `pillar`: `all | <pillarId>`
* `focus`: `baseline_ready | break_less | ship_faster | reduce_friction`
* `timeframe`: `24h|7d|30d`
* `segment`: `all|flight_ready|preflight|telemetry_missing`
* `vitality`: `0|1`
* `leverage`: `default|hi_low|hi_med|custom`
* `q`: search query (optional)
* `view`: optional sub-view anchors (e.g., `closest_wins`, `top_blocker`)

---

## **3) Component Spec**

## **3.1 Team Command Bar**

### **Controls**

* Search services (typeahead)
* Pillar selector (All + each pillar)
* Focus mode selector (canonical)
* Timeframe selector (affects vitality and Momentum series)
* Segment filter (All / Flight Ready / Pre-flight / Telemetry Missing)
* Vitality toggle (On by default for Squad Console — team leads need operational context)
* Leverage filter toggle (On default)

### **Behavior rules**

* Changing a control updates URL params.
* URL param change triggers data reload (or cached keyed by params).

---

## **3.2 The Huddle Header (Squadron Dashboard)**

**Goal:** orient quickly with an instrument-panel cockpit feeling — per-pillar readiness, closest wins, trajectory, coverage, circuits attention, and main blocker all at a glance.

### **Contains**

1. **Overall Readiness Spine** (full-width distribution bar)
2. **Pillar Readiness Panel** (per-pillar mini ReadinessSpine bars with service counts — left column)
3. **Closest Wins** (count of "1 step away" services — right column)
4. **Trajectory** (momentum sparkline + direction label — right column)
5. **Telemetry Coverage** (confidence metric — right column)
6. **Circuits Need Attention** (always visible, not gated by vitality toggle)
7. **Main Blocker** (top blocker summary, one-line, curated)
8. Quick actions:
   * "View Closest Wins" → Opens the **Closest Wins Drawer**
   * "View Circuits" → Opens the **Vitality Drawer** for degrading/down circuits
   * "View Blocker" → Opens the **Upgrade Brief Drawer** for the top blocker recommendation

### **V3 Design Decisions (Locked)**

* **Pillar Readiness is always visible in the Huddle.** Answers the TL question: "Which pillars are my team strong/weak on?" Uses the same ReadinessSpine (mini variant) per pillar.
* **Circuits Attention is always visible in the Huddle.** The vitality toggle (`showVitality`) only controls Circuits and Momentum columns in the Services Table — it does NOT affect the Huddle.
* **2-column instrument panel layout:** Pillar Readiness on the left, Closest Wins + Trajectory + Coverage stacked on the right. Dense, glanceable, cockpit-inspired.
* **Backend-owned pillar metadata.** The frontend receives `pillarOrder`, `pillarNames`, and `pillarDescriptions` from the API — no config lookups in the View layer.

### **Visual requirements**

* Scannable distribution bar using canonical states.
* Per-pillar bars use ReadinessSpine variant="mini" with compact inline counts.
* Must be calm and readable — instrument panel, not alarm panel.
* If segment filter active, show that context.

### **Data required**

"""ts
interface TeamHuddle {
  teamId: string;
  teamName: string;
  readinessDistribution: ReadinessDistribution;
  totalServices: number;
  closestWinsCount: number;
  coveragePct: number;
  lastEvaluatedAt: string;
  topBlocker: UpgradeRecommendation | null;
  momentum: MomentumData;
  pillarReadiness: {
    distributions: Record<string, ReadinessDistribution>;
    pillarOrder: string[];
    pillarNames: Record<string, string>;
    pillarDescriptions: Record<string, string>;
    totalServices: number;
  };
  coachingLabels: {
    signalCoverage: TopicVocabulary;
    closestWins: TopicVocabulary;
    nominalCircuits: TopicVocabulary;
  };
}
"""

### **Closest Wins definition (locked)**

* Services missing exactly **one** required baseline item for the selected pillar (or overall if pillar=all).
* "Telemetry Missing" does not count as "closest win"; it counts as "unblock."

---

## **3.3 Suggested Upgrades (Curated, the core doing engine)**

**Goal:** convert reality into action. This is the team's weekly plan surface.

### **Display rules**

* Show **3–5 items** max by default.
* "See all upgrades" expands into grouped lists:
  1. Unblock Telemetry
  2. Reach Baseline
  3. Reduce Friction
  4. Optimize
* Each item follows: **Judgement → Evidence → Next Action**.

### **Upgrade card anatomy**

"""
Title (actionable)
Implication (why it matters) - 1–2 lines
Remediation (how to fix) - 1–2 lines

Impact: HIGH   Effort: S/M   Moves: 9 services toward Flight Ready
Evidence: 90% coverage · last eval 4m ago
Primary signals: GitOps manifests, CI pipeline

[View Evidence] [Open Services]
"""

### **Sorting**

* Default uses Leverage Filter: High impact + Low/Med effort.
* Must never sort by "worst teams/services," but can sort by "moves most services" when focus=baseline_ready.

### **Data required**

"""ts
type TeamUpgrade = {
  id: string;
  group: "unblock_telemetry" | "reach_baseline" | "reduce_friction" | "optimize";
  title: string;
  implication: string;
  remediation: string;
  impact: "HIGH" | "MEDIUM" | "LOW";
  effort: "S" | "M" | "L";
  scope: {
    servicesAffected: number;
    servicesMovedTowardReadiness: number;
  };
  evidence: {
    evaluatedCoveragePct: number;
    lastEvalTs: string;
    primarySignals: Array<{ key: string; source: string }>;
  };
  drilldowns: {
    evidenceUrl: string;
    servicesUrl: string;
  };
  vitality?: {
    kind: "circuit" | "none";
    state: "nominal" | "degrading" | "down";
    healthRatio?: number;
  };
  flightHours?: {
    timeframe: "24h" | "7d" | "30d";
    series: number[];
  };
};
"""

---

## **3.4 Team Vitality (optional module, calm)**

**Goal:** protect Circuits as habits of excellence; responsibility without panic.

### **Visibility**

* Vitality toggle controls Circuits/Momentum columns in the Services Table only.
* Circuits Attention in the Huddle is always visible regardless of toggle (see 3.2).

### **Contents**

* Circuit counts: Nominal / Degrading / Down recently
* A list of **max 3 "Fading Circuits"** with fastest fix path
* Team Momentum sparkline (secondary)

### **Visual requirements**

* Nominal circuits: subtle pulse
* Degrading circuits: subtle fade indicator + "fading" label
* No alarm language; no red sirens

### **Data required**

"""ts
type TeamVitality = {
  circuitsNominal: number;
  circuitsDegrading: number;
  circuitsDownRecently: number;
  degradingItems: Array<{
    circuitId: string;
    title: string;
    summary: string;
    timeToBreak?: string;
    remediation: string;
    drilldownUrl: string;
    healthRatio: number;
  }>;
  flightHours: {
    timeframe: "24h" | "7d" | "30d";
    series: number[];
  };
};
"""

---

## **3.5 Services Table (team scope, scannable)**

**Goal:** deep navigation without audit vibes.

### **Default ordering**

* A→Z or org-defined ordering.
* Optional sorting allowed (user-initiated):
  * Next Step (services with an actionable step float up)

**Never default to "worst."** Sort by readiness state, vitality, cores, circuits, or coverage is not allowed (anti-surveillance).

### **Columns (minimum)**

Always:

* Service name (link to Flight Deck)
* Readiness state (for current pillar selection)
* Next Step (progress-first cascade: closestWin priority, topUpgrade fallback) with typeLabel subtitle
* Cores count (foundation snapshot)

When vitality=1:

* Circuits state (nominal/degrading)
* Momentum sparkline (tiny)

### **Row anatomy**

* Clicking any status must never be a dead end:
  * Readiness → show blockers
  * Telemetry Missing → show what's missing + how to connect
  * Next Step → open upgrade brief drawer (remediation + evidence)

### **Data required**

"""ts
type TeamServiceRow = {
  serviceId: string;
  serviceName: string;
  readiness: "flight_ready" | "preflight" | "telemetry_missing";
  nextStep?: {
    title: string;
    typeLabel: string;
    typeDescription: string;
    recommendationKey: string;
  };
  coresCount: number;
  vitality?: {
    circuitsState: "nominal" | "degrading" | "none";
    healthRatio?: number;
  };
  flightHours?: {
    series: number[];
  };
  urls: {
    flightDeckUrl: string;
  };
};
"""

### **Perf requirements**

* Table must be virtualized.
* Drilldown details fetched on demand.

### **Design Decisions (Locked)**

These decisions prevent the Services Table from becoming a surveillance tool:

1. **`closestWin` and `topSuggestedUpgrade` merged into a single `nextStep` field.** Priority cascade: closestWin first (progress-first per charter P1), topUpgrade as fallback.
2. **`typeLabel` shown as subtitle** per charter 3-question contract: What is the state? / Why it matters? / What to do next?
3. **Sort restricted to Name (default A→Z) and Next Step only.** No sort by readiness state, coverage, cores, circuits, or vitality — prevents "sort → screenshot → blame" pattern.
4. **Cores count visible at team altitude.** Not sortable.
5. **Circuits display uses state text** ("N nominal" / "M degrading"), not "X/Y" ratio.
6. **Next Step click opens upgrade brief drawer** with full evidence, remediation, and drilldown.

---

## **3.6 Trust Panel (stable right rail)**

**Goal:** reinforce permission, trust, and correct usage.

### **Contents (short, stable)**

* Why Ground Control exists (2 lines)
* Coverage + "what Telemetry Missing means"
* Evidence contract: "Every status is clickable to signals + timestamp"
* How to use: "Pick one closest win per week"

No preachiness. Calm and practical.

---

## **4) Interaction Flows (the ritual loop)**

### **Flow 1 — Orient**

Open page → see Team Runway + Pillar Readiness + Closest Wins + Coverage + Circuits Attention.

### **Flow 2 — Choose**

Switch Focus Mode → Suggested Upgrades and "Top Blocker" update.

### **Flow 3 — Act**

Click an upgrade → open:

* Evidence view (signals + rule + timestamps)
* Filtered service list
* Flight Deck deep-link for implementation details

### **Flow 4 — Validate**

Vitality module shows Circuit recovery; Momentum sparkline ticks over time; "This changed since last week" appears in a small factual strip (optional future).

---

## **5) Data endpoint suggestion**

### **`GET /team/<teamId>/console?pillar=&focus=&timeframe=&segment=&leverage=&q=`**

Returns:

* `TeamHuddle` (includes pillarReadiness)
* `TeamUpgrade[]`
* `TeamVitality` (optional / only when vitality=1)
* `TeamServiceRow[]`

Cache by params with short TTL (1–3 min).

---

## **6) Empty/Edge states**

* **Telemetry Missing dominates:**
  Show a single calm message in Huddle:
  "Most insights require GitOps/CI signals. Start with Unblock Telemetry."
  Then surface Unblock upgrades first.
* **No upgrades under leverage filter:**
  "No high-leverage upgrades detected under current filters."
  Suggest switching focus/pillar or broadening leverage.
* **All Circuits Nominal, not all Flight Ready (Tier B — Crew Encouragement):**
  Warm acknowledgment in Vitality Attention: "All circuits nominal across N services."
  Redirect: "M services still pre-flight — keep building."
  No Bowie theme. Subtle check icon, muted styling.
* **All Circuits Nominal AND all services Flight Ready (Tier C — Earned Celebration):**
  Rare, earned celebration. Bowie accent: "Ground Control to Major Tom — full orbit."
  Sparkles, warm styling. Short and brief.

---

## **7) Acceptance Criteria (must pass)**

* No default ranking by worst services.
* Top of page is the Huddle: runway truth + pillar readiness + closest wins + coverage.
* Pillar readiness always visible in Huddle — shows per-pillar distribution for the team's services.
* Circuits Attention always visible in Huddle — not gated by vitality toggle.
* Vitality toggle only controls Circuits/Momentum columns in Services Table.
* Suggested Upgrades are curated (3–5) and always include implication/remediation/evidence.
* Momentum remains secondary: sparkline-only, no "levels," no leaderboards.
* All view state encoded in URL params and shareable.
