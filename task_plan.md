# Away-From-Home Departure Adjustment — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When the user leaves home before an event, detect it via GPS, calculate walk-back time, shift reminders earlier by that amount, and notify the user — so they're never late because the system assumed they were at home.

**Architecture:** Instead of re-calling the Google Maps API (expensive, complex), we add a walk-back time offset (`away_walk_back_min`) computed from haversine distance. This offset is stored per-event in the daily_plan sheet and subtracted from departure time by the reminder engine. WF4's state machine computes it on every GPS ping; a combined "location update" loop writes both `at_home` and `away_walk_back_min` to the sheet. State transitions (`should_leave`, departure detection, urgent alerts) are also adjusted to use the effective departure time.

**Tech Stack:** n8n workflow JSON, JavaScript (inside n8n code nodes), Google Sheets API, ntfy

---

## Design Decisions

### Walk-back buffer vs. full recalculation from current location

**Chosen: Walk-back buffer.** Reasons:
- No extra Google Maps API calls (cost, latency)
- No structural refactoring of the existing arrival-recalculation path
- Correctly models the primary scenario: user needs to return home before leaving for the event
- Composes cleanly with WF2 traffic refresh (independent layers, no stale data interaction)

### Single offset column vs. per-event adjusted departure column

**Chosen: Single `away_walk_back_min` column.** The reminder engine subtracts it from the best-known departure time (`traffic_updated_departure || departure_time`) on the fly. This avoids stale interactions when WF2 updates traffic data independently.

### Walk-back estimation

Formula: `haversine_distance_meters / 80` (80 m/min ≈ 4.8 km/h walking speed).
- Threshold: only activate when >5 min away (ignore trivial distances like checking the mailbox)
- Cap: 60 min maximum (beyond that, user needs transportation, not a bigger buffer)
- Significance threshold: only update sheet when walk-back changes by >=3 min (avoid API churn)

### Event chains: walk-back only applies to home-origin events

Walk-back offset is ONLY relevant when `origin_type === 'home'`. In multi-event chains (Home → A → B), after arriving at A, event B's origin becomes A's location. Adding walk-back-to-home time for event B would be wrong — the user departs from A, not home. The reminder engine and state machine both gate on `origin_type === 'home'` before applying the offset.

### Use traffic-updated departure as base

WF2 updates `traffic_updated_departure` every 2 hours. Both WF4's state machine and WF3's reminder engine use `traffic_updated_departure || departure_time` as the base departure time. The walk-back offset subtracts from this base, ensuring consistency between state transitions and reminders.

### Notification strategy

- Notify on first detection (walk_back transitions 0 → >5)
- Notify on significant increase (>5 min shift since last notification)
- No "welcome home" notification (unnecessary noise)
- If walk_back > 45 min, message suggests transportation instead of walking

---

## File Map

| File | Action | What changes |
|------|--------|-------------|
| `co-location-handler.json` | Modify | State machine code (walk-back calc, effective departure, polling), rename+extend at-home update path, add away notification node |
| `co-reminder-engine.json` | Modify | Departure time priority chain in "Find Due Reminders" code node |
| `README.md` | Modify | Document new behavior in WF4 section |

### New daily_plan column

| Column | Type | Written by | Read by |
|--------|------|-----------|---------|
| `away_walk_back_min` | Integer (0 = at home) | WF4 | WF3 (reminder engine), WF4 (state machine) |

---

## Task 0: Add `away_walk_back_min` Column to Google Sheets

**Prerequisites:** Access to the Google Sheet used by the system.

This MUST be done before activating any workflow changes.

- [ ] **Step 1: Add column to `daily_plan` sheet**

Open the Google Sheet. In the `daily_plan` sheet, add a new column header: `away_walk_back_min`. Place it after the existing `at_home` column. Leave all existing rows blank (the system treats blank/0 as "no offset").

- [ ] **Step 2: Verify column is readable**

Confirm the column name matches exactly: `away_walk_back_min` (lowercase, underscores). n8n reads column names from the header row.

---

## Task 1: State Machine — Walk-Back Calculation + Effective Departure

