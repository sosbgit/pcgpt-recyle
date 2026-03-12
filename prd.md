---
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys, step-05-domain, step-06-innovation, step-07-project-type, step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish, step-12-complete]
status: complete
classification:
  projectType: Self-hosted web platform with persistent backend/data-processing services
  domain: Modular geospatial intelligence platform
  complexity: High
  projectContext: Greenfield
inputDocuments:
  - _bmad-output/planning-artifacts/product-brief-panoptes-2026-03-07.md
  - _bmad-output/planning-artifacts/brainstorming-report.md
  - docs/core/00-brainstorming-brief.md
workflowType: 'prd'
briefCount: 1
researchCount: 0
brainstormingCount: 1
projectDocsCount: 1
---

# Product Requirements Document - panoptes

**Author:** Nikku
**Date:** 2026-03-08

## Executive Summary

Panoptes is a modular geospatial intelligence platform that fuses six foundational layers into a coherent intelligence architecture: geography and geospatial analytics (the spatial foundation that anchors all intelligence to place and enables spatial reasoning across domains); people and place context (Census, demographic, housing, economic, and resilience data embedded in the architecture — not optional enrichment — that answers *who is affected, at what scale, and under what conditions*); economic and business context (trade flows and economic indicators that establish baseline conditions against which change becomes meaningful); observational feeds (domain-specific data streams that generate raw signal material); intelligence and fusion (the processing layer that transforms inputs into anomaly detection, confidence-scored findings, and actionable intelligence products); and operator experience (immersive, information-dense web applications built for serious practitioners who make decisions under uncertainty).

The cockpit is constant. The domains are variable. Adding a domain changes what Panoptes can see — it does not change what Panoptes is.

**Panoptes v1 is the platform's first operational module:** a self-hosted movement intelligence cockpit centered on ADS-B and AIS data, designed to surface early disruption signals in aircraft and maritime movement before they are clearly visible in public reporting. V1 establishes the platform's permanent core — the watch engine, intelligence processing layer, signal schema, confidence framework, and cockpit surfaces. The v1 wedge is bounded and tractable. The platform identity is not.

The platform is built for a single-operator, self-hosted deployment at v1. The broader platform and commercial path are preserved, not yet activated.

### What Makes This Special

A serious individual operator monitoring aircraft and maritime movement for supply chain and logistics intelligence has no accessible, always-on system that translates the continuous noise of public movement data into pre-shock signals. Feed viewers require the operator to find the signal. Panoptes surfaces it for them.

The defining gap is not data availability — it is the intelligence processing layer between raw feeds and actionable findings. Few publicly accessible tools at this price point and accessibility level do this for an individual operator today: watch persistently, maintain baselines, detect deviations, and deliver ranked, confidence-scored findings with traceable evidence.

Key differentiators:
- **Signal-first, not map-first.** The default experience is "here is what the system found" — not a globe to explore.
- **Compound signals.** When multiple independent data streams converge on the same conclusion in the same geography simultaneously, the resulting signal is more reliable than any single-source anomaly. This is what feed viewers cannot produce.
- **Absence as signal.** Panoptes detects what stopped, went dark, or deviated — not only what is present and active.
- **Persistent watch engine.** The system monitors continuously; the operator focuses on judgment, not data discovery.
- **Geography and Census as architecture.** Spatial foundation and people-and-place context are embedded in the intelligence layer from the start — not decorative layers added after the fact.
- **Modular design.** Adding a domain changes what the platform can see. It does not change what the platform is.

## Project Classification

- **Project Type:** Self-hosted web platform with persistent backend/data-processing services
- **Domain:** Modular geospatial intelligence platform
- **Complexity:** High
- **Project Context:** Greenfield

## Success Criteria

### User Success

Panoptes v1 is a single-operator, self-hosted system. Traditional acquisition and engagement metrics do not apply. User success is operational and intelligence-quality focused — the system is doing its job when the operator no longer needs to hunt for the signal.

| Criterion | What It Measures |
|---|---|
| **Signal Desk becomes the default first check** | Operator opens Signal Desk as the reliable starting point when beginning a monitoring session — not a periodic fallback after manually checking raw feeds |
| **Drill-down completability** | Operator can complete the full Signal Desk → Evidence Surface → Spatial View path without leaving the cockpit or requiring raw feed access |
| **Quiet state trust** | Operator does not feel compelled to manually monitor raw feeds when Signal Desk shows NOMINAL — confirmed silence is accepted as meaningful |
| **Compound signal confirmation rate** | Proportion of compound signals (two or more signal families converging) that the operator judges as genuinely useful or worth acting on |
| **Signal lead time** | At least some confirmed findings precede public confirmation in shipping indices, publications, or official reporting; tracked qualitatively at v1 |
| **Calibration convergence** | Operator dismissal rate on escalated signals trends downward over time as threshold calibration improves |

**The "aha" moment:** The first time a compound signal fires and the operator can trace it from the Signal Desk through the Evidence Surface to the specific vessel dwell and flight deviation data — and the finding holds up under scrutiny.

### Business Success

At v1, business objectives are architectural and strategic — preserving the commercial path while building the right foundation:

