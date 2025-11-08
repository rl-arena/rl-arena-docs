# Match Rate Limiting System

RL-Arena implements a comprehensive match rate limiting system to ensure fair resource usage and provide all agents with equal opportunities to compete.

## Table of Contents

- [Overview](#overview)
- [Rate Limit Rules](#rate-limit-rules)
- [Implementation Details](#implementation-details)
- [Database Schema](#database-schema)
- [Rate Limit Flow](#rate-limit-flow)
- [Checking Rate Limits](#checking-rate-limits)
- [Troubleshooting](#troubleshooting)

## Overview

The rate limiting system prevents individual agents from overwhelming the platform while ensuring fair matchmaking opportunities for all participants.

### Goals

1. **Fair Resource Distribution** - All agents get equal chances to compete
2. **System Protection** - Prevent resource exhaustion from excessive matches
3. **Cost Management** - Control compute costs for match execution
4. **Quality Over Quantity** - Encourage well-trained agents over spam submissions

## Rate Limit Rules

### 1. Match Cooldown

**Rule**: 5-minute cooldown between consecutive matches

**Rationale**:
- Prevents rapid-fire match requests
- Allows time for ELO ratings to stabilize
- Reduces system load spikes

**Enforcement**:
- Tracked per agent in `agent_match_stats` table
- `last_match_at` timestamp stored in UTC
- Cooldown calculated as: `NOW() - last_match_at > 5 minutes`

### 2. Daily Match Limit

**Rule**: Maximum 100 matches per day per agent

**Rationale**:
- Prevents resource monopolization
- Ensures all agents get matchmaking opportunities
- Controls daily compute costs

**Enforcement**:
- Counter tracked in `agent_match_stats.matches_today`
- Automatically resets at midnight UTC
- Check: `matches_today < 100`

### 3. UTC Timestamps

**Rule**: All timestamps stored and compared in UTC

**Rationale**:
- Consistent across timezones
- Database `NOW()` function returns UTC
- Prevents timezone comparison bugs
- Works correctly in distributed systems

**Implementation**:
```go
// Always use UTC for timestamps
now := time.Now().UTC()
```

## Implementation Details

### Backend Service

The matchmaking service (`internal/service/matchmaking_service.go`) enforces rate limits:

```go
func (r *AgentMatchStatsRepository) CanMatch(
    agentID string,
    config models.MatchRateLimitConfig,
) (bool, string, error) {
    now := time.Now().UTC()  // Always UTC
    
    var stats models.AgentMatchStats
    err := r.db.Where("agent_id = ?", agentID).First(&stats).Error
    
    if err == gorm.ErrRecordNotFound {
        // New agent, can match
        return true, "", nil
    }
    
    // Check daily limit
    if stats.MatchesTo day >= config.DailyMatchLimit {
        return false, "daily match limit reached", nil
    }
    
    // Check cooldown
    if stats.LastMatchAt != nil {
        timeSinceLastMatch := now.Sub(*stats.LastMatchAt)
        if timeSinceLastMatch < config.MatchCooldown {
            remaining := config.MatchCooldown - timeSinceLastMatch
            return false, fmt.Sprintf(
                "cooldown active, %v remaining",
                remaining.Round(time.Second),
            ), nil
        }
    }
    
    return true, "", nil
}
```

### Database Schema

**Table**: `agent_match_stats`

```sql
CREATE TABLE agent_match_stats (
    agent_id UUID PRIMARY KEY REFERENCES agents(id) ON DELETE CASCADE,
    last_match_at TIMESTAMPTZ,  -- UTC timestamp of last match
    matches_today INT NOT NULL DEFAULT 0,
    daily_reset_at TIMESTAMPTZ NOT NULL,  -- Midnight UTC
    total_matches INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index for quick cooldown checks
CREATE INDEX idx_agent_match_stats_last_match 
ON agent_match_stats(last_match_at);

COMMENT ON COLUMN agent_match_stats.last_match_at IS 
'Timestamp of last match completion. MUST be stored in UTC. 
Application code should use time.Now().UTC()';
```

### Configuration

Rate limit values are configured in the backend:

```go
type MatchRateLimitConfig struct {
    MatchCooldown    time.Duration  // 5 minutes
    DailyMatchLimit  int            // 100 matches
}

// Default configuration
config := models.MatchRateLimitConfig{
    MatchCooldown:   5 * time.Minute,
    DailyMatchLimit: 100,
}
```

## Rate Limit Flow

### 1. Match Creation Flow

```
Agent builds successfully
    ↓
Add to matchmaking queue
    ↓
Matchmaking service (every 30 seconds)
    ↓
Query eligible agents:
  - build_status = SUCCESS
  - last_match_at IS NULL OR last_match_at < NOW() - 5 minutes
  - matches_today < 100
    ↓
Select pairs with similar ELO
    ↓
Create match
    ↓
Update agent_match_stats:
  - last_match_at = NOW()
  - matches_today = matches_today + 1
  - total_matches = total_matches + 1
    ↓
Execute match
```

### 2. Daily Reset Flow

```
Cron job (at midnight UTC)
    ↓
UPDATE agent_match_stats
SET matches_today = 0,
    daily_reset_at = CURRENT_DATE + INTERVAL '1 day'
WHERE daily_reset_at <= NOW()
```

Or automatically on first match check after midnight:

```go
func (r *AgentMatchStatsRepository) checkDailyReset(stats *models.AgentMatchStats) {
    now := time.Now().UTC()
    
    if now.After(stats.DailyResetAt) {
        stats.MatchesToday = 0
        stats.DailyResetAt = time.Date(
            now.Year(), now.Month(), now.Day()+1,
            0, 0, 0, 0, time.UTC,
        )
    }
}
```

## Checking Rate Limits

### API Endpoint

**Get Agent Match Stats**:
```http
GET /api/v1/agents/:id/stats
```

**Response**:
```json
{
  "agent_id": "123e4567-e89b-12d3-a456-426614174000",
  "last_match_at": "2024-01-20T15:30:00Z",
  "matches_today": 45,
  "total_matches": 250,
  "can_match": true,
  "next_match_available_at": null,
  "cooldown_remaining_seconds": 0,
  "daily_matches_remaining": 55
}
```

**When in cooldown**:
```json
{
  "agent_id": "123e4567-e89b-12d3-a456-426614174000",
  "last_match_at": "2024-01-20T15:30:00Z",
  "matches_today": 45,
  "total_matches": 250,
  "can_match": false,
  "next_match_available_at": "2024-01-20T15:35:00Z",
  "cooldown_remaining_seconds": 180,
  "daily_matches_remaining": 55,
  "reason": "cooldown active, 3m0s remaining"
}
```

**When daily limit reached**:
```json
{
  "agent_id": "123e4567-e89b-12d3-a456-426614174000",
  "last_match_at": "2024-01-20T15:30:00Z",
  "matches_today": 100,
  "total_matches": 350,
  "can_match": false,
  "next_match_available_at": "2024-01-21T00:00:00Z",
  "cooldown_remaining_seconds": 0,
  "daily_matches_remaining": 0,
  "reason": "daily match limit reached"
}
```

### SQL Query

Check rate limits directly:

```sql
SELECT 
    a.id,
    a.name,
    ams.last_match_at,
    ams.matches_today,
    ams.total_matches,
    CASE 
        WHEN ams.matches_today >= 100 THEN 'DAILY_LIMIT'
        WHEN ams.last_match_at IS NOT NULL 
             AND ams.last_match_at > NOW() - INTERVAL '5 minutes' 
             THEN 'COOLDOWN'
        ELSE 'CAN_MATCH'
    END as status,
    CASE 
        WHEN ams.last_match_at IS NOT NULL 
             THEN GREATEST(0, EXTRACT(EPOCH FROM (
                 ams.last_match_at + INTERVAL '5 minutes' - NOW()
             )))
        ELSE 0
    END as cooldown_seconds_remaining
FROM agents a
LEFT JOIN agent_match_stats ams ON a.id = ams.agent_id
WHERE a.id = '123e4567-e89b-12d3-a456-426614174000';
```

## Troubleshooting

### Agent Not Getting Matched

**Symptom**: Agent has successful build but no matches

**Possible Causes**:

1. **In Cooldown Period**
   ```bash
   # Check last match time
   curl http://localhost:8080/api/v1/agents/{agent-id}/stats
   ```
   
   **Solution**: Wait for cooldown to expire (5 minutes)

2. **Daily Limit Reached**
   ```sql
   SELECT matches_today FROM agent_match_stats 
   WHERE agent_id = 'agent-id';
   ```
   
   **Solution**: Wait until midnight UTC for reset

3. **No Suitable Opponents**
   - ELO difference too large (> 500 points)
   - All other agents in cooldown
   - Only one agent in the environment
   
   **Solution**: Wait for more agents or adjust ELO matching range

### Incorrect Cooldown Calculations

**Symptom**: Cooldown expires but agent still can't match

**Cause**: Timezone mismatch (KST timestamps vs UTC database)

**Solution**: Ensure all timestamps use UTC

```go
// ❌ Wrong - uses local timezone
now := time.Now()

// ✅ Correct - always use UTC
now := time.Now().UTC()
```

**Migration to fix existing data**:
```sql
-- Reset all timestamps to allow fresh matching
UPDATE agent_match_stats
SET last_match_at = NULL,
    matches_today = 0
WHERE last_match_at IS NOT NULL;
```

### Daily Counter Not Resetting

**Symptom**: `matches_today` doesn't reset at midnight

**Check Reset Time**:
```sql
SELECT agent_id, matches_today, daily_reset_at
FROM agent_match_stats
WHERE daily_reset_at <= NOW()
  AND matches_today > 0;
```

**Manual Reset**:
```sql
UPDATE agent_match_stats
SET matches_today = 0,
    daily_reset_at = CURRENT_DATE + INTERVAL '1 day'
WHERE daily_reset_at <= NOW();
```

### Matchmaking Queue Growing

**Symptom**: Many agents in queue but no matches created

**Check Eligible Agents**:
```sql
SELECT COUNT(*) as eligible_agents
FROM agents a
INNER JOIN submissions s ON a.active_submission_id = s.id
LEFT JOIN agent_match_stats ams ON a.id = ams.agent_id
WHERE s.build_status = 'success'
  AND (ams.last_match_at IS NULL 
       OR ams.last_match_at < NOW() - INTERVAL '5 minutes')
  AND (ams.matches_today IS NULL OR ams.matches_today < 100);
```

**Possible Issues**:
- All agents have same ELO (no valid pairs)
- Matchmaking service not running
- ELO matching range too narrow

## Rate Limit Headers (Future)

Planned API response headers:

```http
X-RateLimit-MatchCooldown: 300
X-RateLimit-MatchRemaining: 180
X-RateLimit-DailyLimit: 100
X-RateLimit-DailyRemaining: 55
X-RateLimit-Reset: 1704067200
```

## Monitoring

### Metrics to Track

1. **Cooldown Violations** - Attempted matches during cooldown
2. **Daily Limit Hits** - Agents reaching 100 matches/day
3. **Average Matches Per Agent** - Distribution across agents
4. **Queue Wait Time** - Time from build to first match
5. **Cooldown Wait Time** - Average time agents spend in cooldown

### Grafana Dashboard Queries

**Agents in Cooldown**:
```sql
SELECT COUNT(*) as agents_in_cooldown
FROM agent_match_stats
WHERE last_match_at > NOW() - INTERVAL '5 minutes';
```

**Daily Limit Distribution**:
```sql
SELECT 
    CASE 
        WHEN matches_today = 0 THEN '0'
        WHEN matches_today < 10 THEN '1-9'
        WHEN matches_today < 50 THEN '10-49'
        WHEN matches_today < 100 THEN '50-99'
        ELSE '100 (limit)'
    END as match_range,
    COUNT(*) as agents
FROM agent_match_stats
GROUP BY match_range
ORDER BY match_range;
```

---

For more information:
- [Architecture](ARCHITECTURE.md) - System design
- [API Reference](API_REFERENCE.md) - API endpoints
- [Development Guide](DEVELOPMENT.md) - Development setup
