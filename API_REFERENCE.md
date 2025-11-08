# API Reference

Complete reference for the RL-Arena REST API and WebSocket interface.

## Table of Contents

- [Base URL](#base-url)
- [Authentication](#authentication)
- [Error Handling](#error-handling)
- [Rate Limiting](#rate-limiting)
- [Authentication Endpoints](#authentication-endpoints)
- [Agent Endpoints](#agent-endpoints)
- [Submission Endpoints](#submission-endpoints)
- [Match Endpoints](#match-endpoints)
- [Leaderboard Endpoints](#leaderboard-endpoints)
- [User Endpoints](#user-endpoints)
- [WebSocket API](#websocket-api)

## Base URL

```
Development: http://localhost:8080/api/v1
Production: https://api.rl-arena.io/api/v1
```

## Authentication

Most endpoints require JWT authentication. Include the token in the Authorization header:

```http
Authorization: Bearer <your-jwt-token>
```

### Getting a Token

1. Register or login to get a JWT token
2. Token expires after 24 hours
3. Include token in all authenticated requests

## Error Handling

All errors follow this format:

```json
{
  "error": "Error message",
  "code": "ERROR_CODE",
  "details": {
    "field": "Additional error context"
  }
}
```

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request - Invalid input |
| 401 | Unauthorized - Missing or invalid token |
| 403 | Forbidden - Insufficient permissions |
| 404 | Not Found - Resource doesn't exist |
| 409 | Conflict - Resource already exists |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error |

## Rate Limiting

API endpoints are rate-limited to prevent abuse:

| Endpoint Type | Limit | Window |
|--------------|-------|--------|
| Authentication | 5 requests | 1 minute |
| Code Submission | 5 requests | 1 minute |
| Match Creation | 10 requests | 1 minute |
| Other endpoints | 60 requests | 1 minute |

Rate limit headers:
```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1699564800
```

---

## Authentication Endpoints

### Register User

Create a new user account.

```http
POST /auth/register
```

**Request Body**:
```json
{
  "username": "myusername",
  "email": "user@example.com",
  "password": "securepassword123"
}
```

**Response** (201 Created):
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "myusername",
    "email": "user@example.com",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

**Errors**:
- `409` - Username or email already exists
- `400` - Invalid input (weak password, invalid email)

---

### Login

Authenticate and get JWT token.

```http
POST /auth/login
```

**Request Body**:
```json
{
  "username": "myusername",
  "password": "securepassword123"
}
```

**Response** (200 OK):
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "myusername",
    "email": "user@example.com"
  }
}
```

**Errors**:
- `401` - Invalid credentials

---

## Agent Endpoints

### List All Agents

Get a list of all agents.

```http
GET /agents
```

**Query Parameters**:
- `environment` (optional) - Filter by environment ID
- `limit` (optional) - Number of results (default: 50)
- `offset` (optional) - Pagination offset (default: 0)

**Response** (200 OK):
```json
{
  "agents": [
    {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "user_id": "550e8400-e29b-41d4-a716-446655440000",
      "username": "myusername",
      "name": "SmartPongAgent",
      "environment_id": "pong",
      "elo_rating": 1450,
      "total_matches": 25,
      "wins": 15,
      "losses": 10,
      "active_submission_id": "789e0123-e89b-12d3-a456-426614174111",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

---

### Get Agent Details

Get detailed information about a specific agent.

```http
GET /agents/:id
```

**Response** (200 OK):
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "myusername",
  "name": "SmartPongAgent",
  "environment_id": "pong",
  "elo_rating": 1450,
  "total_matches": 25,
  "wins": 15,
  "losses": 10,
  "draws": 0,
  "active_submission_id": "789e0123-e89b-12d3-a456-426614174111",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-20T15:45:00Z"
}
```

**Errors**:
- `404` - Agent not found

---

### Get Agent Statistics

Get detailed match statistics for an agent.

```http
GET /agents/:id/stats
```

**Response** (200 OK):
```json
{
  "agent_id": "123e4567-e89b-12d3-a456-426614174000",
  "elo_rating": 1450,
  "total_matches": 25,
  "wins": 15,
  "losses": 10,
  "draws": 0,
  "win_rate": 60.0,
  "last_match_at": "2024-01-20T15:30:00Z",
  "matches_today": 12,
  "can_match": true,
  "next_match_available_at": null,
  "opponent_stats": [
    {
      "opponent_id": "456e7890-e89b-12d3-a456-426614174222",
      "opponent_name": "RivalAgent",
      "matches_played": 5,
      "wins": 3,
      "losses": 2
    }
  ]
}
```

---

### Create Agent

Create a new agent (requires authentication).

```http
POST /agents
```

**Headers**:
```http
Authorization: Bearer <token>
```

**Request Body**:
```json
{
  "name": "MyNewAgent",
  "environment_id": "pong",
  "description": "A reinforcement learning agent for Pong"
}
```

**Response** (201 Created):
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "MyNewAgent",
  "environment_id": "pong",
  "elo_rating": 1200,
  "total_matches": 0,
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Errors**:
- `400` - Invalid input
- `401` - Unauthorized
- `409` - Agent name already exists for this user

---

### Update Agent

Update agent details (requires authentication, must own agent).

```http
PUT /agents/:id
```

**Headers**:
```http
Authorization: Bearer <token>
```

**Request Body**:
```json
{
  "name": "UpdatedAgentName",
  "description": "New description"
}
```

**Response** (200 OK):
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "UpdatedAgentName",
  "description": "New description",
  "updated_at": "2024-01-20T15:45:00Z"
}
```

**Errors**:
- `401` - Unauthorized
- `403` - Not the owner
- `404` - Agent not found

---

### Delete Agent

Delete an agent (requires authentication, must own agent).

```http
DELETE /agents/:id
```

**Headers**:
```http
Authorization: Bearer <token>
```

**Response** (204 No Content)

**Errors**:
- `401` - Unauthorized
- `403` - Not the owner
- `404` - Agent not found

---

## Submission Endpoints

### Submit Agent Code

Submit code for an agent (requires authentication).

```http
POST /submissions
```

**Headers**:
```http
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

**Form Data**:
- `agent_id` (string) - Agent ID
- `code` (file) - ZIP file containing agent code

**ZIP Structure**:
```
my_agent.zip
├── agent.py          # Required: Agent class implementation
├── requirements.txt  # Required: Python dependencies
└── model.pkl         # Optional: Trained model weights
```

**Response** (201 Created):
```json
{
  "id": "789e0123-e89b-12d3-a456-426614174111",
  "agent_id": "123e4567-e89b-12d3-a456-426614174000",
  "version": "1.0.2",
  "status": "queued",
  "code_path": "/storage/submissions/789e0123.zip",
  "created_at": "2024-01-20T16:00:00Z"
}
```

**Errors**:
- `400` - Invalid ZIP file or missing required files
- `401` - Unauthorized
- `403` - Not the agent owner
- `429` - Submission rate limit exceeded (5 per day)

---

### Get Submission Status

Get submission details and build status.

```http
GET /submissions/:id
```

**Response** (200 OK):
```json
{
  "id": "789e0123-e89b-12d3-a456-426614174111",
  "agent_id": "123e4567-e89b-12d3-a456-426614174000",
  "version": "1.0.2",
  "status": "success",
  "docker_image": "registry.example.com/agents/789e0123:latest",
  "build_started_at": "2024-01-20T16:00:05Z",
  "build_completed_at": "2024-01-20T16:03:15Z",
  "created_at": "2024-01-20T16:00:00Z"
}
```

**Status Values**:
- `queued` - Waiting for build
- `building` - Currently building Docker image
- `success` - Build completed, ready for matches
- `failed` - Build failed (check logs)

---

### Get Build Status

Get detailed build status.

```http
GET /submissions/:id/build-status
```

**Response** (200 OK):
```json
{
  "submission_id": "789e0123-e89b-12d3-a456-426614174111",
  "status": "building",
  "phase": "Running",
  "docker_image": null,
  "started_at": "2024-01-20T16:00:05Z",
  "estimated_completion": "2024-01-20T16:05:00Z"
}
```

---

### Get Build Logs

Get build logs for debugging.

```http
GET /submissions/:id/build-logs
```

**Response** (200 OK):
```json
{
  "submission_id": "789e0123-e89b-12d3-a456-426614174111",
  "logs": "Step 1/5 : FROM python:3.10-slim\n ---> abc123def456\nStep 2/5 : COPY requirements.txt .\n..."
}
```

---

### Rebuild Submission

Retry a failed build.

```http
POST /submissions/:id/rebuild
```

**Headers**:
```http
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "id": "789e0123-e89b-12d3-a456-426614174111",
  "status": "queued",
  "message": "Rebuild queued successfully"
}
```

**Errors**:
- `400` - Submission not in failed state
- `401` - Unauthorized
- `403` - Not the owner

---

### List Agent Submissions

Get all submissions for an agent.

```http
GET /submissions/agent/:agentId
```

**Query Parameters**:
- `limit` (optional) - Number of results (default: 20)
- `offset` (optional) - Pagination offset (default: 0)

**Response** (200 OK):
```json
{
  "submissions": [
    {
      "id": "789e0123-e89b-12d3-a456-426614174111",
      "version": "1.0.2",
      "status": "success",
      "is_active": true,
      "created_at": "2024-01-20T16:00:00Z"
    }
  ],
  "total": 1
}
```

---

## Match Endpoints

### Create Manual Match

Create a match between two agents (requires authentication).

```http
POST /matches
```

**Headers**:
```http
Authorization: Bearer <token>
```

**Request Body**:
```json
{
  "agent_1_id": "123e4567-e89b-12d3-a456-426614174000",
  "agent_2_id": "456e7890-e89b-12d3-a456-426614174222",
  "environment_id": "pong"
}
```

**Response** (201 Created):
```json
{
  "id": "match-abc123-def456",
  "environment_id": "pong",
  "submission_1_id": "789e0123-e89b-12d3-a456-426614174111",
  "submission_2_id": "789e0456-e89b-12d3-a456-426614174222",
  "status": "queued",
  "created_at": "2024-01-20T17:00:00Z"
}
```

**Errors**:
- `400` - Invalid agent IDs or environment
- `401` - Unauthorized
- `429` - Match creation rate limit exceeded

---

### Get Match Details

Get match information and results.

```http
GET /matches/:id
```

**Response** (200 OK):
```json
{
  "id": "match-abc123-def456",
  "environment_id": "pong",
  "agent_1": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "SmartPongAgent",
    "username": "user1"
  },
  "agent_2": {
    "id": "456e7890-e89b-12d3-a456-426614174222",
    "name": "RivalAgent",
    "username": "user2"
  },
  "winner_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "completed",
  "score_1": 21,
  "score_2": 18,
  "total_steps": 1500,
  "execution_time_sec": 45.2,
  "replay_json_url": "/storage/replays/match-abc123.json",
  "replay_html_url": "/storage/replays/match-abc123.html",
  "created_at": "2024-01-20T17:00:00Z",
  "completed_at": "2024-01-20T17:01:30Z"
}
```

---

### List Matches

Get a list of matches.

```http
GET /matches
```

**Query Parameters**:
- `environment` (optional) - Filter by environment
- `agent_id` (optional) - Filter by agent
- `status` (optional) - Filter by status (queued, running, completed, failed)
- `limit` (optional) - Number of results (default: 20)
- `offset` (optional) - Pagination offset

**Response** (200 OK):
```json
{
  "matches": [
    {
      "id": "match-abc123-def456",
      "environment_id": "pong",
      "agent_1_name": "SmartPongAgent",
      "agent_2_name": "RivalAgent",
      "winner_id": "123e4567-e89b-12d3-a456-426614174000",
      "status": "completed",
      "created_at": "2024-01-20T17:00:00Z"
    }
  ],
  "total": 1
}
```

---

### Get Match Replay

Download match replay.

```http
GET /matches/:id/replay
```

**Query Parameters**:
- `format` (required) - `json` or `html`

**Response** (200 OK):
- Content-Type: `application/json` (for JSON format)
- Content-Type: `text/html` (for HTML format)

**JSON Format**:
```json
{
  "match_id": "match-abc123-def456",
  "environment": "pong",
  "total_steps": 1500,
  "winner": "agent_1",
  "frames": [
    {
      "step": 0,
      "observation": [0.5, 0.5, 0.0, 0.5],
      "action_1": 1,
      "action_2": 0,
      "reward_1": 0.0,
      "reward_2": 0.0
    }
  ]
}
```

**HTML Format**: Returns interactive HTML page with replay visualization

---

### Get Match Replay URL

Get URL for match replay (without downloading).

```http
GET /matches/:id/replay-url
```

**Response** (200 OK):
```json
{
  "match_id": "match-abc123-def456",
  "replay_json_url": "http://localhost:8080/storage/replays/match-abc123.json",
  "replay_html_url": "http://localhost:8080/storage/replays/match-abc123.html"
}
```

---

### List Agent Matches

Get all matches for a specific agent.

```http
GET /matches/agent/:agentId
```

**Query Parameters**:
- `limit` (optional) - Number of results (default: 20)
- `offset` (optional) - Pagination offset

**Response** (200 OK):
```json
{
  "matches": [
    {
      "id": "match-abc123-def456",
      "opponent_id": "456e7890-e89b-12d3-a456-426614174222",
      "opponent_name": "RivalAgent",
      "result": "win",
      "score": "21-18",
      "elo_change": +15,
      "created_at": "2024-01-20T17:00:00Z"
    }
  ],
  "total": 1
}
```

---

## Leaderboard Endpoints

### Get Leaderboard

Get ranked list of agents.

```http
GET /leaderboard
```

**Query Parameters**:
- `environment` (optional) - Filter by environment
- `limit` (optional) - Number of results (default: 100)
- `offset` (optional) - Pagination offset

**Response** (200 OK):
```json
{
  "leaderboard": [
    {
      "rank": 1,
      "agent_id": "123e4567-e89b-12d3-a456-426614174000",
      "agent_name": "SmartPongAgent",
      "username": "user1",
      "elo_rating": 1650,
      "total_matches": 50,
      "wins": 35,
      "losses": 15,
      "win_rate": 70.0
    },
    {
      "rank": 2,
      "agent_id": "456e7890-e89b-12d3-a456-426614174222",
      "agent_name": "RivalAgent",
      "username": "user2",
      "elo_rating": 1520,
      "total_matches": 40,
      "wins": 25,
      "losses": 15,
      "win_rate": 62.5
    }
  ],
  "total": 2,
  "updated_at": "2024-01-20T17:30:00Z"
}
```

---

### Get Environment Leaderboard

Get leaderboard for a specific environment.

```http
GET /leaderboard/environment/:envId
```

**Response**: Same format as general leaderboard, filtered by environment

---

## User Endpoints

### Get Current User

Get authenticated user's profile.

```http
GET /users/me
```

**Headers**:
```http
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "myusername",
  "email": "user@example.com",
  "created_at": "2024-01-15T10:30:00Z",
  "stats": {
    "total_agents": 3,
    "total_matches": 75,
    "total_wins": 45
  }
}
```

---

### Update Current User

Update user profile.

```http
PUT /users/me
```

**Headers**:
```http
Authorization: Bearer <token>
```

**Request Body**:
```json
{
  "email": "newemail@example.com"
}
```

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "myusername",
  "email": "newemail@example.com",
  "updated_at": "2024-01-20T18:00:00Z"
}
```

---

## WebSocket API

### Connect to WebSocket

```
ws://localhost:8080/api/v1/ws
```

**Query Parameters**:
- `token` (required) - JWT authentication token

**Example**:
```javascript
const ws = new WebSocket('ws://localhost:8080/api/v1/ws?token=YOUR_JWT_TOKEN');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
};
```

### Message Types

#### Build Status Update

```json
{
  "type": "build_status",
  "submission_id": "789e0123-e89b-12d3-a456-426614174111",
  "status": "success",
  "docker_image": "registry.example.com/agents/789e0123:latest",
  "timestamp": "2024-01-20T16:03:15Z"
}
```

#### Match Created

```json
{
  "type": "match_created",
  "match_id": "match-abc123-def456",
  "agent_1_id": "123e4567-e89b-12d3-a456-426614174000",
  "agent_2_id": "456e7890-e89b-12d3-a456-426614174222",
  "timestamp": "2024-01-20T17:00:00Z"
}
```

#### Match Completed

```json
{
  "type": "match_completed",
  "match_id": "match-abc123-def456",
  "winner_id": "123e4567-e89b-12d3-a456-426614174000",
  "elo_changes": {
    "123e4567-e89b-12d3-a456-426614174000": 15,
    "456e7890-e89b-12d3-a456-426614174222": -15
  },
  "timestamp": "2024-01-20T17:01:30Z"
}
```

---

## Additional Resources

- [Swagger UI](http://localhost:8080/swagger/index.html) - Interactive API documentation
- [Getting Started](GETTING_STARTED.md) - Quick start guide
- [Architecture](ARCHITECTURE.md) - System architecture
- [Rate Limiting](RATE_LIMITING.md) - Detailed rate limiting information
