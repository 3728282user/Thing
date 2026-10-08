# Tidewright: Design and Build Guide

**Deliverable:** A single, self-contained `tidewright.html` file. No external dependencies, no network calls, no build step. Vanilla HTML, CSS, and JavaScript. Save data in `localStorage`.

**Target playtime:** 30 to 40 hours to the first full city restoration, with an open-ended Endless Tide afterward.

**Tone:** Quiet, industrial, slightly melancholic. A lone mechanic restoring a drowned city, one machine at a time.

---

## 1. Premise

You are the only mechanic left in a sealed workshop at the bottom of a drowned city. Your job is to pull salvage from the flooded ruins, rebuild machines, and restore ten districts so the city can breathe again. The water does not stop moving. Every tide changes what is possible, and every machine you run adds pressure to a system that must be vented.

The game's emotional arc: the city comes back district by district, and the Keeper's Log (see Section 11) slowly reveals who lived there and why they left.

---

## 2. Core Design Pillars

1. **Numbers go up, but you manage them.** Pressure and tides make passive growth an active problem.
2. **Every system feeds the progression.** No mechanic exists only for flavor. Each one changes what you buy, when you buy it, or how fast the next district arrives.
3. **Planning beats clicking.** Clicking is important early, but by midgame the decisions are about tide timing, blueprint routing, and machine placement.
4. **Long sessions and short sessions both work.** Offline progress and a tide forecast make it possible to play for five minutes or five hours.

---

## 3. Resources

| Resource | Symbol | Earned by | Spent on | Persists through Reclamation? |
|---|---|---|---|---|
| Scrap | S | Clicking, machines, routes | Machines, click upgrades, routes, districts | No |
| Parts | P | Sorter Frame and above, dives, routes | Repairs, gear, districts, blueprints (indirectly), routes | No |
| Blueprint Fragments | BF | Dives (rare), districts (fixed) | Blueprint nodes | No |
| Reclamation Score | RS | Reclamation reset | Foundation tree | Yes |
| Pressure | Pr (0 to 100) | Machines add load; bleed removes it | Not spent. It is a gauge. | No |
| Lifetime Scrap | LS | Every scrap earned | Used to compute RS | Yes |

**Number formatting:** Use suffixes up to 1 quadrillion (K, M, B, T, Qa, Qi), then scientific notation (e.g. `3.42e18`). Always show at least 3 significant digits.

---

## 4. Core Loop

1. Click to pull scrap from the water. Clicks grow stronger with Hands upgrades.
2. Buy machines. Each one produces Scrap per second and adds pressure load.
3. When pressure climbs toward Strain, vent it with the Vent Valve mini-game.
4. Send divers on salvage dives to gather Parts and occasionally Blueprint Fragments.
5. Spend Blueprint Fragments on the blueprint tree, which changes the rules of the next purchase.
6. Restore districts by meeting machine and Parts thresholds. Each district unlocks a new layer.
7. Build shipping routes between restored districts for passive income.
8. When the city is mostly whole, Reclaim: reset, keep score, and start a stronger run.

---

## 5. Machines

Each machine has: a base cost, a cost growth factor of **1.15** per owned unit, a Scrap/s output, a pressure load (per unit, per second), and an optional Parts/s output.

Cost of the nth purchase: `cost = baseCost * 1.15^owned`. Buying in bulk uses the closed-form geometric sum.

