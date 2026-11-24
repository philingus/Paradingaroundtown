# Crosstown: get across the Macy's 100th parade route

**Product:** Crosstown, a navigation agent built on Google Maps Platform (Routes API, Weather API, Maps JavaScript API) for one niche, painful problem: on Thursday, November 26, 2026, the 100th Macy's Thanksgiving Day Parade steps off at 8:30 a.m. from W 77th St & Central Park West and runs to Herald Square until about noon. For those hours the route (plus the balloon staging zone on CPW up to 86th St) is a 2.5-mile wall. Every West Sider going east, and everyone in Midtown going across 6th Ave, hits it. Generic nav apps know a street is "closed". They don't know you can't *cross* it, that a subway goes *under* it, or that a crosstown bus can't get *through* it.

Crosstown acts on your behalf: it detects that your trip crosses the parade, plans around it (walk around the south end, walk around the north end, or go under it by subway), adjusts for strollers and wheelchairs, watches wind for the balloons, and drafts an ETA text to whoever is waiting for you. It only sends that text after you approve it.

## Run the prototype

1. Get a free Google Maps Platform demo key: https://developers.google.com/maps/demo-key
2. `python3 -m http.server 8000` in this folder, then open `http://localhost:8000/?key=YOUR_KEY` (or paste the key in the page). The key stays in your browser and is never committed.
3. Pick a scenario and press **Plan my crossing**. The **Agent trace** panel shows every tool call.

| File | What it is |
|---|---|
| `index.html` | Working prototype: map, agent loop, ranking, human approval step |
| `tools.json` | The MCP tool surface external agents use (schemas + read/write annotations) |
| `README.md` | This design doc, organized by the workshop scorecard |

**What the prototype demonstrates (live API calls, not mocks):**
- Routes API `computeRoutes` for a direct walk, two walking detours using intermediate waypoints, and a TRANSIT option departing at the user's Thanksgiving-morning time.
- A crossing check: decodes each route polyline and tests it segment-by-segment against the parade barrier line. Subway legs are exempt because they run underground. Walking and bus legs that cross are rejected. This is how it catches a transit plan whose crosstown bus can't get through.
- Weather API hourly forecast (up to 120 h ahead, paged) filtered to 8 a.m.-12 p.m. on Nov 26, compared with NYC's balloon limits of 23 mph sustained and 34 mph gusts. You can simulate calm or gusty conditions for the demo.
- Preference-aware ranking (stroller: penalize stairs; wheelchair: penalize subway until elevator status is verified), arrive-by checking ("leave 12 min earlier"), and a human-approval gate before any message goes out.
- Restraint: the Chelsea → West Village scenario never crosses the route, so the agent says "no action needed" and drafts nothing.

**Honest limits:** barrier coordinates are approximate corners from the published route. The 6 a.m.-1 p.m. crossing-restriction window is an assumption (NYPD controls actual crossing points). Station elevator status is a designed tool, not wired up yet. Grading should treat the MCP server as a design: `tools.json` is the contract, and `index.html` implements the same tools client-side.

---

## 1. Stack & Surface

**Frontend / application layer (UI):** map with the parade barrier in red and candidate routes, trip form (from, to, leave at, arrive by, mobility, who to notify), plan cards, approval buttons, agent trace. This is where users configure preferences and approve actions.

**Backend & data layer:**
- *Data objects:* `ParadeBarrier` (polyline + time window, versioned per event), `CrossingOption` (route, minutes, ETA, surface-crossing flag, notes), `TripIntent` (origin, destination, depart_at, arrive_by, mobility, contact), `WindOutlook` (max sustained / gust for parade hours), `Plan` (ranked options + chosen one), `ApprovalToken`.
- *Business logic:* barrier intersection test, subway-underground exemption, mobility penalties, wind penalty, arrive-by check.
- *Upstream APIs:* Google Routes API (deterministic routing), Google Weather API (forecast), Maps JavaScript API (display).

**How an agent connects:** through an **MCP server** (`crosstown-mcp`, schemas in `tools.json`): `get_parade_disruptions`, `get_wind_outlook`, `plan_crossing`, `watch_trip`, `send_eta_message`. Why MCP and not just a REST API or a browser agent: the person in need is already talking to an assistant ("get us to Aunt Rosa's by 11, we have a stroller"). MCP gives any assistant typed tools with read/write annotations, so the model picks `plan_crossing` instead of scraping a map UI. Under the hood those tools wrap deterministic traditional APIs (Routes, Weather), so the probabilistic part is limited to understanding intent and choosing tools. The routing math is never left to the LLM.

## 2. Value & Model Impact

**Where Crosstown creates value:** it is an **intelligence engine** plus an **execution layer**, not a system of record. Google owns the map and the routes. Crosstown's value is the event-specific logic on top: which streets are uncrossable and when, that subways go under and buses can't go through, what a stroller or wheelchair changes, and when wind changes crowd behavior.

