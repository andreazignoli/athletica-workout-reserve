---
layout: page
title: Athletica WR API - Documentation
permalink: /api/guide
---

# Athletica WR API - Client Guide

**API Version:** 1.1.1
**Contact:** andrea@athletica.ai
**Copyright © 2025 Andrea Zignoli and Athletica.ai**

---

## Introduction

The Athletica WR API allows you to compute Workout Reserve (WR) metrics from athletic training data.

### Data Requirements

**Sampling Rate:** The algorithm works best with **1 Hz data** (one data point per second). While the implementation is agnostic to sampling rates, we strongly recommend 1-second intervals for optimal accuracy.

**Timestamps:** Timestamps should represent **seconds from the beginning of the activity**, starting at 0. For example, a 10-minute session would have timestamps from 0 to 600.

**Handling Pauses:** The API automatically handles timestamp gaps by filling missing seconds with value=0. You can send data with or without gaps - the server will ensure continuous timestamp sequences for accurate EWM calculations. However, for optimal data quality, we recommend including pauses with value=0 in your client data when possible.

**Sport-Specific Metrics:**
- **Cycling & Rowing:** Use **mechanical power output** (watts)
- **Running:** Use **speed** (m/s or km/h - be consistent)
- **Football & Team Sports:** Use **metabolic power** (watts) for best results

The algorithm is flexible and can work with any continuous effort metric, but the above recommendations are based on extensive validation in each sport.

---

## Getting Started

### Base URL

```
https://3jqargtyza.execute-api.eu-central-1.amazonaws.com/prod
```

This is the production API endpoint hosted in the EU (Frankfurt region). 

### Authentication

All API requests require an API key in the request header:

```
x-api-key: YOUR_API_KEY_HERE
```

**Example:**
```bash
curl -X POST https://3jqargtyza.execute-api.eu-central-1.amazonaws.com/prod/session-ewm \
  -H "x-api-key: YOUR_API_KEY_HERE" \
  -H "Content-Type: application/json" \
  -d @request.json
```

Replace `YOUR_API_KEY_HERE` with the API key provided to you.

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
    "6": 270.15,
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
  "api_version": "1.1.0"
}
```

| Field | Description |
|-------|-------------|
| `max_ewm_per_tau` | Maximum effort values across different time scales (6s to 3600s) |

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
        "6": 270.15,
        "12": 284.52,
        "30": 299.68,
        ...
      }
    },
    {
      "session_id": "session_789",
      "max_ewm_per_tau": {
        "6": 435.0,
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
    "6": 435.0,
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
  "api_version": "1.1.0"
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
    "6": 435.0,
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
  "contributions": {
    "AN_ALA_perc": 0.0,
    "AN_LA_perc": 0.0,
    "AE_MAX_perc": 45.8,
    "AE_THRES_perc": 27.3,
    "AE_perc": 26.9
  },
  "api_version": "1.1.0",
  "wr_version": "3.0.0"
}
```

| Field | Description |
|-------|-------------|
| `timestamps` | Timestamps from your data (seconds) |
| `WR_reserve` | Workout Reserve at each timestamp (0-1 scale) |
| `limiting_tau` | Which physiological system is most stressed at each timestamp |
| `contributions` | Energy system contributions as percentages (sum to 100%) |
| `contributions.AN_ALA_perc` | Anaerobic Alactic % (tau ≤ 12s, neuromuscular/power) |
| `contributions.AN_LA_perc` | Anaerobic Lactic % (tau = 30-60s, anaerobic capacity) |
| `contributions.AE_MAX_perc` | Aerobic Max % (tau = 120-300s, VO2max) |
| `contributions.AE_THRES_perc` | Aerobic Threshold % (tau = 600-1200s, lactate threshold) |
| `contributions.AE_perc` | Aerobic % (tau = 1800-3600s, endurance) |

#### Interpreting Results

