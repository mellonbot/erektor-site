# The Big Erektor — System Architecture Spec (Draft 4)

Status: working draft. Draft 3 added the battery-swap ERS architecture, computed capacity/fleet math, and two-phase production targets. **Draft 4 replaces the central always-online ERS registry model with a local-facility-autonomy model with async reconciliation** (5.5), resolves tenant hierarchy depth as N-level by composition (5.6), splits "lost" detection into local/cross-facility tiers (5.3), and confirms post-dock in-situ leg-swap feasibility (2.5, 8). Settled decisions are stated as such; remaining open items are flagged inline and listed in Section 8.

**Reading note on numbers:** Section 7 (Capacity & Fleet Math) is computed from two *assumed* inputs that have not been confirmed — average legs per build, and the day-values assigned to the short/medium/long delivery thirds. Both are labeled where they appear. Every fleet figure scales linearly off them; treat the numbers as a sized model to be re-run against real inputs, not as settled requirements.

---

## 1. Product Overview

The Erektor is a modular heavy industrial robot that:

1. Transports raw materials around the factory floor
2. Holds and assembles the aluminum frame of a commercial EV charging station
3. Continues down the line for internal/charging-spec outfitting
4. Self-loads (with the finished charging station) onto a flatbed semi trailer for OTR transport
5. Self-unloads by walking off the trailer edge
6. Places the finished charging station gently at its marked destination

The robot and the product it builds share a structural skeleton: the charging station's own aluminum rail system becomes the Erektor's rigid chassis once legs are bolted on. Before that bolt-on, there is no shared rigidity — each leg is independently stable.

---

## 2. Mechanical Architecture

### 2.1 Leg (base unit)

* Each **leg** is a fully independent, self-contained system.
* **No wiring or support shared with any other leg** — own power, own control, own radio.
* Ground contact: **3 caster wheels per leg**, arranged for standalone tripod stability (a leg can stand upright unsupported).
* Caster configuration per leg: 2 free-spinning casters + 1 caster fixed to the drive-axis motor.
* **2 motors per leg**, each with a planetary gearbox (higher torque, lower speed, some backlash — more consequential on the lift axis than the drive axis):

  * **Drive axis** — a powered wheel at the foot (wheel-leg hybrid). One caster per leg is fixed to this drive-axis motor; the other two casters free-spin. Gait/actuator sizing is governed by the worst case (trailer-edge transition under full payload), not steady-state floor travel.
  * **Lift axis** — raises/lowers the leg, used for ground clearance during stepping and for the trailer-edge transition.
* Motor control: Teknic ClearPath-SC integrated servo motors (drive electronics in the motor housing), commanded via one **Teknic ClearCore board per leg**.
* Power: 24V to the ClearCore (logic/control), 56V direct from battery to motor +/- pins (bypasses the board).
* Radio: one **XBee module per leg**, wired to the ClearCore, for leg-to-controller communication.

### 2.2 Module (leg pair)

* A **module** = one right leg + one left leg.
* A module is a **logical/control grouping only, not a mechanical assembly.** Any left leg can pair with any right leg; the two legs are separately mounted and commanded together, with no shared mechanical structure between them. Pairing is arbitrary and decided fresh each session (see 2.4).

### 2.3 Frame attachment (rigidity source)

* Legs bolt to a **universal standard mounting bracket** on the charging station's aluminum rail — **fixed attachment points, never variable per product.**
* The rail must bear full structural load (4-point-lift-equivalent) **anywhere along its long sides**, not just at fixed hardpoints — this is what allows different charging station lengths to use the same fixed leg attachment geometry. State as a hard requirement for every frame variant.
* Because attachment points are fixed and known in advance, **no dynamic docking/sensing logic is required** — docking positions are a lookup, not a measurement.

### 2.4 Variable module count

* Builds require **3 to 6 modules** (6–12 legs) depending on charging station length — **not a fixed 8-leg/4-module unit.** "Erektor" is not a persistent physical unit; it is a temporary grouping of independently-claimed legs for the duration of one controller session.
* Leg-to-module pairing is **arbitrary each session** — legs are not persistently paired. Consequently, **"Erektor" has no persistent identity**: the only persistent entities are individual legs, controllers, and sessions. "A build" is a temporary grouping of independently-claimed legs for the duration of one controller session.

