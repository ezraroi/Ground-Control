# Ground Control Charter v4.0

**Voice • Tone • UX Behavior • Theme Coherence**

> **Foundation Documents:** For immutable definitions (hierarchy, states, relationships), see the [Constitution](./Ground_Control_Constitution.md). For implementation guidance (UX specs, guardrails, configuration), see the [Product Definition](./Product_Definition_Ground_Control_V4.md).

---

## 0. What This Charter Is For

Ground Control is a developer-first system that drives **engineering excellence bottom-up** by making reality visible and improvement achievable.

This charter ensures Ground Control's **voice, tone, and UX behavior stay consistent and coherent over time**. It is the authority on *how we communicate*, not on *what the system does* (see Constitution) or *how it's built* (see Product Definition and Technical Spec).

**Primary audience:** Engineers  
**Secondary:** Tech leads / team leads (planning + reflection)  
**Optimizes for:** Engineering excellence, craftsmanship, time saved, friction reduced, safer deploys, fewer incidents, confidence

---

## 1. Principles (Stable)

These change rarely.

1. **Clarity beats theme.**  
   Theme is an accent layer; meaning must be instantly understandable without references.

2. **Progress over perfection.**  
   Default views celebrate improvement, near-wins, and completions—never shame.

3. **Evidence-backed truth.**  
   Every judgement (blocked, baseline, risk, vitality state) must be inspectable: signals, thresholds/window, evaluation time, contract version.

4. **Purpose is part of the product contract.**  
   Ground Control is not a scoreboard. The system must always answer:  
   - *What is the state?*  
   - *Why it matters?*  
   - *What to do next?*  
   with minimal friction.

5. **Vitality is the "now" that outranks the "nice-to-have."**  
   When vitality is degrading, Ground Control must treat it as **priority work**—calmly, without drama, but unmistakably.

6. **Risk exists to prioritize action.**  
   Risk is never presented as condemnation; it guides the next best upgrade.

7. **Engineers' time is sacred.**  
   Prefer small, high-leverage recommendations over exhaustive lists. Curate by default.

8. **Autonomy matters.**  
   Always provide paths to choose focus (e.g., "Reduce Friction", "Ship Faster", "Break Less", "Baseline Ready").  
   Autonomy is expressed through **focus + prioritization**, not through "build your own rules" UX (for now).

9. **Trust + transparency over enforcement.**  
   Ground Control treats engineers as adults. Where misuse is possible (no-auth constraints), rely on **system truth (physics)** and **visibility (social friction)** rather than permission gates.

10. **Trajectory over ranking.**  
    The default experience must never push the product into "league table" behavior. Comparison (if ever) must be opt-in, scoped, and honest about telemetry coverage.

11. **Theme must not reduce trust.**  
    No humor in high-severity contexts. Delight is earned and rare.

---

## 2. Tone Model (Stable, Context-Driven)

Tone is selected by context, not by author preference.

### Tier A — Mission Control (Default)

**Use for:** Statuses, evidence, blocked, risk, vitality state changes, system messages.  
Calm, factual, direct. No jokes.

### Tier B — Crew Encouragement (Nudges)

**Use for:** Near-complete prompts, planning guidance, "next win" suggestions.  
Invitational, achievable, outcome-focused.

### Tier C — Celebration (Earned Only)

**Use for:** New completions, meaningful stability holding, major improvements.  
Short, rare, warm.

**Hard rule:** Tier C never appears on Blocked / High risk / vitality-degrading contexts.

---

## 3. Lexicon (Language Contract)

> **For canonical state definitions** (Readiness, Circuit states, Badge levels), see [Constitution Sections 4-7](./Ground_Control_Constitution.md).

This section defines **preferred terminology and phrasing** for user-facing copy.

### Recommendations Language

| Prefer | Avoid |
|--------|-------|
| Suggested Upgrades | Violations |
| Pre-flight Checklist | Non-compliant |
| Next steps | Infractions |
| Needs attention | Failures |
| Exposure, hotspot, watchlist | "Bad service/team" |

### Achievement Language (Path B)

| Term | Voice Guidance |
|------|----------------|
| **Cores** | "Permanent achievements" — built once, kept forever |
| **Circuits** | "Living health indicators" — must be carried, not owned; make invisible maintenance labor visible |
| **Momentum** | "Trajectory feedback" — never call it a score, rank, or currency |

### Risk Language (Non-Shaming)

| Prefer | Avoid |
|--------|-------|
| Needs attention | Dangerous |
| Exposure | Broken |
| Hotspot | Irresponsible |
| Watchlist | "Your team is failing" |

### Momentum Cues (Visual Only)

Momentum (sparkline/trend arrow) is a **visual indicator** of trajectory over time.  
It reflects the daily rate from active Circuits and must not introduce new jargon or hidden scoring systems.

