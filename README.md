WhichWay — architecture and decisions (as built, v01)

Status: working vertical slice delivered as whichway-v01.zip on 2026-09-01. Read this before starting new work — it supersedes parts of the three earlier briefs (Revised_AI_Critique, Critical_Review_Kimi, MVP_Final_Summary).

Settled with Ani
Question	Decision
Build shape	Static frontend + small Node/Express backend proxy (not React/Vite)
Traffic data	TomTom free tier primary, HERE as second opinion
Google	Adapter written but dormant — inert without a key
Preferences in v1	Avoid tolls/highways/ferries · vehicle profiles · weather/monsoon-aware · multi-stop optimisation
Scope	Both intra-Hyderabad and intercity Telangana, one map
API keys	Build key-less with graceful degradation; Ani adds keys later
Delivery	Zip (no folder connected to the session)
Extra asks	Ride comfort, convenience, and mileage as first-class scored dimensions
Two things that changed versus the earlier briefs

1. Google is the wrong oracle, and not for cost reasons. Google Routes does not expose per-segment road class. The "major-road percentage" and "road hierarchy" features the product is differentiated on are simply not in its response. TomTom returns them natively (sectionType), plus a physics fuel model with gradient terms and live/historic/free-flow durations in one call — 20,000 routing requests/month free, verified Sept 2026, not the 2,500 initially assumed. ORS returns real OSM waytype/surface per segment. So the free providers give richer route structure than the paid one.

Knock-on: the MVP doc's "no OSM initially, add it later as a context layer" inverts. With an OSM-based router, road hierarchy is native and free on day one.

2. The waypoint sweep is unnecessary. The d_artery continuous-optimisation framing correctly identifies the problem and picks the wrong method — it reverse-engineers a black box's cost function at 15-20 calls per corridor. Two reasons it is not needed:

Commitment points on a real artery are already discrete. You can only join or leave the ORR at an interchange. The curated corridor table has 62 real junctions; a Hyderabad→Nizamabad trip probes three or four of them, and each probe returns a scoreable route rather than a data point about someone else's router.
Once you can score continuity yourself, you never need to infer the router's preference at all.

The sweep survives as an offline research tool if you ever want to characterise a provider's behaviour. It does not belong in the request path.

The idea that carries the product: route continuity

"Take the ORR at Narsingi and stay on it to Medchal, don't get off at Dundigal" is a statement about how many times the driver changes road, not about time, cost, or road class. 60 km on one carriageway asks nothing; the same ground in nine named segments asks for nine decisions.

Measured directly from the step list, free, no search required:

longestRunShare — longest unbroken stretch ÷ total distance
fragmentedShare — distance in sub-5 km segments
arteryShare — distance on named NH/SH/ORR-grade roads

Both of the first two are needed. A route can spend 190 of 230 km on NH-44 and still be miserable because the first 25 km were seven short hops through the old city (high run-share, terrible fragmentation). A route down district roads in 10 km legs is the opposite — not fragmented, just off the artery. Fixtures pin both cases.

The briefs call this "route integrity" in §9 and then under-use it.

Pipeline
discovery (plain · preference variants · context-avoid · corridor probes)
  → geometric dedupe
  → features (each value carries provenance + confidence)
  → scoring (non-linear utilities → confidence-shrunk weights → Pareto flag → constraints)
  → explanation (every sentence derived from the factors that separated the routes)

Seven dimensions: Time · Cost · Comfort · Continuity · Road quality · Traffic reliability · Surroundings. Presets are named weight vectors over exactly those seven (config/presets.json) and carry no separate logic.

Three design positions worth keeping:

Non-linearity. Time takes the worse of a proportional and an absolute reading — ratio alone understates +40 min on a long trip, absolute alone understates it on a short one. Cost has a ₹25 deadband. U-turns get a convex penalty.
Confidence shrinks weights, does not default values. A missing feature's dimension stops voting rather than voting the mean. This is what prevents a speed-profile guess at road class from quietly deciding a recommendation.
Pareto flags, never filters. Dominated routes are shown and labelled. Dropping one silently leaves the user unable to tell "considered and rejected" from "never found".
Telangana context data

config/telangana/ — 10 corridors / 62 junctions (ORR, NH-44 N, NH-163, Rajiv Rahadari, NH-65, NH-65 W, NH-765, PVNR Expressway, Inner Ring Road, Nizamabad–Karimnagar), 20 market zones with time-of-day peak windows, 20 chronic waterlogging points.

Hand-curated local knowledge, not survey data, and labelled as such on every surface. Market penalties scale with the hour in Asia/Kolkata; flood penalties scale with live Open-Meteo rainfall, so they cost nothing dry and dominate in a downpour. Neither ever blocks a route — the destination may be the market.

Verification

41 offline checks in tests/verify.js, plus tests/mock-osrm.js for exercising the full HTTP path without a public router. The suite runs the real Narsingi→Nizamabad case across four fixtures and asserts that Relaxed picks the ORR corridor while Fast picks the quicker stop-start route — if a preset cannot separate those two, the model is doing nothing a stopwatch could not.

Four real bugs it caught, worth not reintroducing:

trafficDelay: 0 vs null — key-less providers reporting zero delay earned a "high confidence · live flow data" badge. Absence and zero are different values.
Market peak windows read off server-local time inverted every penalty outside IST. Timezone now pinned.
Route ids collided across discovery strategies (providers number from zero), breaking the shortlist and map selection.
Discovery legitimately returned 22 distinct routes. Shortlisted to 6, always keeping the outright fastest and cheapest.
Next steps
Add a TomTom key, re-run the same OD pairs with live traffic. Weights in presets.json are starting points, not calibrated constants.
Log recommendation vs. actual choice on 30-50 real Hyderabad trips. The validating question is not accuracy — it is whether WhichWay consistently picks the route Ani would have picked.
Only then: Overpass enrichment, historical reliability, personalisation.

Open and unresolved: toll cost is a per-km estimate (no free plaza-level tariff API for Telangana; NHAI data on data.gov.in is the upgrade path), and there is no free source for live construction or road-closure data in Telangana.