**Files:**
- Modify: `co-location-handler.json` — "Select Active Event + Run State Machine" code node (lines 242-243 of the JSON, the `jsCode` string)

This is the foundational change. All other tasks depend on this.

- [ ] **Step 1: Add walk-back calculation to state machine output**

In the "Select Active Event + Run State Machine" code node, add after the at-home detection section (after the `eventsNeedingAtHomeUpdate` computation, before the polling interval section):

```javascript
// --- Away-from-home walk-back calculation ---
// When user is away from home with upcoming events departing from home,
// estimate walk-back time. Only applies when origin_type is 'home' —
// event chains (A → B) don't need walk-back since user is already at origin.
let awayWalkBackMin = 0;
let awayWalkBackChanged = false;
let awayNotify = false;

// Only compute walk-back if active event departs from home
const eventDepartsFromHome = event.origin_type === 'home'
  || (event.origin_coords || '') === HOME_COORDS;

if (!userIsAtHome && trackableEvents.length > 0 && eventDepartsFromHome) {
  const distFromHome = haversine(userLat, userLng, homeLat, homeLng);
  const rawWalkBack = Math.ceil(distFromHome / 80); // 80 m/min ≈ 4.8 km/h walking
  awayWalkBackMin = (rawWalkBack > 5) ? Math.min(rawWalkBack, 60) : 0;

  const storedWalkBack = parseInt(event.away_walk_back_min) || 0;
  awayWalkBackChanged = Math.abs(awayWalkBackMin - storedWalkBack) >= 3;

  // Notify on first detection or significant increase
  if (awayWalkBackMin > 0 && (storedWalkBack === 0 || (awayWalkBackMin - storedWalkBack) >= 5)) {
    awayNotify = true;
  }
}

// When returning home or event doesn't depart from home, clear the offset
if (userIsAtHome || !eventDepartsFromHome) {
  const storedWalkBack = parseInt(event.away_walk_back_min) || 0;
  if (storedWalkBack > 0) {
    awayWalkBackMin = 0;
    awayWalkBackChanged = true;
  }
}
```

Add to the return object (alongside existing fields):

```javascript
  away_walk_back_min: awayWalkBackMin,
  away_walk_back_changed: awayWalkBackChanged,
  away_notify: awayNotify,
  effective_departure: (event.traffic_updated_departure || event.departure_time),
```

- [ ] **Step 2: Adjust state transitions to use effective departure time**

In the same code node, REPLACE the existing state transition section. Find:

```javascript
if (currentState === 'planned' && now >= new Date(depTime.getTime() - 20 * 60 * 1000)) {
```

Replace the state transition block with:

```javascript
// Use traffic-updated departure if available (consistent with WF3 reminder engine)
const bestDepTime = new Date(event.traffic_updated_departure || event.departure_time);

// Effective departure accounts for walk-back time when away from home
const storedWalkBack = (event.origin_type === 'home' || (event.origin_coords || '') === HOME_COORDS)
  ? (parseInt(event.away_walk_back_min) || 0)
  : 0;
const effectiveDepTime = new Date(bestDepTime.getTime() - storedWalkBack * 60 * 1000);

// --- State transitions (using effective departure) ---
if (currentState === 'planned' && now >= new Date(effectiveDepTime.getTime() - 20 * 60 * 1000)) {
  newState = 'should_leave';
  shouldUpdate = true;
}

// Departure detection: user is >500m from origin AND within 30 min of effective departure
const distFromOrigin = haversine(userLat, userLng, originLat, originLng);
const departureThreshold = Math.max(500, accuracy * 2);
if ((currentState === 'planned' || currentState === 'should_leave')
    && distFromOrigin > departureThreshold
    && now >= new Date(effectiveDepTime.getTime() - 30 * 60 * 1000)) {
  newState = 'departed';
  shouldUpdate = true;
}

// Arrival detection: unchanged (doesn't depend on departure time)
const distFromDest = haversine(userLat, userLng, destLat, destLng);
const arrivalRadius = Math.max(200, accuracy * 2);
if (currentState === 'departed' && distFromDest <= arrivalRadius) {
  newState = 'arrived';
  shouldUpdate = true;
}

// Should-leave alert: at home past departure time (existing)
let sendUrgentAlert = false;
if (currentState === 'should_leave' && distFromOrigin < 200 && now > depTime) {
  sendUrgentAlert = true;
}

// Away-from-home urgent alert: away and past effective departure
if (!userIsAtHome && currentState === 'should_leave' && now > effectiveDepTime) {
  sendUrgentAlert = true;
}
```