**Important distinction:**
- **Momentum** shows trajectory (how things are trending over 7d/30d/90d)
- **Vitality** shows current state (NOMINAL/DEGRADING/DOWN right now)

These are separate concepts and must not be conflated in UI or documentation.

**Momentum Display Principles (P-M1 through P-M9):**

See [Product Definition Section 6 — Momentum Display Principles](./Product_Definition_Ground_Control_V4.md) for the full table. Key rules for copy/UX review:

- **P-M2 Non-Competitive:** Momentum must never be sortable, rankable, or comparable between entities.
- **P-M3 Inspectable:** Every Momentum number must be explainable in one hover/click (e.g., "3 active circuits x 10/day").
- **P-M6 No Alarm State:** Declining trend uses neutral color (muted foreground), not amber/red. The warning comes from Vitality, not Momentum.
- **P-M9 Single Visual Source:** All momentum visual decisions (icon, sparkline sizes, trend colors) are defined in `momentum-visuals.ts`. No local overrides.

---

## 4. UX Behavior Patterns (Medium Stability)

Patterns evolve as the product grows, but changes must preserve principles.

### P1) Progress-First Defaults

- Default sorting and summaries emphasize **Closest Wins** and improvement trajectory.  
- No default "worst performers" views.  
- Recognition (new completions, meaningful stability holding) should be visible but not noisy.

### P2) Curate by Default

- Show **3–5** suggested upgrades (team/service) before "see all"  
- Group actions as: **Unblock Telemetry → Protect Vitality → Reach Baseline → Reduce Friction → Optimize**  
- Avoid listing every failing check by default

### P3) Judgement → Evidence → Next Action

Every negative state must provide:

- What it means (short)  
- Why it happened (evidence)  
- What to do next (action path)

Examples:

| State | Copy Pattern |
|-------|--------------|
| Telemetry Missing | "Missing" + "how to connect" + owner hint |
| Go/No-Go pending | Show the 1-2 remaining baseline items + direct link |
| Hold | Show top risk driver + top recommended upgrade |
| Circuit Degrading/Down | Show window truth + what changed + fix path |

### P4) Missing Signals Are Not Silent Failures

- Missing critical signals → **Telemetry Missing** (Blocked), not "failed"  
- Always show coverage (% evaluated) where comparisons exist

### P5) Focus Modes (Autonomy Control)

Provide a small, persistent control that reorders recommendations and sorting:

- Reduce Friction  
- Ship Faster  
- Break Less  
- Baseline Ready

### P6) Foundation vs Vitality Must Be Obvious

In Service and Team contexts:

- Users must quickly distinguish **what is permanent (Cores)** from **what is current (Circuits)**.  
- Circuits must visually communicate window truth (e.g., "nominal", "degrading", "down"), always backed by evidence.

### P7) Vitality-First Prioritization (Calm Urgency)

- When any Circuit is Degrading or Down, Ground Control prioritizes **protecting/restoring vitality** before suggesting "build-once" improvements.  
- Urgency comes from **truth + proximity to down + fixability**, not from alarms, shame, or dramatic language.

### P8) Momentum Stays Secondary

- Momentum must never replace Mastery/Readiness states.  
- It must be displayed as **trajectory instrumentation**, not as a central status or identity.  
- **No public ranking** based on Momentum.

### P9) Transparency Beats Permissions (No-Auth Reality)

When user actions influence how the system is experienced:

- Preserve system truth (window outcomes never change due to a click).  
- Preserve visibility (actions are attributable and inspectable in context).  
- Use transparency to create healthy accountability without governance theater.

### P10) Spotlights Are Nudges, Not Enforcement

- Spotlights **bias recommendation ordering**, not hide or override truth.  
- Only one spotlight may be active at a time.  
- Spotlight multipliers are capped (max 2.0x), never overpowering hard-priority telemetry logic.  
- UI may display a banner and badge for affected recommendations, but must never present spotlight as mandatory compliance.  
- No leaderboards or ranking tied to spotlight participation.

### P11) Standards Transparency

- The Flight Manual overlay provides read-only access to protocol definitions, evaluation physics, and vocabulary.  
- It is a centered HUD (not a drawer), URL-driven via `?manual=` params, and accessible from any view via the CommandBar or contextual entry points on pillar labels, rulepack names, and achievement badges.

---

## 5. Theme Constraints (Medium Stability)

**Theme:** Mission Control foundation + subtle Space/Bowie accent.

### What the Theme May Do