**WR Reserve Values:**
- **0.7-1.0**: Very high reserve - athlete working well below capacity
- **0.4-0.7**: Moderate reserve - sustainable effort level
- **0.2-0.4**: Low reserve - approaching limits
- **0.0-0.2**: Very low reserve - near maximum capacity

**Limiting Tau (indicates limiting system):**
- **6-12 seconds**: Very short sprints / explosive power (team sports)
- **12-60 seconds**: Neuromuscular/sprint power systems
- **120-600 seconds**: Anaerobic capacity systems
- **1200-3600 seconds**: Aerobic/endurance systems

---

## Implementation Patterns

### Understanding the Workflow

To compute WR for an athlete during a new session, you need:

1. **Build athlete profile** (Endpoint 1): Process 3-6 weeks of historical sessions
2. **Establish baseline** (Endpoint 2): Calculate athlete's performance ceiling
3. **Monitor real-time WR** (Endpoint 3): Track reserve during new sessions

**Key insight:** After processing historical data once, you can store the `grand_max_ewms` profile and reuse it for days/weeks, only updating periodically when athletes achieve breakthrough performances.

### Pattern 1: Initial Setup (One-Time)

When onboarding a new athlete with 6 weeks of training history (30 sessions):

```
Day 1: Initial backfill
├─ Endpoint 1: 30 calls (one per historical session)
├─ Endpoint 2: 1 call (compute grand_max_ewms from all 30 sessions)
└─ Store grand_max_ewms in your database

Total: 31 API calls (one-time cost)
```

### Pattern 2: Daily Operations (Efficient)

**Option A: Store profiles locally (recommended)**

```
Each training day:
├─ Endpoint 3: 1 call (monitor WR using stored grand_max_ewms)
└─ Store new session data locally

Weekly profile update:
├─ Endpoint 1: 7 calls (process last week's sessions)
├─ Endpoint 2: 1 call (update grand_max_ewms with recent sessions)
└─ Update stored grand_max_ewms

Daily cost: 1 call per athlete session
Weekly overhead: 8 calls per athlete
```

**Option B: Always recompute (inefficient, not recommended)**

```
Each training day:
├─ Endpoint 1: 30 calls (reprocess entire history)
├─ Endpoint 2: 1 call (recompute baseline)
└─ Endpoint 3: 1 call (monitor WR)

Daily cost: 32 calls per athlete session (32x more expensive!)
```

### Pattern 3: Profile Update Strategy

**Conservative (monthly updates):**
- Update `grand_max_ewms` once per month
- Cost: ~8 API calls per athlete per month
- Best for: Stable athletes, off-season training

**Standard (weekly updates):**
- Update `grand_max_ewms` once per week
- Cost: ~8 API calls per athlete per week
- Best for: Regular training, most use cases

**Aggressive (daily updates):**
- Update `grand_max_ewms` after every session
- Cost: 2 additional calls per session (Endpoint 1 + 2)
- Best for: Peak training periods, rapid adaptation tracking

### Capacity Planning by Tier

Assumptions:
- Athletes train 5 days/week average (260 sessions/year)
- Weekly profile updates (8 calls/athlete/week)
- 1 WR monitoring call per session
- Initial backfill: 31 calls per athlete (one-time)

#### **Starter Tier: $49/month (1,000 req/day)**

**Steady state capacity:**
- Daily budget: 1,000 calls
- Calls per athlete per day: 5/7 sessions × 1 call = 0.71 calls/day + (8/7 weekly overhead) = ~1.85 calls/day
- **Capacity: ~500 active athletes**

**Initial backfill capacity:**
- Backfill cost: 31 calls per athlete
- Can onboard: ~32 athletes per day
- Time to onboard 500 athletes: ~16 days

**Use cases:**
- **Endurance sports coaching**: Cycling, running, triathlon coaches (lower frequency, 3-5 sessions/week)
- **Personal training studios**: 50-200 athletes with varied schedules
- **Small team sports**: Local clubs, youth teams
- **Boutique fitness apps**: Niche markets, specialized training
- **Research projects**: Academic studies, training methodology validation
- **Beta testing**: Pilot programs before full deployment