- [ ] **Step 3: Tighten polling interval when away from home**

In the polling interval section of the same code node, add before the `return` statement:

```javascript
// Away from home with upcoming events → tighter polling for walk-back tracking
if (!userIsAtHome && trackableEvents.length > 0
    && newState !== 'departed' && newState !== 'arrived') {
  recommendedInterval = Math.min(recommendedInterval, 5);
}
```

- [ ] **Step 4: Expand location update event list to include walk-back changes**

Replace the existing `eventsNeedingAtHomeUpdate` computation with a combined check:

```javascript
// Combined location-state updates: fires when EITHER at_home or walk_back changes
const eventsNeedingLocationUpdate = allRows.filter(row =>
  row.user_id === userId
  && row.plan_date === today
  && !terminalStates.includes(row.current_state)
  && (row.is_online !== 'TRUE' && row.is_online !== true)
  && (
    ((row.at_home === 'TRUE' || row.at_home === true) !== userIsAtHome)
    || awayWalkBackChanged
  )
);
```

And update the return object — replace `at_home_updates` with:

```javascript
  location_updates: eventsNeedingLocationUpdate.length > 0
    ? JSON.stringify(eventsNeedingLocationUpdate.map(r => r.event_id))
    : null,
```

Remove the old `at_home_updates` field from the return object.

- [ ] **Step 5: Add new fields to `force_cancel` early return**

The consent-safety gate near the top of the state machine has an early return for `force_cancel`. Add the new fields so downstream nodes don't encounter undefined values:

```javascript
  // Add to the force_cancel return object:
  away_walk_back_min: 0,
  away_walk_back_changed: false,
  away_notify: false,
  location_updates: null,
```

- [ ] **Step 6: Verify state machine changes compile**

Open `co-location-handler.json`, locate the "Select Active Event + Run State Machine" node's `jsCode` field, apply all changes from steps 1-4. Verify the JavaScript is syntactically valid by pasting into a JS linter or Node.js REPL.

- [ ] **Step 7: Commit**

```bash
git add co-location-handler.json
git commit -m "feat: add walk-back calculation and effective departure to WF4 state machine"
```

---

## Task 2: Location Update Path — Write Walk-Back to Sheet

**Files:**
- Modify: `co-location-handler.json` — Rename and extend 3 existing nodes + their connections

**Depends on:** Task 1

The existing at-home update path (nodes: "At-Home Update Needed?", "Prepare At-Home Updates", "Loop At-Home Updates", "Update At-Home Flag") becomes a combined location update path that writes both `at_home` AND `away_walk_back_min`.

- [ ] **Step 1: Rename "At-Home Update Needed?" → "Location Update Needed?"**

In `co-location-handler.json`, find the node with `"name": "At-Home Update Needed?"` (id: `206f8156-...`). Change:
- `"name"` to `"Location Update Needed?"`
- Update the condition's `leftValue` from `$('Select Active Event + Run State Machine').item.json.at_home_updates` to `$('Select Active Event + Run State Machine').item.json.location_updates`

Also update ALL connection references: find `"At-Home Update Needed?"` in the `connections` object and rename to `"Location Update Needed?"`.

- [ ] **Step 2: Rename + modify "Prepare At-Home Updates" → "Prepare Location Updates"**

Find node with `"name": "Prepare At-Home Updates"` (id: `fd6c8136-...`). Change `"name"` to `"Prepare Location Updates"`. Replace the `jsCode`:

```javascript
// Parse the list of event IDs that need location state updated
const stateData = $('Select Active Event + Run State Machine').item.json;
const eventIds = JSON.parse(stateData.location_updates);
const newAtHome = stateData.user_is_at_home;
const walkBackMin = stateData.away_walk_back_min;

return eventIds.map(eventId => ({ json: {
  event_id: eventId,
  at_home: newAtHome,
  away_walk_back_min: walkBackMin
}}));
```

Update connection references from `"Prepare At-Home Updates"` to `"Prepare Location Updates"`.

- [ ] **Step 3: Rename "Loop At-Home Updates" → "Loop Location Updates"**

Find node with `"name": "Loop At-Home Updates"` (id: `1474c737-...`). Change `"name"` to `"Loop Location Updates"`. Update all connection references.

- [ ] **Step 4: Rename + extend "Update At-Home Flag" → "Update Location State"**

Find node with `"name": "Update At-Home Flag"` (id: `faf92e0c-...`). Change `"name"` to `"Update Location State"`. Add `away_walk_back_min` to the `fieldsUi.values` array:

```json
{
  "column": "away_walk_back_min",
  "fieldValue": "={{ $json.away_walk_back_min }}"
}
```

Update connection references from `"Update At-Home Flag"` to `"Update Location State"`.

- [ ] **Step 5: Verify all connection references updated**

Search the entire `connections` object in `co-location-handler.json` for any remaining references to the old node names: `"At-Home Update Needed?"`, `"Prepare At-Home Updates"`, `"Loop At-Home Updates"`, `"Update At-Home Flag"`. All should be replaced with the new names.

- [ ] **Step 6: Commit**

```bash
git add co-location-handler.json
git commit -m "feat: extend at-home update path to write walk-back offset"
```

---

## Task 3: Away-From-Home Notification

**Files:**
- Modify: `co-location-handler.json` — Add 2 new nodes + connections

**Depends on:** Task 2

When the user leaves home with upcoming events, send a push notification with walk-back estimate and return-by time.

- [ ] **Step 1: Add "Away Notify?" if-node**

Add a new node after the "Loop Location Updates" done-output (output index 1). This node checks whether an away notification should fire.

```json
{
  "name": "Away Notify?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "parameters": {
    "conditions": {
      "options": { "caseSensitive": true, "leftValue": "" },
      "conditions": [
        {
          "leftValue": "={{ $('Select Active Event + Run State Machine').item.json.away_notify }}",
          "rightValue": "true",
          "operation": { "type": "string", "operation": "equals", "sameType": false }
        }
      ],
      "combinator": "and"
    }
  },
  "id": "<generate-uuid>",
  "position": [910, 1620]
}
```

- [ ] **Step 2: Add "Send Away Alert" httpRequest node**

Add a new ntfy notification node for the away-from-home alert:

```json
{
  "name": "Send Away Alert",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "POST",
    "url": "=http://localhost:8080/{{ $('Select Active Event + Run State Machine').item.json.ntfy_topic }}",
    "authentication": "predefinedCredentialType",
    "options": {},
    "nodeCredentialType": "httpHeaderAuth",
    "sendHeaders": true,
    "headerParametersUi": {
      "parameter": [
        { "name": "X-Title", "value": "Away From Home" },
        { "name": "X-Priority", "value": "4" },
        { "name": "X-Tags", "value": "walking" }
      ]
    },
    "sendBody": true,
    "bodyType": "raw",
    "body": "={{ (() => { const sm = $('Select Active Event + Run State Machine').item.json; const wb = sm.away_walk_back_min; const title = sm.title; const dep = sm.effective_departure || sm.departure_time || ''; const depDate = dep ? new Date(new Date(dep).getTime() - wb * 60000) : null; const returnBy = depDate ? String(depDate.getHours()).padStart(2,'0') + ':' + String(depDate.getMinutes()).padStart(2,'0') : '?'; return wb > 45 ? `You're ~${wb} min walk from home. Consider transport. Return by ${returnBy} for ${title}.` : `You're ~${wb} min walk from home. Return by ${returnBy} for ${title}.`; })() }}"
  },
  "credentials": {
    "httpHeaderAuth": { "id": "", "name": "ntfy-auth" }
  },
  "id": "<generate-uuid>",
  "position": [1130, 1620]
}
```