| # | Machine | Base cost (S) | Scrap/s | Pressure load/s | Parts/s | Unlock condition |
|---|---|---|---|---|---|---|
| 1 | Skimmer | 15 | 0.1 | 0.2 | 0 | Start |
| 2 | Dredge Rake | 100 | 0.8 | 0.6 | 0 | 10 Skimmers |
| 3 | Pump Engine | 1,100 | 5 | 2.0 | 0 | 10 Dredge Rakes |
| 4 | Sorter Frame | 12,000 | 30 | 6.0 | 0.05 | 10 Pump Engines, District 1 restored |
| 5 | Hydro Lift | 130,000 | 180 | 14 | 0.2 | 10 Sorter Frames |
| 6 | Foundry Press | 1.5M | 1,000 | 35 | 1.0 | 10 Hydro Lifts, District 3 restored |
| 7 | Deep Winch | 20M | 6,000 | 90 | 4.0 | 10 Foundry Presses |
| 8 | Tidal Engine | 300M | 40,000 | 220 | 20 | 10 Deep Winches, District 6 restored |

**Notes:**
- Machines 1 to 3 are in the **Low Deck** and are the machines that flood during High Tide (see Section 8).
- Machines 4 to 8 are in the **High Deck** and only flood during Storm Surge events (see Section 8).
- Output is multiplied by all global multipliers (blueprints, districts, routes, Foundation, Strain penalty).

---

## 6. Pressure and Venting

### 6.1 Pressure gauge

The gauge `Pr` runs from 0 to 100. It changes every tick:

```
Load    = sum over machines of (pressureLoad * count) * pressureMultiplier
Bleed   = 2.0 + bleedBonuses           (base bleed is 2.0 per second)
dPr/dt  = Load - Bleed                  (clamped so Pr stays in 0..100)
```

Load grows as machines are added. Bleed comes from blueprint upgrades and districts.

### 6.2 Thresholds

| Pr range | State | Effect |
|---|---|---|
| 0 to 69 | Steady | No penalty |
| 70 to 99 | **Strain** | All production reduced by 25%. The gauge turns amber. |
| 100 | **Jam** | All machines stop for 15 seconds. The gauge flashes red and a vent is forced at the end of the jam. |

### 6.3 The Vent Valve mini-game

The player can vent at any time, subject to a cooldown of **8 seconds** (reduced by blueprints).

**How it works:**
- A horizontal bar appears. A needle sweeps left to right and back, at a speed that increases as the gauge rises (speed = 1.0 at Pr 0, 1.8 at Pr 100).
- The player holds the Vent button to open the valve. Releasing the button at the right moment fires the vent.
- The bar has three zones: a **Perfect** zone (12% wide, centered), a **Good** zone (flanking, 20% wide each side), and a **Miss** zone (the rest).

**Results:**

| Release in | Pressure removed | Bonus |
|---|---|---|
| Perfect | 60 | Next 30 seconds: +10% Scrap/s ("Clean Steam") |
| Good | 30 | None |
| Miss | 10 | Costs 5 Scrap per point of gauge removed |

**Design intent:** Perfect venting is the best state, but it is never mandatory. Players can coast on Good vents with no real penalty. Perfect vents become important late, when the Clean Steam bonus stacks with other multipliers.

**Accessibility:** Provide an "Auto-Vent Assist" option that fires automatically inside the Good zone for 40% of the normal bleed reward. Also provide a larger-zone setting.

---

## 7. Clicking and Hands

Click output: `clickValue = 1 * (1 + 0.5 * handsLevel) * clickMultiplier`

- **Hands level** is bought with Scrap at increasing cost (starts at 25 S, grows 1.3x per level).
- Clicking spawns a small floating number and a short water-splash sound.
- **Hold-to-pull:** Holding the pointer down auto-clicks at 4 clicks per second. This is unlocked by the blueprint node "Steady Grip."

---

## 8. Tides

The tide is a single shared cycle that drives many systems.

### 8.1 Tide cycle

- Full cycle length: **20 real minutes**. Offline time advances it too.
- The tide level `T` is a smoothed sine wave in the range -1 (Low) to +1 (High).

```
T(t) = sin(2π * t / 1200)     // t in seconds, cycle is 1200 s
```

### 8.2 Tide phases