### 2.5 Process / rigidity state machine

Per-build state progression:

1. **Independent transport** — legs mobile individually, no shared rigidity; each leg self-propels via its drive-axis wheel with casters carrying load (2.1).
2. **Convergence & dock** — legs bolt to fixed rail mounting points; rigidity builds incrementally as each leg attaches.
3. **Unified rigid structure** — all claimed legs attached; frame + legs behave as one rigid body.
4. **Outfitting** — internal/charging-spec components added.
5. **Self-load for OTR transport** — full rigid structure + full payload; subject to FMCSA cargo securement requirements (49 CFR §393.100–136).
6. **Self-unload** — walk off trailer edge; highest-risk stability/structural event (asymmetric loading, incline, CoG crossing the trailer lip).
7. **Placement** — compliant force control for gentle set-down at marked location.

**Failure severity differs by stage:** a leg failure pre-dock is an isolated mobility loss (other legs unaffected). A leg failure post-dock is a structural failure — one of a small number of load-bearing points compromised on a now-rigid body. ERS and any diagnostics layer should treat these as different severity classes.

**Post-dock in-situ swap: confirmed feasible (Draft 4).** A single leg can be unbolted and replaced while the remaining legs hold a rigid, loaded structure — full de-rigidizing is not always required. This is conditional, not universal: it depends on which leg position fails (load redistribution margin varies by position relative to CoG), so the SOP's post-dock failure section needs a **decision table keyed by leg position** (swappable in-situ vs. non-swappable/abort-and-de-rigidize), not a single flowchart. Open items this creates:

* **Mechanical (blocking):** define swappable vs. non-swappable leg positions, load margin numbers (rated leg capacity vs. worst-case redistributed load at N-1 legs), and swap procedure/timing (safe-state, unbolt, insert spare, re-torque to spec, re-verify via controller).
* **Software:** whether a re-bolted leg requires a fresh claim/pairing handshake mid-session or inherits the failed leg's operational identity — a new state-machine sub-case (see 4.2, 4.4).
* **SOP:** decision-table structure depends on the mechanical answer above; cannot be finalized until swappable positions are known.
* Swap-time estimate (once known) feeds spare-pool sizing for post-dock failures (8.1), which is a separate input from the pre-dock line-side buffer (7.5) — pre-dock failures are already solved by the existing 20-leg buffer with no structural implications.

### 2.6 Dual load-rating requirement

The aluminum frame must be engineered against two independent load cases:

* Charging-station service loads (static mounting, thermal, vibration/environmental)
* Robot-chassis dynamic loads (walking/gait forces, torsional stress during single-leg-lift trailer transition, combined weight during self-transport)

The robot-chassis case, especially the trailer-edge transition, is likely the governing load case and should drive structural spec — not the reverse.

---

## 3. Safety & Compliance Contexts (kept separate — do not conflate)

1. **Factory floor operation** — near-human operation; likely ISO/TS 15066 or ANSI/RIA R15.08 (mobile robot) territory rather than fixed-arm standards (ISO 10218 series).
2. **OTR transport / self-securing** — FMCSA 49 CFR §393.100–136 cargo securement.
3. **Unload & placement** — compliant force control, tip-over margin, stability during trailer-to-ground transition.

Each context has different governing standards and default assumptions; treat as distinct compliance domains.

---

## 4. Connectivity Protocol (Controller ↔ Legs)

### 4.1 Hardware

* Controller: touchscreen unit powered by Raspberry Pi 5, running an XBee coordinator.
* Each leg: independent XBee radio node, joined to the controller's session — no shared bus, no leg-to-leg communication.
* The XBee's hardware address (SH+SL) is the **sole operational identity key** for a leg — a new XBee module means new identity info (see 4.6 for how this coexists with permanent mechanical identity).

### 4.2 Session lifecycle

