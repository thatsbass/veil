# Veil

[![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go&logoColor=white)](https://go.dev/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Veil is a universal LLM gateway. It lets you keep using your existing AI coding
tools — Claude CLI, Cursor, Aider, Codex — while routing their traffic to more
cost-effective models in the background.

It exposes the **Anthropic** and **OpenAI** API surfaces, translates incoming
requests to a configurable provider (DeepSeek by default), and returns responses
in the exact format your tool expects. The tool is unaware that a translation is
happening.

```
Claude CLI / Cursor / Aider / Codex
              |
       Bearer vl_live_xxx
              |
          Veil API
       /v1/messages
       /v1/chat/completions
       /v1/responses
              |
        DeepSeek API
```

## Table of contents

- [Motivation](#motivation)
- [Architecture](#architecture)
- [Repository structure](#repository-structure)
- [Getting started](#getting-started)
- [License](#license)

## Motivation

AI coding tools have converged on a small set of provider APIs. This creates an
opportunity: instead of paying the premium price of the default provider, a
single translation layer can serve the same tools with cheaper models.

Veil implements this layer. It sits between your local tools and the model
provider, translates the Anthropic and OpenAI wire formats to the upstream
provider, and transparently handles authentication, quota enforcement, usage
metering, and billing.

Key properties:

- **Drop-in compatibility.** Existing tools keep their native configuration.
  Only the base URL and API key change.
- **Cost reduction.** Requests are forwarded to DeepSeek by default, cutting API
  spend by up to 90% depending on usage patterns.
- **Provider isolation.** The upstream client, payment processor, auth provider,
  and mailer are each isolated behind an interface. Swapping one out is a
  one-line change in the composition root.
- **Self-service platform.** A web dashboard and a CLI let users manage API
  keys, monitor usage, and upgrade plans without contacting an operator.

## Architecture

```
                        +------------------------------+
                        |       veil-dashboard         |  React / Vite / Clerk
                        |  (usage, keys, billing)      |
                        +--------------+---------------+
                                       |  Bearer <Clerk JWT>
                                       v
+---------------+         +------------------------------+
| Local tools   |  Bearer |            veil-api          |  Go / Fiber
| Claude CLI    | vl_live |  /v1/messages                |
| Cursor        | ------> |  /v1/chat/completions        |
| Aider         |   xxx   |  /v1/responses               |
| Codex         |         |  /api/* (dashboard)          |
+---------------+         +--------------+---------------+
                                       |  format translation
                                       v
                              +-----------------------+
                              |   Upstream provider   |  DeepSeek (default)
                              +-----------------------+
```

The gateway is responsible for routing, format translation, quotas, and billing.
See the [`veil-api`](veil-api/README.md#architecture) README for the detailed
request flow and package layout.

## Repository structure

This repository is a monorepo that tracks three independently versioned
components. Each component is fully self-documented in its own README:

| Component | Mission | Stack | Documentation |
|-----------|---------|-------|---------------|
| `veil-api` | The gateway: request routing, format translation, quotas, billing | Go (Fiber, pgx, Redis, Stripe, Clerk) | [README](veil-api/README.md) |
| `veil-cli` | Command-line client: authentication, tool configuration, terminal dashboard | Go (Cobra, Bubble Tea, Lipgloss) | [README](veil-cli/README.md) |
| `veil-dashboard` | Web dashboard: usage monitoring, API key management, billing | React, Vite, TypeScript, Tailwind CSS | [README](veil-dashboard/README.md) |

## Getting started

A typical local setup runs the gateway first, then either the CLI or the
dashboard on top of it. Each component has its own setup instructions:

1. **Start the gateway** — [`veil-api`](veil-api/README.md#getting-started)

   ```bash
   cd veil-api
   cp .env.example .env
   make docker-up     # PostgreSQL + Redis
   make run           # server + auto-migrations, Swagger on :3000
   ```

2. **Connect your tools** — [`veil-cli`](veil-cli/README.md#installation)

   ```bash
   cd veil-cli
   make install
   veil auth login
   veil use claude
   ```

3. **Or use the web dashboard** — [`veil-dashboard`](veil-dashboard/README.md#getting-started)

   ```bash
   cd veil-dashboard
   npm install
   echo "VITE_CLERK_PUBLISHABLE_KEY=pk_test_xxx" > .env.local
   echo "VITE_API_URL=http://localhost:3000" >> .env.local
   npm run dev
   ```

## License

MIT. See the license file in each component repository for details.