- Shape naming (Ground Control, Mission Map, Flight Deck, Squad Console)  
- Add subtle visual accents (telemetry vibe)  
- Add occasional celebratory lines (Tier C only)  
- Use Momentum visuals (glow/pulse/fade) to support vitality comprehension  
- Reward engineer curiosity via a styled console boot sequence (hidden channel, not product UI — see Product Definition D.2)

### What the Theme Must Not Do

- Reduce readability/contrast  
- Introduce jargon that hides meaning  
- Add jokes during risk/blocked contexts  
- Require cultural knowledge to understand the UI  
- Push the product into RPG/fictional lore tone

### Design Constraints

- Prioritize high contrast and legibility (especially Wall Mode)  
- Limit accent usage per screen (avoid visual noise)  
- Keep iconography simple and consistent

---

## 6. Examples (Illustrative, Not Canonical)

### Mission Control (Tier A)

- "Telemetry Missing: CI duration signal unavailable."  
- "Go/No-Go pending: 1 baseline item remaining."  
- "Circuit degrading: 1 crash detected in the last 24h window."

### Encouragement (Tier B)

- "Closest win: complete Probe Baseline to unlock safer rollouts."  
- "Next waypoint: reduce CI flake rate to protect hotfix speed."

### Celebration (Tier C — Rare)

- "Flight Ready achieved. Good flight."  
- "Vitality holding steady — good craft."

---

## 7. Evolution Protocol

### Versioning

- Charter is versioned (v4.0, v4.1…)  
- **Principles** and **Tone Model** change rarely and require deliberate review  
- **Patterns** can evolve; log notable changes  
- **Examples** can change freely

### Change Triggers (When to Revisit)

- Repeated confusion about a term (especially Vitality vs Momentum)  
- Engineers report "audit vibes"  
- New feature introduces a new state or recommendation class  
- Theme starts to overpower trust/clarity  
- Momentum begins to behave like a score (pressure for leaderboards)  
- Teams experience vitality as "noise" (signals/thresholds/window need refinement)

### Ownership

- Assign a lightweight "Charter Owner"  
- PR checklist uses this charter as acceptance criteria

---

## 8. PR Checklist (Operational Guardrail)

Before shipping UI/copy changes, confirm:

- [ ] Does the screen default to progress-first (Closest Wins / trajectory)?  
- [ ] Are negative states paired with evidence + next action?  
- [ ] Did we avoid shame/compliance language?  
- [ ] Are theme accents optional and not used in high severity?  
- [ ] Are comparisons honest about coverage and telemetry gaps?  
- [ ] Are Cores vs Circuits visually and conceptually distinct?  
- [ ] Does vitality degrade/restore as **window truth** (not vibes)?  
- [ ] Is Momentum secondary and non-competitive (no "rank me" affordances)? **(P-M1, P-M2)**  
- [ ] Is Momentum non-sortable at every level — no column sort, no URL param, no API support? **(P-M2)**  
- [ ] Are Momentum cues backed by inspectable evidence (not "magic")? **(P-M3)**  
- [ ] Does declining momentum trend use neutral color, not amber/red? **(P-M6)**  
- [ ] Under no-auth constraints, do interactions preserve **truth + transparency**?

---

## 9. Product Language Reference

> **For canonical state definitions and values**, see [Constitution Sections 6-7, 11](./Ground_Control_Constitution.md).

### View Names (Canonical)

| View | Name | Purpose |
|------|------|---------|
| Product | Ground Control | — |
| Org Main View | **Mission Map** | Teams/Groups overview |
| Team View | **Squad Console** | Planning + reflection |
| Service View | **Flight Deck** | Status + upgrade workshop |
| Standards Overlay | **Flight Manual** | Protocol definitions + evaluation physics |
| Pillar Focus (future) | **System Console** | Pillar-wide drilldown |
| TV/Wall Mode (future) | **Mission Broadcast** | Display mode |

### Entity Naming

| Entity | UI Term | Data Model |
|--------|---------|------------|
| Org Unit | Group | `Group` |
| Team | Team | `Team` |
| Service | Service | `Service` |
| High-level outcome | Pillar (or "System" sparingly) | `Pillar` |
| Capability module | Rulepack | `Rulepack` |
| Evaluator | Check | `Check` |
| Engine output | Badge | `Badge` |

### Recommendation Labels

| Label | Usage |
|-------|-------|
| **Suggested Upgrades** | Primary section title |
| **Pre-flight Checklist** | Alternate label (optional) |
| **Next best upgrade** | Single-item summary in tables |
| **Opportunities** | "See all" lists |

### Recommendation Grouping (Ordered)

1. **Unblock Telemetry**  
2. **Protect Vitality**  
3. **Reach Baseline**  
4. **Reduce Friction**  
5. **Optimize**

---

*This charter governs voice, tone, and UX behavior. For system definitions, see the Constitution. For implementation guidance, see the Product Definition.*