1. **Power-on / discovery** — controller scans and enumerates all XBee nodes in range.
2. **Assignment** — operator selects how many legs (6–12, per 3–6 module build) to claim.
3. **Claim/lock** — controller binds selected legs to itself for the session; a claimed leg must reject competing claims from any other controller until released.
4. **Session duration** — legs remain bound to that controller from claim until return; **no communication between ERS and legs while deployed** (store-and-forward model, not live tracking).
5. **Release** — on return, legs un-bind and return to "available" status.

### 4.3 Identity & asset lifecycle rule

* **Identity is bound to electronics, not mechanical structure.** If the ClearCore/XBee assembly is replaced, the leg is treated as a **new asset** with a new serial — regardless of whether frame, motors, or gearboxes are reused.
* **Reissue record required:** the old serial must be marked retired/superseded with an explicit pointer to the new serial, noting what was replaced (electronics) vs. reused (mechanical components), to preserve traceability of carried-over hardware.
* **Wear/usage history is tracked against the permanent mechanical serial (4.6), not the electronics-tied asset serial.** An electronics swap rolls the operational identity but does not reset motor-hours / gearbox-wear / cycle-count history, because that history follows the mechanical frame. This is what makes annual maintenance (5.4) genuinely usage-accurate.

### 4.4 Failure/edge cases requiring explicit rules (not yet defined)

* Leg drops out mid-session (mechanical/comms failure) — does the session continue short, or is a spare leg substituted live? Post-dock, this is also the structural failure case from 2.5, where in-situ swap is now confirmed feasible for swappable leg positions — see 2.5 for the pending mechanical/software sub-decisions this creates, including whether a re-bolted leg needs a fresh claim/pairing handshake mid-session.
* Legs claimed but session aborted before departure — auto-release timeout vs. manual release, to avoid a claimed-but-unused leg blocking other controllers indefinitely.
* **Simultaneous claim race (two controllers claim the same leg near-simultaneously) — resolved at the facility level (Draft 4):** the facility's local ERS ledger is the sole arbiter for claims originating at that facility, consistent with the local-authority model in 5.5 (no dependency on a central registry). Cross-facility claim races (the same leg claimed near-simultaneously by sessions at two different facilities) resolve via the same physical-possession-wins rule as 5.5 — whichever facility's controller successfully pairs with the leg's electronics first is the confirmed claimant; the other facility's ledger is corrected on next sync.

### 4.5 Capacity check (needs verification against hardware spec)

Controller's XBee coordinator must reliably enumerate and hold up to **12 concurrent joined nodes** (largest build: 6 modules × 2 legs) plus headroom — confirm against the specific XBee series/firmware's max simultaneous connection count.

### 4.6 Two-layer identity model

Each leg carries **two distinct identities** that serve different purposes and change on different lifecycles:

* **Electronics-tied asset serial (XBee SH+SL address)** — the *current operational identity*. This is what the controller discovers, claims, and binds for a session, and what ERS uses for deployment/session/lost tracking. **Changes when the ClearCore/XBee electronics are replaced** — per 4.3, that constitutes a new asset.
* **Mechanical-frame stamped serial** — the *permanent physical identity* of the leg's mechanical structure, stamped on the leg's own frame. **Never changes** as long as the frame exists. This is the anchor for wear/usage history (motor hours, gearbox wear, cycle counts) that must survive electronics replacement.

**The reissue record (4.3) is the join between the two layers:** when electronics change, the asset serial rolls over while the mechanical serial stays constant, so wear history follows the mechanical frame rather than resetting. Annual maintenance (5.4) triggers against mechanical-frame lifetime hours, not against whatever the current electronics serial has accumulated.

---

## 5. ERS — Erektor Return System

### 5.1 Scope

ERS is **both a physical reconditioning line and the software that tracks it.** Two distinct things sharing a name — keep them separate when specifying:

* **ERS-physical** — a slow serial conveyor at each production facility, running from the incoming-shipment side to the point where step 1 of charging-station manufacturing begins. Returned legs enter at one end and exit ready-to-claim at the other (see 5.7).
* **ERS-software** — desktop application tracking every leg through that line and across the fleet: check-in, state, battery-pack association, maintenance triggers, lost detection, multi-facility registry (5.3–5.6).

