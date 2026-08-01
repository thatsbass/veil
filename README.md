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

## Repository layout

This repository is a monorepo that tracks three independently versioned
components. Each component is fully self-documented in its own README:

| Component | Mission | Stack | Documentation |
|-----------|---------|-------|---------------|
| `veil-api` | The gateway: request routing, format translation, quotas, billing | Go (Fiber, pgx, Redis, Stripe, Clerk) | [README](veil-api/README.md) |
| `veil-cli` | Command-line client: authentication, tool configuration, terminal dashboard | Go (Cobra, Bubble Tea, Lipgloss) | [README](veil-cli/README.md) |
| `veil-dashboard` | Web dashboard: usage monitoring, API key management, billing | React, Vite, TypeScript, Tailwind CSS | [README](veil-dashboard/README.md) |

## Getting started

Each component has its own setup instructions:

- **Gateway** — [`veil-api`](veil-api/README.md#getting-started)
- **CLI client** — [`veil-cli`](veil-cli/README.md#installation)
- **Web dashboard** — [`veil-dashboard`](veil-dashboard/README.md#getting-started)

A typical local setup consists of running the gateway, then either the CLI or
the dashboard on top of it.

## License

MIT. See the license file in each component repository for details.
