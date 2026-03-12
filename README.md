# Autodeployed Agent Example

**Deploy AI agents to production with a single `git push`** — this project demonstrates true continuous deployment for AWS Bedrock AgentCore agents using GitHub Actions.

## Overview

Every commit to `main` automatically:
- 🚀 **Deploys a new version** of each agent to AWS Bedrock
- 🏗️ **Builds and pushes** Docker images to Amazon ECR
- 📦 **Tracks deployment history** via timestamped artifacts

**No manual intervention required.** Push your code, and your agents are live in minutes.

## How It Works

### Continuous Deployment Pipeline

**Each push to `main` triggers a full deployment cycle:**

```
git push origin main
      ↓
GitHub Actions detects the push
      ↓
AWS credentials configured via OIDC
      ↓
Docker image built → pushed to ECR → deployed to Bedrock
      ↓
New version is live! ✨
```

The deployment triggers on:
- Direct pushes to the `main` branch
- Merged pull requests to `main`

### Deployment Architecture

The GitHub Actions workflow ([.github/workflows/deploy-agent.yml](.github/workflows/deploy-agent.yml)) uses the `@aws/agentcore` npm CLI backed by AWS CDK:

1. Checks out code
2. Installs Node.js and `@aws/agentcore` CLI
3. Installs `uv` (required by the agentcore CLI for Python agents)
4. Configures AWS credentials via OIDC
5. Runs `agentcore deploy -y` which builds the Docker image, pushes to ECR, and deploys to Bedrock AgentCore

### Setup Requirements

#### Prerequisites

- Node.js 20+
- [uv](https://github.com/astral-sh/uv) package manager
- AWS account with CDK bootstrapped in your target region

#### CDK Bootstrap (one-time per account/region)

The `@aws/agentcore` CLI uses AWS CDK internally. Before the first deploy, bootstrap your account:

```bash
npm install -g @aws/agentcore
npx cdk bootstrap aws://YOUR_ACCOUNT_ID/us-east-1
```

#### AWS Configuration

Configure AWS authentication for GitHub Actions using OIDC:

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
   - AWS CloudFormation (CDK stack management)
   - `sts:AssumeRole` for CDK bootstrap roles (`cdk-hnb659fds-*`)

4. Add the role ARN as a GitHub secret: `AWS_ROLE_ARN`

### Manual Deployment

```bash
npm install -g @aws/agentcore
agentcore deploy -y
```

## Development

### Prerequisites

- Python 3.13+
- [uv](https://github.com/astral-sh/uv) package manager
- Node.js 20+ and `@aws/agentcore` CLI
- AWS credentials configured

### Local Setup

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install Python dependencies
uv sync

# Install agentcore CLI
npm install -g @aws/agentcore
```

### Local Agent Development

```bash
# Run the agent locally with a live reload server
agentcore dev
```

Then in another terminal:
```bash
curl -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Hello!"}'
```

### Adding a New Agent

1. Add a new entry to `agentcore/agentcore.json`:
   ```json
   {
     "type": "AgentCoreRuntime",
     "name": "my-new-agent",
     "build": "Container",
     "entrypoint": "my_agent.py",
     "codeLocation": ".",
     "runtimeVersion": "PYTHON_3_13"
   }
   ```

2. Commit and push:
   ```bash
   git add agentcore/agentcore.json my_agent.py
   git commit -m "Add my-new-agent"
   git push origin main
   ```

The workflow automatically deploys the new agent alongside existing ones.

## Project Structure

```
agent.py                          # Agent entrypoint
agentcore/
  agentcore.json                  # Agent configuration
  aws-targets.json                # Deployment target (account/region)
  cdk/                            # CDK app (managed by agentcore CLI)
Dockerfile                        # Container definition
.github/workflows/deploy-agent.yml  # Deployment workflow
```

## Deployment Versioning & History

Every deployment is tracked and versioned automatically:

### Artifacts
Each successful deployment creates artifacts:
- **Naming**: `agentcore-config-{commit-sha}`
- **Contents**: `agentcore/agentcore.json` and `agentcore/aws-targets.json`
- **Retention**: 90 days

### Version Tracking
- Each `git push` to `main` creates a new agent version in AWS Bedrock
- Docker images are tagged with the commit SHA for traceability
- Deployment artifacts link code changes to production versions

This means you can:
- 📜 Review deployment history across all agents
- 🔍 Trace production issues back to specific commits
- ⏮️ Roll back to any previous configuration
- 📊 Audit who deployed what and when