- **Operational self-sufficiency** — the system runs persistently without requiring the operator to babysit infrastructure
- **Modular architecture validation** — adding a new data domain does not require rearchitecting the cockpit or intelligence layer; the modularity principle holds in practice, not just in design
- **Commercial path preservation** — the data model, signal schema, and watch engine are structurally extensible to multi-operator and multi-tenant use without fundamental rework
- **API terms clarity** — provider terms for OpenSky Network, ADS-B Exchange, and AIS sources are reviewed and understood before production-scale ingestion begins

### Technical Success

| Metric | Target |
|---|---|
| **Feed processing continuity** | ADS-B and AIS feeds processing without extended gaps; feed health visible on Signal Desk at all times |
| **Baseline establishment** | Established baselines for all active watches within a defined collection window after watch creation |
| **Watch coverage completeness** | All configured watches actively monitored; no silent failures without operator notification |
| **Self-hosted stability** | System runs reliably on Contabo VPS without requiring active daily maintenance by the operator |

### Measurable Outcomes

| KPI | Measurement Approach |
|---|---|
| **Feed uptime** | Continuous — feed health status visible on Signal Desk domain pulse |
| **Signals judged worth investigation per active watch region** | Operator-assessed; tracked qualitatively per session — ratio of signals reviewed to signals dismissed without action |
| **Compound signal confirmation rate** | Count of multi-family convergence events per month that the operator confirms as substantive, not noise |
| **Confirmed lead time instances** | Qualitative log: "signal fired on [date], public confirmation on [date]" |
| **False escalation rate** | Ratio of operator-dismissed signals to total escalated signals; tracked per watch condition |
| **Time to calibrated baseline** | Days from watch creation to first stable, trusted baseline per watch condition |

**MVP is considered successful when:**
1. System runs persistently on Contabo VPS without active daily maintenance over weeks, not just hours
2. Signal processing pipeline has been exercised for all five primary signal families under live data
3. At least one compound disruption pattern fires and the operator can trace it cleanly through Evidence Surface to underlying data
4. Signal Desk becomes the operator's consistent first check without routinely needing to fall back to raw feeds
5. Provider terms for OpenSky Network, ADS-B Exchange, and AIS sources reviewed and ingestion architecture confirmed compliant

## Product Scope

### MVP Strategy

**MVP Approach:** Foundation MVP — v1 simultaneously establishes the platform's permanent core (watch engine, intelligence processing layer, cockpit surfaces, signal schema, confidence framework) and delivers operational value through the first admitted domain (movement intelligence). It is not a throwaway prototype. Everything built at v1 is intended to remain. The v1 wedge is bounded; the platform identity is not.

This approach is deliberate: the movement intelligence module is the most tractable first domain, produces immediately useful findings for the operator, and exercises every layer of the platform architecture. The architectural extensibility constraint — that the data model, watch model, and spatial model must preserve hooks for future domains and context layers — means v1 build decisions have platform-scale consequences.

**Resource requirements:** Solo builder at v1. Panoptes is self-built by the operator. Scope must be achievable by one person operating a self-hosted system. This is a defining constraint on what "MVP" means here — not what a team could build, but what a single skilled builder can bring to operational readiness.

### MVP — Minimum Viable Product

V1 is a self-hosted, single-operator movement intelligence cockpit. Scope is bounded to the movement intelligence module only. Every item below is required for the system to deliver its core value proposition:

- ADS-B and AIS feed ingestion (continuous, always-on)
- Intelligence processing layer: baseline establishment, anomaly detection, absence detection, cross-domain correlation, confidence scoring
- Five primary signal families: Port Dwell-Time Anomaly, Cargo Route Deviation, Chokepoint Throughput Change, Vessel Traffic Density Anomaly, Compound Disruption Pattern
- Secondary signals: commercial route mass deviation, AIS vessel dark events, GPS jamming zone formation
- Cockpit surfaces: Signal Desk, Evidence Surface, Spatial View, Watch Configuration
- Self-hosted on Contabo VPS; always-on; feed health monitoring; session summary; explicit NOMINAL state

**Architectural Extensibility Requirement (v1 constraint, not feature scope):** V1 implementation is bounded to the movement intelligence module. However, the v1 data model, watch model, signal model, and spatial model must preserve structural hooks for: geography and geospatial analytics as a first-class layer; people-and-place and Census context as future-admissible enrichment at the entity and geography level; and additional observational domains as they earn platform admission. This is a design constraint on the v1 foundation — not an implementation commitment. The v1 architecture must not become a movement-only dead end.

### Post-MVP Roadmap

**Immediate post-v1 priority — First Expansion Module:**
- Satellite orbital data (CelesTrak / Space-Track) — satellite pass correlation; compound pre-event patterns
- NOTAM filings — airspace restriction context for route deviation signals
- Military flight baseline + geopolitical watch class as an opt-in module with defined responsible-use boundaries

**Phase 2 (v2 — Temporal depth and investigation workflows):**
- Timeline View — event reconstruction and temporal pattern replay; requires historical data depth that accumulates during v1 operation
- Briefing View — AI-generated intelligence summaries; depends on Timeline View foundation
- Event Investigation and Reconstruction as a full operator mission alongside Continuous World Monitoring

**Phase 3 (v3+ — Pattern research and commercial path):**
- Pattern Discovery and Leading Indicator Research as a third operator mission
- Multi-tenant architecture enabling team-scale deployment
- API output layer for structured data consumers
- Commercial licensing path activated
- Expansion domains admitted as they pass the signal test, trust test, and identity test (environmental, infrastructure, geopolitical, and others)
- Broader people-and-place and Census-grounded signal use cases expand on the structural hooks embedded in the v1 platform architecture; these context layers are foundational to the platform from the start — v1 preserves the attachment points, and later phases build operational signal use cases and modules on top of them