**What's defensible vs commoditizable:**
- *Defensible:* the curated, time-windowed **event closure dataset** (barrier geometry, crossing points, staging zones, versioned per event and reusable for the NYC Marathon, Pride, the Village Halloween Parade, UN General Assembly week), crossing-aware **verified execution** (every plan is re-checked against the barrier before it's shown), and over time, proprietary data on which detours actually worked.
- *Commoditizable:* the map UI and the plan cards. Any assistant can render a route.

**Business model and agent access:**
- An ad/attention model would be cannibalized: agents skip our screen, so no impressions. We don't use one.
- We charge **usage-based**: per `plan_crossing` call and per `watch_trip` session, sold to (a) AI assistants that need event-aware routing, and (b) hotels, venues, and event organizers who want to give guests a "how do I get there on parade day" answer. The consumer app stays free as the trust and data-collection surface.
- Under usage pricing, agent access **expands TAM**: tourists in hotels, people who'd never install a parade app, and anyone asking their assistant in natural language become customers at the exact moment of need, and volume scales with automated calls instead of eyeballs.
- **Disintermediation risk:** assistants could try to rebuild closure logic from news articles. Our protection is accuracy and freshness of the closure dataset plus verified execution. Scraped closure lists can't tell you a bus route is blocked or that the 72nd St crossing is closed right now.

## 3. Boundary Design

| Capability | Exposed to external agents? | Why |
|---|---|---|
| Closure status for a time (`get_parade_disruptions`) | **Yes, processed output** | Agents get "active / window / barrier description", not the raw geometry dataset, which is our moat |
| Wind outlook vs balloon limits | **Yes, read-only** | Cheap, public-weather-derived, builds trust |
| Ranked crossing plans (`plan_crossing`) | **Yes, read-only** | The core product. Returns options + ETAs, never raw closure polygons |
| Trip monitoring (`watch_trip`) | **Yes, scoped write** | Creates a watch on the user's own plan only. Idempotent, no external side effects |
| Sending ETA texts (`send_eta_message`) | **Yes, gated** | Messages an external person, so it requires a `confirmation_token` from the user approving the exact text |
| Raw event closure dataset, crossing-point inventory | **No, retained** | Proprietary. Licensed separately to the city and partners |
| User location history, saved contacts | **No, retained** | Privacy. Agents get a `contact_id`, never the phone number |
| Live turn-by-turn on parade morning, editing preferences, managing contacts | **No, deep link** | `crosstown://plan/{id}` opens the native map for the hands-on part |
| Booking rides or paying for anything | **No, not in v1** | Spending money needs a native confirmation screen, not an agent call |

**Unauthorized browser agents:** we block automated scraping of the web app (rate limits and bot detection) and point agents to the MCP server instead. Sanctioned access is metered and scoped. Scraped access bypasses the approval gate, which is exactly the thing we can't allow for messaging.

These decisions follow from Section 2: we expose **processed outputs** because that's what usage pricing meters, retain **raw data** because that's the moat, and separate **read from write** because the only write with real-world consequences (texting a family member) needs a human.

## 4. Workflow & Guardrails

**One end-to-end workflow** (the prototype's UWS scenario, and what `index.html` does step by step in its Agent trace):

1. **Intent:** "We're at W 75th & Amsterdam. Get us to Aunt Rosa's on E 74th & Lex by 11. We have a stroller. Leaving 9:45."
2. **Authentication:** user signs in once with OAuth. The assistant gets scopes `plans:read` and `watch:write`. `messages:send` is never granted standing; each send needs a fresh confirmation token.
3. **Context retrieval:** saved mobility preference (stroller), contact `Aunt Rosa`, `get_parade_disruptions(09:45)` → active, `get_wind_outlook(2026-11-26)` → Weather API forecast for parade hours.
4. **Multi-step tool execution:** `plan_crossing` calls Routes API four times: direct walk, walk via the south end (6th Ave & W 32nd), walk via the north end (CPW & W 88th), transit.
5. **Result verification:** direct walk is flagged because it crosses CPW inside the window. Every alternative is re-checked against the barrier: subway legs pass, any surface walk or bus leg across the route fails and is dropped. Options are ranked with the stroller penalty on subway stairs and checked against the 11:00 deadline. If the best plan is late, the agent says "leave N minutes earlier" instead of pretending.
6. **Human-in-the-loop:** the agent drafts "Hi Aunt Rosa, parade is blocking the direct way. I'm taking: … ETA 10:38." The user taps **Approve** (prototype opens the SMS composer; the user still presses send) or **Don't send**.
7. **Watch:** `watch_trip` re-plans every 5 minutes and only interrupts if the best option changes or the ETA slips past arrive-by.

**Designing for non-determinism:**
- *Latency:* all routing is deterministic API calls. The model only parses intent and picks tools. Detour calls are independent, so production runs them in parallel with a 3-second budget each.
- *Graceful tool errors:* if Weather fails, the agent plans without wind and says so. If one detour fails, it's skipped and logged. If the direct route can't be computed, the agent stops and sends nothing. If every option still crosses the barrier, it escalates to the user ("leave after 1 p.m. or check NYPD crossing points") rather than guessing.
- *Scoped permissions:* read tools can't cause side effects. `watch_trip` only touches the caller's own plan. `send_eta_message` is rejected without a per-message confirmation token.
- *Human approval required before:* messaging anyone, any spend (rides: not in v1), and changing saved preferences. Never needed for read-only planning, so the agent stays fast.
- *Doing nothing is a valid output:* if the trip doesn't cross the route (Chelsea → West Village), the agent reports "no action needed" and drafts nothing.

**Sources for event facts:** Macy's 100th parade date/time and route via Fox 5 NY's 2026 street-closure guide and Rockefeller Center's viewing guide. Demo key scope via Google's Maps Demo Key page.
