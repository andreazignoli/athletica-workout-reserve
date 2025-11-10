---
layout: page
title: Athletica WR API - Documentation
permalink: /api/guide
---

# Athletica WR API - Client Guide

**API Version:** 1.0.0
**Contact:** andrea@athletica.ai
**Copyright © 2025 Andrea Zignoli and Athletica.ai**

---

## Introduction

The Athletica WR API allows you to compute Workout Reserve (WR) metrics from athletic training data. 

---

## Getting Started

### Base URL

```
https://YOUR_API_ID.execute-api.YOUR_REGION.amazonaws.com/prod
```

Your API administrator will provide you with the complete base URL. 

### Authentication

All API requests require an API key in the request header:

```
x-api-key: YOUR_API_KEY_HERE
```

**Example:**
```bash
curl -X POST https://api.example.com/prod/session-ewm \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d @request.json
```

---

## Quick Start

The typical workflow involves three steps:

1. **Analyze individual sessions** → Get maximum effort values per session
2. **Establish performance baseline** → Determine athlete's ceiling across multiple sessions
3. **Monitor real-time WR** → Track reserve capacity during workouts

---

## API Endpoints

### Endpoint 1: Session Analysis

Analyze a single training session to determine maximum effort values.

**POST** `/session-ewm`

#### Request