The cockpit surfaces, watch engine, and intelligence processing layer built for v1 are the permanent foundation. The platform's six foundational layers — geography and geospatial analytics, people and place context, economic and business context, observational feeds, intelligence and fusion, and operator experience — are the architectural constant. What changes across versions is what the system can see and who can use it.

### Risk Mitigation

**Technical Risks:**

| Risk | Mitigation |
|---|---|
| Baselines require time to establish; system appears inactive on first run | NOMINAL state communicates active collection — not silence; baseline-in-progress is a named, visible state |
| Feed continuity: provider outages or rate limits cause data gaps | Separate ingestion process per provider; graceful degradation; degraded state surfaced on Domain Pulse; no silent failures |
| Compound signal false positive rate undermines operator trust | Confidence scoring at compound and constituent level; operator dismiss-with-reason; calibration convergence tracked over time |
| Provider terms constrain ingestion architecture post-build | API terms review is an explicit MVP gate before production-scale operation — not an afterthought |
| Platform extensibility hooks lost during build | Architectural extensibility requirement is a named design constraint documented in Domain Requirements and Product Scope; not a future backlog item |

**Market Risks:**

| Risk | Mitigation |
|---|---|
| Signals don't achieve meaningful lead time under live data | Lead time tracking begins on first run; qualitative log maintained; expectation set that not all signal families may fire naturally during the initial evaluation window |
| v1 scope too broad for solo builder to reach operational readiness | Scope is hard-bounded to movement intelligence module; no scope creep permitted into other domains at v1 |

**Resource Risks:**

| Risk | Mitigation |
|---|---|
| Solo builder capacity | Scope is set for single-builder delivery; no team-scale features (multi-tenancy, collaboration, API output) in v1 |
| Self-hosted infrastructure complexity | Containerized deployment (Docker / Compose) minimizes environment management overhead; backend health surfaced via Signal Desk, not server logs |

## User Journeys

### Journey 1: The Morning Watch (Primary Operator — Daily Success Path)

**The operator.** A serious individual analyst who tracks a handful of strategic maritime chokepoints and cargo corridors. They've been doing this manually for years — cross-referencing FlightAware, MarineTraffic, and shipping news each morning before their actual work begins. It takes an hour. Most mornings it turns up nothing.

**Opening scene.** It's early morning. The operator opens Panoptes. Signal Desk loads. The Horizon View shows three zones: *Attention Required* (one item), *Monitoring* (all other active watches, showing NOMINAL), *Domain Pulse* (ADS-B and AIS feeds both green). The session summary shows: two signals since last session, one escalated, watch coverage 100%.

**Rising action.** The operator reads the escalated signal: a Port Dwell-Time Anomaly at Jebel Ali, confidence 71%, flagged at 03:14 UTC. They click through to the Evidence Surface. The contributing layers are visible: vessels with dwell times measurably exceeding the rolling baseline; two of them are container vessels in known Red Sea diversion patterns. Raw AIS position data is accessible directly from the evidence panel.

**Climax.** The operator recognizes the pattern. They've seen this kind of clustering before a rate spike. The signal arrived before any shipping index published this morning. They act on it.

**Resolution.** The operator dismisses the second (sub-threshold) signal with a reason note. They spend 12 minutes total in Panoptes. The manual feed review they used to do doesn't happen today. Signal Desk was enough.

**Requirements revealed:** Signal Desk Horizon View; session summary; NOMINAL state as explicit confirmation; Evidence Surface with contributing layers; raw source access from evidence panel; signal confidence score display; operator dismiss-with-reason workflow.

---

### Journey 2: The First Compound Signal (Primary Operator — High-Value Discovery)

**The moment the system proves itself.** It has been running for three weeks. Baselines are established. The operator has dismissed four signals — two were noise, two were real but already in the news before the signal fired. Then one morning:

**Opening scene.** The Signal Desk shows *Attention Required* with two items stacked — a Compound Disruption Pattern at confidence 84%, and below it the two constituent signals that fed it: a Chokepoint Throughput Change (Strait of Hormuz, throughput measurably below rolling baseline) and a Cargo Route Deviation (Gulf-Europe cargo flights deviating measurably from established corridors).

**Rising action.** The operator drills into the compound signal first. Evidence Surface shows three independent contributing data streams converging on the same geography within the same time window. They switch to Spatial View — not the default, but triggered from the evidence panel. The spatial layer shows the vessel thinning and the flight corridor shift simultaneously. The patterns don't share a confirmed cause yet; that's the operator's job to determine.

**Climax.** The operator checks the major shipping publications. Nothing. They check the news. Nothing. The signal is hours old. They have lead time.

**Resolution.** This is the "aha" moment. The operator can trace the compound finding from the Signal Desk through the Evidence Surface to specific vessel and flight deviation data, and the finding holds under scrutiny. From this session forward, Signal Desk is their first open every morning — not a check, but a habit.

**Requirements revealed:** Compound Disruption Pattern signal family; Evidence Surface with multi-family contributing layer display; Spatial View triggered from evidence (not as default); cross-domain correlation display (ADS-B + AIS simultaneously); signal timestamp and age display; confidence score at compound and constituent level.

