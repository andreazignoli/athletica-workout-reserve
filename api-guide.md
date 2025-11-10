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
- Personal training studios (50-200 athletes)
- Small team sports organizations
- Boutique fitness apps
- Research projects

#### **Professional Tier: $199/month (10,000 req/day)**

**Steady state capacity:**
- Daily budget: 10,000 calls
- Calls per athlete per day: ~1.85 calls/day
- **Capacity: ~5,000 active athletes**

**Initial backfill capacity:**
- Can onboard: ~320 athletes per day
- Time to onboard 5,000 athletes: ~16 days

**Use cases:**
- Multi-location training facilities
- College/university athletic programs
- Growing fitness platforms
- Professional sports organizations (multiple teams)

#### **Enterprise Tier: $799/month (100,000 req/day)**

**Steady state capacity:**
- Daily budget: 100,000 calls
- Calls per athlete per day: ~1.85 calls/day
- **Capacity: ~50,000 active athletes**

**Initial backfill capacity:**
- Can onboard: ~3,200 athletes per day
- Time to onboard 50,000 athletes: ~16 days

**Use cases:**
- Large fitness platforms (Peloton-scale)
- National sports federations
- Military/tactical athlete programs
- Enterprise wellness platforms

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

## Pricing

### Subscription Tiers

#### **Starter - $49/month**

Perfect for small applications, prototypes, and indie developers.

- **Request Volume:** 1,000 requests/day (30K/month)
- **Rate Limit:** 5 requests/second
- **Burst Capacity:** 10 requests
- **Support:** Email support (72-hour response time)
- **SLA:** Best effort (no uptime guarantee)
- **Documentation:** Full API documentation and examples
- **Ideal For:**
  - Prototyping and development
  - Small fitness apps (<100 active users)
  - Personal training platforms
  - Research projects

#### **Professional - $199/month**

For production applications and growing fitness platforms.

- **Request Volume:** 10,000 requests/day (300K/month)
- **Rate Limit:** 10 requests/second
- **Burst Capacity:** 20 requests
- **Support:** Priority email support (24-hour response time)
- **SLA:** 99% uptime target
- **Documentation:** Full API documentation + integration examples
- **Ideal For:**
  - Production fitness applications
  - Training platforms with 100-1,000 active users
  - Small to medium teams
  - Commercial training apps

#### **Enterprise - $799/month**

For large-scale platforms and high-volume applications.

- **Request Volume:** 100,000 requests/day (3M/month)
- **Rate Limit:** 50 requests/second
- **Burst Capacity:** 100 requests
- **Support:** Priority email + phone/video support (4-hour response time)
- **SLA:** 99.5% uptime commitment with service credits
- **Documentation:** Full documentation + dedicated onboarding session
- **Additional Features:**
  - Custom integration assistance
  - Monthly usage reports and analytics
  - Dedicated account manager
  - Algorithm version lock (optional)
- **Ideal For:**
  - Large fitness platforms (1,000+ active users)
  - Professional sports teams and organizations
  - Enterprise training systems
  - High-volume commercial applications

#### **Custom/Volume Pricing**

For organizations requiring more than 100K requests/day, contact us for custom enterprise solutions:

- **Dedicated infrastructure** - Isolated deployment for your organization
- **Custom SLA agreements** - Tailored uptime and response commitments
- **Volume discounts** - Reduced per-request pricing at scale
- **White-label options** - Subject to approval and licensing terms
- **Multi-region deployment** - Deploy in your preferred AWS region
- **Custom integrations** - Direct database access, webhooks, custom endpoints

**Contact:** andrea@athletica.ai

### Overage Pricing

If you exceed your daily quota:
- **Starter:** $0.05 per 100 additional requests
- **Professional:** $0.04 per 100 additional requests
- **Enterprise:** $0.03 per 100 additional requests

Overage charges are billed monthly. Consider upgrading to a higher tier if you consistently exceed your quota.

### Cost Calculator

Estimate your monthly costs:

| Active Users | Avg. Sessions/User/Week | Est. Daily Requests | Recommended Tier | Monthly Cost |
|--------------|-------------------------|---------------------|------------------|--------------|
| 10-50 | 2-3 | 200-500 | Starter | $49 |
| 50-200 | 3-4 | 500-2,000 | Starter | $49 |
| 200-500 | 3-5 | 2,000-7,000 | Professional | $199 |
| 500-2,000 | 4-5 | 7,000-30,000 | Professional | $199 |
| 2,000-5,000 | 4-6 | 30,000-80,000 | Enterprise | $799 |
| 5,000+ | 4-6 | 80,000+ | Custom | Contact us |

*Assumes 3 API calls per session (1 session-ewm, periodic grand-max updates, 1 wr-predict)*

### Payment Terms

- **Billing Cycle:** Monthly, billed in advance
- **Payment Methods:** Invoice (ACH, wire transfer), credit card
- **Free Trial:** 7-day trial available (Starter tier limits)
- **Cancellation:** Cancel anytime - no long-term contracts required
- **Refunds:** No refunds for partial months (see Terms of Service)

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

Current API version: **1.0.0**

All responses include version information:
- `api_version`: Overall API version
- `wr_version`: WR algorithm version (Endpoint 3 only)

We follow semantic versioning (MAJOR.MINOR.PATCH).

---

**Copyright © 2025 Andrea Zignoli and Athletica.ai. All rights reserved.**

This API and its algorithms are proprietary. Unauthorized use, reproduction, or distribution is prohibited.
