# Development Guide

This guide covers setting up a development environment for contributing to RL-Arena.

## Table of Contents

- [Development Setup](#development-setup)
- [Project Structure](#project-structure)
- [Development Workflow](#development-workflow)
- [Code Style and Standards](#code-style-and-standards)
- [Testing](#testing)
- [Debugging](#debugging)
- [Common Development Tasks](#common-development-tasks)

## Development Setup

### Prerequisites

Install the following tools:

- **Go 1.21+** - [Download](https://golang.org/dl/)
- **Python 3.10+** - [Download](https://www.python.org/downloads/)
- **Node.js 18+** - [Download](https://nodejs.org/)
- **PostgreSQL 15+** - [Download](https://www.postgresql.org/download/)
- **Docker** - [Download](https://www.docker.com/get-started)
- **kubectl** - [Install](https://kubernetes.io/docs/tasks/tools/)
- **Minikube** (for local K8s) - [Install](https://minikube.sigs.k8s.io/docs/start/)
- **Git** - [Download](https://git-scm.com/downloads/)

### Clone Repositories

```bash
# Create workspace
mkdir rl-arena-dev && cd rl-arena-dev

# Clone all repositories
git clone https://github.com/rl-arena/rl-arena-backend.git
git clone https://github.com/rl-arena/rl-arena-executor.git
git clone https://github.com/rl-arena/rl-arena-web.git
git clone https://github.com/rl-arena/rl-arena-env.git
git clone https://github.com/rl-arena/rl-arena-docs.git
```

### Backend Setup (Go)

```bash
cd rl-arena-backend

# Install dependencies
go mod download

# Copy environment file
cp .env.example .env

# Edit .env with your settings
# DB_HOST=localhost
# DB_PORT=5433
# DB_USER=postgres
# DB_PASSWORD=postgres
# DB_NAME=rl_arena
# JWT_SECRET=your-dev-secret-key

# Start PostgreSQL with Docker
docker-compose up -d db

# Run migrations
go run cmd/server/main.go migrate up

# Install development tools
go install github.com/cosmtrek/air@latest  # Hot reload
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest  # Linter

# Run with hot reload
air

# Or run normally
go run cmd/server/main.go
```

Backend runs on `http://localhost:8080`

### Executor Setup (Python)

```bash
cd rl-arena-executor

# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Windows:
venv\Scripts\activate
# On Unix/macOS:
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Install development dependencies
pip install pytest pytest-cov black ruff mypy

# Generate gRPC code
python -m grpc_tools.protoc \
    -I./proto \
    --python_out=. \
    --grpc_python_out=. \
    ./proto/executor.proto

# Start Minikube (for local Kubernetes)
minikube start

# Run executor
python -m executor.server
```

Executor runs on `localhost:50051` (gRPC)

### Web Setup (React)

```bash
cd rl-arena-web

# Install dependencies
npm install

# Copy environment file
cp .env.example .env

# Edit .env
# VITE_API_URL=http://localhost:8080/api/v1
# VITE_WS_URL=ws://localhost:8080/api/v1/ws

# Run development server
npm run dev
```

Frontend runs on `http://localhost:5173`

### Environment Library Setup (Python)

```bash
cd rl-arena-env

# Install in development mode
pip install -e .

# Or with dev dependencies
pip install -e ".[dev]"

# Run tests
pytest
```

## Project Structure

### Backend (Go)

```
rl-arena-backend/
├── cmd/
│   └── server/
│       └── main.go              # Entry point
├── internal/
│   ├── api/
│   │   ├── handlers/            # HTTP handlers
│   │   ├── middleware/          # Middleware
│   │   └── router.go            # Routes
│   ├── config/                  # Configuration
│   ├── models/                  # Data models
│   ├── repository/              # Data access
│   ├── service/                 # Business logic
│   └── websocket/               # WebSocket
├── pkg/                         # Shared packages
├── migrations/                  # SQL migrations
├── docs/                        # Documentation
├── go.mod
└── go.sum
```

**Key Files**:
- `cmd/server/main.go` - Application entry point
- `internal/service/matchmaking_service.go` - Auto-matchmaking logic
- `internal/service/elo_service.go` - ELO rating calculations
- `internal/repository/*_repository.go` - Database queries

### Executor (Python)

```
rl-arena-executor/
├── executor/
│   ├── server.py                # gRPC server
│   ├── k8s_runner.py           # Kubernetes runner
│   ├── match_runner.py         # Match execution
│   ├── replay_recorder.py      # Replay recording
│   ├── validation.py           # Code validation
│   └── config.py               # Configuration
├── orchestrator/
│   └── run_match.py            # Runs in K8s pods
├── proto/
│   └── executor.proto          # gRPC definitions
├── k8s/
│   └── deployment.yaml         # K8s manifests
├── tests/                      # Test files
└── requirements.txt
```

**Key Files**:
- `executor/server.py` - gRPC service implementation
- `executor/k8s_runner.py` - Creates and monitors K8s Jobs
- `orchestrator/run_match.py` - Match orchestration inside pods

### Web (React)

```
rl-arena-web/
├── src/
│   ├── components/             # React components
│   │   ├── common/             # Reusable components
│   │   ├── leaderboard/        # Leaderboard components
│   │   ├── replay/             # Replay player
│   │   └── submission/         # Agent submission
│   ├── pages/                  # Page components
│   ├── hooks/                  # Custom hooks
│   ├── services/               # API clients
│   ├── store/                  # State management
│   └── utils/                  # Utilities
├── public/                     # Static assets
└── package.json
```

**Key Files**:
- `src/services/api.js` - REST API client
- `src/services/websocket.js` - WebSocket client
- `src/components/replay/ReplayCanvas.jsx` - Replay visualization

### Environment (Python)

```
rl-arena-env/
├── rl_arena/
│   ├── envs/                   # Game environments
│   │   ├── pong.py             # Pong environment
│   │   └── ...
│   ├── replay/                 # Replay generation
│   └── __init__.py
├── tests/                      # Test files
└── setup.py
```

**Key Files**:
- `rl_arena/envs/pong.py` - Pong environment implementation
- `rl_arena/replay/html_generator.py` - HTML replay generation

## Development Workflow

### 1. Create Feature Branch

```bash
git checkout -b feature/amazing-feature
```

### 2. Make Changes

Edit code following style guidelines (see below)

### 3. Run Tests

```bash
# Backend
cd rl-arena-backend
go test ./...

# Executor
cd rl-arena-executor
pytest

# Web
cd rl-arena-web
npm test

# Environment
cd rl-arena-env
pytest
```

### 4. Lint Code

```bash
# Backend (Go)
golangci-lint run

# Executor (Python)
black executor/ tests/
ruff check executor/ tests/
mypy executor/

# Web (JavaScript/React)
npm run lint
```

### 5. Commit Changes

```bash
git add .
git commit -m "feat: add amazing feature"
```

Use conventional commits:
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation
- `style:` - Code style
- `refactor:` - Code refactoring
- `test:` - Tests
- `chore:` - Maintenance

### 6. Push and Create PR

```bash
git push origin feature/amazing-feature
```

Then create a Pull Request on GitHub.

## Code Style and Standards

### Go (Backend)

**Format**:
```bash
go fmt ./...
goimports -w .
```

**Conventions**:
- Use `gofmt` for formatting
- Follow [Effective Go](https://golang.org/doc/effective_go)
- Use meaningful variable names
- Add comments for exported functions
- Handle errors explicitly

**Example**:
```go
// GetAgent retrieves an agent by ID
func (s *AgentService) GetAgent(id string) (*models.Agent, error) {
    agent, err := s.repo.FindByID(id)
    if err != nil {
        return nil, fmt.Errorf("failed to get agent: %w", err)
    }
    return agent, nil
}
```

### Python (Executor, Environment)

**Format**:
```bash
black .
```

**Lint**:
```bash
ruff check .
mypy .
```

**Conventions**:
- Follow [PEP 8](https://pep8.org/)
- Use type hints
- Use docstrings for functions/classes
- Maximum line length: 88 (Black default)

**Example**:
```python
def get_action(self, observation: np.ndarray) -> int:
    """
    Select an action based on the observation.
    
    Args:
        observation: Current game state
        
    Returns:
        Integer representing the action to take
    """
    # Implementation
    return action
```

### JavaScript/React (Web)

**Format**:
```bash
npm run lint
npm run format
```

**Conventions**:
- Use ESLint configuration
- Follow React best practices
- Use functional components and hooks
- PropTypes for type checking
- Meaningful component and variable names

**Example**:
```jsx
import PropTypes from 'prop-types';

function LeaderboardRow({ agent, rank }) {
  return (
    <tr>
      <td>{rank}</td>
      <td>{agent.name}</td>
      <td>{agent.elo_rating}</td>
    </tr>
  );
}

LeaderboardRow.propTypes = {
  agent: PropTypes.shape({
    name: PropTypes.string.isRequired,
    elo_rating: PropTypes.number.isRequired,
  }).isRequired,
  rank: PropTypes.number.isRequired,
};

export default LeaderboardRow;
```

## Testing

### Backend Tests (Go)

```bash
# Run all tests
go test ./...

# Run with coverage
go test -cover ./...

# Run specific package
go test ./internal/service

# Run specific test
go test -run TestELOService ./internal/service
```

**Example Test**:
```go
func TestELOService_CalculateRating(t *testing.T) {
    service := NewELOService()
    
    newRating := service.CalculateRating(1200, 1200, true, 10)
    
    assert.Greater(t, newRating, 1200)
}
```

### Executor Tests (Python)

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=executor --cov-report=html

# Run specific test
pytest tests/test_k8s_runner.py -v
```

**Example Test**:
```python
def test_match_runner():
    runner = MatchRunner()
    result = runner.run_match(agent1, agent2)
    assert result.status == "SUCCESS"
    assert result.winner_agent_id in [agent1.id, agent2.id]
```

### Web Tests (React)

```bash
# Run tests
npm test

# Run with coverage
npm test -- --coverage

# Run in watch mode
npm test -- --watch
```

**Example Test**:
```jsx
import { render, screen } from '@testing-library/react';
import LeaderboardRow from './LeaderboardRow';

test('renders agent name and rating', () => {
  const agent = { name: 'TestAgent', elo_rating: 1500 };
  render(<LeaderboardRow agent={agent} rank={1} />);
  
  expect(screen.getByText('TestAgent')).toBeInTheDocument();
  expect(screen.getByText('1500')).toBeInTheDocument();
});
```

## Debugging

### Backend (Go)

**Using Delve**:
```bash
# Install Delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug
dlv debug cmd/server/main.go
```

**VS Code**: Add to `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Server",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/cmd/server",
      "env": {
        "DB_HOST": "localhost"
      }
    }
  ]
}
```

### Executor (Python)

**Using pdb**:
```python
import pdb; pdb.set_trace()
```

**VS Code**: Add to `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Executor",
      "type": "python",
      "request": "launch",
      "module": "executor.server",
      "console": "integratedTerminal"
    }
  ]
}
```

### Web (React)

**Browser DevTools**:
- Chrome DevTools (F12)
- React DevTools extension
- Console logging with `console.log()`

**VS Code**: Add to `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Chrome",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:5173",
      "webRoot": "${workspaceFolder}/src"
    }
  ]
}
```

## Common Development Tasks

### Add New API Endpoint

1. **Define handler** (`internal/api/handlers/`):
```go
func (h *Handler) NewEndpoint(c *gin.Context) {
    // Implementation
    c.JSON(200, gin.H{"message": "success"})
}
```

2. **Add route** (`internal/api/router.go`):
```go
router.GET("/new-endpoint", handler.NewEndpoint)
```

3. **Add Swagger docs**:
```go
// NewEndpoint godoc
// @Summary      New endpoint description
// @Tags         tag
// @Accept       json
// @Produce      json
// @Success      200  {object}  Response
// @Router       /new-endpoint [get]
func (h *Handler) NewEndpoint(c *gin.Context) {
    // ...
}
```

4. **Regenerate Swagger**:
```bash
swag init -g cmd/server/main.go -o docs
```

### Add New Environment

1. **Create environment** (`rl_arena/envs/new_game.py`):
```python
import gymnasium as gym
from gymnasium import spaces

class NewGameEnv(gym.Env):
    metadata = {"render_modes": ["human"]}
    
    def __init__(self):
        super().__init__()
        self.action_space = spaces.Discrete(3)
        self.observation_space = spaces.Box(low=0, high=1, shape=(4,))
    
    def reset(self, seed=None, options=None):
        super().reset(seed=seed)
        # Return initial observation
        return observation, info
    
    def step(self, action):
        # Execute action
        return observation, reward, terminated, truncated, info
```

2. **Register environment** (`rl_arena/__init__.py`):
```python
from gymnasium.envs.registration import register

register(
    id='rl-arena/NewGame-v0',
    entry_point='rl_arena.envs:NewGameEnv',
)
```

3. **Add HTML generator** (`rl_arena/replay/newgame_html.py`)

4. **Update backend** with new environment ID

### Database Migration

1. **Create migration file** (`migrations/XXX_description.sql`):
```sql
-- Up migration
CREATE TABLE new_table (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

-- Down migration (in separate file or comment)
DROP TABLE new_table;
```

2. **Apply migration**:
```bash
go run cmd/server/main.go migrate up
```

### Update gRPC Proto

1. **Edit proto file** (`proto/executor.proto`):
```protobuf
message NewRequest {
    string field = 1;
}
```

2. **Regenerate code**:

Backend (Go):
```bash
protoc --go_out=. --go-grpc_out=. proto/executor.proto
```

Executor (Python):
```bash
python -m grpc_tools.protoc \
    -I./proto \
    --python_out=. \
    --grpc_python_out=. \
    ./proto/executor.proto
```

3. **Update implementations** in both backend and executor

---

For more information:
- [Architecture](ARCHITECTURE.md) - System design
- [API Reference](API_REFERENCE.md) - API documentation
- [Contributing](https://github.com/rl-arena/.github/blob/main/CONTRIBUTING.md) - Contribution guidelines
