# **Ground Control — Mission Map (Main View) V2**

## **“Runway Campaigns” Wireframe & Implementation Spec**

### **Purpose of this screen**

Mission Map is the **cultural landing page**: it repeatedly reinforces that **engineering excellence is allowed**, and turns that permission into **ownership \+ action**.

This view must do two things simultaneously:

1. **Anchor direction** in a calm, truthful north star (**Readiness Spine**).  
2. **Drive doing** with curated, high-leverage work (**Campaign Cards**).

**Not a leaderboard. Not a red dashboard.**  
It is a coaching cockpit at org scale (60 teams / \~1300 services).

---

## **1\) UX Narrative (what Roi wants people to feel)**

When an engineer opens Ground Control on a Monday:

* They immediately see **where we’re heading** (Flight Ready truth, not opinion).  
* They immediately see **what to do next** (closest wins, impact/effort).  
* They feel **permission to care** (calm tone, purpose, evidence, no shame).  
* They see excellence as **habit and trajectory**, not perfection (Vitality shows current state via Circuits; Momentum shows trajectory over time).

---

## **2\) Page Layout (desktop first, responsive to widescreen \+ TV mode)**

### **High-level structure**

1. **Global Command Bar** (filters \+ focus)  
2. **Readiness Spine** (org runway distribution — scannable, not rank)  
3. **Campaign Deck** (3–4 dynamic cards — the “face” that drives action)  
4. **Team Matrix** (60 teams — scannable rows, not sorted by “worst”)  
5. Optional: **Right Rail** (Purpose \+ trust contract \+ quick help)

### **Wireframe (desktop)**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Ground Control · Mission Map                             [Share] [Help]      │
│ [Search services/teams]  [Pillar: All ▼] [Focus: Baseline Ready ▼]           │
│ [Timeframe: 7d ▼] [Vitality Layer: Off/On] [Leverage: High Impact+Low Effort]│
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ READINESS SPINE (Org-wide Runway)                                             │
│ [Flight Ready ███████] [Pre-flight ████████████] [Telemetry Missing ████]     │
│ 1300 services · Coverage: 92% evaluated · Last eval: 4m ago                   │
│ (Click segment filters below; never shows “worst teams”)                      │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ CAMPAIGNS (Closest Wins)                                                      │
│ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐     │
│ │ Campaign Card │ │ Campaign Card │ │ Campaign Card │ │ Campaign Card │     │
│ └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┬───────────────┐
│ TEAMS (60)                                                    │ PURPOSE PANEL │
│ ┌───────────────────────────────────────────────────────────┐│ (Stable)      │
│ │ Team Row                                                   ││ - Why GC      │
│ ├───────────────────────────────────────────────────────────┤│ - Trust rules │
│ │ Team Row                                                   ││ - How to use  │
│ ├───────────────────────────────────────────────────────────┤│               │
│ │ ... (virtualized)                                          ││               │
│ └───────────────────────────────────────────────────────────┘│               │
└──────────────────────────────────────────────────────────────┴───────────────┘
```

### **TV Mode (future, but design-compatible now)**

* Hides most controls; cycles Focus Mode and/or pillar.  
* Big Readiness Spine \+ 3 Campaign Cards \+ top 10 teams by “closest wins” **without labeling as rank** (e.g., “Teams closest to baseline today”).

---

## **3\) Global Command Bar (stateful, URL-driven)**

### **Controls (left → right)**

* **Search** (services/teams; typeahead; client-side debounce)  
* **Pillar selector**: `All | Operational Excellence | Production Health | ...`  
* **Focus Mode** (canonical):  
  * Baseline Ready  
  * Break Less  
  * Ship Faster  
  * Reduce Friction  
* **Timeframe** (for vitality \+ Momentum sparkline): 24h / 7d / 30d  
* **Vitality Layer toggle** (Off by default): shows Circuits \+ momentum cues at org/team/campaign level  
* **Leverage Filter toggle** (On by default):  
  * “High Impact \+ Low/Med Effort”  
  * Advanced: Impact \= High/Med/Low, Effort \= S/M/L

### **URL params (must be implemented)**

No-auth, shareable, persistent state.

Example:  
`/mission-map?pillar=operational_excellence&focus=baseline_ready&timeframe=7d&vitality=1&segment=preflight&leverage=himed`

**Required params**

* `pillar`: string | `all`  
* `focus`: enum  
* `timeframe`: `24h|7d|30d`  
* `vitality`: `0|1`  
* `segment`: `all|flight_ready|preflight|telemetry_missing`  
* `leverage`: `default|hi_low|hi_med|custom`  
* `q`: search query

---

## **4\) Component Spec**

## **4.1 Readiness Spine (Org-wide Runway)**

**Goal:** show “where we are” without ranking or blame.

### **Visual requirements**

* Horizontal distribution bar segmented into:  
  * ✅ Flight Ready  
  * 🟡 Pre-flight  
  * ⛔ Telemetry Missing  
* Shows counts \+ %.  
* Shows **Coverage** (% evaluated) and **Last evaluation time**.  
* Clicking a segment applies a filter (`segment=`) that affects campaigns \+ team matrix.

### **Behavior requirements**

* **Reactive to Pillar selection**:  
  * If pillar \= All: distribution reflects overall readiness.  
  * If pillar \= X: distribution reflects pillar mastery readiness.  
* **Never** convert into a ranked list on click.  
  * Segment click filters the team matrix but keeps default ordering (alphabetical or org structure).

### **Data required (API)**

```ts
type ReadinessSpine = {
  scope: { pillarId: string | "all" };
  totals: { services: number; teams: number }; // teams = teams with active services (not all Backstage teams)
  counts: {
    flightReady: number;
    preFlight: number;
    telemetryMissing: number;
  };
  evaluatedCoveragePct: number; // evaluated services / total
  lastEvalTs: string; // ISO
};