---

### Journey 3: False Escalation and Threshold Calibration (Primary Operator — Edge Case)

**Learning to trust the signal.** In week two, the operator gets an escalated signal that doesn't hold up. A Vessel Traffic Density Anomaly at the Singapore Strait, confidence 68%. They drill into the Evidence Surface. The clustering is real, but the operator recognizes the cause immediately — it's Chinese New Year weekend; port traffic patterns in this region follow a predictable seasonal shift.

**Opening scene.** The operator dismisses the signal with a reason: "Seasonal traffic — CNY clustering, not anomaly." They note that the watch condition needs adjustment for this recurring pattern.

**Rising action.** They navigate to Watch Configuration. The Singapore Strait watch is listed with its current threshold and baseline window settings. The operator adjusts the threshold and baseline window parameters to account for the known seasonal pattern — this is a deliberate operator decision, not an automatic system change.

**Resolution.** The next CNY period, the watch behaves as the operator intended. The operator's calibration has made the watch more precise. The false escalation rate for this watch trends down over time as the operator continues to tune it based on what they observe. The system reflects the operator's domain knowledge because the operator applied it — the calibration is explicit and operator-owned.

**Requirements revealed:** Operator dismiss-with-reason workflow; Watch Configuration surface with threshold and baseline window controls; per-watch calibration with explicit operator control; session-level dismissal tracking (feeds into calibration convergence metric).

---

### Journey 4: System Setup and Watch Configuration (Operator as Administrator)

**Onboarding the platform.** The operator has deployed Panoptes to their Contabo VPS. The system is running. Now they need to configure it for their actual focus areas before it can do anything useful.

**Opening scene.** The operator opens Watch Configuration — the first thing they do after verifying feed health. The Domain Pulse shows ADS-B (OpenSky + ADS-B Exchange) and AIS (NOAA) feeds as active and ingesting.

**Rising action.** They create their first watch: Strait of Hormuz, maritime, all five signal families enabled, default thresholds, 30-day rolling baseline window. They create three more: Suez Canal, Strait of Malacca, and a cargo corridor watch (Gulf-Europe). They set escalation thresholds manually for each — tighter on the high-confidence chokepoints, wider on the corridor watches where variance is higher.

**Climax.** The system acknowledges the watches and begins baseline establishment. Signal Desk now shows all four watches in the *Monitoring* zone, each with a baseline-in-progress status. The operator knows: nothing meaningful will fire until baselines are established. NOMINAL is not silence — it's confirmed data collection in progress.

**Resolution.** Over the following days, the watches transition to active baseline status. The operator receives their first sub-threshold items in the Monitoring zone. They adjust one threshold based on what they see. The system is now calibrated to their watchlist.

**Requirements revealed:** Watch Configuration surface (geography selection, signal family toggle, threshold controls, baseline window setting); Domain Pulse / feed health display on Signal Desk; baseline-in-progress status communication; NOMINAL state as active confirmation (not emptiness); multi-watch management.

---

### Journey Requirements Summary

| Capability Area | Revealed By |
|---|---|
| Signal Desk — Horizon View layout (Attention Required / Monitoring / Domain Pulse) | Journeys 1, 2, 4 |
| Session summary (signals since last session, escalations, watch failures) | Journey 1 |
| NOMINAL state as explicit, meaningful confirmation | Journeys 1, 4 |
| Evidence Surface with contributing layers and raw source access | Journeys 1, 2 |
| Compound Disruption Pattern signal with constituent signal display | Journey 2 |
| Spatial View triggered from evidence, not as default | Journey 2 |
| Cross-domain correlation display (ADS-B + AIS simultaneously) | Journey 2 |
| Signal confidence score at compound and constituent level | Journeys 1, 2 |
| Signal timestamp and age display | Journey 2 |
| Operator dismiss-with-reason workflow | Journeys 1, 3 |
| Watch Configuration surface with full threshold and baseline controls | Journeys 3, 4 |
| Per-watch calibration with explicit operator control | Journey 3 |
| Feed health / Domain Pulse visible on Signal Desk at all times | Journeys 1, 4 |
| Baseline-in-progress status communication | Journey 4 |

**Platform alignment note (v1 structural requirement, not feature scope):** The Evidence Surface and Spatial View must attach findings to named geographic entities (chokepoint, corridor, port, region) at the data model level — not only to raw coordinate positions. This geographic entity anchoring is the structural hook that will enable people-and-place and Census context to be associated with findings in future platform layers. V1 does not implement that context; it preserves the attachment point for it.

## Innovation & Novel Patterns

### Detected Innovation Areas

**1. Architectural inversion: findings surface, not feeds**

The dominant paradigm for public movement data tools is the feed viewer — a display layer over raw ADS-B or AIS data. The operator is responsible for finding the signal. Panoptes inverts this: the default operator experience is a ranked list of findings the system generated. The raw feed is accessible from evidence, not the entry point. This is an architectural choice with product consequences — it determines interface design, processing priorities, and what "done" means for the system.

**2. Compound signal detection as a primary output type**

Most anomaly detection systems surface individual-source deviations. Panoptes treats cross-stream convergence as its own output type — a Compound Disruption Pattern is not a derived view of two signals; it is a first-class finding with its own confidence score, ranked separately. The intelligence value of convergence across independent data streams (ADS-B + AIS simultaneously in the same geography) is materially higher than either stream alone. This requires the architecture to maintain cross-domain correlation state, not just per-feed anomaly state.