#### **Professional Tier: $199/month (10,000 req/day)**

**Steady state capacity:**
- Daily budget: 10,000 calls
- Calls per athlete per day: ~1.85 calls/day
- **Capacity: ~5,000 active athletes**

**Initial backfill capacity:**
- Can onboard: ~320 athletes per day
- Time to onboard 5,000 athletes: ~16 days

**Use cases:**
- **Multi-sport training platforms**: Combining endurance (cycling, running) with team sports
- **College/university athletic programs**: Multiple teams across various sports
- **Professional sports organizations**: Several teams, multiple squads
- **Growing fitness platforms**: 1,000-5,000 users, production environments
- **Corporate wellness programs**: Enterprise employee fitness tracking
- **Virtual racing platforms**: Online cycling/running competitions with WR leaderboards
- **Fitness wearable integrations**: Apps integrating with Garmin, Wahoo, Zwift

#### **Enterprise Tier: $799/month (100,000 req/day)**

**Steady state capacity:**
- Daily budget: 100,000 calls
- Calls per athlete per day: ~1.85 calls/day
- **Capacity: ~50,000 active athletes**

**Initial backfill capacity:**
- Can onboard: ~3,200 athletes per day
- Time to onboard 50,000 athletes: ~16 days

**Use cases:**
- **Large fitness platforms**: Peloton-scale, 10,000+ active users
- **National sports federations**: Olympic committees, national teams across all sports
- **Military/tactical athlete programs**: Special forces, military fitness assessment systems
- **Enterprise wellness platforms**: Fortune 500 employee health programs
- **Live sports broadcasting**: Real-time WR display during competitions, TV overlays
- **Gaming/esports**: Real-time WR integration in fitness gaming (Zwift, Rouvy, virtual races)
- **Streaming platforms**: Twitch/YouTube fitness streamers with live WR metrics
- **Multi-tenant SaaS**: White-label fitness solutions serving multiple sub-clients
- **High-frequency trading sports**: Daily/multiple daily competitions with instant WR feedback

### Real-World Example Calculation

**Scenario:** Training platform with 1,000 athletes, 70% train 5x/week, 30% train 3x/week

**Daily API calls:**
```
Active sessions per day:
- 700 athletes × 5 sessions/week ÷ 7 days = 500 sessions/day
- 300 athletes × 3 sessions/week ÷ 7 days = 129 sessions/day
- Total: 629 sessions/day

WR monitoring (Endpoint 3):
- 629 calls/day

Weekly profile updates (Endpoints 1+2):
- 1,000 athletes × 8 calls/week ÷ 7 days = 1,143 calls/day

Total daily calls: 629 + 1,143 = 1,772 calls/day
```

**Recommended tier:** Professional ($199/month) - uses only 18% of daily quota, room for growth

### Cost Optimization Best Practices

#### 1. **Store Profiles Locally** ✅

```python
# Store in your database
athlete_profiles = {
    "athlete_123": {
        "grand_max_ewms": {...},
        "last_updated": "2025-11-01",
        "sessions_since_update": 5
    }
}

# Reuse profile for multiple sessions
def monitor_session(athlete_id, session_data):
    profile = athlete_profiles[athlete_id]

    # Use stored profile - only 1 API call!
    response = requests.post(
        f"{API_URL}/wr-predict",
        json={
            "player_id": athlete_id,
            "session_id": session_id,
            "data_points": session_data,
            "grand_max_ewms": profile["grand_max_ewms"]  # Stored profile
        }
    )
    return response.json()
```

#### 2. **Batch Profile Updates** ✅

```python
# Update profiles weekly, not daily
def update_athlete_profile(athlete_id, last_7_sessions):
    # Process recent sessions
    sessions_data = []
    for session in last_7_sessions:
        result = call_endpoint_1(session)  # 7 calls
        sessions_data.append(result)

    # Update baseline with recent + stored old baseline
    updated_profile = call_endpoint_2(sessions_data)  # 1 call

    # Store updated profile
    athlete_profiles[athlete_id] = updated_profile

    # Total: 8 calls per athlete per week
```