// V3: Organization Vitality (time-based circuit health)
type OrgVitality = {
  nominalCircuitsCount: number;      // circuits currently passing
  degradingCircuitsCount: number;    // circuits approaching expiry window
  downCircuitsCount: number;         // circuits that have expired
  totalCircuitsCount: number;        // total evaluated circuits
  nominalPct: number;                // 0-100, percentage of nominal circuits
  window: MomentumTimeframe;         // '7d' | '30d' | '90d'
  trend: MomentumTrend;              // 'up' | 'down' | 'flat'
  allNominal: boolean;               // true when all circuits are nominal (celebration trigger)
};
```

---

## **4.2 Campaign Deck (3–4 cards, curated, dynamic)**

**Goal:** this is the “doing engine.” It should make action feel obvious and valuable.

### **Card selection principles**

* Must be **Closest Wins**, not static banners.  
* Must follow **Judgement → Evidence → Next Action**.  
* Must be filtered by **Leverage Filter** (impact/effort).  
* Must maintain diversity:  
  * Avoid 4 cards that are basically “unblock telemetry” unless focus demands it.  
  * Prefer 1–2 baseline campaigns \+ 1 vitality-protection campaign \+ 1 friction campaign.

### **Campaign card anatomy (wireframe)**

```
┌─────────────────────────────────────────────────────────┐
│ Title (Actionable)                                      │
│ 1-line WHY (Implication)                                │
│                                                         │
│ Impact: HIGH   Effort: S/M   Scope: 8 teams · 120 svcs   │
│ Closest Wins: 34 services are 1 step away                │
│ Evidence: 92% coverage · last eval 4m ago
| Actionability: [Ready | Blocked by Telemetry]              │
│                                                         │
│ Next Action (Remediation):                               │
│ “Connect GitOps manifests for rollout analysis”          │
│ [View Evidence]  [Open Campaign]                         │
└─────────────────────────────────────────────────────────┘
```

### **Campaign types (minimum set)**

Campaigns should come from three sources, depending on focus:

1. **Unblock Telemetry** (only when relevant)  
2. **Baseline Readiness** (relatively stable, compounding improvements)  
3. **Vitality Protection** (Circuits in DEGRADING or DOWN state requiring attention)

### **“Closest Wins” definition (locked)**

* For **Cores/Foundation**: services missing exactly **N=1** rule for a baseline requirement.  
* For **Circuits/Vitality**: services where circuit health is **degrading** (near window break) *or* recently down and can be restored with a single fix path.  
* A service is only a 'Closest Win' if its `telemetryMissing` status for the required check is `false`

The UI should label “Closest Wins” as “1 step away” and never “failing services.”

### **Leverage score (for sorting cards)**

Implemented server-side if possible; otherwise client-side with provided fields.

```ts
leverageScore = (impactWeight / effortWeight) * scopeWeight * confidenceWeight
// impactWeight: High=3 Med=2 Low=1
// effortWeight: S=1 M=1.6 L=2.4
// scopeWeight: log(servicesAffected + 1)
// confidenceWeight: evaluatedCoveragePct / 100
```

### **Data required (API)**

```ts
type CampaignCard = {
  id: string;
  pillarId: string | "all";
  focusMode: "baseline_ready" | "break_less" | "ship_faster" | "reduce_friction";
  title: string; // action-oriented
  implication: string; // 1–2 lines, calm
  remediation: string; // 1–2 lines, concrete
  impact: "HIGH" | "MEDIUM" | "LOW";
  effort: "S" | "M" | "L";
  scope: {
    teamsAffected: number;
    servicesAffected: number;
    closestWinsServices: number; // exactly N away
  };
  evidence: {
    evaluatedCoveragePct: number;
    lastEvalTs: string;
    primarySignals: Array<{ key: string; source: string }>;
  };
  vitality?: {
    kind: "circuit" | "none";
    state: "nominal" | "degrading" | "down";
    healthRatio?: number; // 0..1 (for glow/fade)
  };
  flightHours?: {
    timeframe: "24h" | "7d" | "30d";
    series: number[]; // sparkline points (non-comparable)
  };
  drilldowns: {
    openCampaignUrl: string; // pre-filtered view
    evidenceUrl: string; // shows top failing rule evidence
  };
};
```

---

## **4.3 Team Matrix (60 teams, scannable, non-ranking)**

**Goal:** provide structured navigation \+ local truth without turning into "who's worst."

### **Design Decisions (Locked)**

These decisions prevent the Team Matrix from becoming a surveillance tool:

1. **Section heading is "Teams"**, not "Team Readiness." The matrix is a navigation surface, not a report card.
2. **Sort options: only `Name` (default A→Z) and `Closest Wins` (opt-in, progress-oriented).** No `coverage` or `services` sort — sorting by coverage enables ranking ("sort → screenshot → blame").
3. **Always-visible per row:** readiness bar, closest wins, coverage %, momentum sparkline, **topUpgrade** (coaching action).
4. **Raw `coreCount` is not shown at org altitude.** Foundation detail lives in Squad Console / Flight Deck where coaching context exists.
5. **Circuit display is vitality-toggle-only**, showing "N nominal / M degrading" (not "X/Y" ratio). Ratios create scorecard energy.
6. **Every row must answer the charter's 3-question contract:** What is the state? (readiness bar) / Why it matters? (coverage) / What to do next? (topUpgrade).

### **Ordering**

Default ordering: **Org structure** or **A→Z**.  
Optional sort: "Closest wins (count)" but **never default**.

### **Team row anatomy**

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│ Team Name (N svcs) | Readiness Bar | Closest Wins | Coverage % | Momentum | Top Upgrade | > │
│                                                                                    │
│ Vitality (if toggled): Circuits nominal 14 · degrading 3  (pulse/fade cues)        │
└────────────────────────────────────────────────────────────────────────────────────┘
```