| Phase | T range | Effects |
|---|---|---|
| Ebb Low | T < -0.6 | **Ebb Sites** open: special salvage locations that pay Parts and BF at 2x odds. Duration of the window is shown in the forecast. |
| Rising / Falling | -0.6 to 0.6 | Normal operation |
| High Water | T > 0.6 | Salvage dive yields +40%. **Low Deck machines flood** unless a Tide Damper is installed. |

### 8.3 Flooding and repair

- When High Water begins, each Low Deck machine has a flood chance of **35%** (reduced to 10% with Tide Dampers, see the blueprint tree).
- A flooded machine produces 0 and shows a repair icon.
- **Repair cost:** `repairCost = machineBaseCost * 0.001 * count` in Parts (minimum 1 Part per machine).
- Players can choose to repair immediately, or wait for the tide to fall. Waiting costs nothing but lost output.

### 8.4 Tide forecast

A forecast bar at the top of the screen shows the next 60 minutes of tide, marking Ebb windows and High Water in color. This forecast is the main planning tool for the game and should always be visible.

### 8.5 Storm Surge (events)

A rare event triggered at random during High Water (5% chance per High Water phase after the first district is restored). A Storm Surge floods the **High Deck** as well, for 90 seconds. Storm Surge is also the source of rare **Distress Salvage** (see Section 9.4).

---

## 9. Salvage Dives

### 9.1 The diver

The diver has three gear slots. Each can be upgraded with Parts and blueprints.

| Gear | Effect |
|---|---|
| Lamp | Increases the blueprint fragment chance per dive |
| Hull | Reduces the chance of losing gear on deep dives |
| Tether | Increases maximum depth available |

### 9.2 Dive mechanics

- The player chooses a **depth** from 1 to the diver's max depth (starts at 3, rises with Tether upgrades).
- Each dive takes a **cooldown of 5 minutes** (reduced by Salvage blueprints) and returns with results after that time. The diver can be recalled early for half the results.
- **Parts yield:** `parts = depth * 2 * (1 + districtBonus) * (tideYieldBonus)`, where `tideYieldBonus` is 1.4 during High Water.
- **Blueprint fragment chance:** `bfChance = 0.04 * depth * (1 + lampBonus)`. Each dive rolls once.
- **Gear loss chance:** `lossChance = 0.02 * depth * (1 - hullReduction)`. If lost, the gear piece must be replaced for Parts before the next dive. Shallow dives (depth 1 or 2) cannot lose gear.

### 9.3 Ebb Sites

During Ebb Low, a special site appears in the forecast. Ebb Sites have doubled BF odds and cannot be dived at any other time. They are the main way to speed up the blueprint tree. Only one Ebb Site appears per Ebb window.

### 9.4 Distress Salvage

During a Storm Surge, a distress signal appears on the dive map. It has a fixed high Parts reward and a guaranteed Blueprint Fragment, but it can only be reached by a diver with at least Depth 4 and a Tether upgrade. It is the single most valuable dive in the game and the reason to invest in Tether early.

---

## 10. Blueprint Tree

### 10.1 Structure

The blueprint tree has four branches with **six nodes each** (24 nodes total), plus **four Cross Nodes** that unlock only after the player takes one node from two specific branches. Each node costs Blueprint Fragments (BF).

| Branch | Theme | Focus |
|---|---|---|
| **Hands** | Clicking and manual control | Click power, auto-click, vent efficiency |
| **Engines** | Machine output | Machine multipliers, cost reduction, new machine behavior |
| **Pressure** | Vent and bleed | Bleed rate, vent cooldown, flood protection |
| **Salvage** | Dives and parts | Dive cooldown, yield, gear |

### 10.2 Node list

**Hands**
- H1 Firm Grip (2 BF): Click value +50%
- H2 Steady Grip (4 BF): Unlocks hold-to-pull at 4 clicks/s
- H3 Clean Hands (6 BF): Miss vents cost 0 Scrap
- H4 Long Reach (8 BF): Click value scales with Low Deck count (+2% per machine)
- H5 Sure Hand (12 BF): Perfect zone widens by 4%
- H6 Second Wind (18 BF): Perfect vents extend Clean Steam to 45 seconds

