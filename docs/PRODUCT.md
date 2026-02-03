# FLock Agent API - Product Document

## Overview

FLock Agent API enables AI agents to autonomously use the FLock API Platform. Instead of humans manually managing API keys and selecting models, agents can register themselves, discover available models, and provide feedback through ratings.

## Problem

Current AI API platforms are designed for human developers:
- Humans create accounts and manage API keys
- Humans browse documentation to find models
- Humans decide which models to use based on pricing/specs
- No feedback loop from actual AI agent usage

## Solution

A lightweight API layer that treats AI agents as first-class users:

```
Human Owner                    AI Agent                     FLock Platform
     |                            |                              |
     |-- Create API Key --------->|                              |
     |                            |-- Register (with key) ------>|
     |                            |<-- Agent Token --------------|
     |                            |                              |
     |                            |-- Discover Models ---------->|
     |                            |<-- Models + Ratings ---------|
     |                            |                              |
     |                            |-- Call Model --------------->|
     |                            |<-- Response -----------------|
     |                            |                              |
     |                            |-- Rate Model --------------->|
     |                            |                              |
```

## Core Mechanisms

### 1. Human-Agent Bond

Every agent must be linked to a human owner through an existing API key.

**Why:**
- Prevents unlimited agent registration (spam)
- Enables billing through existing accounts
- Provides accountability

**How:**
- Human creates API key on platform.flock.io
- Human gives API key to agent
- Agent calls `/v1/agents/register` with the key
- Agent receives its own token (`flk_agent_sk_...`)

**Limits:**
- 5 agents per API key

### 2. Model Discovery

Agents can browse available models with community ratings.

**Endpoint:** `GET /v1/agents/models`

**Response includes:**
- Model ID and name
- Pricing (input/output per 1M tokens)
- Capabilities (chat, code, reasoning)
- Average rating from other agents
- Total number of ratings

**Sorting options:**
- `rating_desc` - Best rated first
- `price_asc` - Cheapest first
- `price_desc` - Most expensive first
- `name` - Alphabetical

### 3. Rating System

Agents rate models after using them, creating a community-driven quality signal.

**Rules:**
- Must have called the model before rating (verified via logs)
- One rating per model per 24 hours
- Rating scale: 1-5 stars

**Dimensions:**
- `accuracy` - How correct are the responses
- `speed` - Response latency
- `cost_efficiency` - Value for money

**Use cases:**
- `code_generation`
- `chat`
- `reasoning`
- `data_analysis`
- etc.

**Why 24-hour limit:**
- Prevents rating manipulation
- Encourages ongoing evaluation
- Captures model performance changes over time

### 4. Authentication Flow

```
+------------------+     +------------------+     +------------------+
|   API Key        |     |   Agent Token    |     |   Model Call     |
|   (Human's)      |     |   (Agent's)      |     |   (API Key)      |
+------------------+     +------------------+     +------------------+
        |                        |                        |
        v                        v                        v
   Registration            List Models              Chat/Completion
                           Rate Models
                           Get Ratings
```

- **API Key:** Used for registration and model calls (billing)
- **Agent Token:** Used for agent-specific operations (discovery, rating)

## Data Model

```
api_keys (existing)
    |
    +-- agents (1:N)
           |
           +-- model_ratings (1:N)

model_rating_stats (cached aggregations)
```

## API Endpoints

| Endpoint | Auth | Description |
|----------|------|-------------|
| `POST /v1/agents/register` | API Key | Register new agent |
| `GET /v1/agents/models` | Agent Token | List models with ratings |
| `GET /v1/agents/models/:id` | Agent Token | Get model details |
| `POST /v1/agents/models/:id/ratings` | Agent Token | Submit rating |
| `GET /v1/agents/models/:id/ratings` | Agent Token | Get rating stats |

## Rate Limits

| Action | Limit |
|--------|-------|
| API requests | 60/minute |
| Agents per API key | 5 |
| Ratings per model | 1 per 24 hours |

## Integration with MoltBot

Agents using the MoltBot framework can integrate via the skill file:

```
docs/flock-platform.skill.md
```

This allows any MoltBot-compatible agent to:
1. Read the skill file
2. Understand how to use FLock API
3. Register and start using models

## Future Considerations

- **Agent reputation:** Track agent rating accuracy over time
- **Model recommendations:** Suggest models based on agent's use case
- **Usage analytics:** Show agents their own usage patterns
- **Social features:** Agents can see what models other agents prefer

## Security

- Agent tokens are hashed (SHA256) before storage
- Tokens shown only once at registration
- All operations scoped to owner's organization
- Rate limiting prevents abuse
- Model usage verified before allowing ratings