**3. Absence as affirmative signal**

Detecting what stopped, went dark, or deviated from expected presence is treated as an affirmative intelligence output — not a gap in data, not a display artifact. AIS vessel dark events, route deviation from established corridors, throughput thinning below baseline — these are findings, not absences. This requires absence modeling at the watch level: the system must know what "expected presence" looks like to recognize its disappearance.

**4. Persistent watch engine for individual operators**

The intelligence capability pattern — persistent monitoring, baseline establishment, anomaly detection, confidence-scored findings — has previously been accessible only at institutional pricing. Panoptes builds this for a single serious practitioner running self-hosted infrastructure. The innovation is not the technique; it is the accessibility layer and the operator model. This shapes every product decision: single-user, self-hosted, no managed service, no editorial cycle.

**5. Modular domain architecture: cockpit-constant, domains-variable**

The platform's intelligence layer, cockpit surfaces, and watch engine are designed to be domain-agnostic at the structural level. Movement intelligence is the first admitted domain; the architecture is designed so that adding a second domain does not require rebuilding the cockpit or intelligence layer. This is an architectural pattern for intelligence platforms — not just a scalability concern. It is what separates a movement tool from a platform.

**6. Geography and people-and-place context as intelligence architecture, not decoration**

Conventional geospatial tools treat geography as a display layer and demographic or Census context as optional enrichment — filters or overlays applied after the signal is already formed. Panoptes is designed with geography and geospatial analytics as a first-class structural layer: findings are anchored to named geographic entities, not only raw coordinates, and the data model is built so that people-and-place context — Census, demographic, housing, economic, and resilience data — can be associated with findings, regions, and entities at the architecture level. This means signals can ultimately be grounded not only in movement behavior, but in who is affected, where, at what scale, and under what conditions. This is a platform-level innovation in how intelligence systems relate movement data to human and geographic context. Full people-and-place signal implementation is not part of v1 scope; the innovation is that the architecture makes it structurally possible from the start, rather than requiring a later rearchitecture to admit it.

### Market Context & Competitive Landscape

| Category | What It Offers | What It Misses |
|---|---|---|
| Feed viewers (FlightAware, MarineTraffic) | Real-time display of public movement data | No anomaly detection, no baseline comparison, no compound signals |
| Globe-first tools (Cesium-style) | Immersive 3D visualization | Globe-first, not signal-first; showcase tools, not operational cockpits |
| Commercial intelligence platforms (Palantir, Maxar) | Analyst-grade output with compound signals | Institutional pricing; inaccessible to individual operators |
| News / market data | Downstream confirmation | Signal arrives after the window has closed |

The accessible gap is specific: a persistent, always-on intelligence layer with compound signal detection, confidence scoring, and evidence traceability — for an individual operator, at self-hosted cost.

### Validation Approach

Innovation in Panoptes is validated through operational confirmation, not user surveys:

- **Compound signal fires and holds up** — the primary validation event. At least one compound disruption pattern fires under live data, the operator traces it through the Evidence Surface, and the finding holds under scrutiny. This is an MVP success criterion.
- **Signal lead time confirmation** — at least some confirmed findings precede public reporting. Tracked qualitatively by the operator as a running log. Not a guaranteed outcome in any fixed window, but the metric is being tracked from day one.
- **Quiet state acceptance** — the operator stops checking raw feeds when Signal Desk shows NOMINAL. This validates that the absence-as-signal and persistent watch engine innovations are trusted, not just present.
- **Calibration convergence** — false escalation rate trends down over time, validating that the watch engine is learnable and operator-adjustable.
- **Architectural extensibility preserved** — geography and people-and-place hooks remain intact and unused at v1 close; the architecture has not drifted into a movement-only dead end.

### Risk Mitigation

| Innovation Risk | Mitigation |
|---|---|
| Baselines take time to establish; system appears empty on first run | NOMINAL state communicates active data collection — not silence. Baseline-in-progress is a visible, named state. |
| Compound signals may fire on correlated noise, not genuine convergence | Confidence scoring at compound and constituent level; operator dismiss-with-reason workflow; calibration history builds operator trust over time |
| Absence detection requires knowing expected presence | Rolling baseline windows per watch condition; absence signals are explicitly modeled, not inferred from data gaps |
| Provider terms may constrain ingestion architecture | API terms review is an explicit MVP gate before production-scale operation |
| Modular architecture adds complexity at v1 | Architecture constraint, not feature delivery — modularity is enforced at data model and interface levels without requiring additional v1 domains to be implemented |
| Geography and people-and-place architecture hooks drift out of scope during v1 build | Explicit structural requirement in domain requirements and product scope sections; hooks are a design constraint on the data model, not a feature to be delivered or deferred |

## Domain-Specific Requirements

### Provider Terms and Data Source Compliance

ADS-B Exchange, OpenSky Network, and NOAA AIS each have terms of service governing ingestion rate, data storage, and commercial use. API terms review is an explicit MVP success criterion. At the domain requirements level, the ingestion architecture must treat terms-compliance as a first-class design constraint — not a post-build retrofit. The system must not begin production-scale ingestion before provider terms are reviewed and the architecture confirmed compliant.

