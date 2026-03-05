# Roadmap

## Current Status (March 2026)

### ✅ Stable — Standard Deployment (EC2)

Single-user OpenClaw on AWS with Bedrock. Production-ready.

- One-click CloudFormation deploy (Linux/Mac/China)
- 10 Bedrock models, Graviton ARM, VPC Endpoints
- SSM Session Manager access (no public ports)
- Gateway token in SSM SecureString (never on disk)
- Supply-chain hardened (no `curl | sh`)
- S3 Files Skill auto-installed
- Docker sandbox for code isolation
- Kiro conversational deployment guide

### 🔨 In Development — Multi-Tenant Platform (AgentCore)

Multi-user OpenClaw with tenant isolation, permission control, and audit.

| Component | Status | Notes |
|-----------|--------|-------|
| Agent Container (Plan A + E enforcement) | ✅ Complete | System prompt injection + response audit |
| Auth Agent (approval workflow) | ✅ Complete | Risk assessment, 30-min auto-reject, token issuance |
| Safety module (input validation) | ✅ Complete | Memory poisoning detection, path traversal checks |
| Identity module (ApprovalToken) | ✅ Complete | Max 24h TTL, issue/validate/revoke |
| Observability (CloudWatch) | ✅ Complete | Structured JSON logs per tenant |
| CloudFormation (EC2 + ECR + SSM) | ✅ Complete | One-stack infrastructure |
| Gateway Tenant Router | ✅ Implemented | Tenant derivation + AgentCore invocation |
| Auth Agent input validation | ✅ Implemented | Prompt injection detection on approval messages |
| End-to-end integration testing | 🔨 In progress | Gateway → Router → AgentCore → Container flow |
| Auth Agent channel delivery | 🔨 In progress | Actually send notifications via WhatsApp/Telegram |
| Human approver reply parsing | ⬜ Not started | Parse "approve"/"reject" from messaging responses |

---

## Near-Term (Q2 2026)

### End-to-End Integration

Wire the complete message flow: OpenClaw Gateway → Tenant Router → AgentCore Runtime → Agent Container → response back to user. This is the critical path to making multi-tenant work.

- [ ] Integration test suite for the full message flow
- [ ] Tenant Router systemd service (auto-start on EC2 boot)
- [ ] OpenClaw webhook configuration to forward messages to Tenant Router

### Auth Agent Channel Delivery

Replace logging stubs with actual WhatsApp/Telegram API calls for approval notifications.

- [ ] Send approval notifications to admin's WhatsApp/Telegram
- [ ] Parse admin replies ("approve", "reject", "approve temporary 2h")
- [ ] Handle edge cases: admin offline, multiple admins, message delivery failure

### Cost Validation

Real-world cost comparison between EC2-only and AgentCore deployments.

- [ ] Benchmark AgentCore cold start latency
- [ ] Measure cost at 10, 100, 1000 conversations/day
- [ ] Document break-even point (when AgentCore becomes cheaper than EC2)

---

## Mid-Term (Q3 2026)

### Hard Enforcement via AgentCore Gateway (MCP Mode)

Current Plan A (system prompt injection) is soft enforcement — the LLM can be tricked via prompt injection. AgentCore Gateway with MCP mode would intercept tool calls at the API level, providing hard enforcement.

- [ ] Evaluate AgentCore Gateway MCP mode for tool-call interception
- [ ] Implement MCP-based permission checks (replace Plan A)
- [ ] Keep Plan E audit as defense-in-depth

### Multi-Admin Support

- [ ] Multiple approvers per tenant (any one can approve)
- [ ] Approval delegation (admin can delegate to another admin)
- [ ] Approval audit trail in CloudWatch

### Tenant Self-Service

- [ ] Tenant onboarding via messaging ("I'm new, set me up")
- [ ] Self-service permission requests with pre-approved templates
- [ ] Usage dashboard per tenant (tokens, cost, tools used)

---

## Long-Term (Q4 2026+)

### Lightsail Integration

AWS launched [OpenClaw on Lightsail](https://aws.amazon.com/blogs/aws/introducing-openclaw-on-amazon-lightsail-to-run-your-autonomous-private-ai-agents/) for single-user deployments. Explore multi-tenant extension on Lightsail for simpler infrastructure.

### Permissions Vending Machine

Temporary IAM permission elevation for OpenClaw agents ([Issue #29](https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock/issues/29)). Time-scoped IAM policies attached to the agent's role, auto-revoked after expiration.

### AgentCore Memory Integration

Persistent cross-session memory via AgentCore Memory service. Currently optional (`memory.py` exists but not wired to production flow).

- [ ] Enable memory persistence across microVM restarts
- [ ] Memory poisoning detection on load (not just on write)
- [ ] Per-tenant memory isolation verification

### Observability Dashboard

- [ ] CloudWatch dashboard template (per-tenant metrics)
- [ ] Cost anomaly detection (alert on unusual Bedrock spend)
- [ ] Permission denial trends (identify misconfigured tenants)

---

## Contributing

We welcome contributions at any stage. The most impactful areas right now:

1. **Integration testing** — help us validate the end-to-end flow
2. **Auth Agent channel delivery** — implement actual WhatsApp/Telegram notification sending
3. **Cost benchmarking** — real-world AgentCore vs EC2 cost data

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.