#### 3. **Smart Update Triggers** ✅

Only update profiles when needed:

```python
def should_update_profile(athlete_id):
    profile = athlete_profiles[athlete_id]

    # Update if:
    # - More than 7 days since last update
    days_since = (datetime.now() - profile["last_updated"]).days
    if days_since >= 7:
        return True

    # - Accumulated 5+ sessions since last update
    if profile["sessions_since_update"] >= 5:
        return True

    # - Start of new training phase (macro cycle change)
    if training_phase_changed(athlete_id):
        return True

    # - WR reserve consistently low (athlete may be adapting)
    # Track recent WR values and update if athlete is pushing limits frequently
    recent_wr_values = get_recent_wr_values(athlete_id, last_n_sessions=5)
    if recent_wr_values and all(wr < 0.3 for wr in recent_wr_values):
        return True  # Athlete pushing hard, likely adapting - update baseline

    return False

# After each session, increment counter
def after_session(athlete_id, wr_reserve_values):
    profile = athlete_profiles[athlete_id]
    profile["sessions_since_update"] += 1

    # Store WR values for trend analysis
    store_wr_history(athlete_id, wr_reserve_values)

    # Check if profile update needed
    if should_update_profile(athlete_id):
        update_athlete_profile(athlete_id)
        profile["sessions_since_update"] = 0
        profile["last_updated"] = datetime.now()
```

#### 4. **Efficient Backfilling** ✅

When onboarding athletes, spread backfills over time:

```python
# Onboard 50 athletes per day (1,550 calls)
# Instead of 500 athletes in one day (15,500 calls - would exceed quota)

def onboard_athletes_gradually(athlete_list, athletes_per_day=50):
    for i in range(0, len(athlete_list), athletes_per_day):
        batch = athlete_list[i:i+athletes_per_day]

        for athlete in batch:
            # Backfill historical data
            backfill_athlete_history(athlete)  # 31 calls per athlete

        # Wait until next day for next batch
        schedule_next_batch(date=tomorrow)
```

### Migration Strategy for Existing Platforms

If you're migrating from another system:

**Phase 1: Pilot (Week 1-2)**
- Onboard 10-50 athletes
- Test integration
- Validate data quality
- Use Starter tier ($49/month)

**Phase 2: Gradual Rollout (Week 3-8)**
- Onboard 50-100 athletes per day
- Monitor API usage
- Optimize profile update frequency
- Upgrade to Professional tier when needed

**Phase 3: Full Deployment (Month 3+)**
- Complete athlete base migration
- Establish automated profile update workflows
- Monitor costs and optimize
- Scale to Enterprise tier if needed

### API Call Cost Estimation Tool

Use this formula to estimate your needs:

```
Daily API calls =
    (Active sessions per day × 1) +                    # WR monitoring
    (Total athletes × 8 ÷ 7) +                         # Weekly profile updates
    (New athletes per day × 31)                        # Initial backfills

Monthly API calls = Daily API calls × 30

Example:
- 2,000 athletes, 1,000 daily sessions, 20 new athletes/day
- Daily: (1,000 × 1) + (2,000 × 8 ÷ 7) + (20 × 31) = 3,907 calls/day
- Monthly: 3,907 × 30 = 117,210 calls/month
- Recommended tier: Professional (10K/day quota)
```

---

## Complete Example

### Python

```python
import requests

# Configuration
API_URL = "https://3jqargtyza.execute-api.eu-central-1.amazonaws.com/prod"
API_KEY = "YOUR_API_KEY_HERE"
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

const API_URL = 'https://3jqargtyza.execute-api.eu-central-1.amazonaws.com/prod';
const API_KEY = 'YOUR_API_KEY_HERE';

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
  "api_version": "1.1.0"
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
  "api_version": "1.1.0"
}
```
→ Ensure all required fields are present in your request.