- [ ] **Step 3: Wire connections**

Add to the `connections` object:

1. "Loop Location Updates" done-output (index 1) → "Away Notify?" (instead of nothing)
2. "Away Notify?" true-output (index 0) → "Send Away Alert"
3. "Away Notify?" false-output (index 1) → nothing (end of path)

```json
"Loop Location Updates": {
  "main": [
    [{ "node": "Update Location State", "type": "main", "index": 0 }],
    [{ "node": "Away Notify?", "type": "main", "index": 0 }]
  ]
},
"Away Notify?": {
  "main": [
    [{ "node": "Send Away Alert", "type": "main", "index": 0 }],
    []
  ]
}
```

Note: "Loop Location Updates" currently connects its done-output to nothing (implicit end). Change it to route to "Away Notify?".

- [ ] **Step 4: Commit**

```bash
git add co-location-handler.json
git commit -m "feat: add away-from-home push notification to WF4"
```

---

## Task 4: Reminder Engine — Walk-Back Offset in Departure Time

**Files:**
- Modify: `co-reminder-engine.json` — "Find Due Reminders" code node (lines 56-58 of the JSON, the `jsCode` string)

**Depends on:** Task 1 (column must exist), independent of Tasks 2-3

This is the change that makes reminders actually fire earlier when the user is away.

- [ ] **Step 1: Update departure time priority chain**

In the "Find Due Reminders" code node in `co-reminder-engine.json`, find this line:

```javascript
const effectiveDep = row.traffic_updated_departure || row.departure_time;
const depTime = new Date(effectiveDep);
```

Replace with:

```javascript
const baseDep = row.traffic_updated_departure || row.departure_time;
// Walk-back only applies to events departing from home (not event chains)
const isHomeOrigin = row.origin_type === 'home' || !row.origin_type;
const walkBackMin = isHomeOrigin ? (parseInt(row.away_walk_back_min) || 0) : 0;
const effectiveDep = walkBackMin > 0
  ? new Date(new Date(baseDep).getTime() - walkBackMin * 60 * 1000).toISOString()
  : baseDep;
const depTime = new Date(effectiveDep);
```

- [ ] **Step 2: Update the "Build Reminder Message" code to show walk-back context**

In the "Build Reminder Message" code node, find the `remind_now` case and add walk-back context:

```javascript
    case 'remind_now':
      title = 'LEAVE NOW';
      const walkBack = parseInt(r.away_walk_back_min) || 0;
      const walkBackNote = walkBack > 0 ? ` (includes ${walkBack} min walk home)` : '';
      message = `${prefix}Leave NOW for ${r.title}! ${r.commute_minutes} min ${r.commute_mode}${walkBackNote}`;
      break;
```

- [ ] **Step 3: Verify no other departure time references need updating**

Check the "Find Due Reminders" code for any other uses of `row.departure_time` that should use `effectiveDep` instead. The `remind_dressed` section uses `depTime` (which is now walk-back-adjusted) — this is correct because dressed reminders should also shift earlier.

- [ ] **Step 4: Commit**

```bash
git add co-reminder-engine.json
git commit -m "feat: add walk-back offset to reminder engine departure calculation"
```

---

## Task 5: Update Urgent Alert Message for Away Case

**Files:**
- Modify: `co-location-handler.json` — "Send Urgent Alert" node

**Depends on:** Task 1

The existing urgent alert says "You're still at home but should have left!" — this is wrong when the user is AWAY from home. Update the message to handle both cases.

- [ ] **Step 1: Update "Send Urgent Alert" body to be location-aware**

Find the "Send Urgent Alert" node (id: `40b67e9c-...`). Replace the `body` field:

```
={{ $json.user_is_at_home ? 'You\\'re still at home but should have left for ' + $json.title + '!' : 'You\\'re away from home and should have left for ' + $json.title + '! ~' + ($json.away_walk_back_min || '?') + ' min walk back.' }}
```

Note: the state machine now outputs `user_is_at_home` and `away_walk_back_min` which are available here.

- [ ] **Step 2: Commit**

```bash
git add co-location-handler.json
git commit -m "feat: make urgent alert message location-aware"
```