**Engines**
- E1 Tuned Gears (2 BF): All machine output +25%
- E2 Bulk Discount (4 BF): Machine cost growth factor reduced from 1.15 to 1.13
- E3 Overclock (6 BF): Tidal Engine output +100%
- E4 Shared Shaft (8 BF): Each Low Deck machine adds +0.5% output to every other Low Deck machine
- E5 Foundry Standard (12 BF): Parts from Foundry Presses doubled
- E6 Cascade Engines (18 BF): Every 10 machines in a deck add +5% to that deck's output

**Pressure**
- P1 Pressure Relief (2 BF): Bleed +1.0 per second
- P2 Vent Cooldown (4 BF): Cooldown reduced to 5 seconds
- P3 Tide Dampers (6 BF): Flood chance reduced from 35% to 10%
- P4 Clean Steam Plus (8 BF): Clean Steam gives +15% instead of +10%
- P5 Pressure Sink (12 BF): Load from Low Deck machines reduced by 20%
- P6 Safety Valve (18 BF): Jam duration halved, forced vent restores 50 Pr

**Salvage**
- S1 Deeper Lines (2 BF): Max dive depth +2
- S2 Lamp Lens (4 BF): Blueprint fragment chance +50%
- S3 Quick Dive (6 BF): Dive cooldown reduced to 3 minutes
- S4 Reinforced Hull (8 BF): Gear loss chance halved
- S5 Tide Reader (12 BF): Ebb Sites appear with 2 additional sites per window
- S6 Full Haul (18 BF): Dive recall returns full results

### 10.3 Cross Nodes

Cross Nodes unlock only after the player owns one node from each of two named branches.

| Cross Node | Requires | Effect | Cost |
|---|---|---|---|
| X1 Wet Hands | H3 + P1 | Clicks feed Pressure bleed (click adds 0.05 Pr removal) | 15 BF |
| X2 Dredge Economy | E4 + S3 | Dives reduce machine repair costs by 50% | 15 BF |
| X3 Steady Current | P3 + H5 | High Water no longer slows Perfect vent timing | 20 BF |
| X4 Deep Engine | E6 + S5 | Ebb Site yields are added to machine production for 10 minutes | 25 BF |

**Design intent:** The best builds cross branches. A pure Engines build will hit a hard wall on flooding and gear loss. Cross Nodes are where the most interesting decisions live.

### 10.4 Respec

Blueprints can be fully reset once per district, at a cost of 50% of spent BF. This keeps early mistakes recoverable without making the tree trivial.

---

## 11. Districts and the Keeper's Log

### 11.1 Districts

The city has **ten districts**. Each district has a restoration threshold:

| # | District | Requirement (total machines owned) | Parts cost | Permanent bonus | Keeper's Log entries |
|---|---|---|---|---|---|
| 1 | Wharf | 10 | 0 | +10% Scrap/s | 1 |
| 2 | Mill Row | 25 | 40 | +1 max dive depth | 1 |
| 3 | Brass Quarter | 50 | 150 | +5% Scrap/s | 2 |
| 4 | Lamp Street | 90 | 400 | +1 Bleed/s | 2 |
| 5 | Cistern | 150 | 1,000 | +10% Parts yield | 2 |
| 6 | Foundry Yard | 230 | 2,500 | +10% Scrap/s | 2 |
| 7 | Signal Hill | 350 | 6,000 | +2% per district to Pr bleed | 3 |
| 8 | Glass Market | 500 | 14,000 | Unlocks Shipping Routes (see 12) | 3 |
| 9 | Dry Dock | 700 | 35,000 | +15% Scrap/s | 3 |
| 10 | Spire | 950 | 80,000 | +25% Scrap/s, unlocks Endless Tide | 4 |

