# Tasker Setup Guide

Complete configuration for the Android Tasker app that feeds GPS and WiFi data to the Commute Optimizer system. Tasker runs on each user's phone and communicates with WF4 (Location Handler) via HTTP webhook.

---

## Overview

```
[Tasker on Phone]
    |
    ├── Profile: GPS Polling (timer-based)
    │   └── Task: Send Location Update
    │       └── HTTP POST → WF4 webhook
    │           └── Response: { interval, at_home }
    │               └── Tasker adjusts next poll timer
    │
    ├── Profile: Home Geofence Exit
    │   └── Task: Send Immediate Location Update
    │
    ├── Profile: ntfy Listener (stop_tracking)
    │   └── Task: Disable GPS polling
    │
    └── Profile: Morning Kickstart (optional)
        └── Task: Enable GPS polling at 04:00
```

---

## Webhook Endpoint

| Property | Value |
|----------|-------|
| URL | `https://<your-n8n-host>/webhook/location-update` |
| Method | `POST` |
| Content-Type | `application/json` |
| Authentication | Header auth — send the token configured in n8n's `webhook-auth` credential |

### Request Payload

```json
{
  "user_id": "valentin",
  "lat": 44.4268,
  "lng": 26.1025,
  "accuracy": 15,
  "speed": 4.2,
  "wifi_at_home": "1",
  "wifi_at_home_at": 1710648000
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `user_id` | string | Yes | Identifies the user. Must match `user_id` in the `user_config` Google Sheet. |
| `lat` | number | Yes | GPS latitude (-90 to 90). |
| `lng` | number | Yes | GPS longitude (-180 to 180). |
| `accuracy` | number | No | GPS accuracy in meters. Used to dynamically adjust arrival/departure detection radius. Default assumption: 50m. |
| `speed` | number | No | Speed in km/h. Informational — logged but not currently used in decisions. |
| `wifi_at_home` | string | No | `"1"` if currently connected to home WiFi SSID, `"0"` if not. WiFi detection is preferred over GPS for at-home status because it works indoors and is more reliable. |
| `wifi_at_home_at` | number | No | Unix epoch **seconds** when the WiFi status was last checked. The system treats WiFi data older than 20 minutes as stale and falls back to GPS. |

### Response (200 OK)

```json
{
  "status": "ok",
  "interval": 5,
  "at_home": true,
  "at_home_source": "wifi"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Always `"ok"` on success. |
| `interval` | number | Recommended minutes until the next GPS ping. Tasker should use this to set the next poll timer. |
| `at_home` | boolean | Whether the system considers the user at home. Informational for Tasker. |
| `at_home_source` | string | How at-home was determined: `"wifi"`, `"gps"`, or `"gps_after_wifi_disconnect"`. |

### Error Response (400)

```json
{
  "error": "invalid input",
  "interval": 10
}
```

Returned when required fields are missing or coordinates are out of range. Tasker should retry after the suggested interval.

---

## Polling Interval Tiers

The system dynamically adjusts polling frequency based on proximity to the next departure time. These tiers are configured in n8n via the `TRACKING_INTERVAL_TIERS` variable (format: `"60:20,30:10,15:5,5:3"`).

| Minutes until departure | Polling interval | Rationale |
|------------------------|-----------------|-----------|
| > 60 min | 20 min | Far from departure — conserve battery |
| 30–60 min | 10 min | Approaching departure window |
| 15–30 min | 5 min | Close to departure — need accurate tracking |
| < 15 min | 3 min | Critical window — tightest tracking |
| Departed (commuting) | 3 min | Arrival detection needs frequent pings |
| Away from home (not departed) | 5 min | Walk-back tracking (capped) |
| No trackable events | 15 min | Idle — minimal tracking |

**Tasker must read the `interval` field from each response and schedule the next ping accordingly.** Do not use a fixed timer.

---

## Tasker Profiles and Tasks

### Profile 1: GPS Polling (Core)

**Trigger:** Timer — dynamic interval from webhook response.

**Task: Send Location Update**

1. **Get Location v2** — source: GPS, timeout: 30s
2. **Variable Set** `%wifi_home` — value: `1` if connected to home SSID, `0` otherwise
3. **Variable Set** `%wifi_home_at` — value: `%TIMES` (current epoch seconds)
4. **HTTP Request**
   - Method: `POST`
   - URL: `https://<your-n8n-host>/webhook/location-update`
   - Headers: `Authorization: Bearer <your-webhook-token>`
   - Body:
     ```
     {"user_id":"%user_id","lat":"%LOC_LAT","lng":"%LOC_LON","accuracy":"%LOC_ACC","speed":"%LOC_SPD","wifi_at_home":"%wifi_home","wifi_at_home_at":"%wifi_home_at"}
     ```
5. **Parse JSON** — extract `interval` from response body
6. **Wait** `%interval` minutes
7. **Goto** step 1

**WiFi detection (step 2):** Use Tasker's `%WIFII` variable or a `Wifi Connected` state check against your home SSID. Example:
```
If %WIFII ~ *YourHomeSSID*
  Variable Set %wifi_home = 1
Else
  Variable Set %wifi_home = 0
End If
```

### Profile 2: Home Geofence Exit (Recommended)

**Purpose:** Eliminates polling latency when leaving home. Without this, the system may not detect you left home for up to 15-20 minutes (idle polling tier).

**Trigger:** State → Location → Geofence Exit
- Center: your home coordinates (matching `HOME_COORDS` n8n variable)
- Radius: match `HOME_RADIUS_METERS` n8n variable (typically 200m)

**Task: Send Immediate Location Update**

1. **Get Location v2** — source: GPS, timeout: 30s
2. Run the same HTTP Request as Profile 1 (send location update)
3. (No need to schedule next poll — the regular polling profile handles it)

This ensures the system detects your departure from home within seconds, triggering the walk-back adjustment and tighter polling immediately.

### Profile 3: ntfy Stop Tracking Listener

**Purpose:** When all events are done, WF4 sends a stop-tracking notification via ntfy. Tasker should stop GPS polling to save battery.

**Trigger:** Event → ntfy notification received with `extras.action = "stop_tracking"`

How to detect this depends on your ntfy client setup. Options:
- **AutoNotification** plugin: intercept ntfy app notifications matching "Stop GPS Tracking"
- **ntfy subscriber API**: Tasker HTTP GET on `https://<ntfy-host>/<your-topic>/json?poll=1` filtered for stop_tracking action
- **Direct intent:** If using ntfy Android app, it can fire a broadcast intent that Tasker catches

**Task: Stop GPS Polling**

1. **Stop** the GPS Polling task (Profile 1)
2. Optionally show a notification: "GPS tracking stopped — all events done"

### Profile 4: Morning Kickstart (Optional)

**Purpose:** Ensure GPS polling is active when the daily plan is generated.

**Trigger:** Time → 04:00

**Task:**

1. **Enable** Profile 1 (GPS Polling)
2. **Run** the Send Location Update task once immediately

This ensures WF1 (Nightly Planning at 4 AM) has a fresh location data point.

---

## n8n Variables Referenced

These n8n variables affect Tasker's behavior indirectly (via webhook responses):

| Variable | Example | Effect on Tasker |
|----------|---------|-----------------|
| `HOME_COORDS` | `44.4268,26.1025` | Determines at-home detection. Match your Tasker geofence center to this. |
| `HOME_RADIUS_METERS` | `200` | At-home GPS radius. Match your Tasker geofence radius to this. |
| `TRACKING_INTERVAL_TIERS` | `60:20,30:10,15:5,5:3` | Controls the `interval` value in responses. |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Webhook returns 400 | Missing `user_id`, `lat`, or `lng` | Check Tasker variable names — `%LOC_LAT` not `%LOC_lat` |
| `interval` always 15 | No trackable events today | Normal — check if WF1 ran and daily_plan has events |
| `at_home` wrong despite being home | WiFi data stale (>20 min) or GPS inaccurate | Ensure `wifi_at_home_at` is current epoch seconds, not milliseconds |
| GPS polling doesn't stop | ntfy stop_tracking not received | Check ntfy topic matches, check AutoNotification/intent filter |
| Battery drain | Polling too frequently | Verify Tasker reads `interval` from response; check TRACKING_INTERVAL_TIERS |
