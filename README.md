# Production AI Agent with Amazon Bedrock AgentCore

A **complete, production-ready AI agent** deployed on AWS with persistent memory, automatic scaling, and enterprise-grade observability. Built following AWS's official tutorial with **Claude Sonnet 4.5**.

---

## Table of Contents

1. [What We Built Today](#what-we-built-today)
2. [Understanding Amazon Bedrock AgentCore](#understanding-amazon-bedrock-agentcore)
3. [Architecture Deep Dive](#architecture-deep-dive)
4. [Memory System Explained](#memory-system-explained)
5. [The Lazy Loading Pattern](#the-lazy-loading-pattern)
6. [Session Management](#session-management)
7. [How Everything Works Together](#how-everything-works-together)
8. [Setup and Deployment](#setup-and-deployment)
9. [Testing Memory Features](#testing-memory-features)
10. [Project Structure](#project-structure)
11. [Next Steps](#next-steps)
12. [Resources](#resources)

---

## What We Built Today

We successfully implemented a **production-grade AI agent** that demonstrates the complete journey from local development to cloud deployment:

### ✅ Accomplishments

1. **Agent Deployment**
   - Deployed to Amazon Bedrock AgentCore Runtime
   - Using Claude Sonnet 4.5 (latest Anthropic model)
   - Automatic scaling and managed infrastructure
   - No Docker required (direct code deploy)

2. **Memory Integration** 
   - **Short-term Memory (STM)**: Maintains conversation history within sessions
   - **Long-term Memory (LTM)**: Persists facts and preferences across sessions
   - Memory survives container restarts
   - Cross-session fact retrieval working

3. **Tools and Capabilities**
   - Calculator tool for mathematical operations
   - Memory-aware conversation handling
   - Actor-based identity management

4. **Enterprise Features**
   - CloudWatch logging
   - X-Ray distributed tracing
   - GenAI Observability Dashboard
   - IAM role-based security

---

## Understanding Amazon Bedrock AgentCore

### What is AgentCore?

Amazon Bedrock AgentCore is an **enterprise-grade framework** for building, deploying, and operating generative AI agents securely at scale. Think of it as "AWS Lambda for AI agents" but with specialized features for agentic workloads.

### Why AgentCore Exists

Traditional deployment of AI agents faces several challenges:
- **Infrastructure complexity**: Containerization, scaling, load balancing
- **Memory management**: Agents need to remember across conversations
- **Security**: Executing arbitrary code safely
- **Observability**: Understanding agent reasoning and performance
- **Cost**: Efficient resource utilization for long-running conversations

AgentCore solves these by providing **managed services** for each concern.

### AgentCore Services (Independent Components)

AgentCore is modular - use what you need:

| Service | Purpose | When to Use | Our Implementation |
|---------|---------|-------------|-------------------|
| **Runtime** | Managed hosting for agents | Deploy any agent | ✅ Implemented |
| **Memory** | Persistent conversation storage | Need history or preferences | ✅ Implemented |
| **Gateway** | Host MCP tools as APIs | Connect to business systems | ⬜ Not used |
| **Code Interpreter** | Secure Python execution | Data analysis, calculations | ⬜ Next step |
| **Browser Tool** | Cloud-based web browsing | Web scraping, testing | ⬜ Not used |
| **Identity** | User authentication | Multi-user agents | ⬜ Not used |
| **Observability** | Monitoring and debugging | Production visibility | ✅ Implemented |

### Three Layers of Access

AgentCore provides three levels of abstraction:

```
┌─────────────────────────────────────────────────┐
│  Layer 3: AgentCore Starter Toolkit (CLI)       │
│  • agentcore configure                          │
│  • agentcore launch                             │
│  • agentcore invoke                             │
│  → Highest level, easiest to use               │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│  Layer 2: AgentCore SDK (Python)                │
│  • BedrockAgentCoreApp                          │
│  • AgentCoreMemoryConfig                        │
│  • AgentCoreMemorySessionManager                │
│  → Programmatic, framework integrations         │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│  Layer 1: Control & Data Plane APIs (boto3)     │
│  • CreateAgentRuntime                           │
│  • InvokeAgentRuntime                           │
│  • CreateMemory                                 │
│  → Lowest level, maximum control                │
└─────────────────────────────────────────────────┘
```

**We used all three:**
- **CLI** for deployment and testing
- **SDK** for memory integration in code
- **APIs** indirectly through SDK calls

---

## Architecture Deep Dive

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     AWS Cloud                                │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │   Amazon Bedrock AgentCore Runtime                     │ │
│  │                                                        │ │
│  │   ┌──────────────────────────────────────────────┐    │ │
│  │   │  Container Instance                          │    │ │
│  │   │  (Pinned to Session ID)                      │    │ │
│  │   │                                              │    │ │
│  │   │  ┌────────────────────────────────────────┐  │    │ │
│  │   │  │  tutorial_agent.py                     │  │    │ │
│  │   │  │                                        │  │    │ │
│  │   │  │  • Claude Sonnet 4.5                  │  │    │ │
│  │   │  │  • Calculator Tool                    │  │    │ │
│  │   │  │  • Memory Session Manager             │  │    │ │
│  │   │  │  • Lazy Loaded Agent Instance         │  │    │ │
│  │   │  │                                        │  │    │ │
│  │   │  │  Python 3.12 Runtime                  │  │    │ │
│  │   │  └────────────────────────────────────────┘  │    │ │
│  │   │                                              │    │ │
│  │   │  Lifecycle:                                  │    │ │
│  │   │  • Up to 8 hours runtime                    │    │ │
│  │   │  • 15 min idle timeout                      │    │ │
│  │   │  • Dedicated per session                    │    │ │
│  │   └──────────────────────────────────────────────┘    │ │
│  │                                                        │ │
│  │   Features:                                            │ │
│  │   • Automatic scaling                                  │ │
│  │   • Session isolation (microVMs)                       │ │
│  │   • Streaming support                                  │ │
│  │   • Multiple invocations per session                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                            │                                 │
│                            ↓                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │   Amazon Bedrock AgentCore Memory                      │ │
│  │                                                        │ │
│  │   Memory ID: agent_deployment_tutorial_agent_mem-...  │ │
│  │                                                        │ │
│  │   ┌──────────────────┐  ┌──────────────────────────┐  │ │
│  │   │ Short-Term       │  │ Long-Term Memory (LTM)   │  │ │
│  │   │ Memory (STM)     │  │                          │  │ │
│  │   │                  │  │ Extraction Strategies:   │  │ │
│  │   │ • Conversation   │  │ • User facts             │  │ │
│  │   │   history        │  │ • Preferences            │  │ │
│  │   │ • Session-scoped │  │ • Session summaries      │  │ │
│  │   │ • Immediate      │  │                          │  │ │
│  │   │   recall         │  │ Cross-session retrieval  │  │ │
│  │   └──────────────────┘  └──────────────────────────┘  │ │
│  │                                                        │ │
│  │   Storage:                                             │ │
│  │   • 30-day retention                                   │ │
│  │   • Actor-based isolation (/users/{actor_id}/...)     │ │
│  │   • Managed scaling                                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                            │                                 │
│                            ↓                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │   Amazon Bedrock (Claude Sonnet 4.5)                   │ │
│  │                                                        │ │
│  │   Inference Profile:                                   │ │
│  │   us.anthropic.claude-sonnet-4-5-20250929-v1:0        │ │
│  │                                                        │ │
│  │   • Cross-region routing                               │ │
│  │   • Automatic failover                                 │ │
│  │   • Optimized latency                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │   Observability (CloudWatch + X-Ray)                   │ │
│  │                                                        │ │
│  │   • CloudWatch Logs                                    │ │
│  │   • X-Ray Distributed Tracing                          │ │
│  │   • GenAI Observability Dashboard                      │ │
│  │   • Service Maps                                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **User invokes agent** via CLI or boto3
2. **AgentCore Runtime** receives request with session ID
3. **Container instance** is retrieved or spawned (pinned to session)
4. **Lazy loading**: Agent checks if instance exists in container
5. **Memory retrieval**: Session manager fetches conversation history + relevant facts
6. **Model invocation**: Claude processes prompt with memory context
7. **Tool execution**: Calculator runs if needed
8. **Memory storage**: Response and facts are stored
9. **Response returned** to user
10. **Observability**: Logs, traces, and metrics captured

---

## Memory System Explained

### The Memory Problem

Traditional stateless AI agents face a challenge:

```
Conversation 1 (Session A):
User: "My favorite color is blue"
Agent: "Got it! I'll remember that."

Conversation 2 (Session B - next day):
User: "What's my favorite color?"
Agent: "I don't know your favorite color."  ❌

Why? No memory persistence between sessions!
```

### AgentCore Memory Solution

AgentCore Memory provides **two complementary types** of memory:

#### 1. Short-Term Memory (STM)

**Purpose**: Maintain conversation flow within a session

**How it works:**
```python
# Session A - Message 1
User: "My favorite number is 42"
# STM stores: conversation_history = [user_msg, agent_response]

# Session A - Message 2 (same session)
User: "What's my favorite number times 5?"
# STM provides: Previous conversation context
# Agent: "210!" ✅ (remembers 42 from STM)
```

**Key characteristics:**
- **Scope**: Session-bound
- **Duration**: While session is active (up to 8 hours)
- **Storage**: Exact conversation transcript
- **Access**: Immediate, sequential
- **Use case**: Maintaining context in ongoing conversations

#### 2. Long-Term Memory (LTM)

**Purpose**: Remember facts and preferences across different conversations

**How it works:**
```python
# Session A (Monday)
User: "I love Earl Grey tea with milk and sugar"
# LTM extracts: 
#   - Fact: "User loves Earl Grey tea"
#   - Preference: "Prefers tea with milk and one sugar"
# Stored at: /users/user123/preferences/tea

# Wait 20-30 seconds for extraction processing...

# Session B (Thursday - completely different session)
User: "What tea do I like?"
# LTM retrieves from: /users/user123/preferences/tea
# Agent: "You love Earl Grey with milk and one sugar" ✅
```

**Key characteristics:**
- **Scope**: Actor-bound (across all sessions)
- **Duration**: 30 days retention
- **Storage**: Extracted facts, not full conversations
- **Access**: Semantic search, relevance-based
- **Processing**: 20-30 seconds extraction delay
- **Use case**: User preferences, personal facts, learned information

### Memory Architecture in Our Agent

#### Storage Hierarchy

```
AgentCore Memory
└── /users/
    └── {actor_id}/              # e.g., "user123"
        ├── /facts/              # General facts about user
        │   ├── fact_1.json
        │   ├── fact_2.json
        │   └── ...
        └── /preferences/        # User preferences
            ├── preference_1.json
            ├── preference_2.json
            └── ...
```

#### Retrieval Configuration

In our code (`tutorial_agent.py`):

```python
retrieval_config={
    f"/users/{actor_id}/facts": RetrievalConfig(
        top_k=3,              # Retrieve top 3 most relevant facts
        relevance_score=0.5   # Minimum similarity threshold
    ),
    f"/users/{actor_id}/preferences": RetrievalConfig(
        top_k=3,
        relevance_score=0.5
    )
}
```

**What this means:**
- When agent is invoked, it searches `/users/user123/facts` and `/users/user123/preferences`
- Returns top 3 most relevant items from each location
- Only includes items with relevance score ≥ 0.5
- This context is provided to Claude along with the prompt

### Memory SDK Integration

Our agent uses Strands-specific memory integrations:

```python
from bedrock_agentcore.memory.integrations.strands.config import (
    AgentCoreMemoryConfig,    # Configures memory connection
    RetrievalConfig            # Defines retrieval parameters
)
from bedrock_agentcore.memory.integrations.strands.session_manager import (
    AgentCoreMemorySessionManager  # Manages memory operations
)
```

**Why Strands-specific?**
- Strands Agents has a `session_manager` interface
- AgentCore provides an implementation of this interface
- Works seamlessly - Strands Agent doesn't know it's using AgentCore Memory
- **Other frameworks**: Similar integrations exist for LangChain, LlamaIndex, CrewAI

### Memory Environment Variables

```python
MEMORY_ID = os.getenv("BEDROCK_AGENTCORE_MEMORY_ID")
```

**How this gets set:**
1. During `agentcore configure`, you enable memory
2. CLI creates a Memory resource in AWS
3. Returns a Memory ID like: `agent_deployment_tutorial_agent_mem-9NdUUl2KoG`
4. CLI automatically sets this as an environment variable in your Runtime container
5. Your code reads it at runtime

**Why environment variable?**
- Different per deployment
- No hardcoding in source code
- Managed by AgentCore infrastructure
- Secure (scoped to your Runtime)

---

## The Lazy Loading Pattern

### The Problem We're Solving

AgentCore Runtime has a unique challenge:

```
Problem Timeline:

1. Container starts
   └─ Python module loads
      └─ Need to create Agent instance
         └─ BUT: Don't have session_id or actor_id yet!

2. First invocation arrives
   └─ NOW we have session_id and actor_id
      └─ But too late - module already loaded!

Question: How do we create an Agent with memory 
          when we don't know the session until invocation?
```

### The Solution: Lazy Loading

**Lazy loading** means "create on first use, reuse thereafter."

#### Our Implementation

```python
# Global variable - survives across invocations in same container
_agent = None

def get_or_create_agent(actor_id: str, session_id: str) -> Agent:
    """
    Get existing agent or create new one with memory configuration.
    Since the container is pinned to the session ID, 
    we only need one agent per container.
    """
    global _agent
    
    if _agent is None:
        # FIRST INVOCATION: Create agent with memory
        memory_config = AgentCoreMemoryConfig(
            memory_id=MEMORY_ID,
            session_id=session_id,      # Now we have it!
            actor_id=actor_id,           # Now we have it!
            retrieval_config={...}
        )
        
        _agent = Agent(
            model=MODEL_ID,
            session_manager=AgentCoreMemorySessionManager(memory_config, REGION),
            system_prompt="...",
            tools=[calculator]
        )
    
    # SUBSEQUENT INVOCATIONS: Return existing agent
    return _agent

@app.entrypoint
def invoke(payload, context):
    """Called on EVERY invocation"""
    # Extract session and actor from invocation context
    actor_id = context.request_headers.get('X-Amzn-Bedrock-AgentCore-Runtime-Custom-Actor-Id', 'user')
    session_id = context.session_id or 'default_session'
    
    # Get or create agent
    agent = get_or_create_agent(actor_id, session_id)
    
    # Use agent
    result = agent(prompt)
    return {"response": ...}
```

### Execution Flow

#### First Invocation in Container

```
Invocation 1:
├─ Container starts (fresh)
├─ Python loads tutorial_agent.py
│  └─ _agent = None (global variable initialized)
├─ invoke() called with session_id="session-A", actor_id="user123"
├─ get_or_create_agent("user123", "session-A")
│  ├─ Check: _agent is None? YES
│  ├─ Create AgentCoreMemoryConfig with session="session-A", actor="user123"
│  ├─ Create Agent with memory
│  └─ Set _agent = <new agent instance>
├─ agent("What is 2+2?")
└─ Return response
```

#### Second Invocation in Same Container

```
Invocation 2:
├─ Same container (still alive)
├─ invoke() called with session_id="session-A", actor_id="user123"
├─ get_or_create_agent("user123", "session-A")
│  ├─ Check: _agent is None? NO (already exists)
│  └─ Return existing _agent
├─ agent("What's my favorite number?")  
│  └─ Has memory from Invocation 1! ✅
└─ Return response with context
```

### Why This Works

**Key insight:** AgentCore Runtime containers are **pinned to session IDs**

```
Session A → Container 1 → _agent instance 1
Session B → Container 2 → _agent instance 2
Session C → Container 3 → _agent instance 3

Each container handles ONE session (for up to 8 hours).
Therefore, ONE agent instance per container is correct!
```

**Benefits:**
1. **Performance**: Don't recreate agent on every invocation
2. **Memory continuity**: Session manager maintains conversation state
3. **Simple**: One agent per session, one session per container
4. **Safe**: Sessions can't interfere (different containers)

### What If Container Restarts?

```
Scenario:
├─ Container dies (timeout, crash, deployment)
├─ New container spawns for next invocation
├─ _agent = None (fresh global variable)
├─ Lazy loading creates new agent instance
└─ Memory? Still works! ✅
    └─ AgentCoreMemorySessionManager reconnects to persistent memory
    └─ Conversation history retrieved from Memory service
```

**This is why Memory is crucial**: It persists beyond container lifecycle.

---

## Session Management

### What is a Session?

A **session** represents a continuous conversation between a user and the agent.

```
Session A (Morning):
  - "Hello"
  - "What's 2+2?"
  - "Thanks!"

Session B (Afternoon):
  - "What's the weather?"
  - "Set a reminder"
```

### Session IDs

**Format requirements:**
- Minimum 33 characters
- Alphanumeric and hyphens
- Example: `my-session-12345-12345-12345-12345-67890-A`

**Why 33+ characters?**
- Security: Prevents session hijacking
- Uniqueness: Ensures no collisions
- Container isolation: Used as key for spawning dedicated microVMs

### Actor IDs

**What is an Actor?**
An actor represents the **user** or **entity** interacting with the agent.

```
Actor: user123
├─ Session A: Morning conversation
├─ Session B: Afternoon conversation
└─ Session C: Evening conversation

All three sessions belong to the same actor (user123).
LTM is stored per actor, shared across all their sessions.
```

**How we pass Actor ID:**

```bash
# Via custom header
uv run agentcore invoke '{"prompt": "..."}' \
  --headers "Actor-Id:user123"

# In code:
actor_id = context.request_headers.get(
    'X-Amzn-Bedrock-AgentCore-Runtime-Custom-Actor-Id', 
    'user'  # default if not provided
)
```

### Session Lifecycle

```
Timeline of a Session:

T+0:00    First invocation
          ├─ Container spawns
          ├─ Agent instance created (lazy loading)
          └─ Response returned

T+0:10    Second invocation (same session)
          ├─ Same container (still alive)
          ├─ Agent instance reused
          └─ Has memory from first invocation

T+8:00    Maximum runtime reached
          └─ Container terminates

OR

T+0:25    15 minutes of inactivity
          └─ Container terminates (idle timeout)
```

**Container Pinning:**
```
Session ID → Hashed → Container Selection

Same session ID → Always routes to same container (if alive)
Different session ID → Different container
```

This ensures:
- Conversation continuity within sessions
- Efficient resource utilization
- Session isolation (security)

---

## How Everything Works Together

### Complete Request Flow

Let's trace a complete request through the system:

```
1. User Invokes Agent
   ↓
   $ uv run agentcore invoke '{"prompt": "What is 2+2?"}' \
       --session-id my-session-...-A \
       --headers "Actor-Id:user123"

2. CLI sends request to AWS
   ↓
   boto3.client('bedrock-agentcore').invoke_agent_runtime(
       agentRuntimeArn="arn:aws:...",
       runtimeSessionId="my-session-...-A",
       payload='{"prompt": "What is 2+2?"}',
       headers={'X-Amzn-Bedrock-AgentCore-Runtime-Custom-Actor-Id': 'user123'}
   )

3. AgentCore Runtime receives request
   ↓
   • Hashes session_id
   • Routes to existing container OR spawns new one
   • Container is pinned to this session

4. Container executes tutorial_agent.py
   ↓
   @app.entrypoint
   def invoke(payload, context):
       # Extract parameters
       actor_id = "user123"  (from headers)
       session_id = "my-session-...-A"  (from context)
       prompt = "What is 2+2?"

5. Lazy Loading Check
   ↓
   agent = get_or_create_agent("user123", "my-session-...-A")
   
   if _agent is None:  # First invocation
       ├─ Create AgentCoreMemoryConfig
       │  ├─ memory_id: From environment
       │  ├─ session_id: "my-session-...-A"
       │  ├─ actor_id: "user123"
       │  └─ retrieval_config: Facts + Preferences paths
       │
       ├─ Create AgentCoreMemorySessionManager
       │  └─ Connects to Memory service
       │
       └─ Create Agent with session manager
   
   else:  # Subsequent invocations
       └─ Return existing agent (already connected to memory)

6. Memory Retrieval
   ↓
   AgentCoreMemorySessionManager:
   ├─ Fetch STM: Previous messages in this session
   ├─ Fetch LTM: Search /users/user123/facts (top_k=3, score≥0.5)
   ├─ Fetch LTM: Search /users/user123/preferences (top_k=3, score≥0.5)
   └─ Build conversation history

7. Agent Processes Request
   ↓
   agent("What is 2+2?")
   ├─ Strands Agent receives prompt
   ├─ Session manager provides: 
   │  ├─ Previous messages from STM
   │  └─ Relevant facts from LTM
   ├─ Builds context for Claude
   └─ Determines if tool needed

8. Claude Invocation
   ↓
   POST to Bedrock API:
   ├─ Model: us.anthropic.claude-sonnet-4-5-20250929-v1:0
   ├─ Messages: [conversation_history + new_prompt]
   ├─ Tools: [calculator definition]
   └─ System prompt: "You are a helpful assistant..."

9. Tool Execution (if needed)
   ↓
   Claude decides: "Need calculator for 2+2"
   ├─ Returns tool_use block
   ├─ Strands executes: calculator(expression="2+2")
   ├─ Result: 4
   └─ Feeds back to Claude

10. Claude Generates Response
    ↓
    "The answer is 4"

11. Memory Storage
    ↓
    AgentCoreMemorySessionManager:
    ├─ Store in STM: [user_message, agent_response]
    ├─ Queue for LTM: Extract facts (if any)
    └─ LTM processing: Background job (20-30s delay)

12. Response Returned
    ↓
    return {
        "response": "The answer is 4"
    }

13. Observability Captured
    ↓
    ├─ CloudWatch Logs: All log statements
    ├─ X-Ray Trace: Service map with timing
    └─ GenAI Dashboard: Model costs, latency, errors

14. CLI Displays Response
    ↓
    Response:
    The answer is 4
```

### Memory Persistence Across Container Restarts

```
Scenario: Container dies and restarts

Container 1 (Initial):
├─ Invocation 1: "My favorite color is blue"
├─ Memory stores: STM + LTM extraction queued
└─ Container dies (timeout)

Container 2 (New):
├─ Invocation 2: "What's my favorite color?"
├─ Lazy loading creates new agent instance
├─ AgentCoreMemorySessionManager connects to Memory
│  ├─ Retrieves STM for session
│  └─ Searches LTM for facts about actor
└─ Response: "Your favorite color is blue" ✅

KEY: Memory persists in AgentCore Memory service,
     not in the container!
```

---

## Setup and Deployment

### Prerequisites

1. **AWS Account** with billing enabled
2. **AWS CLI** configured with credentials
3. **uv** package manager installed
4. **Python 3.12** (managed by uv)
5. **IAM Permissions**: AdministratorAccess (or specific AgentCore policies)

### Model Access

Claude Sonnet 4.5 requires an **inference profile** (not direct model access):
- Model ID: `anthropic.claude-sonnet-4-5-20250929-v1:0` ❌ (doesn't work)
- Inference Profile: `us.anthropic.claude-sonnet-4-5-20250929-v1:0` ✅ (correct)

**No manual model enablement needed** as of late 2024 - models are available by default.

### Step-by-Step Setup

#### 1. Create IAM User for Development

```bash
# In AWS Console:
# IAM → Users → Create user
# - Username: your-name-prototyping
# - Permissions: AdministratorAccess (for prototyping)
# - Create access keys for CLI

# Configure AWS CLI
aws configure
# Enter: Access Key ID, Secret Access Key
# Region: us-west-2
# Output: json

# Verify
aws sts get-caller-identity
```

**Important**: Remove any permissions boundaries that restrict IAM operations.

#### 2. Initialize Project

```bash
# Create project directory
mkdir agentcore-tutorial && cd agentcore-tutorial

# Initialize main project (for CLI tools)
uv init --no-workspace
uv add bedrock-agentcore-starter-toolkit

# Create deployment directory (for agent code)
mkdir agent_deployment
uv init --bare ./agent_deployment
uv --directory ./agent_deployment add \
    strands-agents \
    bedrock-agentcore \
    strands-agents-tools
```

**Why two directories?**
- `./` (root): Development tools (CLI, testing)
- `./agent_deployment/`: Agent runtime code only (gets deployed)
- Keeps deployed container lean and secure

#### 3. Create Agent Code

Create `agent_deployment/tutorial_agent.py`:

```python
"""
Production-Ready AI Agent with Memory
Remembers conversations and user preferences across sessions
"""
import os
from strands import Agent
from strands_tools import calculator
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from bedrock_agentcore.memory.integrations.strands.config import (
    AgentCoreMemoryConfig, 
    RetrievalConfig
)
from bedrock_agentcore.memory.integrations.strands.session_manager import (
    AgentCoreMemorySessionManager
)

app = BedrockAgentCoreApp()

MEMORY_ID = os.getenv("BEDROCK_AGENTCORE_MEMORY_ID")
REGION = os.getenv("AWS_REGION", "us-west-2")
MODEL_ID = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"

# Global agent instance
_agent = None

def get_or_create_agent(actor_id: str, session_id: str) -> Agent:
    """
    Get existing agent or create new one with memory configuration.
    Since the container is pinned to the session ID, 
    we only need one agent per container.
    """
    global _agent
    
    if _agent is None:
        # Configure memory with retrieval for user facts and preferences
        memory_config = AgentCoreMemoryConfig(
            memory_id=MEMORY_ID,
            session_id=session_id,
            actor_id=actor_id,
            retrieval_config={
                f"/users/{actor_id}/facts": RetrievalConfig(
                    top_k=3, 
                    relevance_score=0.5
                ),
                f"/users/{actor_id}/preferences": RetrievalConfig(
                    top_k=3, 
                    relevance_score=0.5
                )
            }
        )
        
        # Create agent with memory session manager
        _agent = Agent(
            model=MODEL_ID,
            session_manager=AgentCoreMemorySessionManager(memory_config, REGION),
            system_prompt="You are a helpful assistant with memory. Remember user preferences and facts across conversations. Use the calculate tool for math problems.",
            tools=[calculator]
        )
    
    return _agent

@app.entrypoint
def invoke(payload, context):
    """AgentCore Runtime entry point with lazy-loaded agent"""
    if not MEMORY_ID:
        return {
            "error": "Memory not configured. Set BEDROCK_AGENTCORE_MEMORY_ID environment variable."
        }
    
    # Extract session and actor information
    actor_id = context.request_headers.get(
        'X-Amzn-Bedrock-AgentCore-Runtime-Custom-Actor-Id', 
        'user'
    ) if context.request_headers else 'user'
    session_id = context.session_id or 'default_session'
    
    # Get or create agent (lazy loading)
    agent = get_or_create_agent(actor_id, session_id)
    
    prompt = payload.get("prompt", "Hello!")
    result = agent(prompt)
    
    return {
        "response": result.message.get('content', [{}])[0].get('text', str(result))
    }

if __name__ == "__main__":
    app.run()
```

#### 4. Configure Agent

```bash
uv run agentcore configure -e agent_deployment/tutorial_agent.py
```

**Prompts and answers:**
- Agent name: Press Enter (accepts default)
- Dependency file: Select `agent_deployment/pyproject.toml`
- Deployment type: 1 (Direct Code Deploy)
- Python version: 3 (Python 3.12)
- Execution role: Press Enter (auto-create)
- S3 bucket: Press Enter (auto-create)
- OAuth: `no`
- Request headers: `no`
- **Memory**: Press Enter (create new memory)
- **Long-term memory**: `yes` ✅ (enables LTM extraction)

This creates `.bedrock_agentcore.yaml` with all configuration.

#### 5. Deploy to AWS

```bash
uv run agentcore launch
```

**What happens:**
1. Creates IAM execution role
2. Provisions Memory (STM+LTM) - takes ~3 minutes
3. Creates S3 bucket for deployment artifacts
4. Packages agent code and dependencies
5. Deploys to AgentCore Runtime
6. Enables CloudWatch logging and X-Ray tracing
7. Returns agent ARN and status

**Wait for**: "Memory: STM+LTM (3 strategies)" to show as ACTIVE (green)

#### 6. Verify Deployment

```bash
uv run agentcore status
```

Output shows:
- Agent ARN
- Endpoint status: READY
- Memory: STM+LTM (3 strategies) - ACTIVE
- Region and account
- CloudWatch log groups
- GenAI Observability Dashboard URL

---

## Testing Memory Features

### Test 1: Basic Invocation

```bash
uv run agentcore invoke '{"prompt": "What is 25 * 4 + 10?"}'
```

**Expected**: "The answer is 110"

**What happened:**
- Agent used calculator tool
- Claude processed the request
- Response returned

### Test 2: Short-Term Memory (Same Session)

```bash
# First message
uv run agentcore invoke '{"prompt": "My favorite number is 42"}' \
  --session-id memory-test-12345-12345-12345-12345-A

# Second message (same session)
uv run agentcore invoke '{"prompt": "What is my favorite number multiplied by 5?"}' \
  --session-id memory-test-12345-12345-12345-12345-A
```

**Expected:** 
- Second response: "210" (42 × 5)
- Agent remembers favorite number from first message

**What happened:**
- STM stored first conversation
- Second invocation retrieved STM
- Agent had context from previous message

### Test 3: Long-Term Memory (Cross-Session)

```bash
# Session A: Store preference
uv run agentcore invoke \
  '{"prompt": "Remember that I absolutely love hot tea, especially Earl Grey, and I prefer it with a splash of milk and one sugar"}' \
  --session-id ltm-test-12345-12345-12345-12345-A \
  --headers "Actor-Id:user123"

# Wait for LTM extraction (20 seconds)
sleep 20

# Session B: Retrieve preference (different session!)
uv run agentcore invoke \
  '{"prompt": "What kind of hot drinks do I like and how do I prefer them prepared?"}' \
  --session-id ltm-test-12345-12345-12345-12345-B \
  --headers "Actor-Id:user123"
```

**Expected:**
- Session B response: "You love Earl Grey tea with milk and one sugar"
- Even though it's a completely different session

**What happened:**
- Session A stored conversation in STM
- LTM extraction ran in background (20s delay)
- Facts extracted and stored at `/users/user123/preferences`
- Session B retrieved facts from LTM using semantic search
- Agent had context from previous session

**Key insight:** Different sessions, **same actor**, facts persist!

### Viewing Logs

```bash
# Get log command from status
uv run agentcore status

# Tail logs (replace with your agent ID)
aws logs tail /aws/bedrock-agentcore/runtimes/agent_deployment_tutorial_agent-DJssS8DTtx-DEFAULT \
  --log-stream-name-prefix "2025/11/10/[runtime-logs" \
  --follow
```

### Observability Dashboard

1. Run `uv run agentcore status`
2. Copy "GenAI Observability Dashboard" URL
3. Open in browser
4. View:
   - Service map (agent → memory → claude)
   - Request traces
   - Latency metrics
   - Model costs

---

## Project Structure

```
agentcore-tutorial/
├── agent_deployment/                  # DEPLOYED CODE
│   ├── tutorial_agent.py             # Main agent (memory-enabled)
│   ├── pyproject.toml                # Runtime dependencies
│   └── .python-version               # Python 3.12
│
├── .bedrock_agentcore/               # Build artifacts (gitignored)
│   └── agent_deployment_tutorial_agent/
│       ├── dependencies.zip          # Packaged dependencies
│       ├── deployment.zip            # Complete deployment package
│       └── dependencies.hash         # Cache invalidation
│
├── .bedrock_agentcore.yaml           # AgentCore configuration (gitignored)
│                                     # Contains: ARNs, Memory ID, settings
│
├── pyproject.toml                    # Development dependencies
│                                     # Contains: bedrock-agentcore-starter-toolkit
│
├── .venv/                            # Virtual environment (gitignored)
│
├── initial_goal.txt                  # Original AWS tutorial
├── initial_goal_transcript.txt       # Video transcript
│
├── .gitignore                        # Excludes: .bedrock_agentcore.yaml, .venv, etc.
│
└── README.md                         # This file
```

### Key Files Explained

**`agent_deployment/tutorial_agent.py`**
- Main agent code
- Defines entrypoint
- Memory integration
- Lazy loading implementation
- Gets deployed to AgentCore Runtime

**`.bedrock_agentcore.yaml`** (gitignored)
- Auto-generated during `agentcore configure`
- Contains:
  - Agent name and ARN
  - Memory ID
  - Execution role ARN
  - S3 bucket name
  - Deployment settings
- **Do not commit**: Contains account-specific identifiers

**`agent_deployment/pyproject.toml`**
- Runtime dependencies only
- Gets packaged into deployment zip
- Keep minimal (affects cold start time)

**Root `pyproject.toml`**
- Development tools
- `bedrock-agentcore-starter-toolkit` for CLI
- Not deployed to Runtime

---

## Next Steps

### Option 1: Add Code Interpreter

Replace calculator with secure Python execution:

```python
# Update imports
from strands_tools.code_interpreter import AgentCoreCodeInterpreter

# Update dependencies
uv --directory ./agent_deployment add strands-agents-tools[agent_core_code_interpreter]

# Update agent creation
tools=[AgentCoreCodeInterpreter(region=REGION).code_interpreter]
```

**Benefits:**
- Complex calculations
- Data analysis (NumPy, Pandas)
- Statistical operations
- Visualizations (Matplotlib)
- CSV/JSON processing

**Test:**
```bash
uv run agentcore invoke \
  '{"prompt": "Calculate mean, median, and std dev of [3, 5, 4, 6, 2, 4, 7]"}'
```

### Option 2: Multi-Agent Orchestration

Use Strands' multi-agent features:
- Agent handoffs
- Swarm patterns
- Graph workflows
- Specialized agents (research, writing, coding)

### Option 3: Production Integration

Invoke from your application:

```python
import boto3
import json

client = boto3.client('bedrock-agentcore', region_name='us-west-2')

response = client.invoke_agent_runtime(
    agentRuntimeArn="arn:aws:bedrock-agentcore:...",
    runtimeSessionId="webapp-user123-session-20250110-...",
    payload=json.dumps({"prompt": "Hello!"}).encode('utf-8'),
    requestHeaders={
        'X-Amzn-Bedrock-AgentCore-Runtime-Custom-Actor-Id': 'user123'
    }
)

result = json.loads(response['response'].read())
print(result['response'])
```

### Option 4: Advanced Memory Strategies

Customize memory extraction:
- Custom fact extraction logic
- Different storage hierarchies
- Temporal decay of facts
- User feedback loops

---

## Cleanup

**Remove all AWS resources:**

```bash
uv run agentcore destroy
```

**What gets deleted:**
- AgentCore Runtime deployment
- Memory resources (STM + LTM)
- S3 bucket and artifacts
- IAM execution role
- CloudWatch log groups (optional)

**Estimated cost savings:** ~$5-20/month depending on usage

---

## Resources

### Official Documentation

- [Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore.html)
- [Strands Agents Framework](https://github.com/awslabs/strands-agents)
- [AgentCore Memory Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore-memory.html)
- [AgentCore Runtime Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore-runtime.html)

### Tutorials and Guides

- [Original Tutorial](initial_goal.txt) - Detailed walkthrough
- [Video Transcript](initial_goal_transcript.txt) - Spoken explanation
- [AWS Builder Center](https://aws.amazon.com/builders/) - More tutorials

### Community

- [AWS re:Post](https://repost.aws/) - Q&A forum
- [Strands GitHub Discussions](https://github.com/awslabs/strands-agents/discussions)
- [AWS GenAI Community](https://community.aws/generative-ai)

---

## Troubleshooting

### Common Issues

**1. Model Access Denied (Claude Sonnet 4.5)**
```
Error: ValidationException - Invocation of model ID ... with on-demand 
throughput isn't supported. Retry with inference profile.
```

**Solution:** Use inference profile ID:
```python
MODEL_ID = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"  # ✅ Correct
# NOT: "anthropic.claude-sonnet-4-5-20250929-v1:0"  # ❌ Wrong
```

**2. Memory Not Working**
```
Agent doesn't remember previous conversations
```

**Check:**
- Is memory ACTIVE? Run `uv run agentcore status`
- Did you pass `--session-id` in subsequent calls?
- Did you pass `--headers "Actor-Id:..."` for LTM tests?
- Did you wait 20 seconds after storing for LTM extraction?

**3. IAM Permission Errors**
```
Error: User is not authorized to perform: iam:GetRole
```

**Solution:**
- Attach `AdministratorAccess` policy to IAM user
- Remove any permissions boundaries
- Wait 30 seconds for propagation

**4. Session ID Too Short**
```
Error: Session ID must be at least 33 characters
```

**Solution:**
```bash
# Bad: --session-id abc123
# Good: --session-id my-session-12345-12345-12345-12345-A
```

---

## Key Learnings

### What We Accomplished

1. ✅ Deployed production AI agent to AWS
2. ✅ Integrated persistent memory (STM + LTM)
3. ✅ Implemented lazy loading pattern
4. ✅ Configured cross-session fact retrieval
5. ✅ Set up enterprise observability
6. ✅ Used Claude Sonnet 4.5 (latest model)

### Core Concepts Mastered

- **AgentCore Runtime**: Serverless compute for agents
- **Memory Architecture**: STM vs LTM, storage hierarchy
- **Lazy Loading**: Why and how to defer agent creation
- **Session Management**: Container pinning, lifecycle
- **Actor-based Identity**: Cross-session user tracking
- **Inference Profiles**: Cross-region model routing

### Production-Ready Patterns

- Environment variable configuration
- Separation of dev/runtime dependencies
- Memory connection handling
- Error handling and fallbacks
- Observability integration

---

## License

MIT

## Author

Christopher Wycoff

**Built following AWS's official AgentCore tutorial with hands-on implementation and deep technical analysis.**

---

*Last updated: November 10, 2025*
*AgentCore version: 1.0.5*
*Strands Agents version: 1.15.0*