### Compliance and Regulatory Scope

No named regulatory compliance framework (HIPAA, GDPR, PCI-DSS, FedRAMP, or equivalent) is in direct v1 scope. The system is self-hosted, single-operator, and processes only public feed data with no external data subjects. However, the following are first-class domain constraints that shape architectural decisions regardless of formal regulatory status:

- Provider terms compliance (data source ingestion and storage)
- Local security and access control (self-hosted deployment)
- Retention policy (structured by data category — see below)
- Operator work-product handling (findings and configurations remain local by default)

### Source Provenance and Traceability

Panoptes findings are evidence-backed by design. The domain requires explicit traceability from finding → contributing source(s) → timestamps and feed state at time of detection. This is a domain integrity requirement, not a UI feature. A confidence-scored finding that cannot be traced to its source inputs is not a valid finding. Traceability must be preserved in the signal/evidence artifact record for the full retention lifetime of the finding.

### Graceful Degradation Requirements

Feed continuity monitoring is necessary but not sufficient. The following are explicit domain requirements:

- **Degraded and stale feed state visibility** — degraded or stale feed state must be visible to the operator at all times; feeds must not silently present last-known data as current
- **No silent failures** — any watch that stops receiving data must surface that state explicitly on the Signal Desk; continued display of last-known watch status without a staleness indicator is a failure mode
- **Confidence accounting for partial source availability** — compound signal confidence scoring must reflect degraded or partial source availability; a compound signal derived from a partially degraded feed must not be presented as fully sourced

### Data Retention Policy

Four structurally distinct retention categories are required. Exact durations are implementation decisions and are not specified here:

| Category | Description |
|---|---|
| **Raw feed retention** | Raw ingested ADS-B/AIS messages; highest volume, shortest useful life after normalization and processing |
| **Normalized / entity-state retention** | Processed positional and behavioral state per tracked entity (vessel, aircraft); forms the baseline window; retained across the active baseline period |
| **Signal / evidence artifact retention** | The finding record, contributing sources, confidence score, and feed state at time of detection; retained for the full period the signal is accessible to the operator |
| **Operator work-product retention** | Dismissal records, watch configuration history, calibration adjustments; operator-owned and should persist by default until operator-defined deletion or retention policy applies |

### Operational Sensitivity

ADS-B and AIS are public feeds; no data classification or export control issues attach to the source data. Intelligence outputs generated by the system — named findings, watchlist configurations, evidence records, operator work product — are produced by and for the operator. These must remain local by default. At v1 (self-hosted, no external API) this is structurally guaranteed; it should be stated as an explicit design principle to preserve as the platform evolves.

### Platform Alignment (Domain Constraint, Not v1 Feature)

Domain requirements must preserve geography and geospatial-entity anchoring across all layers — findings, evidence artifacts, spatial context, and retention records. Findings must be associated with named geographic entities (chokepoint, corridor, port, region) at the data model level, not only with raw coordinate positions. This is the structural condition that enables people-and-place and Census context to attach to entities and geographies in future platform layers. V1 does not implement that context. It must not make it structurally impossible.

## Platform-Specific Technical Requirements

### Project-Type Overview

Panoptes is a self-hosted, single-operator web platform composed of two tightly coupled subsystems:

1. **Web application cockpit** — a browser-based SPA (Signal Desk, Evidence Surface, Spatial View, Watch Configuration) serving a single authenticated operator on a self-hosted instance
2. **Persistent backend processing services** — always-on feed ingestion, intelligence processing, baseline management, and signal generation that run continuously regardless of operator presence

These subsystems are architecturally distinct. The cockpit consumes what the backend produces. The backend operates whether or not the cockpit is open. Requirements for each are different in character.

### Web Application Architecture

| Concern | Decision / Requirement |
|---|---|
| **Application type** | Single-page application (SPA); no multi-page navigation model |
| **Browser support** | Modern evergreen browsers (Chrome, Firefox, Edge); no legacy browser support required; self-hosted operator context |
| **SEO** | Not applicable — self-hosted, not publicly indexed |
| **Responsive / mobile design** | Not required at v1 — single-operator desktop cockpit; mobile is explicitly out of v1 scope |
| **Real-time updates** | Required — Signal Desk and Domain Pulse must reflect feed health and new signals without manual page refresh; WebSocket or equivalent push mechanism |
| **Accessibility** | Functional for a single known operator; public-facing WCAG compliance is not required at v1; standard semantic HTML and keyboard navigation are reasonable baseline |
| **Authentication** | Single-operator local authentication; no multi-user, no SSO, no OAuth integration at v1 |

### Backend Service Architecture

| Concern | Requirement |
|---|---|
| **Operational model** | Always-on; backend services run continuously and do not require operator presence to collect data or generate signals |
| **Feed ingestion services** | Separate ingestion processes per provider (OpenSky Network, ADS-B Exchange, NOAA AIS); must handle provider-specific rate limits, reconnection, and partial outages without cascading failure |
| **Intelligence processing pipeline** | Consumes normalized feed data; maintains baseline state per watch condition; executes anomaly detection, absence detection, and cross-domain correlation on a continuous basis |
| **Signal generation and persistence** | Findings are written to persistent storage with full evidence artifact records; signal state is not held only in memory |
| **Service isolation** | Ingestion, processing, and serving layers should be sufficiently isolated that a failure in one feed does not corrupt baseline state for other feeds or watches |
| **Operator-facing API** | Internal API between backend services and web cockpit; not externally exposed at v1 |

