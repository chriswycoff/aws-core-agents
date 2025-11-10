# AWS AgentCore Production Agent

Production-ready AI agent deployed on **Amazon Bedrock AgentCore** using **Claude Sonnet 4.5**.

## Features

- ✅ **Deployed Agent**: Running on AWS Bedrock AgentCore Runtime
- ✅ **Claude Sonnet 4.5**: Latest Anthropic model via inference profile
- ✅ **Calculator Tool**: Basic math operations using Strands tools
- ✅ **Memory Infrastructure**: STM + LTM provisioned (integration in progress)
- ✅ **Observability**: CloudWatch logs and X-Ray tracing enabled
- ⏳ **Code Interpreter**: Coming next

## Architecture

```
┌─────────────────────────────────────────┐
│   Amazon Bedrock AgentCore Runtime      │
│   ┌─────────────────────────────────┐   │
│   │  tutorial_agent.py              │   │
│   │  • Claude Sonnet 4.5            │   │
│   │  • Calculator tool              │   │
│   │  • Python 3.12                  │   │
│   └─────────────────────────────────┘   │
│                                         │
│   AgentCore Services:                   │
│   • Memory (STM+LTM) - Provisioned      │
│   • Observability - Active              │
└─────────────────────────────────────────┘
```

## Prerequisites

- AWS account with Bedrock access
- AWS CLI configured
- [uv](https://github.com/astral-sh/uv) package manager
- Python 3.12

## Setup

### 1. Clone and Install

```bash
git clone https://github.com/chriswycoff/aws-core-agents.git
cd aws-core-agents
uv init --no-workspace
uv add bedrock-agentcore-starter-toolkit
```

### 2. Install Agent Dependencies

```bash
cd agent_deployment
uv sync
```

### 3. Configure AWS

```bash
aws configure
# Enter your credentials and set region to us-west-2
```

## Deployment

### Configure Agent

```bash
uv run agentcore configure -e agent_deployment/tutorial_agent.py
```

Follow prompts:
- Agent name: Press Enter (default)
- Dependency file: Select `agent_deployment/pyproject.toml`
- Memory: Yes to enable STM+LTM
- Accept defaults for other options

### Deploy to AWS

```bash
uv run agentcore launch
```

### Check Status

```bash
uv run agentcore status
```

## Usage

### Invoke Agent

```bash
uv run agentcore invoke '{"prompt": "What is 25 * 4 + 10?"}'
```

### With Session ID

```bash
uv run agentcore invoke '{"prompt": "Hello!"}' --session-id my-session-12345-12345-12345-12345-67890-A
```

### View Logs

```bash
# Get log command from status output
uv run agentcore status

# Tail logs
aws logs tail /aws/bedrock-agentcore/runtimes/[AGENT-ID] --follow
```

## Project Structure

```
├── agent_deployment/           # Agent runtime code
│   ├── tutorial_agent.py      # Main agent implementation
│   └── pyproject.toml         # Runtime dependencies
├── agentcore_docs/            # AgentCore documentation
├── initial_goal.txt           # Tutorial reference
├── .bedrock_agentcore.yaml    # AgentCore config (gitignored)
├── pyproject.toml             # Dev dependencies
└── README.md                  # This file
```

## Model Configuration

Using inference profile for Claude Sonnet 4.5:
```python
MODEL_ID = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"
```

## Roadmap

- [x] Basic agent with calculator
- [x] Deploy to AgentCore Runtime
- [x] Configure memory infrastructure
- [ ] **Next**: Integrate memory in code (STM+LTM)
- [ ] Add Code Interpreter for advanced analysis
- [ ] Multi-agent orchestration

## Resources

- [AgentCore Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore.html)
- [Strands Agents Framework](https://github.com/awslabs/strands-agents)
- [Original Tutorial](initial_goal.txt)

## Cleanup

To remove all AWS resources:

```bash
uv run agentcore destroy
```

⚠️ This deletes the agent, memory, and S3 artifacts.

## License

MIT

## Author

Christopher Wycoff

