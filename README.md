# Commute Optimizer -- Workflow Reference Guide

A detailed, step-by-step walkthrough of every workflow and every node in the Commute Optimizer system. This document covers all 14 n8n workflows (206 nodes total) that automate daily commute planning, real-time traffic monitoring, departure reminders, GPS-based location tracking, and system health monitoring.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Sub-Workflow D: Event Classifier](#sub-workflow-d-event-classifier-co-event-classifierjson)
3. [Sub-Workflow A: Departure Calculator](#sub-workflow-a-departure-calculator-co-departure-calculatorjson)
4. [Sub-Workflow B: Weather Advisor](#sub-workflow-b-weather-advisor-co-weather-advisorjson)
5. [Sub-Workflow C: Notifier](#sub-workflow-c-notifier-co-notifierjson)
6. [WF1: Nightly Planning](#wf1-nightly-planning-co-nightly-planningjson)
7. [WF2: Traffic Refresh](#wf2-traffic-refresh-co-traffic-refreshjson)
8. [WF3: Reminder Engine](#wf3-reminder-engine-co-reminder-enginejson)
9. [WF4: Location Update Handler](#wf4-location-update-handler-co-location-handlerjson)
10. [WF8: Mid-Day Calendar Sync](#wf8-mid-day-calendar-sync-co-calendar-syncjson)
11. [WF5a: Weekly Cleanup](#wf5a-weekly-cleanup-co-weekly-cleanupjson)
12. [WF5b: Monthly Statistics](#wf5b-monthly-statistics-co-monthly-statisticsjson)
13. [WF6: Error Handler](#wf6-error-handler-co-error-handlerjson)
14. [WF7: Heartbeat](#wf7-heartbeat-co-heartbeatjson)
15. [Cancel Event Webhook](#cancel-event-webhook-co-cancel-eventjson)

---

## System Overview

```
[Google Calendar] <-- read events ---- [WF1 Nightly Planning] --> [Google Sheets: daily_plan]
                                              |                            ^  ^  ^
                                              v                            |  |  |
                                        [Sub-WF A-D]                       |  |  |
                                        (departure calc,                   |  |  |
                                         weather, classify)                |  |  |
                                                                           |  |  |
[Google Maps API] <-- traffic check ---- [WF2 Traffic Refresh] -----------+  |  |
                                                                              |  |
[ntfy + Google Home] <-- reminders ---- [WF3 Reminder Engine] ---- read -----+  |
                                                                                 |
[Tasker on phones] ---- GPS + WiFi --> [WF4 Location Handler] ---- update ------+
       ^                                      |
       +---- interval + stop-tracking --------+

[WF7 Heartbeat]    --> monitors all services, pings healthchecks.io
[WF6 Error Handler] --> catches errors from all workflows, alerts via ntfy
[WF8 Calendar Sync] --> detects mid-day calendar changes, updates daily_plan
[WF5a/b Cleanup]    --> weekly data retention, monthly statistics
```

### How it works

1. **WF1 (4 AM)** reads each user's Google Calendar, classifies events, computes departure times and weather, writes a daily plan to Google Sheets, and sends a morning digest notification.
2. **WF2 (every 2 hours)** re-checks live traffic and updates departure times if conditions changed significantly.
3. **WF3 (every 5 minutes)** scans the plan for due reminders and sends "get dressed", "1 hour", "30 min", "15 min", and "LEAVE NOW" notifications.
4. **WF4 (on each GPS ping)** receives real-time location from phones, runs a state machine (planned -> should_leave -> departed -> arrived), detects at-home status, calculates walk-back time when away from home, and tells the phone how often to poll next. See [Tasker Setup](tasker-setup.md) for phone configuration.
5. **WF8 (every 30 minutes)** catches mid-day calendar changes -- new events, moved events, deleted events -- and updates the plan accordingly.
6. **WF5a (weekly)** and **WF5b (monthly)** handle data retention and statistics.
7. **WF6** catches errors from all workflows and alerts.
8. **WF7** monitors system health and pings a dead man's switch.

### External services used

| Service | Purpose | Auth method |
|---------|---------|-------------|
| Google Calendar API v3 | Read calendar events | OAuth2 (per-user refresh token) |
| Google Maps Distance Matrix | Compute travel times | API key |
| Google Maps Geocoding | Convert addresses to coordinates | API key |
| Google Sheets API | State storage (5 sheets) | OAuth2 |
| OpenWeatherMap One Call 3.0 | Hourly weather forecast | API key |
| ntfy (self-hosted) | Push notifications | Header auth |
| Twilio Messages | SMS fallback | Basic auth |
| Google Home TTS | Voice announcements | Local network script |
| healthchecks.io | Dead man's switch | Ping URL |

### Google Sheets structure

| Sheet | Purpose | Primary writers |
|-------|---------|-----------------|
| `user_config` | User settings, credentials, ntfy topics | Manual + WF5a |
| `daily_plan` | Today's events with state, departure, reminders | WF1, WF2, WF3, WF4, WF8 |
| `commute_history` | Historical trip records for analytics | WF1, WF4 |
| `location_updates` | GPS audit trail | WF4 |
| `error_log` | Sanitized error records | WF6 |

---

## Sub-Workflows

Sub-workflows are reusable building blocks called by the main workflows via n8n's Execute Workflow node. They receive inputs, process data, and return results to the caller.

---

### Sub-Workflow D: Event Classifier (`co-event-classifier.json`)

**Purpose:** Receives a raw Google Calendar event and determines whether it should be processed or skipped, extracting classification flags (online, tentative, force-car, dress-cap) for downstream workflows.
**Called by:** WF1 (Nightly Planning), WF8 (Calendar Sync)
**Inputs:** A single raw Google Calendar event object (fields: `id`, `summary`, `location`, `start`, `end`, `attendees`, `conferenceData`).
**Outputs:** Either `{ skip: true, reason: "..." }` to drop the event, or a classification object with `is_online`, `is_tentative`, `force_car`, `dress_cap`, `adjusted_start`, `adjusted_end`, `location`, `title`, and `event_id`.
**Node count:** 2

#### Step-by-step Flow

**Step 1: Input** (`executeWorkflowTrigger`)
Entry point that receives the raw calendar event data from the parent workflow via the Execute Workflow call.

**Step 2: Classify Event** (`code` -- runs once per item)
Parses the incoming event, defensively handling nested objects that may arrive as JSON strings. Then applies a series of classification checks in order:

1. **All-day event check:** If the event has `start.date` but no `start.dateTime`, it is an all-day event -- returns `skip: true` with reason "all-day event".
2. **Declined event check:** Looks for the user's own entry (`self: true`) in the attendees array. If the user's `responseStatus` is "declined", returns `skip: true` with reason "declined".
3. **No location check:** If the location field is empty or whitespace AND the event has no `conferenceData` (no video meeting link), returns `skip: true` with reason "no location". Events with a conference solution but no location are allowed through.
4. **is_online flag:** Set to `true` if the location contains video meeting keywords (zoom, meet, teams, webex, slack, hangout) OR if the event has `conferenceData.conferenceSolution`.
5. **is_tentative flag:** Set to `true` if the user's own attendee entry has `responseStatus` of "tentative".
6. **force_car flag:** Set to `true` if the event title (summary) contains the tag `[car]` (case-insensitive).
7. **dress_cap extraction:** Parses the title for `[dress:N]` or `[N]` tags. If found, sets `dress_cap` to that integer value (caps which dress reminder intervals fire). Defaults to `0` (no cap).
8. **Adjusted start/end:** Extracts `dateTime` (preferred) or `date` from start/end objects.

Returns the full classification object with all extracted flags and the cleaned location (set to `null` for online events).

---

### Sub-Workflow A: Departure Calculator (`co-departure-calculator.json`)

**Purpose:** Queries the Google Maps Distance Matrix API for transit, driving, and walking travel times, selects the optimal transport mode, adds time buffers, and calculates the departure time for a given event.
**Called by:** WF1 (Nightly Planning), WF4 (Location Handler), WF8 (Calendar Sync)
**Inputs:** `origin` (address or coordinates), `destination` (address or coordinates), `event_start` (ISO datetime), `force_car_mode` (boolean).
**Outputs:** An object with `departure_time` (ISO string), `commute_minutes` (total including buffers), `commute_mode` (selected mode), and `mode_comparison` (all three mode durations). Returns `{ error: true, message: "..." }` on failure.
**Node count:** 9

#### Step-by-step Flow

**Step 1: Input** (`executeWorkflowTrigger`)
Receives origin, destination, event start time, and force-car flag from the parent workflow.

**Step 2: Prepare API Params** (`code` -- runs once for all items)
Converts the `event_start` ISO string to a Unix timestamp (seconds since epoch) for use as the `departure_time` parameter in Google Maps API calls. Passes through origin, destination, and force-car flag.

**Steps 3-5: Three parallel API calls** (all `httpRequest` -- Google Maps Distance Matrix API)

These three nodes run in parallel from the same source:

- **Step 3: Get Transit Time** -- Queries with `mode=transit` and `departure_time`. Authenticated via `google-maps-api` query auth credential. Configured to continue on error.
- **Step 4: Get Driving Time** -- Queries with `mode=driving` and `departure_time`. Same credential and error handling.
- **Step 5: Get Walking Time** -- Queries with `mode=walking` (no departure_time -- walking time is constant). Same credential and error handling.

**Steps 6-8: Tag each result** (all `set` nodes)

Each API response is tagged with its transport mode so they can be identified after merging:

- **Step 6: Tag Transit** -- Adds `transport_mode = "transit"`.
- **Step 7: Tag Driving** -- Adds `transport_mode = "driving"`.
- **Step 8: Tag Walking** -- Adds `transport_mode = "walking"`.

All three tagged results feed into the next node.

**Step 9: Combine Results** (`merge` -- append mode)
Merges the three tagged API responses into a single list (3 items, one per mode).

**Step 10: Compare Modes + Calculate Departure** (`code` -- runs once for all items)
Receives all three tagged responses and applies the mode selection algorithm:

1. **Parse each response:** Extracts `duration_sec`, `duration_text`, and `distance_m` from each Google Maps response. Handles missing/error responses gracefully.
2. **Mode selection:**
   - If `force_car` is true: always selects "driving".
   - Otherwise: compares transit, walking (only if under 20 minutes), and driving -- picks the shortest.
   - If no options are available, returns an error.
3. **Buffer calculation:**
   - Driving/rideshare: +10 min parking + 7 min reliability + 5 min leaving = 22 min buffer.
   - Transit/walking: +7 min reliability + 5 min leaving = 12 min buffer.
4. **Departure time:** Subtracts (travel time + buffer) from the event start time.
5. Returns the final departure time, total commute minutes, selected mode, and a comparison object.

---

### Sub-Workflow B: Weather Advisor (`co-weather-advisor.json`)

**Purpose:** Fetches hourly weather forecast data for the event location and time window, then generates clothing advice and dangerous condition warnings.
**Called by:** WF1 (Nightly Planning), WF4 (Location Handler), WF8 (Calendar Sync)
**Inputs:** `location_coords` (comma-separated "lat,lng" string), `event_start` (ISO datetime), `event_end` (ISO datetime).
**Outputs:** An object with `weather_worst` (summary string), `clothing_advice` (recommendations), `dangerous_conditions` (array of warnings), plus `min_temp`, `max_temp`, `max_precip_mm`, and `max_wind_ms`.
**Node count:** 4

#### Step-by-step Flow

**Step 1: Input** (`executeWorkflowTrigger`)
Receives location coordinates, event start, and event end from the parent workflow.

**Step 2: Extract Lat/Lng** (`code`)
Splits the `location_coords` string on comma to extract separate `lat` and `lng` numeric values. Passes through `event_start` and `event_end`.

**Step 3: Get Weather Forecast** (`httpRequest` -- OpenWeatherMap One Call 3.0)
Calls `https://api.openweathermap.org/data/3.0/onecall` with lat/lng. Requests only hourly forecast data (excludes current, minutely, daily, alerts). Uses metric units. Configured to continue on fail.

**Step 4: Analyze Weather + Build Advice** (`code`)
Processes the hourly forecast against the event time window:

1. **Error handling:** If the API returned an error, returns a "Weather data unavailable" fallback.
2. **Time window:** Filters hourly data from 1 hour before event start (commute time) through event end.
3. **Worst-case extraction:** Finds min/max temperature, max precipitation (rain + snow), max wind speed, and max precipitation probability.
4. **Clothing advice:** Based on thresholds:
   - Temperature: >30C light/breathable, >20C light layers, >10C jacket, >0C warm coat, <=0C heavy winter coat.
   - Precipitation >50%: bring umbrella.
   - Wind >10 m/s: windbreaker.
   - Temp <5C: gloves and scarf.
5. **Dangerous conditions:**
   - Near-freezing (0-2C) with precipitation: ice risk.
   - Heavy snow (>5mm below 1C).
   - Extreme wind (>70 km/h): consider postponing.
   - Extreme heat (>38C): stay hydrated.

---

### Sub-Workflow C: Notifier (`co-notifier.json`)

**Purpose:** Delivers notification messages across multiple channels (push notification, text-to-speech, SMS) based on the requested channel list, then reports which deliveries succeeded.
**Called by:** WF3 (Reminder Engine)
**Inputs:** `channels` (JSON array string, e.g., `'["push","tts","sms"]'`), `title`, `message`, `priority`, `ntfy_topic`, `event_id`, `user_id`, `phone_number`.
**Outputs:** An object with `delivered` (boolean) and `results` (array of per-channel outcomes).
**Node count:** 10

#### Step-by-step Flow

**Step 1: Input** (`executeWorkflowTrigger`)
Receives notification payload from the parent workflow.

**Step 2: Parse Channels** (`code`)
Parses the `channels` JSON string (defaults to `["push"]` if missing) and sets three boolean flags: `send_push`, `send_tts`, and `send_sms`.

**Steps 3-5: Three parallel channel checks** (all `if` nodes)

- **Step 3: Send Push?** -- If TRUE: proceeds to Step 6.
- **Step 4: Send TTS?** -- If TRUE: proceeds to Step 7.
- **Step 5: Send SMS?** -- If TRUE: proceeds to Step 8.

**Step 6: Send ntfy Push** (`httpRequest` -- POST to `http://localhost:8080/{ntfy_topic}`)
Sends a push notification with title, priority, calendar tag, and an "Cancel Event" action button (X-Actions header with POST URL to the cancel webhook). Body is the message text. Configured to continue on fail.

**Step 7: Speak on Google Home** (`executeCommand`)
Runs the TTS shell script at `/home/pi/commute-automation/tts.sh` to announce the message on a Google Home device. Configured to continue on fail.

**Step 8: Send Twilio SMS** (`httpRequest` -- POST to Twilio Messages API)
Sends an SMS via `https://api.twilio.com/2010-04-01/Accounts/{SID}/Messages.json` with form-urlencoded body (From, To, Body). Configured to continue on fail.

**Step 9: Combine Results** (`merge` -- append mode)
Collects results from all triggered channels into a single list.

**Step 10: Build Result** (`code`)
Returns `delivered: true` if at least one channel succeeded, plus the full results array.

---

## Main Workflows

---

### WF1: Nightly Planning (`co-nightly-planning.json`)

**Purpose:** Runs daily at 4 AM to reconcile stale events from the previous day, then builds a fresh commute plan for each user by fetching their calendar, classifying events, computing departure times, weather, and reminders, and finally sends a morning digest notification and initializes GPS tracking.
**Trigger:** Schedule trigger at 04:00 every day (timezone: Europe/Bucharest)
**Node count:** 28

#### Step-by-step Flow

##### Phase A: Stale Event Reconciliation

**Step 1: 4 AM Daily** (`scheduleTrigger`)
Fires automatically every day at 04:00. This is the entry point for the entire nightly planning workflow.

**Step 2: Read Yesterday's Plan** (`googleSheets` - read)
Reads all rows from the `daily_plan` sheet. This fetches every event row (including yesterday's events) so the next node can identify which ones were never resolved.

**Step 3: Find Stale Events** (`code`)
Scans all plan rows and identifies events whose `end_time + 60 minutes` has passed but whose `current_state` is not terminal (arrived, cancelled, skipped, expired). Assigns `expired` -- unless the user recently requested cancellation (within 60 minutes), in which case it assigns `cancelled`. Increments `state_version`.

**Step 4: Any Stale Events?** (`if`)
Checks whether any stale events were found (item count > 0).

> **If TRUE:**

**Step 5: Update Stale State** (`googleSheets` - update)
Updates each stale event's `current_state` and `state_version` in `daily_plan`. Matches by `event_id`.

**Step 6: Write Stale History** (`googleSheets` - append)
Appends a record for each stale event to `commute_history` with `arrived_on_time: unknown`.

> **If FALSE:** Skips to Step 7.

##### Phase B: Per-User Planning

**Step 7: Read User Config** (`googleSheets` - read)
Reads all rows from `user_config` to get each user's settings.

**Step 8: Loop Users** (`splitInBatches`, batchSize=1)
Iterates over each user. When done, goes to Step 28 (tracking phase).

**Step 9: Detect Dynamic Origin** (`code`)
Sets the user's default origin to `HOME_COORDS` from n8n Variables.

**Step 10: Read Latest Location** (`googleSheets` - read)
Reads up to 10 recent rows from `location_updates`, sorted by timestamp descending.

**Step 11: Check If Travelling** (`code`)
Filters location rows to the current user and checks whether their most recent GPS fix (within 8 hours) is more than 5 km from home using Haversine distance. If so, overrides origin to the user's current location with `origin_type: 'travel'`.

**Step 12: Refresh Calendar Token** (`httpRequest` - POST)
Sends a POST to `https://oauth2.googleapis.com/token` to obtain a fresh OAuth2 access token using the user's `calendar_refresh_token`.

**Step 13: Fetch Calendar Events** (`httpRequest` - GET)
Calls `https://www.googleapis.com/calendar/v3/calendars/primary/events` for today's events.

**Step 13a: Read Today's Existing Plan** (`googleSheets` - read)
Runs in parallel. Reads the current `daily_plan` to detect events that already have active state transitions (avoids resetting them during re-runs).

**Step 14: Extract Calendar Events** (`code`)
Extracts the `items` array from the Calendar API response into individual n8n items.

##### Phase C: Per-Event Processing

**Step 15: Loop Events** (`splitInBatches`, batchSize=1)
Iterates over each calendar event. When done, goes to Step 25 (digest phase).

**Step 16: Classify Event** (`executeWorkflow` - Sub-Workflow D)
Calls the event classifier, passing event id, summary, location, start, end, attendees, and conferenceData.

**Step 17: Should Skip?** (`if`)

> **If TRUE (skip):** Loops back to Step 15 for next event.

> **If FALSE:** Continues to Step 18.

**Step 18: Is Online?** (`if`)

> **If TRUE (online):**

**Step 19: Calculate Online Reminders** (`code`)
Builds a plan row for an online event with `remind_60` and `remind_15` only. No commute, no weather. Goes to Step 24b.

> **If FALSE (in-person):**

**Step 20: Geocode Location** (`httpRequest` - Google Maps Geocoding)
Converts the event location to lat/lng coordinates.

**Step 21: Geocode OK?** (`if`)

> **If TRUE:**

**Step 22: Extract Coordinates** (`set`)
Extracts lat/lng into a `"lat,lng"` string.

**Step 23a: Calculate Departure** (`executeWorkflow` - Sub-Workflow A)
Calls the departure calculator with origin, destination, event start, and force-car flag.

**Step 23b: Get Weather** (`executeWorkflow` - Sub-Workflow B)
Calls the weather advisor with destination coordinates and event start/end times.

**Step 24a: Calculate Reminders + Build Row** (`code`)
Combines all outputs into a complete `daily_plan` row. Computes `remind_dressed` (capped by `dress_cap`, floored at 4 AM), `remind_60`, `remind_30`, `remind_15`, and `remind_now`. Preserves existing `current_state` if the event was already updated mid-day.

> **If FALSE (geocode failed):**

**Step 21b: Handle Geocode Failure** (`code`)
Builds a plan row without commute data. Sets basic reminders relative to event start.

**Step 24b: Upsert Daily Plan Row** (`googleSheets` - upsert)
Writes the row to `daily_plan` matched on `event_id`. Loops back to Step 15 for next event.

##### Phase D: Morning Digest

**Step 25: Re-Read Today's Plan for Digest** (`googleSheets` - read)
Reads `daily_plan` filtered to this user and today's date.

**Step 26: Build Morning Digest** (`code`)
Constructs a human-readable text summary: greeting, origin info, and each event with departure time, commute mode, and clothing advice. Tentative events get a warning prefix. If no events: "No events today -- enjoy your free day!"

**Step 27: Send Morning Digest** (`httpRequest` - POST to ntfy)
Sends the digest to `http://localhost:8080/{ntfy_topic}` with title "Morning Commute Plan", priority 2, and calendar tag. Loops back to Step 8 for next user.

##### Phase E: GPS Tracking Initialization

**Step 28: Re-Read Plan for Tracking Calc** (`googleSheets` - read)
Reads `daily_plan` filtered to today's date (all users).

**Step 29: Calculate Tracking Intervals** (`code`)
Groups events by user, finds each user's earliest departure, and calculates an initial GPS polling interval using configurable tiers from `TRACKING_INTERVAL_TIERS` (e.g., `60:20,30:10,15:5,5:3`). Users with only online events are filtered out.

**Step 30: Anyone Needs Tracking?** (`if`)

> **If TRUE:**

**Step 31: Loop Users for Tracking** (`splitInBatches`, batchSize=1)

**Step 32: Send Start-Tracking Notification** (`httpRequest` - POST to ntfy)
Sends a JSON notification with `action: "start_tracking"`, the computed `interval`, and `earliest_departure`. This instructs the phone to begin GPS polling.

> **If FALSE:** Workflow ends.

---

### WF2: Traffic Refresh (`co-traffic-refresh.json`)

**Purpose:** Periodically recalculates commute/departure times for today's in-person events using live traffic data, and sends a push notification if the departure time has shifted by more than 10 minutes.
**Trigger:** Cron schedule -- every 2 hours from 6 AM to 6 PM (`0 6-18/2 * * *`)
**Node count:** 8

#### Step-by-step Flow

**Step 1: Every 2 Hours 6AM-6PM** (`scheduleTrigger`)
Fires on a cron schedule every 2 hours between 6:00 and 18:00 daily.

**Step 2: Read Daily Plan** (`googleSheets` -- read)
Reads all rows from the `daily_plan` sheet.

**Step 3: Filter Active Non-Online Events** (`code`)
Filters to events that are: planned for today, not online, not in a terminal state, and have destination coordinates.

**Step 4: Loop Events** (`splitInBatches`, batch size 1)
Iterates over the filtered events one at a time.

**Step 5: Recalculate Departure** (`executeWorkflow` -- Sub-Workflow A)
Calls the departure calculator with origin, destination, event start, and force-car flag. Returns updated departure time and commute minutes.

**Step 6: Check If Changed >10 Min** (`code`)
Compares original departure time with the recalculated one. Outputs the difference in minutes and a `changed_more_than_10` flag.

**Step 7: Update Traffic Columns** (`googleSheets` -- update)
Writes `traffic_updated_departure` and `traffic_updated_commute_min` back to `daily_plan` by `event_id`.

**Step 8: Changed >10 Min?** (`if`)

> **If TRUE:** Proceeds to Step 9.

> **If FALSE:** Loops back to Step 4 for next event.

**Step 9: Send Traffic Alert** (`httpRequest` -- POST to ntfy)
Sends a notification with title "Traffic Update", priority 3, traffic-light emoji tag, and body describing the time shift. Loops back to Step 4.

---

### WF3: Reminder Engine (`co-reminder-engine.json`)

**Purpose:** Polls the daily plan every 5 minutes and sends time-based departure reminders (including "get dressed" reminders for at-home users) via push notifications and optionally TTS, with busy-day batching for users with many events.
**Trigger:** Cron schedule -- every 5 minutes from 4 AM to midnight (`*/5 4-23 * * *`)
**Node count:** 9

#### Step-by-step Flow

**Step 1: Every 5 Min 4AM-Midnight** (`scheduleTrigger`)
Fires every 5 minutes between 04:00 and 23:55 daily.

**Step 2: Read Daily Plan** (`googleSheets` -- read)
Reads all rows from `daily_plan`.

**Step 3: Find Due Reminders** (`code`)
The core scheduling logic. For each active event:
1. Determines effective departure time (prefers `traffic_updated_departure` over `departure_time`).
2. Builds standard reminders: `remind_60`, `remind_30`, `remind_15`, `remind_now` (push; `remind_now` also uses TTS).
3. For at-home users, adds "get dressed" reminders at configurable intervals (from `DRESS_REMINDER_INTERVALS`), filtered by `dress_cap`, floored at 4 AM. Tentative events get TTS stripped.
4. A reminder is "due" if its target time falls within the last 5-minute window AND hasn't been sent (checked against `reminders_sent`).
5. If a user has >5 active events, flags as `busy_day`.

**Step 4: Any Due Reminders?** (`if`)

> **If TRUE:** Continues to Step 5.

> **If FALSE:** Workflow ends silently.

**Step 5: Batch Busy-Day Reminders** (`code`)
For busy-day users, batches all "dressed" and "60-min" reminders into a single digest per user. Critical reminders (`remind_30`, `remind_15`, `remind_now`) are never batched.

**Step 6: Loop Reminders** (`splitInBatches`, batch size 1)

**Step 7: Build Reminder Message** (`code`)
Constructs title and body based on reminder type:
- `remind_dressed_*` (>=60 min): "Time to get dressed" + clothing advice.
- `remind_dressed_*` (<60 min): "Leaving in N min -- are you dressed?"
- `remind_60`: "1 Hour Warning" + event details.
- `remind_30`: "30 Minutes" + commute mode.
- `remind_15`: "15 Minutes!"
- `remind_now`: "LEAVE NOW" + commute details.
- Tentative events get a warning prefix.

**Step 8: Send Notification** (`executeWorkflow` -- Sub-Workflow C)
Calls the notifier with user_id, message, title, channels, event_id, and ntfy_topic.

**Step 9: Mark Reminder Sent** (`code`)
Appends the reminder type to `reminders_sent` (deduplicating via Set). For busy-day digests, appends all batched types.

**Step 10: Update Reminders Sent** (`googleSheets` -- update)
Writes updated `reminders_sent` back to `daily_plan` by `event_id`. Loops back to Step 6.

---

### WF4: Location Update Handler (`co-location-handler.json`)

**Purpose:** Receives GPS location updates from a mobile device, runs a state machine to track commute progress (planned -> should_leave -> departed -> arrived), detects at-home status, and recalculates downstream events when the user's origin changes mid-day.
**Trigger:** POST webhook at `/location-update` (header auth)
**Node count:** 27

#### Step-by-step Flow

**Step 1: Location Update** (`webhook`)
Listens for POST requests at `/location-update` with header auth. Expects `user_id`, `lat`, `lng`, `accuracy`, `speed`, and optional `wifi_at_home` / `wifi_at_home_at`. Response is deferred.

**Step 2: Validate Input** (`if`)
Checks that `user_id`, `lat`, `lng` are non-empty and coordinates are in valid ranges (-90 to 90, -180 to 180).

> **If FALSE:**

**Step 3: Error Response** (`respondToWebhook`)
Returns HTTP 400 with `{"error":"invalid input","interval":10}`.

> **If TRUE:**

**Step 4: Log Location** (`googleSheets` -- append)
Appends a row to `location_updates` with timestamp, user_id, lat, lng, accuracy, speed, wifi_at_home, wifi_at_home_at.

**Step 5: Read User's Daily Plan** (`googleSheets` -- read)
Reads from `daily_plan` filtered by user_id and today's date.

**Step 6: Select Active Event + Run State Machine** (`code`)
The core logic node:

1. **Validation:** Checks required n8n Variables exist (`HOME_COORDS`, `HOME_RADIUS_METERS`, `DRESS_REMINDER_INTERVALS`, `TRACKING_INTERVAL_TIERS`).
2. **Event selection:** Filters to non-terminal, non-online physical events with coordinates. Picks earliest `departure_time`.
3. **Consent-safety gate:** If `cancel_requested_at` is set within 60 minutes, short-circuits to `action: force_cancel`.
4. **State transitions** (Haversine distance):
   - **planned -> should_leave:** Current time within 20 min of departure.
   - **planned/should_leave -> departed:** User >max(500m, accuracy*2) from origin AND within 30 min of departure.
   - **departed -> arrived:** User within max(200m, accuracy*2) of destination.
   - **Urgent alert:** User in `should_leave` state, still near origin, past departure time.
5. **Multi-event origin recalculation:** On arrival at event A, builds list of remaining events needing origin updated.
6. **At-home detection:** WiFi-first (fresh within 20 min), GPS fallback (distance from HOME_COORDS).
7. **Dynamic polling interval:** Matches `TRACKING_INTERVAL_TIERS` to minutes-until-departure.

**Step 7: Return Interval** (`respondToWebhook`)
Returns HTTP 200 with `status`, `interval`, `at_home`, `at_home_source`.

**Step 8: What Action?** (`switch`)
Routes on the `action` field:

> **Output 0 (`update_state` or `force_cancel`):**

**Step 9: Re-Read Event Row** (`googleSheets` -- read)
Re-reads the event from `daily_plan` by `event_id` for concurrency check.

**Step 10: Check State Version** (`code`)
Compares `state_version` in the fresh row against expected version. Sets `version_ok: false` on mismatch.

**Step 11: Version OK?** (`if`)

> **If TRUE:**

**Step 12: Update State** (`googleSheets` -- update)
Sets `current_state`, increments `state_version`, records `departed_at` if departing.

**Step 13: Arrived?** (`if`)

> **If TRUE:**

**Step 14: Parse Arrival Data** (`code`)
Calculates on-time status (5-min grace) and arrival delay.

**Step 15: Write Commute History** (`googleSheets` -- append)
Writes 14 columns to `commute_history`.

**Step 16: Has Events To Update?** (`if`)

> **If TRUE:**

**Step 17: Prepare Events for Recalculation** (`code`)
Outputs one item per remaining event with `new_origin_coords`.

**Step 18: Loop Events To Update** (`splitInBatches`, batch size 1)

**Step 19: Read Event Row** (`googleSheets` -- read by event_id)

**Step 20: Recalculate Departure** (`executeWorkflow` -- Sub-Workflow A)

**Step 21: Recalculate Weather** (`executeWorkflow` -- Sub-Workflow B)

**Step 22: Rebuild Row + Clear Unfired Reminders** (`code`)
Merges recalculated data, recomputes all reminder times, preserves already-sent reminders, clears unfired dress reminders. Bumps `state_version`.

**Step 23: Update Event Row** (`googleSheets` -- update)
Writes all recalculated fields by `event_id`.

**Step 24: Departure Changed >10 Min?** (`if`)

> **If TRUE:**

**Step 25: Notify Updated Departure** (`httpRequest` -- POST to ntfy)
Title "Departure Updated", body with new time and shift. Loops back to Step 18.

> **If FALSE:** Loops back to Step 18.

> **If FALSE at Step 11 (version mismatch):**

**Step 11b: Log Version Conflict** (`googleSheets` -- append to `error_log`)
Records the version mismatch for monitoring.

> **Output 1 (`urgent_alert`):**

**Step 8b: Send Urgent Alert** (`httpRequest` -- POST to ntfy)
Priority 5, title "You should have left!", warning tag.

> **Output 2 (`none` / fallback):** Goes directly to at-home check.

**Step 26: Stop Tracking?** (`if`)

> **If TRUE:**

**Step 26b: Stop Tracking** (`httpRequest` -- POST to ntfy)
JSON body with `action: "stop_tracking"`. Priority 2, stop_sign tag.

**Step 27: Location Update Needed?** (`if`)
Checks if any events need `at_home` or `away_walk_back_min` updated (at-home status changed or walk-back time shifted by >=3 min).

> **If TRUE:**

**Step 28: Prepare Location Updates** (`code`)
Outputs one item per event needing location state update, with `at_home` flag and `away_walk_back_min` value.

**Step 29: Loop Location Updates** (`splitInBatches`, batch size 1)

**Step 30: Update Location State** (`googleSheets` -- update by event_id)
Sets the `at_home` and `away_walk_back_min` columns. Loops back to Step 29.

> **When loop completes:**

**Step 31: Away Notify?** (`if`)
Checks if user just left home or walk-back increased significantly (>=5 min).

> **If TRUE:**

**Step 32: Send Away Alert** (`httpRequest` -- POST to ntfy)
Sends push notification with walk-back estimate and "return by" time. Priority 4, walking tag.

#### Away-from-home adjustment

On each GPS ping, WF4 checks if the user is away from home (via WiFi + GPS). If away with upcoming home-origin events, it estimates walk-back time using haversine distance (capped at 60 min) and stores `away_walk_back_min` in the daily plan. WF3 subtracts this offset from departure times, shifting all reminders earlier. State transitions (`should_leave`, departure detection) also use traffic-updated departure adjusted by walk-back. A push notification fires when the user first leaves home or when the walk-back estimate increases significantly. Walk-back only applies to events with `origin_type = 'home'` -- event chain events (A -> B) are unaffected.

---

### WF8: Mid-Day Calendar Sync (`co-calendar-sync.json`)

**Purpose:** Polls Google Calendar every 30 minutes throughout the day and synchronizes any new, changed, or deleted events back into the daily plan, recalculating departure times, weather, and reminders as needed.
**Trigger:** Cron schedule -- every 30 minutes between 6 AM and 10 PM (`*/30 6-22 * * *`)
**Node count:** 33

#### Step-by-step Flow

**Step 1: Every 30 Min 6AM-10PM** (`scheduleTrigger`)

**Step 2: Read User Config** (`googleSheets` -- read from `user_config`)

**Step 3: Loop Users** (`splitInBatches` -- batch size 1)
When done, workflow ends.

**Step 4: Refresh Calendar Token** (`httpRequest` -- POST to Google OAuth2)

**Step 5: Fetch Calendar Events** (`httpRequest` -- GET Google Calendar API)
Today's events, max 50 results.

**Step 6: Extract Calendar Events** (`code`)
Unwraps `items` array.

**Step 7: Read Existing Daily Plan** (`googleSheets` -- read from `daily_plan`)

**Step 8: Diff Calendar vs Plan** (`code`)
Compares calendar events against existing plan. Produces three lists: `new_events`, `changed_events`, `deleted_events`, plus `has_changes` flag.

**Step 9: Any Changes?** (`if`)

> **If FALSE:** Loops back to Step 3.

> **If TRUE:**

#### Section A: New Events

**Step 10: Process New Events** (`code`)
Extracts new events with user_id and ntfy_topic.

**Step 11: Loop New Events** (`splitInBatches` -- batch size 1)
When done: goes to Step 24 (changed events).

**Step 12: Classify New Event** (`executeWorkflow` -- Sub-Workflow D)

**Step 13: Skip This Event?** (`if`)

> **If TRUE:** Loops back to Step 11.

**Step 14: Online Event?** (`if`)

> **If TRUE:**

**Step 20: Build Online Event Row** (`code`)
Simplified row with `remind_60` and `remind_15` only.

> **If FALSE:**

**Step 15: Geocode Location** (`httpRequest` -- Google Geocoding)

**Step 16: Determine Origin + At-Home** (`code`)
Defaults to HOME_COORDS as origin.

**Step 17: Calculate Departure** (`executeWorkflow` -- Sub-Workflow A)

**Step 18: Get Weather** (`executeWorkflow` -- Sub-Workflow B)

**Step 19: Build New Event Row** (`code`)
Complete plan row with all reminders and clothing advice.

**Step 21: Upsert New Event** (`googleSheets` -- upsert by event_id)

**Step 22: Departure Within 2 Hours?** (`if`)

> **If TRUE:**

**Step 23: Alert: New Event Added** (`httpRequest` -- POST to ntfy)

> **If FALSE:** Loops back to Step 11.

#### Section B: Changed Events

**Step 24: Process Changed Events** (`code`)
Extracts changed events with `location_changed` and `time_changed` flags.

**Step 25: Loop Changed Events** (`splitInBatches` -- batch size 1)
When done: goes to Step 34 (deleted events).

**Step 26: Location Changed?** (`if`)

> **If TRUE:**

**Step 27: Re-Geocode Changed Location** (`httpRequest`)

**Step 28: Recalculate Departure** (`executeWorkflow` -- Sub-Workflow A)

**Step 29: Recalculate Weather** (`executeWorkflow` -- Sub-Workflow B)

> **If FALSE (time-only change):** Goes directly to Step 30.

**Step 30: Build Update Row** (`code`)
Recalculates departure, reminders, and diff_minutes.

**Step 31: Update Changed Event** (`googleSheets` -- update by event_id)

**Step 32: Departure Changed Significantly?** (`if`)

> **If TRUE:**

**Step 33: Alert: Event Updated** (`httpRequest` -- POST to ntfy)
Priority 4 if moved >15 min earlier, otherwise priority 3.

> **If FALSE:** Loops back to Step 25.

#### Section C: Deleted Events

**Step 34: Process Deleted Events** (`code`)
Sets state to "cancelled" (if cancel_requested_at within 60 min) or "skipped".

**Step 35: Loop Deleted Events** (`splitInBatches` -- batch size 1)
When done: goes to Step 38.

**Step 36: Mark Event Skipped/Cancelled** (`googleSheets` -- update by event_id)

**Step 37: Alert: Event Removed** (`httpRequest` -- POST to ntfy)

**Step 38: Check If Tracking Should Stop** (`code`)
Checks if user has any remaining active, non-online events for today.

**Step 39: Stop Tracking?** (`if`)

> **If TRUE:**

**Step 40: Send Stop Tracking** (`httpRequest` -- POST to ntfy)
JSON body with `action: "stop_tracking"`.

> **If FALSE:** Loops back to Step 3.

---

## Support Workflows

---

### WF5a: Weekly Cleanup (`co-weekly-cleanup.json`)

**Purpose:** Purges stale data from Google Sheets on a weekly schedule -- location updates older than 7 days and error log entries older than 90 days -- then stamps the cleanup timestamp in user config.
**Trigger:** Every Sunday at 4:00 AM (`0 4 * * 0`)
**Node count:** 11

#### Step-by-step Flow

**Step 1: Sunday 4 AM** (`scheduleTrigger`)

**Step 2: Read Location Updates** (`googleSheets` -- read from `location_updates`)

**Step 3: Find Old Rows** (`code`)
Identifies rows older than 7 days. Sorts by row number descending for safe bottom-up deletion.

**Step 4: Delete Old Locations** (`splitInBatches` -- batch size 1)

**Step 5: Delete Row** (`googleSheets` -- delete from `location_updates`)
Deletes one row per iteration. Loops back to Step 4.

> **Done:**

**Step 6: Read Error Log** (`googleSheets` -- read from `error_log`)

**Step 7: Find Old Errors** (`code`)
Identifies entries older than 90 days. Sorts descending.

**Step 8: Delete Old Error Rows** (`splitInBatches` -- batch size 1)

**Step 9: Delete Error Row** (`googleSheets` -- delete from `error_log`)
Loops back to Step 8.

> **Done:**

**Step 10: Read User Config** (`googleSheets` -- read from `user_config`)

**Step 11: Update last_cleanup_at** (`googleSheets` -- update `user_config`)
Sets `last_cleanup_at` to current timestamp for all users. Matched by `user_id`.

---

### WF5b: Monthly Statistics (`co-monthly-statistics.json`)

**Purpose:** Calculates per-user commute statistics for the previous month (trip counts, on-time rates, delay averages, transport modes, API cost estimates), sends a digest notification, then cleans up data older than 90 days.
**Trigger:** 1st of every month at 8:00 AM (`0 8 1 * *`)
**Node count:** 14

#### Step-by-step Flow

**Step 1: 1st of Month 8 AM** (`scheduleTrigger`)

**Step 2: Read Commute History** (`googleSheets` -- read from `commute_history`)

**Step 3: Calculate Monthly Stats** (`code`)
Filters to last month. Groups by user. Calculates: total trips, on-time %, average delay, mode breakdown. Estimates API usage costs.

**Step 4: Loop Users** (`splitInBatches` -- batch size 1)

**Step 5: Build Monthly Digest** (`code`)
Formats stats into a human-readable summary.

**Step 6: Send Monthly Digest** (`httpRequest` -- POST to ntfy)
URL: `http://localhost:8080/reminders-{user_id}`. Title "Monthly Commute Report", priority 2, bar_chart tag.

> **Done:**

**Step 7: Read Daily Plan for Cleanup** (`googleSheets` -- read from `daily_plan`)

**Step 8: Find Old Daily Plan Rows** (`code`)
Finds rows older than 90 days.

**Step 9: Delete Old Plan Rows** (`splitInBatches` -- batch size 1)

**Step 10: Delete Plan Row** (`googleSheets` -- delete)
Loops back to Step 9.

> **Done:**

**Step 11: Read Commute History for Cleanup** (`googleSheets` -- read from `commute_history`)

**Step 12: Find Old History Rows** (`code`)
Finds entries older than 90 days.

**Step 13: Delete Old History Rows** (`splitInBatches` -- batch size 1)

**Step 14: Delete History Row** (`googleSheets` -- delete)
Loops back to Step 13. Workflow ends when done.

---

### WF6: Error Handler (`co-error-handler.json`)

**Purpose:** Catches unhandled errors from any workflow, sanitizes the message to remove sensitive data (coordinates, API keys), logs to Google Sheets, sends an ntfy alert, and falls back to a local file if both fail.
**Trigger:** Error Trigger -- activated when any workflow with this error handler fails
**Node count:** 7

#### Step-by-step Flow

**Step 1: Catch Errors** (`errorTrigger`)
Receives the failing workflow's name, execution ID, and error details.

**Step 2: Sanitize Error** (`code`)
Truncates to 200 chars, replaces GPS coordinates with `[REDACTED_COORD]`, replaces hex strings (API keys) with `[REDACTED_KEY]`, strips shell metacharacters.

**Step 3: Log to Error Sheet** (`googleSheets` -- append to `error_log`) *[continueOnFail]*
Writes timestamp, workflow_name, error_message, execution_id, resolved. Runs in parallel with Step 4.

**Step 4: Alert via ntfy** (`httpRequest` -- POST) *[continueOnFail]*
Sends priority 4 notification to `system-errors` topic with rotating_light tag.

**Step 5: Check If Both Failed** (`code`)
Inspects both outputs for errors. Sets `both_failed` flag.

**Step 6: Both Failed?** (`if`)

> **If TRUE:**

**Step 7: Fallback to File** (`code`)
Appends error to `/home/pi/.n8n/error-fallback.log` using `fs.appendFileSync`.

> **If FALSE:** Workflow ends.

---

### WF7: Heartbeat (`co-heartbeat.json`)

**Purpose:** Runs health checks every 30 minutes, verifying Google Sheets, ntfy, and planning are operational. Triggers catch-up runs if the daily plan is missing. Sends failure alerts and daily "all OK" summaries. Pings an external dead man's switch.
**Trigger:** Every 30 minutes between 4 AM and midnight (`*/30 4-23 * * *`)
**Node count:** 13

#### Step-by-step Flow

**Step 1: Every 30 Min 4AM-Midnight** (`scheduleTrigger`)
Fans out to three parallel checks:

**Step 2a: Check Sheets Access** (`googleSheets` -- read 1 row from `user_config`) *[continueOnFail]*

**Step 2b: Check ntfy Access** (`httpRequest` -- POST heartbeat ping to `system-errors`) *[continueOnFail]*

**Step 2c: Check Daily Plan** (`googleSheets` -- read from `daily_plan`) *[continueOnFail]*

**Step 3: Evaluate Health** (`code`)
Checks: Sheets connectivity, ntfy connectivity, planning status (after 5 AM, are today's rows present?), cleanup recency (last_cleanup_at within 48 hours). Sets `should_retrigger_wf1` and `is_daily_summary` flags.

**Step 4: Should Re-trigger?** (`if`)

> **If TRUE (no plan, 5-9 AM):**

**Step 5: Catch-up Planning** (`executeWorkflow` -- triggers WF1)

**Step 6: Catch-up WF2** (`executeWorkflow` -- triggers WF2)

> **If FALSE:** Continues to Step 7.

**Step 7: Any Failures?** (`if`)

> **If TRUE:**

**Step 8: Send Failure Alert** (`httpRequest` -- POST to ntfy)
Priority 4, title "Health Check Failed", warning tag. Body lists all failures.

**Step 9: Ping Dead Man's Switch** (`httpRequest` -- GET to `https://hc-ping.com/{UUID}`) *[continueOnFail]*

**Step 10: Is Daily Summary Time?** (`if`)

> **If TRUE (8 AM window):**

**Step 11: Send System OK** (`httpRequest` -- POST to ntfy)
Priority 2, title "System Health: OK", white_check_mark tag.

> **If FALSE:** Workflow ends.

---

### Cancel Event Webhook (`co-cancel-event.json`)

**Purpose:** HTTP endpoint that allows users to cancel a tracked event (via the "Cancel Event" action button in push notifications), updating state in Google Sheets and notifying the device to stop GPS tracking.
**Trigger:** POST webhook at `/webhook/cancel-event` (header auth with `X-Cancel-Token`)
**Node count:** 7

#### Step-by-step Flow

**Step 1: Cancel Event** (`webhook` -- POST `/webhook/cancel-event`)
Authenticates via `cancel-auth` header credential. Request body contains `event_id` and `user_id`. Response deferred.

**Step 2: Read Event Row** (`googleSheets` -- read from `daily_plan`)
Reads all rows to find the matching event.

**Step 3: Check State + Cancel** (`code`)
Finds the event by `event_id` and `user_id`. Validates:
1. **Not found:** Returns `do_update: false`, reply "Event not found".
2. **Terminal state:** Returns `do_update: false`, reply "Already terminal: {state}".
3. **Valid cancel:** Returns `do_update: true`, incremented `state_version`, and `ntfy_topic`.

**Step 4: Should Update?** (`if`)

> **If TRUE:**

**Step 5: Write Cancel** (`googleSheets` -- update `daily_plan`)
Sets `current_state: "cancelled"`, `cancel_requested_at: now`, incremented `state_version`. Matched by `event_id`.

**Step 6: Stop Tracking** (`httpRequest` -- POST to ntfy)
Sends notification with title "Event Cancelled -- Stop Tracking", priority 2, stop_sign tag. JSON body includes `action: "stop_tracking"`.

> **If FALSE:** Goes directly to Step 7.

**Step 7: Reply** (`respondToWebhook`)
Returns HTTP 200 with the reply message ("Event cancelled", "Event not found", or "Already terminal: {state}").