### Performance Targets

| Area | Requirement |
|---|---|
| **Feed ingestion throughput** | Must sustain continuous ADS-B and AIS ingestion within provider rate limits without data gaps under normal operation |
| **Signal processing latency** | Findings should be available to the operator on the Signal Desk within a defined window after the anomaly condition is met; exact latency target is an implementation decision, but the system must not batch-process signals only on operator session open |
| **Web application responsiveness** | Signal Desk loads within acceptable time on initial open; drill-down to Evidence Surface completes without perceptible delay under normal data volumes |
| **Baseline computation** | Rolling baseline windows must be recomputed incrementally as new data arrives; full recomputation on each cycle is not acceptable at production data volumes |

### Deployment and Infrastructure Requirements

| Concern | Requirement |
|---|---|
| **Target environment** | Contabo VPS; Linux; operator-managed |
| **Deployment model** | Self-hosted; no additional managed cloud dependency required beyond the chosen self-hosted infrastructure |
| **Containerization** | Containerized deployment strongly preferred (Docker / Compose) to ensure reproducible environment and service isolation on a single VPS |
| **Startup and restart behavior** | Services must recover from restart without loss of baseline state or signal history; state must be persisted, not held in process memory |
| **Monitoring and health** | Feed health and service status must be surfaced to the operator via Signal Desk Domain Pulse; backend health must not require operator access to server logs for routine monitoring |

### External Integration Requirements

| Integration | Purpose | Notes |
|---|---|---|
| **OpenSky Network API** | ADS-B feed (commercial aviation focus) | Subject to provider rate limits and terms; review before production-scale ingestion |
| **ADS-B Exchange API** | ADS-B feed (broader coverage) | Subject to provider terms; commercial use terms require review |
| **NOAA AIS** | Maritime feed (US coastal/offshore) | Public data; ingestion terms to be confirmed |
| **Additional AIS providers** | Global maritime coverage as needed | Provider-specific; terms review required before addition |

No payment processor, external auth provider, analytics service, or third-party SaaS integration is required at v1.

### Implementation Considerations

- **State persistence is non-negotiable.** Baseline windows, watch configurations, signal history, and evidence artifacts must survive process restarts. In-memory-only state is a failure mode for an always-on system.
- **Provider terms govern ingestion design.** Rate limit handling, data caching, and storage approach must be shaped by terms review before production deployment — not after.
- **Single-operator simplicity at v1.** No multi-user session handling, no permission matrix, no tenant isolation. Architecture should not carry the complexity of features it does not need.
- **Platform-aligned data model.** The data model for watches, signals, entities, and geographies must preserve extensibility hooks for future domains and context layers (see Domain Requirements and Product Scope). This is a v1 design constraint, not a future task.

## Functional Requirements

### Feed Ingestion and Data Collection

- **FR1:** The system can ingest ADS-B position and flight data continuously from multiple providers without requiring operator presence
- **FR2:** The system can ingest AIS vessel position and maritime data continuously from multiple providers without requiring operator presence
- **FR3:** The system can handle individual provider outages or rate-limit events without cascading failure to other ingestion streams
- **FR4:** The system can normalize raw feed data from multiple providers into a unified entity-state representation anchored to named geographic entities

### Watch Management and Configuration

- **FR5:** The operator can create a watch condition targeting a defined geographic area, chokepoint, or corridor
- **FR6:** The operator can enable or disable specific signal families per watch condition
- **FR7:** The operator can configure escalation thresholds per watch condition
- **FR8:** The operator can configure baseline window parameters per watch condition
- **FR9:** The operator can modify watch configuration parameters; compatible changes may preserve existing baseline state, while materially incompatible changes (such as geography, corridor scope, or baseline window) must trigger an explicit baseline reset and re-establishment rather than silently preserving invalid baseline data
- **FR10:** The operator can view all active watch conditions and their current status from a single surface

### Intelligence Processing and Baseline Management

- **FR11:** The system can establish a rolling baseline per active watch condition from accumulated feed data
- **FR12:** The system can detect vessel dwell-time anomalies at monitored ports or anchorage zones against the established baseline
- **FR13:** The system can detect cargo flight route deviations from established corridor baselines
- **FR14:** The system can detect chokepoint throughput changes against rolling baselines
- **FR15:** The system can detect vessel traffic density anomalies (clustering or thinning) in monitored maritime zones
- **FR16:** The system can detect the absence or disappearance of expected entities or traffic patterns (absence detection) as an affirmative finding
- **FR17:** The system can detect AIS vessel dark events at or near monitored strategic chokepoints
- **FR18:** The system can detect GPS jamming zone formation from ADS-B position and velocity incoherence near monitored corridors
- **FR19:** The system can detect commercial route mass deviations from established corridor baselines
- **FR20:** The system can detect convergence of multiple independent signal families on the same geography within a time window and generate a compound finding
- **FR21:** The system can assign a confidence score to each finding reflecting the strength and completeness of contributing evidence
- **FR22:** The system can adjust compound signal confidence to account for degraded or partial source availability at the time of detection

### Signal Desk and Operator Situational Awareness