Responsibilities:

* Tracking every leg returning from a delivery
* Managing 56V and 24V battery packs (swap-and-charge model, see 5.7)
* Managing brief cleaning for lifespan
* Managing annual maintenance cycling (see 5.4)
* Preventing false "lost" reports across facilities (see 5.5)

ERS is fundamentally an **asset lifecycle system tracking individual legs**, not Erektor units — since builds are variable-count leg groupings (Section 2.4), the leg is the persistent trackable entity. **Battery packs are a second, separately-tracked asset class** (see 5.7) — they detach from legs and have their own circulation and lifecycle.

### 5.2 Per-leg state machine

```
Available (in pool, at head of line)
  → Claimed by controller (session start)
  → Deployed (in transit / at delivery — no ERS comms during this state)
  → Returned / checked in          [enters conveyor]
  → Inspection
  → Cleaning
  → Battery swap (depleted 56V + 24V packs out, charged packs in)
  → Diagnostics / health check
  → Controller check-in / re-registration
  → Available (back in pool)       [exits conveyor at line-start]
```

Both 56V and 24V packs are present **on every leg** (settled). Charging no longer happens on the leg — see 5.7.

Branch states: **Failed inspection** → pulled from rotation (not redeployed) until resolved. **Due for annual maintenance** → routed to the separate maintenance facility (see 5.4) instead of back to Available. On a serial conveyor both branch states require a **physical divert point**, not just a software flag (see 5.7).

### 5.3 Departure logging (required addition — not yet built)

Because ERS does not communicate with legs during deployment, a leg that never returns anywhere leaves no trace and can never be flagged as overdue. **Fix required:** controller sends one lightweight departure event to the facility's **local** ERS at claim/session-start (leg serials, controller ID, destination, timestamp) — a bookend, not ongoing tracking. Without this, "lost" cannot be computed at all.

**"Lost" is a two-tier concept (Draft 4— supersedes the single-network-wide definition):**

* **Local lost (real-time, safety-critical, offline-capable):** checked out per the *facility's own* departure log longer than its expected-return threshold, with no local check-in. This is computed entirely against the facility's own ledger and must never wait on network connectivity or central sync — a facility flags this unilaterally the instant its own threshold is crossed.
* **Cross-facility lost (sync-dependent, informational):** a leg presumed locally lost that has actually checked in at a *different* facility whose events haven't yet synced to the shared control plane (5.5). This is resolved through reconciliation, not local detection, and carries an explicit **reconciliation-lag SLA (target: 15 min)** — a leg only escalates from "local lost" to "confirmed lost" once that lag window closes with no cross-facility match.

### 5.4 Annual maintenance

* A state-transition rule inside ERS (not a separate system) — reuses existing per-leg cycle count / elapsed-time data.
* **Scope: leg-level.** An individual leg hitting its threshold goes to the shop on its own; its most-recent module partner is unaffected and simply pairs with any available leg next session (consistent with arbitrary per-session pairing, 2.4).
* **Trigger: leg-level, usage-based**, read against the mechanical-frame lifetime hours (4.6) so an electronics swap doesn't distort the schedule.
* **Maintenance facility is fully decoupled from production** (settled, see 6): annual service runs at a separate facility on its own schedule, independent of production capacity. Legs out for annual service still count against deployable fleet (see 7.4). 
2 days inside Induistries Direct Erektor Service Department
### 5.5 Multi-facility architecture

