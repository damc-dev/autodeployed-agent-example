# Autodeployed Agent Example

**Deploy AI agents to production with a single `git push`** — this project demonstrates true continuous deployment for AWS Bedrock AgentCore agents using GitHub Actions.

## Overview

Every commit to `main` automatically:
- 🚀 **Deploys a new version** of each agent to AWS Bedrock
- 🏗️ **Builds and pushes** Docker images to Amazon ECR
- 📦 **Tracks deployment history** via timestamped artifacts
- ⚡ **Runs in parallel** for multi-agent deployments

**No manual intervention required.** Push your code, and your agents are live in minutes.

## How It Works

### Continuous Deployment Pipeline

**Each push to `main` triggers a full deployment cycle:**

```
git push origin main
      ↓
GitHub Actions detects the push
      ↓
Workflow extracts all agents from .bedrock_agentcore.yaml
      ↓
Parallel deployment begins for each agent
      ↓
Docker image built → pushed to ECR → deployed to Bedrock
      ↓
New version is live! ✨
```

The deployment triggers on:
- Direct pushes to the `main` branch
- Merged pull requests to `main`

### Deployment Architecture

The GitHub Actions workflow ([.github/workflows/deploy-agent.yml](.github/workflows/deploy-agent.yml)) uses a matrix strategy to deploy multiple agents efficiently:

**Setup Job:**
1. Extracts all agent names from `.bedrock_agentcore.yaml` using `yq`
2. Installs Python and `uv` package manager
3. Installs dependencies once (`uv sync`)
4. Caches the environment for reuse

**Deploy Jobs (parallel):**
1. Restores the cached environment
2. Configures AWS credentials
3. Runs `agentcore launch --agent {agent-name}` for each agent
4. Saves agent-specific `.bedrock_agentcore.yaml` as an artifact

This approach eliminates redundant dependency installation, significantly speeding up multi-agent deployments.

### Setup Requirements

#### AWS Configuration

You need to configure AWS authentication for GitHub Actions. Two options:

**Option 1: OIDC (Recommended)**

1. Create an IAM OIDC identity provider for GitHub Actions:
   ```
   Provider URL: https://token.actions.githubusercontent.com
   Audience: sts.amazonaws.com
   ```

2. Create an IAM role with trust policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::YOUR_ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
             "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main"
           }
         }
       }
     ]
   }
   ```

3. Attach necessary permissions to the role:
   - Amazon ECR (push images)
   - AWS CodeBuild (trigger builds)
   - Amazon Bedrock AgentCore (deploy agent)

4. Add the role ARN as a GitHub secret: `AWS_ROLE_ARN`

**Option 2: Access Keys**

1. Create an IAM user with programmatic access
2. Attach necessary permissions (same as above)
3. Add GitHub secrets:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`

### Manual Deployment

To deploy a specific agent:

```bash
uv sync
uv run agentcore launch --agent <agent-name>
```

## Development

### Prerequisites

- Python 3.12+
- [uv](https://github.com/astral-sh/uv) package manager
- AWS credentials configured

### Local Setup

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync
```

### Local Agent Run

```bash
# Run the agent locally (if applicable)
uv run python agent.py
```

### Local Invocation
You can invoke the agent locally for testing:

```bash
curl -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Hello!"}'
```

### Adding a New Agent

Adding a new agent is simple and automatically deployed:

```bash
# Configure a new agent interactively
uv run agentcore configure --agent <new-agent-name>

# Commit and push
git add .bedrock_agentcore.yaml
git commit -m "Add new agent"
git push origin main
```

**What happens next:**
1. Configure command updates `.bedrock_agentcore.yaml` with your new agent
2. Push to `main` triggers the deployment workflow
3. Workflow automatically detects the new agent
4. New agent is built and deployed alongside existing agents in parallel
5. Your new agent is live! 🎉

**Example:**
```bash
# Add a new agent called "summarizer_agent"
uv run agentcore configure --entrypoint summarizer_agent.py

# Commit and push to deploy
git add .bedrock_agentcore.yaml
git commit -m "Add summarizer agent"
git push origin main

# ✅ Agent automatically deployed to AWS Bedrock
```


## Project Structure

- `agent.py` - Agent entrypoint
- `.bedrock_agentcore.yaml` - Agent configuration
- `Dockerfile` - Container definition
- `.github/workflows/deploy-agent.yml` - Deployment workflow

## Deployment Versioning & History

Every deployment is tracked and versioned automatically:

### Artifacts
Each successful deployment creates timestamped artifacts:
- **Naming**: `bedrock-agentcore-config-{agent-name}-{commit-sha}`
- **Retention**: 90 days
- **Purpose**: Full audit trail of deployments and configuration changes

### Version Tracking
- Each `git push` to `main` creates a new agent version in AWS Bedrock
- Docker images are tagged with the commit SHA for traceability
- Deployment artifacts link code changes to production versions

This means you can:
- 📜 Review deployment history across all agents
- 🔍 Trace production issues back to specific commits
- ⏮️ Roll back to any previous configuration
- 📊 Audit who deployed what and when