**400 Empty Data**
```json
{
  "error": "Internal error: data_points cannot be empty",
  "api_version": "1.1.0"
}
```
→ Your `data_points` array must contain at least one data point.

---

## Best Practices

### Data Collection

- **Collect data at 1 Hz (1-second intervals)** for optimal results
- **Start timestamps at 0** at the beginning of the activity
- **Include pauses with value=0** - do not skip timestamps during rest periods
- **Use sport-specific metrics:**
  - Cycling/Rowing: Mechanical power (watts)
  - Running: Speed (m/s or km/h, be consistent)
  - Football/Team Sports: Metabolic power (watts)
- **Ensure data is properly calibrated** (power meters, GPS devices, etc.)

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

## Pricing

**Choose the model that fits your business:**
- **Direct API Access** → Pay per request volume (best for developers and individual platforms)
- **Platform Partnership** → Pay per team/athlete (best for sports software providers and resellers)
- **Strategic Partnership** → Custom arrangements (best for broadcasting, gaming, major platforms)

---

### Option 1: Direct API Access
*For developers, coaching businesses, and platforms building WR-powered applications*

**Best for:** Individual developers, endurance coaches, research projects, fitness apps

#### **Developer — $49/month**

- **Request Volume:** 1,000 requests/day (≈30K/month)
- **Rate Limit:** 5 requests/second, 10 burst
- **Support:** Email (72h response)
- **SLA:** Best effort
- **Ideal For:**
  - Prototyping and testing
  - Research projects
  - Small apps (<100 active users)
  - Personal training platforms
  - Endurance coaching (cycling, running, triathlon)

#### **Professional — $199/month**

- **Request Volume:** 10,000 requests/day (≈300K/month)
- **Rate Limit:** 10 requests/second, 20 burst
- **Support:** Priority email (24h response)
- **SLA:** 99% uptime
- **Ideal For:**
  - Commercial training platforms
  - Sports science applications
  - Apps with 100–1,000 active users
  - Multi-sport coaching platforms
  - University athletic programs

#### **Enterprise — $799/month**

- **Request Volume:** 100,000 requests/day (≈3M/month)
- **Rate Limit:** 50 requests/second, 100 burst
- **Support:** Email + phone/video (4h response)
- **SLA:** 99.5% uptime with service credits
- **Additional Features:**
  - Custom integration support
  - Monthly usage analytics
  - Dedicated account manager
  - Algorithm version pinning
- **Ideal For:**
  - Large fitness platforms (10K+ users)
  - National sports federations
  - Enterprise wellness programs
  - High-volume commercial applications

**API Capacity Reference:**
- **Developer**: ~500 active athletes (training 5x/week)
- **Professional**: ~5,000 active athletes
- **Enterprise**: ~50,000 active athletes

---

### Option 2: Platform Partnership
*For sports software providers who want to embed and resell WR features*

**Best for:** Team sports platforms, club management software, multi-tenant SaaS providers

**Commercial Rights Included:**
- ✅ Right to embed WR in your platform
- ✅ Right to resell WR features to your customers
- ✅ Co-branded integration ("Powered by Athletica WR")
- ✅ Unlimited API calls per team (within fair use)
- ✅ Priority support and technical partnership

#### **Growth Stage — Team-Based Pricing**

Perfect for platforms still building customer base. Pay only for active teams using WR features.

**Pricing Structure:**

| Active Teams | Price per Team per Month | Monthly Total | Annual Prepay Discount |
|--------------|--------------------------|---------------|------------------------|
| 1-25 teams | $40/team | $40-1,000 | 10% off ($36/team) |
| 26-100 teams | $30/team | $780-3,000 | 15% off ($25.50/team) |
| 101-300 teams | $20/team | $2,020-6,000 | 20% off ($16/team) |
| 301+ teams | Custom pricing | Contact us | Contact us |