---

## Task 6: Update README

**Files:**
- Modify: `README.md`

**Depends on:** Tasks 1-5

- [ ] **Step 1: Add "Away-from-home adjustment" section to WF4 documentation**

In the README's WF4 section, add a subsection describing the new behavior:

> **Away-from-home adjustment:** On each GPS ping, WF4 checks if the user is away from home (via WiFi + GPS). If away with upcoming events, it estimates walk-back time using haversine distance (capped at 60 min) and stores `away_walk_back_min` in the daily plan. The reminder engine subtracts this offset from departure times, effectively shifting all reminders earlier. State transitions (`should_leave`, departure detection) also account for the offset. A push notification fires when the user first leaves home or when the walk-back estimate increases significantly.

- [ ] **Step 2: Update daily_plan sheet schema table**

Add `away_walk_back_min` to the Google Sheets structure documentation.

- [ ] **Step 3: Add Tasker geofence recommendation**

Add a "Recommended Tasker Configuration" section:

> **Home geofence:** Configure a Tasker geofence around your home address (radius matching `HOME_RADIUS_METERS`). On exit, trigger an immediate GPS ping to the WF4 webhook. This eliminates polling latency — the system detects you leaving home within seconds instead of waiting for the next scheduled ping (up to 15-20 min in idle tier).
>
> Tasker profile: `Geofence Exit (Home, radius 200m)` → Task: `HTTP POST to [n8n-url]/webhook/location-update` with current GPS + user_id.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: document away-from-home adjustment and Tasker geofence"
```

---

## Verification Checklist

After all tasks are complete:

- [ ] `away_walk_back_min` column exists in the Google Sheets `daily_plan` sheet
- [ ] Open `co-location-handler.json` in a JSON validator — must be valid JSON
- [ ] Open `co-reminder-engine.json` in a JSON validator — must be valid JSON
- [ ] All JavaScript code in `jsCode` fields parses without syntax errors
- [ ] All node name references in `connections` match actual node names (no broken wires)
- [ ] No references to old node names (`At-Home Update Needed?`, `Prepare At-Home Updates`, `Loop At-Home Updates`, `Update At-Home Flag`)
- [ ] The state machine return object includes: `away_walk_back_min`, `away_walk_back_changed`, `away_notify`, `location_updates` (replacing `at_home_updates`), `effective_departure`
- [ ] The `force_cancel` early return includes the new fields with safe defaults
- [ ] The "What Action?" switch's output 2 (none) routes to "Location Update Needed?" (not the old name)
- [ ] Walk-back offset gated on `origin_type === 'home'` in BOTH WF4 state machine AND WF3 reminder engine
- [ ] State machine uses `event.traffic_updated_departure || event.departure_time` as base departure
- [ ] README accurately reflects the new behavior

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Walk-back estimate inaccurate (straight-line vs actual walking route) | Medium | Low | Haversine is a lower bound; real walk is longer. User arrives early rather than late. |
| GPS ping latency (15-20 min idle polling) | Medium | Medium | Tasker geofence (Task 6) eliminates this. Polling tightens to 5 min once away. |
| Sheet column `away_walk_back_min` doesn't exist yet | Certain | High | Must be added to the Google Sheet before activating. Add as first deploy step. |
| Node rename breaks n8n import | Low | High | Verify all connection references updated. JSON validation catches broken wires. |
| Walk-back offset + traffic update interact badly | Low | Low | No interaction — walk-back subtracts from traffic-updated departure on the fly. |
| Walk-back applied to event-chain events (wrong origin) | N/A | High | Gated on `origin_type === 'home'` in both WF4 and WF3. Event-chain events skip walk-back. |
| `force_cancel` path missing new fields | N/A | Medium | Explicitly added safe defaults (0/false/null) to the `force_cancel` early return. |
| `toLocaleTimeString` fails in n8n Docker (missing ICU) | N/A | Low | Replaced with manual `padStart` formatting. |

## Status
**Tasks 1-6 complete.** Task 0 (add `away_walk_back_min` column to Google Sheets) must be done manually before activating workflows.