```json
{
  "player_id": "athlete_123",
  "session_id": "session_456",
  "data_points": [
    {"timestamp": 0, "value": 100},
    {"timestamp": 1, "value": 150},
    {"timestamp": 2, "value": 200}
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `player_id` | string | Yes | Athlete identifier |
| `session_id` | string | Yes | Session identifier |
| `data_points` | array | Yes | Time series data (power in watts) |
| `data_points[].timestamp` | number | Yes | Time in seconds from session start |
| `data_points[].value` | number | Yes | Power output in watts |

#### Response

```json
{
  "player_id": "athlete_123",
  "session_id": "session_456",
  "max_ewm_per_tau": {
    "12": 284.52,
    "30": 299.68,
    "60": 305.77,
    "120": 309.93,
    "180": 312.08,
    "300": 313.71,
    "600": 315.36,
    "1200": 316.33,
    "1800": 316.72,
    "3600": 317.11
  },
  "api_version": "1.0.0"
}
```

| Field | Description |
|-------|-------------|
| `max_ewm_per_tau` | Maximum effort values across different time scales (12s to 3600s) |

**Note:** Save the `max_ewm_per_tau` values - you'll need them for Endpoint 2.

---

### Endpoint 2: Performance Baseline

Calculate an athlete's performance ceiling across multiple training sessions.

**POST** `/grand-max-ewm`

#### Request

```json
{
  "player_id": "athlete_123",
  "sessions": [
    {
      "session_id": "session_456",
      "max_ewm_per_tau": {
        "12": 284.52,
        "30": 299.68,
        ...
      }
    },
    {
      "session_id": "session_789",
      "max_ewm_per_tau": {
        "12": 450.0,
        "30": 440.0,
        ...
      }
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `player_id` | string | Yes | Athlete identifier |
| `sessions` | array | Yes | Array of sessions with their max_ewm_per_tau from Endpoint 1 |

**Tip:** Include 5-10 sessions from the past 4-12 weeks for best results.

#### Response

```json
{
  "player_id": "athlete_123",
  "grand_max_ewms": {
    "12": 450.0,
    "30": 440.0,
    "60": 450.0,
    "120": 455.0,
    "180": 460.0,
    "300": 465.0,
    "600": 470.0,
    "1200": 475.0,
    "1800": 478.0,
    "3600": 480.0
  },
  "num_sessions": 2,
  "api_version": "1.0.0"
}
```

| Field | Description |
|-------|-------------|
| `grand_max_ewms` | Athlete's performance ceiling values |
| `num_sessions` | Number of sessions analyzed |

**Note:** Save `grand_max_ewms` - you'll use this for Endpoint 3. Update monthly or after breakthrough performances.

---

### Endpoint 3: Real-Time WR Monitoring

Calculate Workout Reserve during a training session.

**POST** `/wr-predict`

#### Request

```json
{
  "player_id": "athlete_123",
  "session_id": "session_new",
  "data_points": [
    {"timestamp": 0, "value": 120},
    {"timestamp": 1, "value": 180},
    {"timestamp": 2, "value": 240}
  ],
  "grand_max_ewms": {
    "12": 450.0,
    "30": 440.0,
    "60": 450.0,
    "120": 455.0,
    "180": 460.0,
    "300": 465.0,
    "600": 470.0,
    "1200": 475.0,
    "1800": 478.0,
    "3600": 480.0
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `player_id` | string | Yes | Athlete identifier |
| `session_id` | string | Yes | Session identifier |
| `data_points` | array | Yes | Current session time series data |
| `grand_max_ewms` | object | Yes | Performance ceiling from Endpoint 2 |

#### Response

```json
{
  "player_id": "athlete_123",
  "session_id": "session_new",
  "timestamps": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
  "WR_reserve": [0.73, 0.65, 0.55, 0.44, 0.32, 0.25, 0.27, 0.30, 0.34, 0.37, 0.41],
  "limiting_tau": [3600, 3600, 3600, 1200, 1200, 120, 120, 120, 120, 120, 120],
  "api_version": "1.0.0",
  "wr_version": "1.0.0"
}
```

| Field | Description |
|-------|-------------|
| `timestamps` | Timestamps from your data (seconds) |
| `WR_reserve` | Workout Reserve at each timestamp (0-1 scale) |
| `limiting_tau` | Which physiological system is most stressed at each timestamp |

#### Interpreting Results

**WR Reserve Values:**
- **0.7-1.0**: Very high reserve - athlete working well below capacity
- **0.4-0.7**: Moderate reserve - sustainable effort level
- **0.2-0.4**: Low reserve - approaching limits
- **0.0-0.2**: Very low reserve - near maximum capacity

**Limiting Tau (indicates limiting system):**
- **12-60 seconds**: Neuromuscular/sprint power systems
- **120-600 seconds**: Anaerobic capacity systems
- **1200-3600 seconds**: Aerobic/endurance systems

---

## Complete Example

### Python

```python
import requests

# Configuration
API_URL = "https://your-api-id.execute-api.region.amazonaws.com/prod"
API_KEY = "your-api-key-here"
HEADERS = {
    "x-api-key": API_KEY,
    "Content-Type": "application/json"
}

# Step 1: Analyze historical sessions
print("Step 1: Analyzing historical sessions...")
sessions_data = []

for session_id in ["session_001", "session_002", "session_003"]:
    response = requests.post(
        f"{API_URL}/session-ewm",
        headers=HEADERS,
        json={
            "player_id": "athlete_123",
            "session_id": session_id,
            "data_points": [
                {"timestamp": 0, "value": 100},
                {"timestamp": 1, "value": 150},
                # ... your session data
            ]
        }
    )

    result = response.json()
    sessions_data.append({
        "session_id": result["session_id"],
        "max_ewm_per_tau": result["max_ewm_per_tau"]
    })
    print(f"  ✓ Analyzed {session_id}")

# Step 2: Establish performance baseline
print("\nStep 2: Establishing performance baseline...")
response = requests.post(
    f"{API_URL}/grand-max-ewm",
    headers=HEADERS,
    json={
        "player_id": "athlete_123",
        "sessions": sessions_data
    }
)

baseline = response.json()
grand_max_ewms = baseline["grand_max_ewms"]
print(f"  ✓ Baseline established from {baseline['num_sessions']} sessions")

# Step 3: Monitor WR during new session
print("\nStep 3: Monitoring WR during workout...")
response = requests.post(
    f"{API_URL}/wr-predict",
    headers=HEADERS,
    json={
        "player_id": "athlete_123",
        "session_id": "session_new",
        "data_points": [
            {"timestamp": 0, "value": 150},
            {"timestamp": 1, "value": 200},
            # ... real-time data
        ],
        "grand_max_ewms": grand_max_ewms
    }
)

wr_data = response.json()
current_wr = wr_data['WR_reserve'][-1]
current_tau = wr_data['limiting_tau'][-1]

print(f"\n  Current WR Reserve: {current_wr:.2f}")
print(f"  Limiting System: {current_tau}s")

if current_wr > 0.7:
    print("  Status: High reserve - can increase intensity")
elif current_wr > 0.4:
    print("  Status: Moderate reserve - sustainable pace")
elif current_wr > 0.2:
    print("  Status: Low reserve - approaching limits")
else:
    print("  Status: Very low reserve - near maximum")
```

### JavaScript

```javascript
const axios = require('axios');

const API_URL = 'https://your-api-id.execute-api.region.amazonaws.com/prod';
const API_KEY = 'your-api-key-here';

const headers = {
  'x-api-key': API_KEY,
  'Content-Type': 'application/json'
};

// Step 1: Analyze session
async function analyzeSession(sessionId, dataPoints) {
  const response = await axios.post(
    `${API_URL}/session-ewm`,
    {
      player_id: 'athlete_123',
      session_id: sessionId,
      data_points: dataPoints
    },
    { headers }
  );
  return response.data;
}

// Step 2: Get performance baseline
async function getBaseline(sessions) {
  const response = await axios.post(
    `${API_URL}/grand-max-ewm`,
    {
      player_id: 'athlete_123',
      sessions: sessions
    },
    { headers }
  );
  return response.data.grand_max_ewms;
}

// Step 3: Monitor WR
async function monitorWR(dataPoints, grandMaxEwms) {
  const response = await axios.post(
    `${API_URL}/wr-predict`,
    {
      player_id: 'athlete_123',
      session_id: 'session_new',
      data_points: dataPoints,
      grand_max_ewms: grandMaxEwms
    },
    { headers }
  );
  return response.data;
}

// Usage
async function main() {
  // Your implementation here
  const wrData = await monitorWR(yourDataPoints, yourGrandMaxEwms);
  console.log('Current WR:', wrData.WR_reserve[wrData.WR_reserve.length - 1]);
}
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad Request - Check your input data |
| 403 | Forbidden - Invalid API key |
| 500 | Server Error - Contact support |

### Error Response Format

```json
{
  "error": "Detailed error message",
  "api_version": "1.0.0"
}
```

### Common Errors

**403 Forbidden**
```json
{"message": "Forbidden"}
```
→ Check your API key is correct and included in the `x-api-key` header.

**400 Bad Request**
```json
{
  "error": "Validation error: Missing required fields: player_id",
  "api_version": "1.0.0"
}
```
→ Ensure all required fields are present in your request.

**400 Empty Data**
```json
{
  "error": "Internal error: data_points cannot be empty",
  "api_version": "1.0.0"
}
```
→ Your `data_points` array must contain at least one data point.

---

## Best Practices

### Data Collection

- Collect data at 1-second intervals for optimal results
- Start timestamps at 0 or the beginning of the effort
- Use consistent units (watts for power data)
- Ensure data is properly calibrated

### Baseline Establishment

- Use 5-10 sessions from the past 4-12 weeks
- Include variety: intervals, threshold work, endurance sessions
- Update baseline monthly or after breakthrough performances
- More sessions = more accurate baseline

### Real-Time Monitoring

- Can update WR every 1-10 seconds during workouts
- Monitor trends over time, not just instantaneous values
- Track changes in limiting_tau to understand which system is fatigued
- Consider sending data in small batches rather than individual points

### Performance Optimization

- Cache the `grand_max_ewms` and reuse for multiple predictions
- Implement retry logic with exponential backoff for network errors
- Validate data client-side before sending to API
- Store API key securely (use environment variables, never hardcode)

---

## Rate Limits

Default limits (may vary based on your plan):
- **10 requests per second**
- **Burst capacity: 20 requests**
- **Daily quota: 10,000 requests**

If you exceed rate limits, you'll receive:
```json
{"message": "Too Many Requests"}
```

Implement exponential backoff retry logic to handle rate limits gracefully.

---

## Support

### Contact

**Email:** andrea@athletica.ai
**Organization:** Athletica.ai

### Reporting Issues

When reporting issues, please include:
- Endpoint being called
- Request payload (sanitized - remove sensitive data)
- Response received
- Expected behavior
- Timestamp of the request

---

## Versioning

Current API version: **1.0.0**

All responses include version information:
- `api_version`: Overall API version
- `wr_version`: WR algorithm version (Endpoint 3 only)

We follow semantic versioning (MAJOR.MINOR.PATCH).

---

**Copyright © 2025 Andrea Zignoli and Athletica.ai. All rights reserved.**

This API and its algorithms are proprietary. Unauthorized use, reproduction, or distribution is prohibited.