**What counts as a "team":**
- Team sports: One squad/roster (e.g., one football club, one basketball team)
- Endurance: Group of 10-15 athletes under same coach/program
- Multi-sport clubs: Each sport counts as separate team

**Includes:**
- Unlimited API calls per team (fair use: ~100-200 calls/team/month)
- 20 requests/second, 50 burst
- Priority email support (24h response)
- 99% uptime SLA
- Co-branding rights
- Monthly usage reports per team

**Example Scenarios:**
- Platform with 10 football clubs → **$400/month** (or $360/month annual prepay)
- Platform with 50 clubs → **$1,500/month** (or $1,275/month annual prepay)
- Platform with 200 clubs → **$4,000/month** (or $3,200/month annual prepay)

#### **Established Platform — Volume Licensing**

For platforms with existing customer base and proven business model.

**Pricing Options:**

**Option A: Annual Platform License**
- **$15,000-30,000/year** (based on expected volume)
- Unlimited API calls (fair use policy)
- Up to 500 active teams
- 50 req/s, 100 burst
- Phone/video support (4h response)
- 99.5% uptime SLA
- Dedicated account manager
- Custom integration assistance

**Option B: Per-Athlete Pricing (Multi-Sport Platforms)**
- **$2-4 per athlete per month** (volume tiered)
- 1-1,000 athletes: $4/athlete/month
- 1,001-5,000 athletes: $3/athlete/month
- 5,001-20,000 athletes: $2/athlete/month
- 20,001+ athletes: Custom pricing

**Option C: Revenue Share Model**
- **Base fee: $5,000-10,000/month**
- **Plus: 5-8% revenue share** on WR-attributed features
- Best for: Platforms with premium tiers powered by WR
- Unlimited API calls
- Full partnership support
- Aligned growth incentives

**Best for:**
- Established club management platforms (100+ teams)
- Multi-sport platforms (1,000+ athletes)
- White-label SaaS providers
- League management systems

---

### Option 3: Strategic Partnership
*For unique applications with massive scale or high-value distribution*

**Best for:** Broadcasting, gaming, streaming platforms, major sports leagues, exclusive partnerships

**Flexible Deal Structures:**

#### **Broadcasting & Media**

- **Per-Event Licensing:** $5,000-20,000 per event (live WR overlays)
- **Season Licensing:** $50,000-200,000 per season (exclusive league rights)
- **Best for:** TV networks, streaming platforms, race broadcasts

#### **Gaming & Virtual Platforms**

- **Annual License:** $50,000-150,000/year (unlimited integration)
- **Revenue Share:** Base fee + 3-5% of premium tier revenue
- **Best for:** Zwift, Rouvy, virtual racing platforms, fitness gaming

#### **Major Platforms & Leagues**

- **Enterprise License:** $100,000-500,000/year
- **Equity Partnership:** Strategic investment + exclusive integration
- **Custom Terms:** White-label, private hosting, co-development
- **Best for:** Premier League analytics, national federations, Peloton-scale platforms

**All Strategic Partnerships Include:**
- Unlimited API calls (custom rate limits)
- 200+ req/s, custom burst
- Direct Slack/phone support (2h response)
- 99.9% uptime SLA with credits
- Dedicated technical team
- Custom feature development
- Co-marketing opportunities
- Exclusive rights (case-by-case)

**Contact:** andrea@athletica.ai for Strategic Partnership discussions

---

### Pricing Decision Guide

**Use this flowchart to choose your option:**

```
Are you building your own app/platform to use WR directly?
├─ YES → Option 1: Direct API Access
│  └─ Choose tier based on athlete volume (see capacity reference)
│
└─ NO → Are you a software provider who will resell WR features?
   ├─ YES → Option 2: Platform Partnership
   │  ├─ Still building customer base? → Growth Stage (team-based pricing)
   │  └─ Established with 100+ teams? → Established Platform (annual license or per-athlete)
   │
   └─ NO → Do you have unique distribution (broadcasting, gaming, major scale)?
      └─ YES → Option 3: Strategic Partnership (custom deal)
```

