# OpenClaw Multi-Tenant Platform (AgentCore Runtime)

> Turn OpenClaw from a single-user assistant into a multi-user platform on AWS — with tenant isolation, permission control, and audit logging. Without modifying OpenClaw itself.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![AWS](https://img.shields.io/badge/AWS-Bedrock-orange.svg)](https://aws.amazon.com/bedrock/)
[![Status](https://img.shields.io/badge/Status-In%20Development-yellow.svg)]()

> ⚠️ **Work in Progress** — Core infrastructure, Agent Container, Auth Agent, and Gateway Tenant Router are implemented. End-to-end integration testing is ongoing. Contributions welcome.

## How It Works

1. **User sends a message** via WhatsApp/Telegram/Discord
2. **EC2 Gateway** receives it, the **Tenant Router** derives a `tenant_id` from channel + user identity
3. **AgentCore Runtime** spins up an isolated Firecracker microVM for that tenant
4. **Agent Container** (Python wrapper) injects the tenant's allowed tools into the system prompt, forwards to OpenClaw subprocess, then audits the response
5. **If a blocked tool is detected**, a permission request goes to the **Auth Agent**, which notifies an admin via messaging for human approval

Each tenant gets complete isolation: separate VM, separate memory, separate filesystem. No cross-tenant data leakage.

## Architecture

```
Users (WhatsApp / Telegram / Discord)
  │
  ▼
┌──────────────────────────────────────────────────────┐
│  EC2 Gateway  (~$35/mo)                              │
│                                                      │
│  OpenClaw Gateway (Node.js, port 18789)              │
│  ├── Receives messages, serves Web UI                │
│  │                                                   │
│  Tenant Router (Python, port 8090)                   │
│  ├── derive_tenant_id(channel, user_id)              │
│  ├── invoke AgentCore Runtime (sessionId=tenant_id)  │
│  └── Return response to Gateway                     │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  AgentCore Runtime  (serverless, pay-per-use)        │
│  Each tenant → isolated Firecracker microVM          │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Agent Container (Docker in ECR)               │  │
│  │                                                │  │
│  │  1. safety.py    → validate input              │  │
│  │  2. permissions  → inject allowed tools        │  │
│  │  3. OpenClaw     → execute (subprocess)        │  │
│  │  4. audit        → scan response for violations│  │
│  │  5. observability→ log to CloudWatch           │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────┬───────────────────────────────┘
                       │ (on permission violation)
                       ▼
┌──────────────────────────────────────────────────────┐
│  Auth Agent  (separate AgentCore session)            │
│                                                      │
│  ├── Format risk-assessed approval notification      │
│  ├── Send to admin via WhatsApp/Telegram             │
│  ├── 30-minute auto-reject timer                     │
│  └── On approval: issue token or update SSM profile  │
└──────────────────────────────────────────────────────┘
```

## What This Adds to OpenClaw

| Capability | OpenClaw alone | This project |
|---|---|---|
| Users | Single user | Multiple tenants, fully isolated |
| Execution | Local process | Serverless microVM per tenant |
| Tool permissions | None | Per-tenant SSM profiles, system prompt injection |
| Response audit | None | Post-execution scan for unauthorized tools |
| Memory poisoning defense | None | Injection pattern detection (13 patterns) |
| Input validation | None | Message truncation, tool name/path validation |
| Approval workflow | None | Human-in-the-loop via messaging, 30-min auto-reject |
| Observability | Local logs | Structured CloudWatch JSON per tenant |

## Security Model

Based on [Microsoft's OpenClaw security guidance](https://www.microsoft.com/en-us/security/blog/2026/02/19/running-openclaw-safely-identity-isolation-runtime-risk/):

**Plan A (Soft Enforcement)**: Tenant's allowed tools list is injected into the system prompt before every request. The LLM knows its boundaries.

**Plan E (Audit)**: After OpenClaw responds, the response is scanned for blocked tool names. Violations are logged to CloudWatch.

**Always Blocked**: `install_skill`, `load_extension`, `eval` — regardless of tenant profile. Prevents supply-chain attacks via [ClawHub](https://www.onyx.app/insights/openclaw-enterprise-evaluation-framework).

**Auth Agent Input Validation**: Approval messages from admins are checked for prompt injection patterns before processing. Prevents attackers from manipulating the approval flow.

> This is soft enforcement, not a hard block. For hard enforcement, the architecture would need AgentCore Gateway (MCP mode). See [Roadmap](ROADMAP.md).

## Repository Structure

```
agent-container/           # Docker image for AgentCore Runtime
├── server.py              # HTTP wrapper: /ping + /invocations (Plan A + E)
├── permissions.py         # SSM profile read/write, permission checks
├── safety.py              # Input validation, memory poisoning detection
├── identity.py            # ApprovalToken lifecycle (max 24h TTL)
├── memory.py              # Optional AgentCore Memory persistence
├── observability.py       # Structured CloudWatch JSON logs
├── openclaw.json          # OpenClaw config template
├── Dockerfile             # Multi-stage: OpenClaw + Python 3.12
└── PERMISSION_SETUP_PROMPT.md

auth-agent/                # Authorization Agent (separate AgentCore session)
├── server.py              # HTTP entry point with input validation
├── handler.py             # Approval notifications, risk assessment, injection detection
├── approval_executor.py   # Execute approve/reject, update SSM
└── permission_request.py  # PermissionRequest dataclass

src/gateway/
└── tenant_router.py       # Gateway → AgentCore routing (tenant derivation + invocation)

src/utils/
└── agentcore.ts           # SessionKey derivation, response formatting

clawdbot-bedrock-agentcore-multitenancy.yaml  # CloudFormation stack
```

## Deployment

### Prerequisites

- AWS CLI with permissions for CloudFormation, EC2, VPC, IAM, ECR, Bedrock AgentCore, SSM, CloudWatch
- Docker installed locally
- Bedrock model access enabled

### Phase 1: Deploy Infrastructure

```bash
aws cloudformation create-stack \
  --stack-name openclaw-multitenancy \
  --template-body file://clawdbot-bedrock-agentcore-multitenancy.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=your-key-pair \
    ParameterKey=OpenClawModel,ParameterValue=global.amazon.nova-2-lite-v1:0

aws cloudformation wait stack-create-complete \
  --stack-name openclaw-multitenancy --region us-east-1
```

### Phase 2: Build and Push Agent Container

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=us-east-1

ECR_URI=$(aws cloudformation describe-stacks \
  --stack-name openclaw-multitenancy --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`MultitenancyEcrRepositoryUri`].OutputValue' \
  --output text)

aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

docker build --platform linux/arm64 -f agent-container/Dockerfile -t $ECR_URI:latest .
docker push $ECR_URI:latest
```

### Phase 3: Create AgentCore Runtime

```bash
EXECUTION_ROLE_ARN=$(aws cloudformation describe-stacks \
  --stack-name openclaw-multitenancy --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`AgentContainerExecutionRoleArn`].OutputValue' \
  --output text)

RUNTIME_ID=$(aws bedrock-agentcore create-agent-runtime \
  --agent-runtime-name "openclaw-multitenancy-runtime" \
  --agent-runtime-artifact '{"containerConfiguration":{"containerUri":"'$ECR_URI':latest"}}' \
  --role-arn "$EXECUTION_ROLE_ARN" \
  --network-configuration '{"networkMode":"PUBLIC"}' \
  --environment-variables "STACK_NAME=openclaw-multitenancy,AWS_REGION=$REGION" \
  --region $REGION \
  --query 'agentRuntimeId' --output text)

aws ssm put-parameter \
  --name "/openclaw/openclaw-multitenancy/runtime-id" \
  --value "$RUNTIME_ID" --type String --overwrite --region $REGION
```

### Phase 4: Start Tenant Router on EC2

```bash
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name openclaw-multitenancy --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' \
  --output text)

aws ssm start-session --target $INSTANCE_ID --region $REGION
```

On the EC2 instance:
```bash
sudo su - ubuntu

# Copy tenant_router.py to instance
# (or clone repo and use src/gateway/tenant_router.py)

export STACK_NAME=openclaw-multitenancy
export AWS_REGION=us-east-1
export AGENTCORE_RUNTIME_ID=$RUNTIME_ID

# Start the tenant router
nohup python3 tenant_router.py > /tmp/tenant-router.log 2>&1 &
```

### Phase 5: Access Gateway UI

```bash
# Terminal 1: port forwarding
aws ssm start-session --target $INSTANCE_ID --region $REGION \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18789"],"localPortNumber":["18789"]}'

# Terminal 2: get token
TOKEN=$(aws ssm get-parameter \
  --name "/openclaw/openclaw-multitenancy/gateway-token" \
  --region $REGION --with-decryption --query 'Parameter.Value' --output text)

echo "http://localhost:18789/?token=$TOKEN"
```

### Phase 6: Configure Enterprise Profiles (Optional)

```bash
STACK_NAME=openclaw-multitenancy REGION=us-east-1 bash setup-enterprise-profiles.sh
```

| Role | Tools | Use case |
|---|---|---|
| `readonly-agent` | web_search only | General staff |
| `finance-agent` | web_search, shell (read-only), file | Financial queries |
| `web-agent` | All tools | Web development |
| `erp-agent` | web_search, shell, file, file_write | ERP operations |

## Day-2 Operations

### Update Auth Agent behavior (no redeployment)
```bash
aws ssm put-parameter \
  --name "/openclaw/openclaw-multitenancy/auth-agent/system-prompt" \
  --type String --overwrite --value "Your updated instructions..."
```

### View tenant logs
```bash
aws logs filter-log-events \
  --log-group-name "/openclaw/openclaw-multitenancy/agents" \
  --filter-pattern '{ $.tenant_id = "wa__8613800138000" }' \
  --region us-east-1
```

### Update container image
```bash
docker build --platform linux/arm64 -f agent-container/Dockerfile -t $ECR_URI:latest .
docker push $ECR_URI:latest
# AgentCore picks up new image on next invocation
```

## Cost

| Component | Cost |
|---|---|
| EC2 c7g.large (always-on) | ~$35/mo |
| EBS 30GB gp3 | ~$2.40/mo |
| VPC Endpoints (optional) | ~$29/mo |
| AgentCore Runtime | Pay-per-invocation |
| ECR storage | ~$0.10/GB/mo |
| Bedrock Nova 2 Lite | $0.30/$2.50 per 1M tokens |

Light usage (100 conversations/day): ~$40-60/month total.

## Cleanup

```bash
aws bedrock-agentcore delete-agent-runtime --agent-runtime-id $RUNTIME_ID --region us-east-1
aws cloudformation delete-stack --stack-name openclaw-multitenancy --region us-east-1
```

## Resources

- [OpenClaw Docs](https://docs.openclaw.ai/)
- [AgentCore Runtime Docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime.html)
- [AgentCore Session Isolation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html)
- [Microsoft OpenClaw Security Guidance](https://www.microsoft.com/en-us/security/blog/2026/02/19/running-openclaw-safely-identity-isolation-runtime-risk/)
- [Roadmap](ROADMAP.md)