Restoring a district also grants **5 Blueprint Fragments** and triggers a short visual of the district's map piece lighting up.

### 11.2 Keeper's Log

The Keeper's Log is a set of short, dated entries (2 to 4 sentences each) written by the previous mechanic who kept the workshop. Entries are unlocked by restoring districts and by collecting BF at set thresholds (every 10 BF gives a log entry, independent of districts).

**Tone:** Plain, human, understated. Entries describe repairs, weather, neighbors who left, and a debt that was never paid. The final entries reveal that the previous mechanic chose to stay with the machines when the city was evacuated. The player is given a final choice at the end (see 13.3).

Log entries are text only, with an optional small illustration. They have no mechanical effect, which keeps the story separate from the balance.

---

## 12. Shipping Routes

Unlocked by restoring Glass Market (District 8).

### 12.1 Route mechanics

- A route connects two restored districts that are **adjacent** on the city map.
- Each route has a build cost in Parts and produces **Scrap/s** based on the combined output of both districts.
- Route income: `routeScrap = 0.03 * (districtAOutput + districtBOutput)`, where output is the district's combined machine Scrap/s.
- Routes **stop working during Storm Surge** and resume after the surge.
- Each district can have at most **3 routes**.

### 12.2 Route map

The city map has an adjacency graph. The player draws routes along it. Adjacency is fixed, so the decision is which routes to build and in what order. Some routes are cheaper but weaker, and a few are "long routes" that pass through the Spire and produce bonus Blueprint Fragments.

| Route type | Build cost (Parts) | Income multiplier | Special |
|---|---|---|---|
| Local | 200 | 0.03 | None |
| Trade | 1,500 | 0.05 | Needs a Shipping Lane Gear (dive reward) |
| Long Route | 8,000 | 0.02 | Yields +1 BF every 15 minutes |

---

## 13. Reclamation (Prestige) and Foundation

### 13.1 When Reclamation unlocks

Reclamation becomes available after restoring **District 4**, and it is the main way to restart the economy with more power.

### 13.2 Reclamation mechanic

When the player Reclaims:
- All machines, Scrap, Parts, districts, routes, and dive gear reset.
- The blueprint tree **also resets** but BF are kept at 100%. This is a deliberate choice that makes the blueprint tree a long-term asset.
- Lifetime Scrap, the Keeper's Log, and Foundation points persist.

**Reclamation Score gain:**

```
RS gain = floor( max(0, log10(LifetimeScrapThisRun) - 9) * 2 ) + districtsRestoredThisRun
```

This means the first reset yields a few RS, and each later reset requires significantly more scrap for the same RS. The formula is logarithmic at first and feels increasingly expensive over time.

### 13.3 Foundation tree

Foundation points (RS) buy permanent starting bonuses. The Foundation tree has 12 nodes in a simple chain with two branches.

| Node | Cost (RS) | Effect |
|---|---|---|
| F1 Salvaged Start | 1 | Begin each run with 500 Scrap |
| F2 Old Gears | 2 | Begin with the Skimmer and 5 Dredge Rakes |
| F3 Loose Pressure | 3 | Bleed +0.5 per second |
| F4 Inherited Log | 4 | Begin with one Dive gear piece at level 1 |
| F5 Known Tides | 6 | Tide forecast shows 2 hours ahead |
| F6 Steady Hand | 8 | Vent cooldown starts at 6 seconds |
| F7 Stored Parts | 10 | Begin each run with 100 Parts |
| F8 Long Memory | 14 | Keeper's Log is preserved; new entries marked |
| F9 Old Foundations | 18 | Begin with District 1 already restored |
| F10 Second Tide | 24 | Unlocks a second tide cycle (offset by 10 minutes) |
| F11 Deep Start | 32 | Begin with Tether level 2 |
| F12 The Last Lantern | 50 | Unlocks the Final Choice (see 13.4) |

