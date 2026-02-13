# Ground Control -- Screenshots

A visual tour of the platform at all three altitudes (organization, team, service) and across its coaching surfaces (drawers, flight manual, telemetry panels).

---

## Mission Map (Organization Level)

Organization-level coaching cockpit. Pillar readiness bars, closest wins count, trajectory spark, and suggested upgrades. The banner reads: "Ground Control is a coaching cockpit -- not a scoreboard."

![Mission Map](Screenshot%202026-02-13%20at%2013.14.41.png)

---

## Teams List

All teams in the organization with readiness bars, closest wins, signal coverage, trajectory sparks, and top recommended upgrade per team. Sortable by name or closest wins.

![Teams List](Screenshot%202026-02-13%20at%2013.14.53.png)

---

## Squad Console -- Huddle Header

Team-level huddle view for a single team. Shows pillar readiness breakdown, closest wins, trajectory with trend direction, and coaching alerts: "Circuits Need Attention" and "Main Blocker" with actionable links.

![Squad Console Huddle](Screenshot%202026-02-13%20at%2013.15.13.png)

---

## Squad Console -- Next Upgrades and Services Table

Team-level view showing three Next Upgrade cards with coaching language ("Why" and "Next Step"), the services table with status, next step, Cores earned, Circuit health, and trajectory. Includes the "Why Ground Control exists" sidebar.

![Squad Console Services](Screenshot%202026-02-13%20at%2013.15.18.png)

---

## Flight Deck (Service Level)

Service-level coaching cockpit for a single service. Shows per-pillar readiness status, 4 nominal circuits, cores earned (1 of 4 required), trajectory (gaining), and the top recommendation with "Why it matters" and "How to do it."

![Flight Deck](Screenshot%202026-02-13%20at%2013.15.27.png)

---

## Flight Deck -- Protocol View (Below Baseline)

Flight readiness progress bar (5 of 8 protocols met). Below-baseline filter showing expanded rulepack detail with blockers: specific rules that need attention, observed values, and coaching text.

![Protocol View](Screenshot%202026-02-13%20at%2013.15.36.png)

---

## Telemetry Drawer -- Not Yet Met

Telemetry signal readout for a rulepack that has not yet met its baseline. Shows conclusion ("2 of 3 blocking rules need attention"), badge-affecting rules, and raw signal checks with observed values (Found, 0/1 patterns, Not found).

![Telemetry Not Met](Screenshot%202026-02-13%20at%2013.15.47.png)

---

## Telemetry Drawer -- Gold Achieved

Telemetry signal readout for a passing rulepack. Shows "Gold achieved" with the scoring criteria (Gold <= 480), the observed signal (Deployment Speed Check: P90 4m 30s), debug data, and a link to the source system (GitLab pipelines).

![Telemetry Gold](Screenshot%202026-02-13%20at%2013.15.52.png)

---

## Upgrade Advisory Drawer

Detailed upgrade recommendation brief. Shows the specific rulepack, "Why This Matters" coaching text, "Next Step" action with concrete instructions, scope, evaluation coverage, and freshness timestamp.

![Upgrade Advisory](Screenshot%202026-02-13%20at%2013.15.58.png)

---

## Flight Manual -- Protocol Detail

The Flight Manual overlay showing a specific protocol (Safety Net) under Production Health. Displays the ambition, standard with threshold, evaluation method (Circuit/Vitality), signal sources, badge semantics (Gatekeeper), urgency profile, and a "Propose change in Git" link.

![Flight Manual Protocol](Screenshot%202026-02-13%20at%2013.16.15.png)

---

## Flight Manual -- Legend

Shared vocabulary reference. Defines all states, types, and categories used across the platform: Readiness States (Flight Ready, Pre-flight, Telemetry Missing), Badge Types (Ladder, Gatekeeper), Achievement Types (Core, Circuit), and Check Statuses.

![Flight Manual Legend](Screenshot%202026-02-13%20at%2013.16.19.png)

---

## Vitality Status Drawer

Circuit health overview showing down circuits that need attention. Lists affected services with the specific degrading circuit (Reliable CI, Right-Sizing) and links to each service's Flight Deck.

![Vitality Status](Screenshot%202026-02-13%20at%2013.16.36.png)

---

## Mission Focus (Spotlight) Drawer

Organization-level spotlight: "Secure the Perimeter." Shows the org focus with coaching action ("Adopt a canary rollout strategy"), impact/effort tags, included protocols (Safe Releases, Safety Net, Service Coupling), and affected services with search.

![Mission Focus](Screenshot%202026-02-13%20at%2013.16.49.png)

---

## Closest Wins Drawer

Shows 7 services that are one upgrade away from Flight Ready. Includes readiness impact projection (0% today to 1% projected), coaching rationale ("These 7 services have done most of the work"), common blockers by rulepack, and per-service detail.

![Closest Wins](Screenshot%202026-02-13%20at%2013.16.54.png)

---

## Console Boot Sequence (Easter Egg)

The developer console easter egg. On load, Ground Control prints a retro system boot: "GROUND CONTROL / SYSTEM CODEX v4.0" with status checks (Telemetry Stream ESTABLISHED, Coaching Protocol ACTIVE, Standards Transparency ONLINE, User Agent ENGINEER DETECTED) and a rotating Bowie lyric -- "The stars look very different today."

![Console Boot Sequence](Screenshot%202026-02-13%20at%2017.04.48.png)

---

## Context Strip (FMA Annunciator)

The Flight Mode Annunciator bar at the bottom of the viewport. Shows the currently active pillar filter ("Delivery Confidence") and its focus description ("Break Less"). This persistent strip provides context when a user has filtered the view, ensuring they always know which lens they are looking through.

![Context Strip](Screenshot%202026-02-13%20at%2017.05.09.png)