Hovering over Readiness mini-bars or Circuits must trigger a `Contextual Callout` showing: 1\) Top blocking rule name, 2\) Link to evidence, 3\) Estimated effort to resolve.

### **Vitality Layer behavior (toggle)**

When `vitality=1`, show:

* counts of nominal vs degrading Circuits  
* subtle pulse for nominal  
* subtle fade overlay for degrading  
* no alarming colors; use icons \+ text; fade is a “trend warning” not a red siren

### **Data required (API)**

```ts
type TeamRow = {
  teamId: string;
  teamName: string;
  readiness: {
    flightReady: number;
    preFlight: number;
    telemetryMissing: number;
    evaluatedCoveragePct: number;
  };
  closestWins: {
    servicesOneStepAway: number;
    topBlockerRulepackId?: string;
    topBlockerSummary?: string; // short
  };
  vitality?: {
    circuitsNominal: number;
    circuitsDegrading: number;
    circuitsDown: number; // V3: circuits that have exceeded expiry window
    healthRatio?: number; // 0..1 aggregate
  };
  flightHours?: {
    timeframe: "24h" | "7d" | "30d";
    series: number[];
  };
  // V11: Curated coaching action — uses canonical UpgradeRecommendation
  topUpgrade: UpgradeRecommendation | null;
  urls: {
    squadConsoleUrl: string;
  };
};
```