- **FR23:** The operator can view all active findings ranked by priority and confidence on a single primary surface
- **FR24:** The operator can view the current status of all active watches (Attention Required / Monitoring / NOMINAL) from the Signal Desk
- **FR25:** The operator can view a session summary showing signals generated since last session, escalated findings, and watch failures
- **FR26:** The operator can view real-time feed health and ingestion status for all active data providers
- **FR27:** The operator can distinguish between a confirmed NOMINAL state (active monitoring, no anomalies) and a degraded or stale state
- **FR28:** The system communicates watch baseline-in-progress status as a named, explicit state
- **FR29:** The Signal Desk updates to reflect new findings and feed health changes without requiring manual page refresh

### Evidence, Investigation, and Provenance

- **FR30:** The operator can drill into any finding and view its contributing data layers, source readings, confidence components, and feed state at time of detection
- **FR31:** The operator can access the retained underlying source records and source-derived evidence for a finding, subject to applicable provider terms and retention policy
- **FR32:** The operator can view the full provenance chain for any finding: finding → contributing sources → timestamps → feed state at detection time
- **FR33:** The operator can view compound findings with constituent signals and their individual confidence scores alongside the compound score
- **FR34:** The operator can trigger a spatial view of signal-relevant geographic and movement data from the evidence view
- **FR35:** The spatial view displays only signal-relevant layers for the finding in context — it is not a default globe or general map browser

### Operator Workflow and Actions

- **FR36:** The operator can dismiss a finding with an associated reason recorded
- **FR37:** The operator can review the history of dismissed findings and their recorded reasons *(v2 — DEFERRED: per epics decision 2026-03-08)*
- **FR38:** The operator can authenticate to access the self-hosted cockpit as a single authorized user

### System Health and Operational Integrity

- **FR39:** The system surfaces degraded or stale feed states explicitly — no silent failures
- **FR40:** The system notifies the operator when any active watch stops receiving data, rather than displaying last-known status as current
- **FR41:** The system persists all baseline state, signal history, evidence artifacts, and watch configurations across process restarts
- **FR42:** The operator can assess overall system and feed health without accessing server logs or backend infrastructure directly

### Data Model and Platform Extensibility

- **FR43:** The system associates all findings, evidence artifacts, and watch conditions with named geographic entities (chokepoints, corridors, ports, regions) at the data model level, in addition to raw coordinate and geometry data where available — not as a replacement for it
- **FR44:** The system maintains structurally distinct retention handling for raw feed data, normalized entity state, signal and evidence artifacts, and operator work product
- **FR45:** The operator can access the full cockpit via a web browser on the self-hosted instance without installing additional client software

## Non-Functional Requirements

### Performance

- **NFR1:** Feed ingestion must sustain continuous throughput for both ADS-B and AIS streams within applicable provider rate limits, without data gaps under normal operating conditions
- **NFR2:** Signal processing must not introduce delays that render findings materially stale by the time they surface on the Signal Desk; the system must not batch-process findings only on operator session open
- **NFR3:** Baseline computation must be incremental as new data arrives; full baseline recomputation on every processing cycle is not acceptable at production data volumes
- **NFR4:** The Signal Desk must load and present current findings within an acceptable time on initial open during normal system operation
- **NFR5:** Evidence Surface drill-down from a finding must complete without perceptible delay under normal data volumes

### Reliability

- **NFR6:** Backend services must recover from unexpected process restarts and restore full operational state — including baseline windows, signal history, watch configurations, and evidence artifacts — without operator intervention
- **NFR7:** Feed ingestion services must reconnect automatically after provider outages or connection failures without requiring operator action
- **NFR8:** The system must not enter a silent operational failure state; any condition that prevents normal watch monitoring must surface to the operator via the Signal Desk
- **NFR9:** The system must sustain continuous operation on self-hosted infrastructure over weeks without requiring active daily maintenance by the operator

### Security and Data Privacy

- **NFR10:** Operator authentication is required to access the cockpit; the interface must not be accessible without valid credentials
- **NFR11:** All persisted data — baselines, signals, evidence artifacts, watch configurations, and operator work product — must be stored locally on the self-hosted instance
- **NFR12:** Intelligence outputs, operator work product, and watchlist configurations must not be transmitted to external services
- **NFR13:** Provider API credentials must be stored and accessed securely; credentials must not be exposed in logs, responses, or cockpit surfaces
- **NFR14:** Traffic between cockpit and backend services must be protected through local-only deployment boundaries and/or encrypted authenticated transport, as the architecture dictates

### Data Integrity

- **NFR15:** Evidence and provenance records must be append-only or versioned; they must not be silently or retroactively altered in any way that breaks the traceability chain from finding to contributing source
- **NFR16:** Baseline state must be protected against corruption during concurrent ingestion and processing operations
- **NFR17:** Watch configuration changes that invalidate existing baseline data must trigger an explicit baseline reset — silently preserving a baseline that no longer reflects the active watch parameters is a data integrity failure
- **NFR18:** Confidence scores must accurately reflect the actual source data available at time of detection; a finding must not be presented as fully sourced when contributing feeds were degraded or unavailable

### Operability

- **NFR19:** The system must be deployable and reproducible from configuration without manual state reconstruction
- **NFR20:** Routine system health assessment must be possible through the cockpit (Signal Desk, Domain Pulse) without requiring operator access to server logs or backend infrastructure
- **NFR21:** Containerized deployment must ensure service isolation such that a failing component does not require full system restart to recover