**Key Differences:**

| Feature | Direct API | Platform Partnership | Strategic Partnership |
|---------|-----------|---------------------|----------------------|
| **Resale Rights** | ❌ No | ✅ Yes | ✅ Yes |
| **Pricing Model** | Per API call | Per team or per athlete | Custom |
| **Best For** | End-user apps | Software resellers | Unique opportunities |
| **Entry Point** | $49/month | $400/month (10 teams) | $50K+/year |
| **Co-branding** | Optional | Required | Negotiable |
| **Support** | Email | Email + technical partnership | Dedicated team |

---

### Cost Comparison Examples

**Scenario 1: Platform with 50 football clubs (25 players/club = 1,250 athletes)**

- **Option 1 (Direct API):** $799/month (Enterprise tier, enough capacity) — ❌ **NO RESALE RIGHTS**
- **Option 2 (Platform Partnership - Growth):** $1,500/month (50 teams × $30/team) — ✅ **INCLUDES RESALE RIGHTS**
- **Option 2 (Platform Partnership - Per-Athlete):** $3,750/month (1,250 athletes × $3/athlete) — ✅ **INCLUDES RESALE RIGHTS**

**Decision:**
- ❌ **You CANNOT use Direct API if you're reselling WR features** (violates Terms of Service)
- ✅ **You MUST use Platform Partnership** if you're embedding/reselling WR to customers
- The $700/month difference ($1,500 vs $799) pays for resale rights + technical partnership

**Why Platform Partnership costs more:**
- Legal right to resell WR features to your customers
- Co-branding and partnership status
- Technical partnership support
- Protection for both parties as you scale

**Scenario 2: Large multi-sport platform with 10,000 athletes**

- **Option 1 (Direct API):** $799/month (Enterprise tier, enough capacity)
- **Option 2 (Platform Partnership - Per-Athlete):** $20,000/month (10K athletes × $2/athlete)
- **Option 2 (Platform Partnership - Annual License):** $30,000/year = $2,500/month

**Recommendation:** If you're **NOT reselling** → Choose Direct API ($799/month)
If you're **reselling to premium tier** → Choose Revenue Share ($5K base + 5-8% revenue)

**Scenario 3: Broadcasting company wanting live WR during Tour de France**

- **Option 1/2:** Not applicable (broadcasting rights required)
- **Option 3 (Strategic Partnership):** $50,000-100,000 for exclusive race coverage

**Recommendation:** Strategic Partnership with per-event or season licensing

---

### Overage Pricing (Direct API Only)

If you exceed your daily quota on Direct API plans:

- **Developer:** $0.05 per 100 additional requests
- **Professional:** $0.04 per 100 additional requests
- **Enterprise:** $0.03 per 100 additional requests

**Note:** Platform Partnership plans include unlimited API calls per team (fair use policy), so overages don't apply.

### Payment Terms

- **Billing Cycle:** Monthly, billed in advance
- **Payment Methods:** Invoice (ACH, wire transfer) or credit card
- **Free Trial:** 7-day free trial (Starter-tier limits)
- **Cancellation:** Cancel anytime — no long-term commitments
- **Refunds:** No refunds for partial months (see Terms of Service)
- **Currency:** All prices in USD
- **Support Email:** andrea@athletica.ai

### Licensing and Reseller Terms

**Intellectual Property Ownership**

All Workout Reserve (WR) algorithms, computational methods, API implementations, and related intellectual property remain the exclusive property of **Athletica Inc. and Andrea Zignoli**. No license, subscription, or partnership grants ownership or derivative rights to the underlying algorithms.

**Permitted Use by API Customers**

- **Starter, Professional, and Enterprise tiers** grant you the right to:
  - Integrate WR metrics into your products and platforms
  - Display WR data to your end users
  - Use WR calculations as part of your commercial offerings
  - Set your own pricing for products that incorporate WR metrics