### **Scale/perf requirements**

* Team Matrix must be **virtualized** (even at 60 rows it’s fine; do it anyway for future).  
* Team rows should not require fetching per-service lists unless user drills in.

---

## **4.4 Purpose \+ Trust Panel (right rail, stable)**

**Goal:** this is the “permission layer” that sets culture without preaching.

### **Content rules**

* Always calm, short, and practical.  
* Never guilt.  
* Never “company propaganda.”

### **Suggested structure**

1. **Why Ground Control exists** (2 lines)  
2. **How we measure fairly** (2 bullets)  
3. **How to use this** (1–2 bullets)

Example copy (template):

* “Ground Control helps teams stay Flight Ready through small, high-leverage upgrades.”  
* “Every status is evidence-backed and recoverable.”  
* “Start with one ‘Closest Win’ per week.”

Optional Bowie accent: one subtle line in tiny text, rarely rotated (e.g., “Stay curious. Stay ready.”) — avoid quotes unless you’re sure.

---

## **5\) Visual System (Mission Control \+ subtle Bowie)**

### **Core rules**

* **Clarity first:** high contrast, readable at distance (TV later).  
* **No drama:** fading is informative, not alarming.  
* **Momentum is a visual layer**, not a new concept:  
  * Nominal Circuit → soft pulse  
  * Degrading Circuit → softened/washed indicator \+ “fading” label  
  * Down Circuit → neutral “inactive” state

### **Cores vs Circuits cues**

* Cores: stable icons, no animation.  
* Circuits: small animated dot/halo only when vitality layer is on.

### **Momentum visualization**