### 13.4 The Final Choice

After restoring all ten districts, the player is offered a choice in the Keeper's Log:

- **Let the water return.** Enter Endless Tide: repeating Reclamation loops with rising thresholds, new blueprint tiers, and a tide that gets stronger each run.
- **Stay and restore.** Start the Endless Tide mode with a permanent +10% to all output and a visual tide that stops.

Both choices are permanent for the save. The ending text changes depending on choice, but there is no mechanical punishment for either.

---

## 14. Progression Arc

Target times assume an average player. Times are cumulative.

| Stage | Hours | Key unlocks | Goal |
|---|---|---|---|
| The Drowned Workshop | 0 to 2 | Clicking, Skimmer, Dredge Rake, Vent Valve, Hands | Learn pressure and venting |
| Salvage Season | 2 to 8 | Pump Engine, diver, first dives, Mill Row | Start blueprint tree |
| Low Tide Work | 8 to 14 | Sorter Frame, tide forecast visible, Brass Quarter | Learn to plan around tides |
| The Tide Turns | 14 to 20 | Hydro Lift, Storm Surge, Lamp Street, first Reclamation | Learn the reset |
| Shipping Lanes | 20 to 28 | Foundry Press, routes, Cistern, Foundry Yard | Build routes and cross nodes |
| Dry City | 28 to 36 | Deep Winch, Signal Hill, Glass Market, Dry Dock | Restore the final districts |
| Last Lantern | 36 to 40 | Tidal Engine, Spire, Final Choice | Make the choice |
| Endless Tide | 40+ | Repeating Reclamation loops | Optimize and push records |

---

## 15. Offline Progress

- The game tracks the timestamp of the last tick.
- On return, offline time is capped at **12 hours**.
- Offline Scrap and Parts are earned at **50%** of the average live rate, computed from the tide simulated in 60-second steps.
- Pressure is simulated for offline time as average Strain: no vents occur offline, so the gauge sits at its end value and applies the Strain penalty.
- On return, show a **Harbor Log** panel summarizing: time away, Scrap gained, Parts gained, floods that occurred, any districts that became affordable, and one line of Keeper's Log text if a new entry was unlocked. The panel is the emotional summary of the absence.

---

## 16. UI Layout

**Top bar:** Scrap counter (large), Parts counter, BF counter, RS counter (once unlocked), settings icon.

**Tide forecast strip:** Full width, below the top bar. Shows the next 60 minutes, colored segments for Low, Rising, High, and Ebb windows. Current position marker.

**Left panel (Workshop):** Click area (a large illustrated water surface), Hands upgrade button, and the pressure gauge with the Vent button beneath it.

**Center panel (Machines):** Vertical list of owned and locked machines grouped by deck. Each row shows count, output per second, pressure load, and buy buttons (x1, x10, x max). Flooded machines show a repair icon.

**Right panel (Map and Dives):** Tabs for the city map (districts, routes), the diver (dive selection, gear, active dive timer), and the blueprint tree.

**Bottom:** Keeper's Log access button, with an unread indicator.

**Responsive behavior:** On narrow screens, collapse to a tabbed layout: Workshop, Machines, Map, Log. The tide forecast stays fixed at the top.

**Hotkeys (desktop):** Space to click, V to vent (hold), 1 to 4 to switch tabs, B for blueprint tree.

---

## 17. Visual and Audio Direction