* **Not** facility-to-facility checking (doesn't scale). **Not** a central always-online registry either (superseded — see below). Instead: **each facility's ERS is a fully self-sufficient node.**
* **Local-facility autonomy (Draft 4, settled):** check-in, check-out, battery-swap logging, maintenance routing, and local lost-detection (5.3) all run against that facility's own local ledger with **zero dependency on central connectivity.** A facility with no network access operates exactly as it would with one — this is a hard requirement of the architecture, not a degraded fallback mode. This is required because ERS is committed as a horizontal multi-tenant SaaS platform (not internal tooling); a central-always-online dependency would mean every customer facility's operations depend on connectivity to Erektor's own infrastructure — a single point of failure incompatible with selling this as a product.
* **Central infrastructure is a reconciliation and reporting layer, never a dependency underneath facility operations.** Facilities push departure/check-in events to their tenant's control-plane partition (5.6) on an interval or on reconnect — **async reconciliation**, not synchronous per-transaction communication.
* **Conflict resolution rule (settled):** for cross-facility disputes — e.g., a leg shows checked-in-locally at two facilities because a departure event hadn't synced yet — **physical possession, confirmed via live XBee pairing (two-layer identity model, 4.6), is authoritative.** Sync timestamps are never used to arbitrate a dispute; whichever facility successfully pairs with the leg's electronics is the confirmed current holder, and the other facility's ledger is corrected retroactively on next sync. In short: **local ledger always wins for local operations; last-confirmed-physical-possession wins for cross-facility disputes.**
* **API-first is a consequence of this, not a separate choice.** Because facility ERS must run fully offline, the facility-local application cannot be a thin client to a central API — it needs its own local API/data layer regardless. Central sync becomes a client *of* that same local API (pushing/pulling reconciliation batches), not a separate architecture. Formalize this boundary explicitly rather than building it accidentally, or a UI-with-embedded-logic version will later need to be gutted to add offline support.
* Open item carried forward: the reconciliation batch protocol itself — what gets synced, how conflicts are surfaced (auto-resolved vs. flagged for human review), and sync frequency/trigger (interval vs. on-reconnect vs. hybrid). See Section 8.

### 5.6 Multi-tenancy (company isolation)

* Each company deploying Erektors is **fully isolated** — no visibility or leg intermingling between companies (e.g., Company A's 300 units and Company B's 450 units never cross).
* Architecture: **one system, hard tenant partitions** (not N separate deployed systems) — company ID as a partition key on every record. Chosen over fully separate systems because it supports the lease program cleanly without later retrofitting cross-tenant access.
* **Tenant hierarchy: N-level by composition, not a hard-coded depth (Draft 4, settled).** The hierarchy falls directly out of the local-facility-autonomy model in 5.5: facility → tenant → manufacturer.

  * Each tenant's facilities sync into *that tenant's own* control-plane partition. Tenant-level visibility is a rollup across that tenant's own synced facilities — nothing more.
  * Manufacturer-tier (you) is a further rollup **across tenant partitions** — structurally the same rollup operation one level up, not a separate mechanism.
  * Because this is composition rather than a fixed schema depth, an intermediate reseller/franchise/white-label layer can be inserted later as another rollup level without redesigning the underlying facility-node model. This is what keeps white-labeling viable without a future rebuild.
* **Three access tiers:**

  1. **Company-level** — sees only their own fleet, fully siloed.
  2. **Manufacturer-level (you)** — spans all companies; required for lease administration, warranty/recall handling, fleet auditing.
  3. **Lease-program logic** (future) — a behavior on top of tier 2: a leg temporarily reassigned from a central pool to a company's partition, with return-to-pool logic on lease end. Adds a "currently leased to" field distinct from ownership.
* **Manufacturer-level (tier 2) access is periodic/on-demand/report-based, not real-time.** Tier 2 is a **query/reporting layer over the (now async-reconciled) tenant partitions**, not a first-class always-on live system — real-time field visibility is the controller's job (Section 4), not ERS's. Consistent with the async reconciliation model in 5.5.

### 5.7 ERS-physical: conveyor + battery-swap architecture

**Chosen architecture: battery swap, not on-leg charging.** Depleted 56V and 24V packs are removed at a swap station and replaced with fully charged packs from a ready shelf; the depleted packs go to an off-line charger bank. This is the load-bearing decision in the whole ERS design — see rationale below.

**Layout — slow serial conveyor.** Legs enter at the incoming-shipment end and creep to the point where step 1 of charging-station manufacturing begins, passing through zones in sequence:

```
[chasis check-in] → [controller check-in] → [inspection] → [battery swap] →  [diagnostics] → [cleaning] →[available, line-start]
                                                |                                  |
                                           divert: failed                     divert: failed
                                            inspection                diagnostics / due for annual
```

**Leg dwell on belt ≈ 1 hour** (clean + swap + check-in). This is the figure that sizes the belt.

**Why swap beats on-leg charging.** On a serial conveyor with on-leg charging, the belt cannot advance a leg until it is charged, so charge time *becomes* belt dwell and the charging zone must physically hold every leg that is mid-charge. Swapping decouples the two: the leg's dwell is the swap operation (~1 hr), while the pack's 3-hr fast-charge happens off-belt in parallel. Charging stops being a throughput constraint and becomes an inventory question (are there enough charged packs on the shelf?).

|                                      | On-leg fast-charge (3 hr)        | ==**Battery swap (chosen)**==                        |
| ------------------------------------ | -------------------------------- | ---------------------------------------------------- |
| Leg dwell on belt                    | 3 hr                             | **.5 hr**                                            |
| Legs resident on belt @ steady-state | ~27                              | **~9**                                               |
| Charging                             | on-belt, couples to belt speed   | **off-belt, independent bank**                       |
| Throughput ceiling                   | belt speed pinned by charge time | **shelf of charged packs (cheap to over-provision)** |
| Added inventory                      | none                             | **pack fleet (in-transit + charging + buffer)**      |
| Scaling phase 1 → steady-state       | lengthen charge zone ~7×         | **modest belt extension; add chargers + packs**      |

**Consequence — ERS is off the critical path.** Under swap, the conveyor can sustain one-station-per-hour provided the charged-pack shelf never starves. Reconditioning is no longer the facility bottleneck it would have been under on-leg charging.

**Battery packs are a separate asset class.** Packs detach from legs and circulate independently, so pack-to-leg association is a *current state*, not a fixed property. ERS-software must track packs on their own identity with their own charge-cycle history and health, and record which pack is in which leg at any moment. Packs are cheaper than legs — over-provisioning the pack fleet is the cheap insurance that keeps the swap station from ever waiting.

**Divert points are physical, not just logical.** A serial belt has no bypass: a leg that fails diagnostics at the far end has ridden the entire line only to be rejected, and will block the exit unless kicked to a spur. The 5.2 branch states each need a real divert mechanism on the conveyor.

⚠️ Manual swap initially. The 56v batteries have a quick disconnect that is easy access, and the 24v battery will need to be charged maybe once every 6 months

---

## 6. Production Line (Separate Discipline — Throughput, not Asset Tracking)

### 6.1 Two-phase output target (settled)

| Phase | Target | Per day | Basis |
|---|---|---|---|
| **Phase 1 (ramp)** | **25 stations/week** (~108/month) | ~3.6/day | Near-term capacity to reach ASAP. The line being built first. |
| **Steady-state** | **1 station/hour, 24/7** = **168/week** (~720/month) | 24/day | Design horizon. ~7× phase 1. |

These are **different facilities in practice**, not one line at two speeds — station count, belt length, charger bank, and fleet size all scale ~7×. Design phase 1 with explicit extension points rather than assuming a speed-up will close the gap.

### 6.2 Coupling rules (settled)

* **Fabrication is independent of ERS return rate.** Legs are manufactured as quickly as possible and will not stop until the purchase order is complete — fabrication is not throttled by how fast legs cycle back.
* **Annual maintenance is fully decoupled** — separate facility, own schedule, no bearing on production capacity.
* **ERS reconditioning is *not* a throughput bottleneck under the swap architecture** (5.7). It would have been under on-leg charging.

### 6.3 Bottleneck candidates (to evaluate against 6.1 targets)

Leg fabrication/assembly · rail-frame fabrication · docking/convergence step · outfitting · **swap-station labor at steady-state** (5.7) · **charged-pack shelf depth** (7.3). The slowest sets facility pace.

---

## 7. Capacity & Fleet Math (computed)

> ⚠️ **Two assumed inputs — all figures scale linearly off them. Confirm before treating as requirements.**
> **(A) Legs per build: 8 assumed.** Spec range is 6–12 (3–6 modules, 2.4). 8 legs - 4 modules is the middle that we will be selling "the big Erektor" as one  unit , *not* a stated product mix. A short-skewed mix (mostly 3-module) puts steady-state fleet nearer ~870; long-skewed, nearer ~1,600.
> **(B) Delivery-third day values: 1.5 / 5.5 / 9.5 days assumed.** Confirmed input was "1–10 days, split in even thirds"; the placement of each third within that range is an assumption.

### 7.1 Dispatch rate

| | Stations/day | × legs per build (A) | **Legs dispatched/day** |
|---|---|---|---|
| Phase 1 | 3.6 | 9 | **~32** |
| Steady-state | 24 | 9 | **~216** |

### 7.2 Deployment cycle time

Delivery mix is **even thirds** short / medium / long (settled). Using assumed midpoints (B): (1.5 + 5.5 + 9.5) ÷ 3 = **5.5 days average**.

### 7.3 Pool sizing (Little's Law: in-system = rate × time-in-system)

| Pool                       | Time in system | Phase 1      | Steady-state      |
| -------------------------- | -------------- | ------------ | ----------------- |
| **In transit (deployed)**  | 5.5 days       | ~176 legs    | **~1,188 legs**   |
| **On ERS conveyor**        | 1 hr           | ~2 legs      | **~9 legs**       |
| In active builds           | ~1 hr          | ~9–20        | ~9–20             |
| **Battery packs charging** | 3 hr           | ~4 pack-sets | **~27 pack-sets** |
| Charged-pack ready shelf   | buffer         | TBD          | TBD               |
| Annual maintenance         | 2 days         | all          | all               |
|                            |                |              |                   |

**The transit pool dominates everything.** At steady-state ~1,188 of ~1,300 legs are simply away on deliveries at any moment. Reconditioning (~9) and build presence (~20) are rounding errors beside it.

### 7.4 Total fleet

|                                         | Phase 1                   | Steady-state              |
| --------------------------------------- | ------------------------- | ------------------------- |
| **Legs (excl. annual-maintenance row)** | **~210**                  | **~1,300**                |
| Pack-sets in charger bank               | ~4                        | ~27–40                    |
| Pack fleet (total)                      | ≈ legs + charging + shelf | ≈ legs + charging + shelf |

⚠️ Both figures exclude legs out for annual service — the one unfilled row, pending service duration (5.4).

### 7.5 Correction: the 20-spare figure

The customer's ~20-spare-leg estimate was sized against **recharge downtime only** — the line-side buffer. It is a reasonable line-side buffer. It is **not the fleet size**, which is ~1,300 at steady-state and dominated by the transit pool the estimate never accounted for. Keep the two numbers separate in all planning: *line-side buffer* ≠ *total fleet*.

### 7.6 Highest-leverage variable

**Transit pool scales linearly with average delivery cycle time.** Every day shaved off the 5.5-day average removes **~216 legs** from the steady-state fleet requirement. Siting facilities closer to customer regions (shifting the mix toward short-haul) is worth far more than any improvement to belt speed, charger count, or recondition time — none of which touch the pool that holds ~90% of the fleet.

---

## 8. Open Questions (action list)

**Resolved in Draft 2:**

* ~~Drive-axis definition~~ → powered foot wheel / wheel-leg hybrid (2.1)
* ~~Module mechanical vs. logical~~ → logical control grouping only (2.2)
* ~~Leg-to-module pairing~~ → arbitrary per session; no persistent Erektor identity (2.4)
* ~~XBee address as sole identity key~~ → sole *operational* identity; permanent mechanical serial handles physical identity (4.1, 4.6)
* ~~Wear tracking granularity~~ → tracked against permanent mechanical-frame serial, survives electronics swap (4.3, 4.6)
* ~~Annual maintenance scope/trigger~~ → leg-level scope, leg-level usage-based trigger against mechanical-frame hours (5.4)
* ~~Multi-facility registry consistency~~ → central always-online registry (5.5) **[SUPERSEDED IN DRAFT 4 — see below]**
* ~~Manufacturer-tier visibility~~ → on-demand/report-based query layer, not real-time (5.6)

**Resolved in Draft 3:**

* ~~56V/24V battery configuration~~ → both packs present per leg (5.2)
* ~~Production target output~~ → two-phase: 25/week ramp, 1/hour steady-state (6.1)
* ~~Fabrication ↔ ERS coupling~~ → fabrication independent; maintenance facility decoupled; ERS off critical path under swap (6.2)
* ~~Spare-leg pool existence~~ → yes, but reframed: ~20 is a *line-side buffer*, not fleet size (7.5)
* ~~ERS physical model~~ → slow serial conveyor, incoming-shipment end to line-start (5.7)
* ~~Charging architecture~~ → battery swap chosen over on-leg charging; charging moved off-belt (5.7)

**Resolved in Draft 4:**

* ~~ERS consistency model~~ → local-facility autonomy, async reconciliation to central, physical-possession-wins conflict rule for cross-facility disputes (5.5). Supersedes the Draft-2 central-always-online model, which was incompatible with ERS's commitment as external-facing multi-tenant SaaS.
* ~~Tenant hierarchy depth~~ → N-level by composition (facility → tenant → manufacturer, extensible for future white-label reseller layer), not a hard-coded depth (5.6)
* ~~"Lost" detection model~~ → two-tier: local (real-time, offline-capable, facility-authoritative) + cross-facility (sync-dependent, 15-min reconciliation-lag SLA) (5.3)
* ~~API-first vs. UI-with-embedded-logic~~ → API-first is forced by the offline requirement — facility-local API is mandatory regardless, central sync is a client of it, not the reverse (5.5)
* ~~Post-dock in-situ leg-swap feasibility~~ → **yes**, confirmed feasible while remaining legs hold rigid load, conditional on leg position (2.5)
* ~~Simultaneous claim race rule~~ → facility-local ledger is sole arbiter for same-facility races; physical-possession-wins for cross-facility races (4.4, 5.5)

**Still open:**

*Blocking the fleet model:*

1. **Legs-per-build actual mix** — assumption (A) in Section 7 uses 9 (midpoint of 6–12). Real product-length mix swings steady-state fleet ~870 to ~1,600. **Highest-value input in the document.**
2. **Delivery-third day values** — assumption (B): 1, 3 and 5 days
3. **Annual-service duration per leg** — last unfilled row in 7.4 (5.4).
4. **Charged-pack ready-shelf depth** — how many hours of buffer before the swap station risks starving (5.7, 7.3).

*Hardware / protocol:*

5. **XBee capacity vs. 12-node largest build** — verify against series/firmware spec (4.5).
6. **Claimed-but-unused leg release policy** — auto-release timeout vs. manual (4.4).
7. **Post-dock swappable leg positions** — which leg positions structurally support in-situ swap under redistributed load, and what are the margin numbers (blocks: mechanical). Feeds swap-time estimate and post-dock spare-pool sizing.
8. **Mid-session re-pair handshake** — after an in-situ leg swap, does the session protocol require a fresh claim/pairing sub-state for that leg, or does it inherit the failed leg's binding? New case to model against the state machine (4.2, 4.4).

*ERS-physical / multi-facility:*

9. **Swap-station automation** — manual at phase 1 with an automation path, or automated from the start (5.7).
10. **Divert-point mechanism** — physical kick-off design for failed-inspection and failed-diagnostics (5.7, 5.2).
11. **Reconciliation batch protocol** — what gets synced, conflict surfacing - human review diagnosis failures), sync frequency/trigger hybrid (5.5).

---

*This document should be treated as project knowledge for future chats in this project. Section 7 figures are a sized model built on two labeled assumptions — re-run them when real inputs land rather than citing them as settled requirements. Draft 4's ERS architecture changes (5.3, 5.5, 5.6) are a structural dependency for all further ERS implementation work — the local-facility-autonomy model should be treated as settled before writing reconciliation code, API contracts, or the SOP's field-failure section.*
