# RL-Arena Architecture

This document provides a comprehensive overview of the RL-Arena platform architecture, including system components, data flow, and design decisions.

## Table of Contents

- [System Overview](#system-overview)
- [Component Architecture](#component-architecture)
- [Data Flow](#data-flow)
- [Technology Stack](#technology-stack)
- [Design Decisions](#design-decisions)
- [Scalability](#scalability)

## System Overview

RL-Arena is a distributed system for competitive reinforcement learning, consisting of five main components:

```
┌─────────────────┐
│   RL-Arena Web  │  (React Frontend)
└────────┬────────┘
         │ REST API / WebSocket
         ▼
┌─────────────────┐
│  Backend Server │  (Go REST API)
└────────┬────────┘
         │ gRPC
         ▼
┌─────────────────┐
│    Executor     │  (Python gRPC Service)
└────────┬────────┘
         │ Kubernetes Jobs
         ▼
┌─────────────────┐
│  Match Pods     │  (Isolated Execution)
└─────────────────┘

┌─────────────────┐
│  RL-Arena Env   │  (pip package)
└─────────────────┘
   Used by agents for local training
```

## Component Architecture

### 1. RL-Arena Web (Frontend)

**Technology**: React 18, Vite, Tailwind CSS

**Responsibilities**:
- User interface for browsing competitions
- Agent submission and management
- Real-time leaderboard display
- Match replay visualization (HTML/JSON formats)
- WebSocket connection for live updates

**Key Features**:
- Responsive design for mobile/desktop
- Real-time notifications via WebSocket
- Interactive replay player with speed control
- Drag-and-drop agent upload

**Structure**:
```
src/
├── components/     # Reusable UI components
├── pages/          # Route pages
├── hooks/          # Custom React hooks
├── services/       # API and WebSocket clients
├── store/          # Zustand state management
└── utils/          # Helper functions
```

### 2. Backend Server (API Layer)

**Technology**: Go 1.21+, Gin framework, PostgreSQL

**Responsibilities**:
- User authentication (JWT-based)
- Agent and submission management
- Match creation and scheduling
- Auto-matchmaking system
- ELO rating calculations
- Rate limiting enforcement
- Build monitoring
- WebSocket hub for notifications

**Key Features**:
- RESTful API with Swagger documentation
- JWT authentication for secure endpoints
- Auto-matchmaking every 30 seconds
- Match rate limiting (5min cooldown, 100/day)
- ELO rating with provisional system
- Kubernetes build monitoring
- WebSocket for real-time updates

**Architecture Pattern**: Clean Architecture

```
internal/
├── api/
│   ├── handlers/       # HTTP request handlers
│   ├── middleware/     # Auth, CORS, logging
│   └── router.go       # Route definitions
├── models/             # Domain models
├── repository/         # Data access layer
├── service/            # Business logic
└── websocket/          # WebSocket hub
```

**Database Schema**:
```
users
├── id (UUID)
├── username
├── email
└── password_hash

agents
├── id (UUID)
├── user_id (FK)
├── name
├── environment_id
├── elo_rating
└── active_submission_id (FK)

submissions
├── id (UUID)
├── agent_id (FK)
├── version
├── code_path
├── docker_image
├── build_status
└── build_logs

matches
├── id (UUID)
├── environment_id
├── submission_1_id (FK)
├── submission_2_id (FK)
├── winner_id (FK)
├── replay_json_path
├── replay_html_path
└── execution_time

agent_match_stats
├── agent_id (FK)
├── last_match_at (UTC)
├── matches_today
├── daily_reset_at
└── total_matches

matchmaking_queue
├── agent_id (FK)
├── submission_id (FK)
├── environment_id
└── queued_at
```

### 3. Executor (Match Engine)

**Technology**: Python 3.10+, gRPC, Kubernetes

**Responsibilities**:
- Execute agent matches in isolated environments
- Manage Kubernetes Jobs for match execution
- Record frame-by-frame replays
- Enforce resource limits (CPU, memory, timeout)
- Return match results to backend

**Key Features**:
- Kubernetes-native execution (Jobs)
- Docker image-based agents
- Pod-level isolation with RBAC
- Resource limits enforcement
- Automatic cleanup of completed jobs
- Replay recording in JSON format

**Execution Flow**:
```
1. Backend → gRPC RunMatch request
2. Executor → Create Kubernetes Job
3. Job Init Containers → Pull agent Docker images
4. Orchestrator Container → Run match
5. Match completes → Record replay
6. Return results → Backend via gRPC
7. Cleanup → Delete Job after completion
```

**Security**:
- Pod isolation (separate namespaces)
- Resource limits (CPU, memory)
- RBAC with minimal permissions
- Non-root container execution
- Dropped Linux capabilities
- Code validation before execution

### 4. RL-Arena Env (Training Library)

**Technology**: Python 3.10+, Gymnasium

**Responsibilities**:
- Provide standardized RL environments
- Local agent training interface
- Replay HTML generation
- Environment versioning

**Key Features**:
- Gymnasium-compatible API
- Easy pip installation
- HTML replay generation
- Multiple game environments (Pong, etc.)
- Extensible for custom environments

**Usage**:
```python
import gymnasium as gym
import rl_arena

env = gym.make('rl-arena/Pong-v0')
observation, info = env.reset()

while True:
    action = agent.get_action(observation)
    observation, reward, terminated, truncated, info = env.step(action)
    if terminated or truncated:
        break
```

## Data Flow

### 1. Agent Submission Flow

```
User → Web UI → Upload ZIP
  ↓
Backend receives submission
  ↓
Create Kubernetes Job (Kaniko builder)
  ↓
Build Docker image from code
  ↓
Push to container registry
  ↓
Update submission status (SUCCESS/FAILED)
  ↓
Add agent to matchmaking queue
  ↓
WebSocket notification to user
```

### 2. Matchmaking Flow

```
Every 30 seconds:
  ↓
Matchmaking service queries eligible agents
  ↓
Filter by:
  - Build status = SUCCESS
  - Last match > 5 minutes ago (UTC)
  - Matches today < 100
  ↓
Select pairs with similar ELO (±100-500)
  ↓
Create match record in database
  ↓
gRPC call to Executor service
  ↓
Executor creates Kubernetes Job
  ↓
Match executes in isolated pod
  ↓
Results returned to Backend
  ↓
Update ELO ratings
  ↓
Save replay files (JSON + HTML)
  ↓
Update agent_match_stats
  ↓
WebSocket notification to users
```

### 3. Replay Viewing Flow

```
User clicks "View Replay" on Web UI
  ↓
Frontend requests replay URL
  ↓
Backend returns replay file path
  ↓
Two format options:
  ├─ HTML: Interactive visualization (default)
  └─ JSON: Frame-by-frame data
  ↓
Frontend loads and renders replay
  ↓
User can:
  - Play/pause
  - Control speed (0.5x, 1x, 2x)
  - Scrub through frames
  - Download for offline viewing
```

## Technology Stack

### Frontend
- **Framework**: React 18.2
- **Build Tool**: Vite 5.4
- **Styling**: Tailwind CSS 3.4
- **Routing**: React Router v6
- **State**: Zustand
- **HTTP**: Axios
- **Real-time**: WebSocket API

### Backend
- **Language**: Go 1.21+
- **Framework**: Gin
- **Database**: PostgreSQL 15+
- **Auth**: JWT (golang-jwt)
- **Logging**: Zap
- **gRPC Client**: google.golang.org/grpc

### Executor
- **Language**: Python 3.10+
- **RPC**: gRPC
- **Orchestration**: Kubernetes
- **Container**: Docker
- **Environment**: Gymnasium

### Infrastructure
- **Container Registry**: Docker Registry
- **Build System**: Kaniko (Kubernetes-native)
- **Orchestration**: Kubernetes
- **Database**: PostgreSQL
- **Storage**: Local filesystem / PVC

## Design Decisions

### 1. Why Kubernetes for Builds and Execution?

**Decision**: Use Kubernetes Jobs for both building agent images and executing matches

**Rationale**:
- **Isolation**: Each build/match runs in separate pod
- **Scalability**: Horizontal scaling across cluster nodes
- **Resource Management**: CPU/memory limits enforced
- **Security**: Pod-level isolation with RBAC
- **Cloud-Native**: Easy deployment to any K8s cluster

### 2. Why gRPC for Executor Communication?

**Decision**: Use gRPC instead of REST for Backend ↔ Executor communication

**Rationale**:
- **Performance**: Binary protocol, faster than JSON
- **Streaming**: Supports bidirectional streaming
- **Type Safety**: Protocol Buffers ensure schema consistency
- **Code Generation**: Auto-generated client/server code

### 3. Why UTC for All Timestamps?

**Decision**: Store all timestamps in UTC, convert to local only for display

**Rationale**:
- **Consistency**: Database NOW() returns UTC
- **No Timezone Issues**: Avoids KST vs UTC comparison bugs
- **Distributed Systems**: Works across multiple time zones
- **Rate Limiting Accuracy**: Cooldown calculations always correct

### 4. Why Match Rate Limiting?

**Decision**: Implement 5-minute cooldown and 100 matches/day limit

**Rationale**:
- **Resource Protection**: Prevents system overload
- **Fair Play**: All agents get equal opportunities
- **Cost Management**: Limits compute costs
- **Quality Over Quantity**: Encourages better agents

### 5. Why ELO Rating System?

**Decision**: Use provisional ELO with dynamic K-factors

**Rationale**:
- **Fast Convergence**: New agents reach true rating quickly
- **Fairness**: Established agents less volatile
- **Proven System**: Chess, Kaggle use similar systems
- **Matchmaking**: Easy to find balanced opponents

### 6. Why Two Replay Formats?

**Decision**: Support both HTML and JSON replay formats

**Rationale**:
- **HTML**: User-friendly, interactive visualization
- **JSON**: Developer-friendly, analyzable data
- **Flexibility**: Users choose based on needs
- **Offline Viewing**: Both formats downloadable

## Scalability

### Horizontal Scaling

**Frontend (Web)**:
- Static assets served from CDN
- Stateless React app
- Scale: Infinite (static files)

**Backend (API)**:
- Stateless REST API
- JWT authentication (no sessions)
- Scale: Add more replicas behind load balancer

**Executor (gRPC)**:
- Stateless gRPC service
- Each request independent
- Scale: Add more executor pods

**Kubernetes Jobs**:
- Each match is separate Job
- Autoscaling based on queue length
- Scale: Limited by cluster capacity

### Vertical Scaling

**Database (PostgreSQL)**:
- Read replicas for queries
- Write to primary only
- Connection pooling
- Indexing on frequently queried fields

**Storage**:
- Replays stored on PersistentVolumeClaim
- Can use object storage (S3, GCS) for unlimited capacity

### Performance Optimizations

**Caching**:
- Leaderboard cached (refresh every 30s)
- Agent stats cached (refresh on match completion)
- Static assets cached by CDN

**Database Indexing**:
```sql
CREATE INDEX idx_agents_elo ON agents(elo_rating DESC);
CREATE INDEX idx_matches_env ON matches(environment_id);
CREATE INDEX idx_submissions_agent ON submissions(agent_id, created_at DESC);
CREATE INDEX idx_agent_match_stats_last_match ON agent_match_stats(last_match_at);
```

**Query Optimization**:
- Pagination for large result sets
- Efficient JOINs with proper indexes
- Batch operations where possible

## Monitoring and Observability

### Logging
- **Backend**: Structured logging with Zap (JSON format)
- **Executor**: Python logging to stdout
- **Kubernetes**: Centralized log aggregation

### Metrics
- Match execution time
- Build success/failure rates
- API response times
- WebSocket connection count
- Active matches count

### Health Checks
- `/health` endpoint on all services
- Kubernetes liveness/readiness probes
- Database connection monitoring

## Security

### Authentication & Authorization
- JWT tokens for API authentication
- Token expiration and refresh
- RBAC in Kubernetes
- User owns agents (authorization checks)

### Code Security
- Agent code validation before build
- Docker image scanning (Trivy)
- Resource limits on execution
- Network policies in Kubernetes

### Data Security
- Password hashing (bcrypt)
- HTTPS/TLS for production
- SQL injection prevention (parameterized queries)
- CORS configuration

---

For more details on specific components, see:
- [Backend API Reference](API_REFERENCE.md)
- [Deployment Guide](DEPLOYMENT.md)
- [Development Guide](DEVELOPMENT.md)