* Always a **sparkline** only.  
* No numeric “level” by default.  
* Optional: tooltip “Momentum reflect sustained vitality \+ completed foundations.”  
* Sparklines are locally normalized (min/max of the team's own 30d range). The Y-axis height is not comparable between different team rows.

---

## **6\) Interaction Flows**

### **Flow A — default landing**

* Pillar \= All, Focus \= Baseline Ready, Vitality \= Off, Leverage \= On  
* Readiness spine shows org distribution  
* Campaign cards show 3–4 highest leverage closest wins  
* Team list shows stable view \+ one top suggested upgrade per team

### **Flow B — pillar selection**

* Updates spine \+ campaigns \+ team readiness mini-bars  
* Keeps ordering stable  
* Keeps URL in sync

### **Flow C — segment click on spine**

* Applies filter (`segment=preflight` etc.)  
* Campaign cards re-calc within segment scope  
* Team rows show same, but counts reflect filtered segment

### **Flow D — campaign click**

* “Open Campaign” leads to a filtered campaign view (can be a mode of Squad Console or a new route)  
* Must preserve “Judgement → Evidence → Next Action” and show scope breakdown (teams/services affected)

### **Flow E — vitality toggle**

* Adds pulse/fade cues \+ circuit counts \+ Momentum sparklines  
* Does not change readiness truth  
* Must remain calm (no sirens)

### **Flow F — vitality drill-down (V3)**

* Clicking on "Degrading Circuits" or "Down Circuits" counts opens **Vitality Brief Drawer**
* Drawer shows:
  * Affected services count and list (top 10)
  * Per-service circuit details (rulepack name, state, timing info)
  * "What This Means" section explaining the state
* When all circuits are nominal (`allNominal=true`), clicking shows **celebration UI**:
  * Thematic message: "Ground Control to Major Tom — orbit stable."
  * Subtitle: "All {count} circuits are nominal."
  * Sparkles icon for visual celebration

---

## **7\) Data Loading Strategy (critical for 1300 services)**

### **Required endpoints (suggested)**

* `GET /mission-map/summary?pillar=&focus=&timeframe=&segment=`  
  returns: `ReadinessSpine`, `CampaignCard[]`, `TeamRow[]`

* `GET /mission-map/vitality/:state/brief?teamId=` (V3)  
  * `:state` = `degrading` | `down`
  * Optional `teamId` filter for team-scoped view
  * Returns: `VitalityBrief` (see below)

```ts
// V3: Vitality Brief Drawer response
type VitalityBrief = {
  state: 'degrading' | 'down' | 'nominal';  // requested state or 'nominal' if all healthy
  circuitsCount: number;                     // total circuits in this state
  servicesAffectedCount: number;             // unique services affected
  topServices: VitalityServiceItem[];        // top 10 services with circuits in state
  allServicesNominal: boolean;               // true = show celebration
  celebration?: {
    text: string;        // "Ground Control to Major Tom — orbit stable."
    subtitle: string;    // "All {n} circuits are nominal."
  };
};

type VitalityServiceItem = {
  serviceId: string;
  serviceName: string;
  teamId: string;
  teamName: string;
  circuits: VitalityCircuitInfo[];
};

type VitalityCircuitInfo = {
  rulepackId: string;
  rulepackTitle: string;
  state: 'nominal' | 'degrading' | 'down';
  lastPassedAt?: string;  // ISO timestamp
  expiresAt?: string;     // for degrading: when it will become DOWN
  lostAt?: string;        // for down: when it became DOWN
};
```

### **Pagination**

* Team rows can load in one call (60 rows).  
* Campaign drilldowns load on demand:  
  * `GET /campaigns/:id` returns affected teams/services list \+ top rules.

### **Caching**

* Cache summary response for short TTL (e.g., 1–3 minutes) keyed by query params.

---

## **8\) Empty/Edge States (must be designed)**

* **Telemetry Missing dominant:** show a calm nudge:  
  * “Most insights require GitOps/CI signals. Start with Unblock Telemetry campaign.”  
* **Low coverage:** always show coverage % and confidence label.  
* **No campaigns available:** show:  
  * “No high-leverage upgrades detected under current filters.”  
  * Suggest switching focus/pillar.  
* **All Circuits Nominal, not all Flight Ready (Tier B — Crew Encouragement):**  
  * Warm acknowledgment: "Fleet vitality holding steady."  
  * Redirect: "{N}% of services at Flight Ready — keep building."  
  * No Bowie theme. Summary card layout, muted styling.  
* **All Circuits Nominal AND all services Flight Ready (Tier C — Earned Celebration):**  
  * Bowie accent: "Ground Control to Major Tom — the fleet is flying."  
  * Centered layout with sparkles icon and glow effect. Rare and earned.

---

## **9\) Acceptance Criteria (from charter, enforced here)**

* No default ranking of teams or services.  
* Campaigns are dynamic closest-win engines, not banners.  
* Every campaign has implication \+ remediation \+ evidence.  
* Vitality layer is optional and calm; never becomes monitoring theater.  
* Momentum are secondary, non-competitive, sparkline-only by default.  
* URL parameters fully reproduce view state (shareable, persistent).  
* The page must feel like coaching and permission, not auditing.

---

## **10\) Implementation Notes for the AI frontend assistant**

* Build components as independent modules:  
  * `CommandBar`  
  * `ReadinessSpine`  
  * `CampaignDeck` / `CampaignCard`  
  * `TeamMatrix` / `TeamRow`  
  * `PurposePanel`  
* Global state is derived from URL params; UI updates write params.  
* Use virtualization for TeamMatrix.  
* Animations must be subtle and accessible; provide “reduce motion” support.  
* Tooltips must surface “why \+ evidence link” not just numbers.

