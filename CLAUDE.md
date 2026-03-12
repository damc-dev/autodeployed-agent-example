# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Run agent locally (serves on port 8080)
uv run python agent.py

# Test locally
curl -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Hello!"}'

# Deploy a specific agent to AWS
uv run agentcore launch --agent <agent-name>

# Configure a new agent
uv run agentcore configure --agent <new-agent-name>

# Build Docker image locally
docker build -t bedrock-agentcore-agent .

# Monitor GitHub Actions deployments
gh run list --workflow=deploy-agent.yml
gh run watch          # watch the latest run
gh run view <run-id>  # view details of a specific run
gh run view <run-id> --log  # view logs
```

## Architecture

This project demonstrates **continuous deployment of AWS Bedrock AgentCore agents** — every push to `main` auto-deploys all configured agents via GitHub Actions.

### Core Components

**`agent.py`** — The agent entrypoint. Wraps a [Strands](https://strandsagents.com) `Agent` in a `BedrockAgentCoreApp`, exposing an `invoke` function decorated with `@app.entrypoint` that receives a `payload` dict and returns a response dict.

**`.bedrock_agentcore.yaml`** — Central config tracking AWS resources per agent: ECR repo, execution role ARNs, CodeBuild project, Bedrock agent IDs. This file is updated by `agentcore launch` and committed back.

**`.github/workflows/deploy-agent.yml`** — Two-stage pipeline:
1. **Setup job**: Extracts agent names from `.bedrock_agentcore.yaml` using `yq`, outputs a JSON matrix
2. **Deploy job**: Runs in parallel per agent, authenticates to AWS via OIDC (preferred) or access keys, then runs `agentcore launch`

### Adding a New Agent

1. Create a new Python file (e.g., `my_agent.py`) following the same `BedrockAgentCoreApp` pattern
2. Run `uv run agentcore configure --agent my-agent` to register it in `.bedrock_agentcore.yaml`
3. Push to `main` — the CI/CD pipeline auto-deploys all agents defined in `.bedrock_agentcore.yaml`

### Key Dependencies

- **bedrock-agentcore**: AWS SDK for deploying/running agents on Bedrock AgentCore
- **strands-agents**: Agent framework (use `from strands import Agent`)
- **bedrock-agentcore-starter-toolkit**: Provides the `agentcore` CLI (dev dependency)

### AWS Infrastructure

The deployment uses:
- ECR for container images
- CodeBuild for building/pushing images
- Bedrock AgentCore runtime for serving
- IAM OIDC for GitHub Actions authentication (configured in repo secrets as `AWS_ROLE_TO_ASSUME`)

## AWS CLI — Check Agent Config & Logs

Agent IDs and region are sourced from `.bedrock_agentcore.yaml`. Use `yq` to extract them:

```bash
AGENT_NAME=$(yq '.default_agent' .bedrock_agentcore.yaml)
AGENT_ID=$(yq ".agents.${AGENT_NAME}.bedrock_agentcore.agent_id" .bedrock_agentcore.yaml)
REGION=$(yq ".agents.${AGENT_NAME}.aws.region" .bedrock_agentcore.yaml)
```

### Check agent runtime configuration

```bash
# Get agent status and full config
aws bedrock-agentcore-control get-agent-runtime \
  --agent-runtime-id "$AGENT_ID" \
  --region "$REGION"

# List all agent runtimes in the account
aws bedrock-agentcore-control list-agent-runtimes --region "$REGION"

# List endpoints (you'll need the endpoint name for log queries)
aws bedrock-agentcore-control list-agent-runtime-endpoints \
  --agent-runtime-id "$AGENT_ID" \
  --region "$REGION"
```

### Check logs (CloudWatch)

Log groups follow the pattern `/aws/bedrock-agentcore/runtimes/<agent-id>-<endpoint-name>`. The endpoint name is typically `DEFAULT`.

```bash
# Discover the log group(s) for this agent
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/bedrock-agentcore/runtimes/${AGENT_ID}" \
  --region "$REGION"

# Live-tail logs (AWS CLI v2)
aws logs tail \
  "/aws/bedrock-agentcore/runtimes/${AGENT_ID}-DEFAULT" \
  --region "$REGION" \
  --follow

# Filter logs by a specific string (e.g. request ID or error keyword)
aws logs filter-log-events \
  --log-group-name "/aws/bedrock-agentcore/runtimes/${AGENT_ID}-DEFAULT" \
  --filter-pattern '"ERROR"' \
  --region "$REGION"
```