**Visuals:**
- Muted, weathered industrial look. Warm amber light against blue-grey water.
- Machines are drawn as simple mechanical sketches with subtle animation (gears turn at the machine's output rate, capped for performance).
- The city map is a stylized top-down illustration with districts that go from dark (flooded) to lit (restored).
- The Vent Valve has a clear, readable bar. Perfect zone is highlighted in warm amber.

**Audio:** All audio uses the Web Audio API with procedural generation (no audio files needed).
- Low ambient hum that rises in pitch with pressure.
- Steam hiss on vent (louder on Perfect).
- Distant bell tones that sync to the tide phase change.
- Machine clanks that reflect the number of machines (sparse early, dense later).
- A mute toggle and separate volume control for ambient and effects.

**Motion:** Keep animations under 300 ms. No screen shake. Respect `prefers-reduced-motion`.

---

## 18. Technical Requirements

**Platform:** Modern desktop and mobile browsers (last two versions of Chrome, Firefox, Safari, Edge).

**Structure:**
- Single file: `tidewright.html`, with embedded CSS and JS.
- Game state in one plain object. Systems update from that object each tick.
- Game loop runs at **10 ticks per second** via `setInterval`. Rendering runs at display rate via `requestAnimationFrame` and only redraws changed elements.

**Number handling:** Use `Number` for values below 1e15. For larger values, store mantissa and exponent as a pair, or use a simple big-number helper. Keep the helper small and tested.

**Save:**
- Autosave to `localStorage` every 10 seconds and on visibility change.
- Save includes a version number for future migrations.
- Export and import via a base64-encoded JSON string.
- Hard reset option with a confirmation step.

**Performance targets:**
- Idle CPU usage under 5% on a mid-range laptop.
- No memory growth over a 4-hour session.
- Dom updates batched per frame.

**Accessibility:**
- All controls keyboard-operable.
- Color is never the only state indicator: flooded machines also have an icon, Strain also has a label.
- Auto-Vent Assist and larger-zone options (see 6.3).

---

## 19. Balance Targets and Testing

**Balance targets (average player, no guides):**
- First district restored: about 1 hour.
- First Reclamation: about 20 hours.
- Ten districts restored: about 36 hours.
- Final Choice reached: about 40 hours.

**Balance checks to run in playtesting:**
- Does Strain feel like a choice, or like a tax? If players never enter it, raise the load on Engine machines. If they always enter it, raise bleed sources.
- Are Ebb Sites ever skipped? If yes, increase BF odds or add a Parts bonus.
- Is any single blueprint branch strictly dominant? If yes, adjust Cross Node costs or add counter effects.
- Does the first Reclamation feel worth it at about 20 hours? Target a 2x to 3x output jump.
- Do offline sessions feel rewarding without making active play pointless? Target offline income around 15% to 25% of an equivalent active session.

**Unit tests to include (in a simple test harness page or console):**
- Machine cost formula matches the closed-form geometric sum.
- Pressure never leaves 0 to 100.
- Tide phase boundaries transition correctly.
- Reclamation score formula produces expected values at boundary inputs.
- Save and load round-trip preserves all state, including BF and districts.

---

## 20. Build Order (Suggested)

1. Core state object, tick loop, and save/load.
2. Click, Scrap, Hands upgrade, and number formatting.
3. Machines with cost growth and output.
4. Pressure gauge, Strain, Jam, and the Vent Valve mini-game.
5. Tide cycle, forecast strip, flooding, and repair.
6. Dives, gear, Ebb Sites, and Distress Salvage.
7. Districts and their thresholds.
8. Blueprint tree and Cross Nodes.
9. Shipping routes.
10. Reclamation and Foundation tree.
11. Keeper's Log and Final Choice.
12. Offline progress and Harbor Log.
13. Audio, visual polish, accessibility options.
14. Balance pass and tests.

---

## 21. Open Questions for the Builder

- Should the city map use a simple grid or a hand-drawn SVG layout? Recommendation: SVG, with districts as paths.
- Should blueprint nodes be drawn as a freeform graph or a fixed layered tree? Recommendation: fixed layers for readability.
- Is the Keeper's Log entries' content written in full by the team, or drafted by the builder? Recommendation: the team writes final text; placeholders are acceptable during build.
- Should Endless Tide scale to infinity or cap at a soft ceiling? Recommendation: soft ceiling with a visible "tide record" that resets each run.
