# Sentorix Documentation

Sentorix is an AI Governance Gateway — a proxy layer that sits between your application and LLM providers (OpenAI, Anthropic, AWS Bedrock). It enforces data privacy and security policies in real time: detecting and redacting PII before prompts ever leave your network, blocking prompt injection attacks, logging every interaction immutably for compliance, and surfacing everything in a real-time compliance dashboard. You change one URL in your existing OpenAI integration; Sentorix handles the rest.

---

## Quick Links

| Document | Description |
|---|---|
| [architecture/overview.md](architecture/overview.md) | System architecture, request flow, component breakdown |
| [development/local-setup.md](development/local-setup.md) | Run gateway + dashboard on your MacBook |
| [development/environment-variables.md](development/environment-variables.md) | Every env var for both applications |
| [development/testing.md](development/testing.md) | Running tests, mock strategy, writing new tests |
| [deployment/cicd-pipeline.md](deployment/cicd-pipeline.md) | How deployments work, trigger, monitor, rollback |
| [deployment/aws-infrastructure.md](deployment/aws-infrastructure.md) | Every AWS resource, why it exists, cost breakdown |
| [deployment/cloudflare-dns.md](deployment/cloudflare-dns.md) | DNS records, ACM certificate, adding subdomains |
| [api/gateway-api.md](api/gateway-api.md) | Customer-facing API reference |
| [runbooks/deploying-manually.md](runbooks/deploying-manually.md) | Manual deployment when CI/CD is broken |
| [runbooks/debugging-ecs.md](runbooks/debugging-ecs.md) | Debugging ECS tasks and services |
| [runbooks/known-issues.md](runbooks/known-issues.md) | Known issues, root causes, workarounds |

---

## Live URLs

| URL | What it is |
|---|---|
| [https://api.sentorix.io](https://api.sentorix.io) | Gateway API — the proxy endpoint |
| [https://app.sentorix.io](https://app.sentorix.io) | Compliance Dashboard |
| [https://docs.sentorix.dev](https://docs.sentorix.dev) | This documentation |

---

## Repository Map

| Repository | Language / Stack | What it contains |
|---|---|---|
| [sentorixhq/sentorix-gateway](https://github.com/sentorixhq/sentorix-gateway) | Python 3.12, FastAPI, Presidio | The proxy gateway — PII detection, policy enforcement, audit publishing |
| [sentorixhq/sentorix-dashboard](https://github.com/sentorixhq/sentorix-dashboard) | Next.js 14, TypeScript, Tailwind | The compliance and analytics dashboard |
| [sentorixhq/sentorix-infra](https://github.com/sentorixhq/sentorix-infra) | Terraform 1.6+ | All AWS infrastructure as code |
| [sentorixhq/sentorix-docs](https://github.com/sentorixhq/sentorix-docs) | Markdown | This repository — single source of truth |

---

## Who Should Read What

### Developers integrating Sentorix into their application
→ Start with [api/gateway-api.md](api/gateway-api.md) — it explains authentication, endpoints, and has working code examples in Python and Node.js.

### Engineers joining the Sentorix team
→ Start here, then read [architecture/overview.md](architecture/overview.md) to understand the system, then [development/local-setup.md](development/local-setup.md) to get running locally. The whole sequence should take about 2 hours.

### DevOps / Platform engineers
→ [deployment/cicd-pipeline.md](deployment/cicd-pipeline.md) covers how deployments work end to end. [deployment/aws-infrastructure.md](deployment/aws-infrastructure.md) documents every AWS resource.

### On-call / Incident response
→ [runbooks/debugging-ecs.md](runbooks/debugging-ecs.md) for live incidents. [runbooks/deploying-manually.md](runbooks/deploying-manually.md) when CI/CD is unavailable. [runbooks/known-issues.md](runbooks/known-issues.md) for known bugs and workarounds.

---

## Domains

| Domain | Purpose |
|---|---|
| `sentorix.io` | Primary product domain — dashboard and API |
| `sentorix.ai` | AI-focused marketing domain |
| `sentorix.dev` | Developer and documentation domain |

All domains are managed on Cloudflare (DNS only, no proxy). TLS is handled by an AWS ACM wildcard certificate covering `*.sentorix.io`.
