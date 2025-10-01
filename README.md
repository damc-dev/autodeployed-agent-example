# Autodeployed Agent Example

This project demonstrates automated deployment of an AWS Bedrock AgentCore agent using GitHub Actions.

## Overview

The agent is automatically deployed to AWS when changes are pushed or merged to the `main` branch. The deployment uses AWS Bedrock AgentCore SDK and is configured to run on AWS infrastructure.

## Agent Configuration

- **Agent ID**: `agent-3aevBK4TEA`
- **Region**: `us-east-1`
- **Platform**: `linux/arm64`
- **Runtime**: Docker
- **Entrypoint**: `agent.py`

## Deployment

### Automated Deployment

The agent automatically deploys on:
- Direct pushes to the `main` branch
- Merged pull requests to `main`

The GitHub Actions workflow ([.github/workflows/deploy-agent.yml](.github/workflows/deploy-agent.yml)):
1. Checks out the code
2. Sets up Python and dependencies with `uv`
3. Configures AWS credentials
4. Runs `agentcore launch` to deploy
5. Saves the `.bedrock_agentcore.yaml` configuration as an artifact

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

To deploy manually:

```bash
uv sync
uv run agentcore launch
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

# Run the agent locally (if applicable)
uv run python agent.py
```

## Project Structure

- `agent.py` - Agent entrypoint
- `.bedrock_agentcore.yaml` - Agent configuration
- `Dockerfile` - Container definition
- `.github/workflows/deploy-agent.yml` - Deployment workflow

## Artifacts

After each successful deployment, the workflow uploads the `.bedrock_agentcore.yaml` configuration as an artifact, retained for 90 days. This helps track configuration changes over time.