- **You may NOT:**
  - Reverse engineer, decompile, or attempt to extract the WR algorithms
  - Resell or redistribute direct API access to third parties
  - Remove or obscure required attribution
  - Create competing WR calculation services

**Platform Partner and Reseller Rights**

- **Platform Partner License holders** may:
  - Embed WR functionality within their platforms
  - Resell WR-powered features to their customers
  - Co-brand integrations as "Powered by Athletica WR"
  - Integrate WR into white-label or multi-tenant solutions

- **Platform Partners require:**
  - A signed Platform Partner Agreement
  - Proper attribution in product documentation (not necessarily in UI)
  - Compliance with data privacy and usage terms
  - Annual renewal and usage reporting

**Attribution Requirements**

All tiers (except Custom/White-Label agreements) must include attribution to **"Athletica WR"** or **"Powered by Athletica WR"** in:

- Product documentation, Terms of Service, or About page
- API integration guides (for developer-facing products)
- Marketing materials mentioning WR functionality (optional but encouraged)

Attribution **does not** need to be displayed in the primary user interface unless specifically requested by the partner.

**Data Privacy and Usage Rights**

- **Your data remains yours:** You retain ownership of all training data submitted to the API.
- **We do not sell your data:** Athletica will never sell client data to third parties.
- **Aggregated research use:** Athletica may use anonymized, aggregated data for research and algorithm improvements (as detailed in the Privacy Policy).
- **GDPR and compliance:** You are responsible for ensuring compliance with GDPR, CCPA, and other privacy regulations when collecting and submitting athlete data.

**Resale and Commercial Use**

- **Standard tiers (Starter, Professional, Enterprise):** You may integrate WR into your products and charge your customers. You may not resell direct API access.
- **Platform Partner License:** You may resell WR-powered features and embed them in multi-tenant platforms. Requires signed agreement.
- **Custom/White-Label:** Full resale and white-label rights available under separate terms.

**Termination and IP Protection**

Upon termination of your subscription:

- API access is immediately revoked
- You must cease using WR metrics in new calculations
- You may retain historical WR data previously computed for your users
- All algorithm access rights expire
- Attribution requirements remain for any legacy integrations still in use

**Enforcement**

Violation of these terms, including reverse engineering attempts or unauthorized resale, will result in:

- Immediate account suspension
- Legal action to protect Athletica's intellectual property
- Liability for damages as defined in the Terms of Service

**Partnership Inquiries**

For partnership opportunities beyond standard API access, including:

- Co-development and joint ventures
- White-label or private hosting
- Academic or institutional licensing
- Strategic integrations with major platforms

**Contact:** andrea@athletica.ai

### Getting Started

1. **Contact us:** Email andrea@athletica.ai with:
   - Your organization name
   - Estimated request volume
   - Preferred tier
   - Brief description of your use case

2. **Receive credentials:** You'll receive:
   - API Gateway URL
   - Unique API key
   - Complete documentation
   - Setup assistance

3. **7-day trial:** Test the API with Starter tier limits for free

4. **Go live:** Upgrade to your selected tier when ready for production

### Fair Use Policy

All tiers include reasonable use expectations:
- API keys are non-transferable and for single organization use only
- No reselling of direct API access (see Terms of Service for integration rules)
- No automated scraping or algorithm reverse engineering attempts
- Compliance with Terms of Service and Privacy Policy

---

## Rate Limits

Rate limits vary based on your subscription tier (see Pricing above).

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

Current API version: **1.1.1**

All responses include version information:
- `api_version`: Overall API version
- `wr_version`: WR algorithm version (Endpoint 3 only)

We follow semantic versioning (MAJOR.MINOR.PATCH).

---

**Copyright © 2025 Andrea Zignoli and Athletica.ai. All rights reserved.**

This API and its algorithms are proprietary. Unauthorized use, reproduction, or distribution is prohibited.
