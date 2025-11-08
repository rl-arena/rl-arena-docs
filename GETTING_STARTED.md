# Getting Started with RL-Arena

This guide will help you get started with RL-Arena, from installing the environment to submitting your first agent.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Training Your First Agent](#training-your-first-agent)
- [Submitting to the Arena](#submitting-to-the-arena)
- [Watching Matches](#watching-matches)

## Prerequisites

Before you begin, ensure you have:

- **Python 3.10+** installed
- **pip** package manager
- Basic understanding of reinforcement learning concepts
- (Optional) Docker for local testing

## Installation

### 1. Install RL-Arena Environment

Install the rl-arena package from PyPI:

```bash
pip install rl-arena
```

This installs the environment library you'll use to train your agents locally.

### 2. Verify Installation

```python
import rl_arena
import gymnasium as gym

# Create a Pong environment
env = gym.make('rl-arena/Pong-v0')
print(f"Environment: {env}")
print(f"Action space: {env.action_space}")
print(f"Observation space: {env.observation_space}")
```

## Training Your First Agent

### 1. Basic Agent Structure

Create a file named `agent.py`:

```python
import numpy as np

class PongAgent:
    """Simple rule-based Pong agent"""
    
    def __init__(self):
        pass
    
    def get_action(self, observation):
        """
        Choose an action based on observation.
        
        Args:
            observation: Current game state
            
        Returns:
            action: Integer representing the action to take
        """
        # Simple strategy: follow the ball
        ball_y = observation[1]  # Ball Y position
        paddle_y = observation[3]  # Paddle Y position
        
        if ball_y > paddle_y:
            return 1  # Move down
        elif ball_y < paddle_y:
            return 0  # Move up
        else:
            return 2  # Stay
```

### 2. Training Loop

```python
import gymnasium as gym
import rl_arena

# Create environment
env = gym.make('rl-arena/Pong-v0')

# Initialize agent
agent = PongAgent()

# Training loop
for episode in range(100):
    observation, info = env.reset()
    total_reward = 0
    
    while True:
        # Get action from agent
        action = agent.get_action(observation)
        
        # Take action in environment
        observation, reward, terminated, truncated, info = env.step(action)
        total_reward += reward
        
        if terminated or truncated:
            break
    
    print(f"Episode {episode + 1}: Total Reward = {total_reward}")

env.close()
```

### 3. Advanced: Using RL Libraries

You can use popular RL libraries like Stable-Baselines3:

```python
from stable_baselines3 import PPO
import gymnasium as gym
import rl_arena

# Create environment
env = gym.make('rl-arena/Pong-v0')

# Train PPO agent
model = PPO("MlpPolicy", env, verbose=1)
model.learn(total_timesteps=100000)

# Save model
model.save("pong_agent")
```

## Submitting to the Arena

### 1. Create Submission Package

Your submission must include:

- `agent.py` - Your agent implementation
- `requirements.txt` - Python dependencies
- (Optional) `model.pkl` or `model.zip` - Trained model weights

**agent.py** structure:

```python
class Agent:
    def __init__(self):
        """Initialize your agent (load models, etc.)"""
        pass
    
    def get_action(self, observation):
        """
        Required method: Return action based on observation
        
        Args:
            observation: Current game state
            
        Returns:
            int: Action to take
        """
        # Your agent logic here
        return action
```

**requirements.txt** example:

```
numpy>=1.24.0
stable-baselines3>=2.0.0
torch>=2.0.0
```

### 2. Package Your Submission

```bash
# Create a zip file with your agent
zip -r my_agent.zip agent.py requirements.txt model.pkl
```

### 3. Submit via Web Interface

1. Go to [RL-Arena Web](http://localhost:5173) (or production URL)
2. Log in or create an account
3. Navigate to the competition page (e.g., "Pong Competition")
4. Click "Submit Agent"
5. Upload your `my_agent.zip` file
6. Wait for the build to complete

### 4. Monitor Build Status

Your submission goes through several stages:

1. **Queued** - Waiting for build
2. **Building** - Creating Docker image
3. **Build Success** - Ready for matches
4. **Build Failed** - Check logs for errors

## Watching Matches

### 1. View Your Agent's Matches

Once your agent is built, it automatically enters the matchmaking queue:

- **Auto-Matchmaking**: Every 30 seconds, the system matches agents with similar ELO ratings
- **Rate Limiting**: 5-minute cooldown between matches, max 100 matches/day
- **ELO Updates**: Rankings updated after each match

### 2. Watch Replays

1. Go to your agent's page
2. Click on any completed match
3. View replay in:
   - **HTML format**: Interactive visualization (default)
   - **JSON format**: Frame-by-frame data for analysis

### 3. Download Replays

```bash
# Download HTML replay
curl -O http://localhost:8080/api/v1/matches/{match-id}/replay?format=html

# Download JSON replay
curl -O http://localhost:8080/api/v1/matches/{match-id}/replay?format=json
```

## Next Steps

Now that you have your first agent running:

- 📊 **Check the [Leaderboard](https://github.com/rl-arena/rl-arena-docs/blob/main/LEADERBOARD.md)** - See how your agent ranks
- 🏗️ **Learn the [Architecture](https://github.com/rl-arena/rl-arena-docs/blob/main/ARCHITECTURE.md)** - Understand how the platform works
- 🎮 **Try Other [Environments](https://github.com/rl-arena/rl-arena-docs/blob/main/ENVIRONMENTS.md)** - Compete in different games
- 🤝 **Read the [Contributing Guide](https://github.com/rl-arena/rl-arena-docs/blob/main/CONTRIBUTING.md)** - Help build new features

## Troubleshooting

### Build Fails

**Problem**: Your agent build fails with import errors

**Solution**: Ensure all dependencies are in `requirements.txt` with correct versions

### No Matches

**Problem**: Your agent isn't getting matched

**Possible causes**:
- Still in 5-minute cooldown period
- Reached daily match limit (100 matches/day)
- No other agents in similar ELO range

**Check status**:
```bash
curl http://localhost:8080/api/v1/agents/{agent-id}/stats
```

### Timeout Errors

**Problem**: Agent takes too long to respond

**Solution**: Each action must return within 5 seconds. Optimize your inference code.

## Getting Help

- 📖 **Documentation**: [rl-arena-docs](https://github.com/rl-arena/rl-arena-docs)
- 💬 **Discussions**: [GitHub Discussions](https://github.com/rl-arena/rl-arena/discussions)
- 🐛 **Issues**: [Report bugs](https://github.com/rl-arena/rl-arena-backend/issues)

---

Happy training! 🚀
