# RL-Arena Documentation

Welcome to the RL-Arena documentation! This repository contains comprehensive guides for the RL-Arena platform - a competitive reinforcement learning environment where AI agents battle against each other.

## 📚 Documentation Index

### Getting Started
- **[Getting Started Guide](GETTING_STARTED.md)** - Install, train, and submit your first agent
- **[Quick Start Tutorial](#quick-start)** - 5-minute introduction to RL-Arena

### Architecture & Design
- **[Architecture Overview](ARCHITECTURE.md)** - System design, components, and data flow
- **[API Reference](API_REFERENCE.md)** - Complete REST API and WebSocket documentation
- **[Rate Limiting System](RATE_LIMITING.md)** - Match rate limits and cooldown system

### Development
- **[Development Guide](DEVELOPMENT.md)** - Set up your dev environment and contribute
- **[Deployment Guide](DEPLOYMENT.md)** - Deploy to production (Docker, Kubernetes)

## 🎯 Quick Start

### 1. Install RL-Arena

```bash
pip install rl-arena
```

### 2. Train Your Agent Locally

```python
import gymnasium as gym
import rl_arena

# Create Pong environment
env = gym.make('rl-arena/Pong-v0')

# Your agent implementation
class MyAgent:
    def get_action(self, observation):
        # Your RL logic here
        return action

# Train
agent = MyAgent()
for episode in range(100):
    obs, info = env.reset()
    while True:
        action = agent.get_action(obs)
        obs, reward, done, truncated, info = env.step(action)
        if done or truncated:
            break
```

### 3. Submit to Arena

Package your agent:
```bash
# Create submission package
zip my_agent.zip agent.py requirements.txt
```

Upload via web interface at [RL-Arena Web](http://localhost:5173)

### 4. Watch Your Agent Compete

- **Auto-Matchmaking**: Agents matched every 30 seconds
- **ELO Rankings**: Track your position on the leaderboard
- **Watch Replays**: View matches in HTML or JSON format

## 🏗️ System Overview

RL-Arena consists of five main components:

```
┌─────────────┐
│  Web (React)│  ← User interface
└──────┬──────┘
       │ REST API / WebSocket
       ▼
┌─────────────┐
│ Backend (Go)│  ← API server, matchmaking, ELO
└──────┬──────┘
       │ gRPC
       ▼
┌─────────────┐
│ Executor    │  ← Match execution engine
│  (Python)   │
└──────┬──────┘
       │ Kubernetes Jobs
       ▼
┌─────────────┐
│ Match Pods  │  ← Isolated agent battles
└─────────────┘

┌─────────────┐
│ RL-Arena Env│  ← pip package for local training
└─────────────┘
```

## 📖 Key Concepts

### Agents
AI agents that compete in various game environments. Each agent:
- Has an ELO rating (starts at 1200)
- Can submit multiple code versions
- Automatically enters matchmaking after successful build

### Submissions
Code uploads for agents. Each submission:
- Built into a Docker image
- Validated for security
- Versioned automatically
- Can be activated for matchmaking

### Matches
Battles between two agents:
- Auto-created by matchmaking system
- ELO-based opponent selection (±100-500 points)
- Executed in isolated Kubernetes pods
- Results recorded with full replay data

### Rate Limiting
Fair play enforcement:
- **Cooldown**: 5 minutes between matches per agent
- **Daily Limit**: 100 matches per day per agent
- **Auto-Reset**: Daily counter resets at midnight UTC

### ELO Rating System
Competitive ranking:
- **Provisional** (< 10 matches): K-factor = 40 (fast convergence)
- **Intermediate** (10-20 matches): K-factor = 32
- **Established** (> 20 matches): K-factor = 24 (stable)

## 🔗 Quick Links

### Repositories
- [rl-arena-backend](https://github.com/rl-arena/rl-arena-backend) - Go REST API server
- [rl-arena-executor](https://github.com/rl-arena/rl-arena-executor) - Python gRPC match executor
- [rl-arena-web](https://github.com/rl-arena/rl-arena-web) - React web frontend
- [rl-arena-env](https://github.com/rl-arena/rl-arena-env) - Python environment library
- [rl-arena-docs](https://github.com/rl-arena/rl-arena-docs) - This documentation

### Resources
- **PyPI Package**: [rl-arena](https://pypi.org/project/rl-arena/)
- **API Docs**: http://localhost:8080/swagger/index.html (when running locally)

## 🛠️ Technology Stack

### Frontend
- **React 18** - UI framework
- **Vite 5** - Build tool
- **Tailwind CSS** - Styling
- **Zustand** - State management

### Backend
- **Go 1.21+** - Programming language
- **Gin** - HTTP framework
- **PostgreSQL 15+** - Database
- **JWT** - Authentication

### Executor
- **Python 3.10+** - Programming language
- **gRPC** - RPC framework
- **Kubernetes** - Container orchestration
- **Docker** - Containerization

### Environment
- **Python 3.10+** - Programming language
- **Gymnasium** - RL framework
- **NumPy** - Numerical computing

## 📋 Common Tasks

### Check Agent Stats
```bash
curl http://localhost:8080/api/v1/agents/{agent-id}/stats
```

### View Leaderboard
```bash
curl http://localhost:8080/api/v1/leaderboard
```

### Download Replay
```bash
# HTML format (interactive visualization)
curl -O http://localhost:8080/api/v1/matches/{match-id}/replay?format=html

# JSON format (frame data)
curl -O http://localhost:8080/api/v1/matches/{match-id}/replay?format=json
```

### Reset Database (Development)
```bash
docker exec -i rl-arena-backend-db-1 psql -U postgres -d rl_arena < scripts/reset_database.sql
```

## 🐛 Troubleshooting

### Build Failed
**Problem**: Agent build fails

**Solution**:
1. Check build logs: GET `/api/v1/submissions/{id}/build-logs`
2. Verify `requirements.txt` has correct dependencies
3. Ensure `agent.py` has required `Agent` class with `get_action()` method

### No Matches
**Problem**: Agent not getting matched

**Causes**:
- Still in 5-minute cooldown
- Reached daily limit (100 matches)
- No opponents with similar ELO

**Check status**:
```bash
curl http://localhost:8080/api/v1/agents/{agent-id}/stats
```

### WebSocket Not Connecting
**Problem**: Real-time updates not working

**Solution**:
1. Verify WebSocket URL: `ws://localhost:8080/api/v1/ws?token={jwt-token}`
2. Check JWT token is valid
3. Verify CORS settings allow WebSocket origin

## 🤝 Contributing

We welcome contributions! See the [Development Guide](DEVELOPMENT.md) for setup instructions.

## 📄 License

RL-Arena is open source:
- **Backend, Executor, Web, Docs**: Apache 2.0 License
- **Environment Library**: Apache 2.0 License

See LICENSE files in each repository.

## 💬 Getting Help

- **Documentation**: You're reading it!
- **API Docs**: http://localhost:8080/swagger/index.html
- **GitHub Issues**: Report bugs and request features

---

<div align="center">
  <sub>Built with ❤️ by the RL-Arena community</sub>
</div>