# Veil

[![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go&logoColor=white)](https://go.dev/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![API](https://img.shields.io/badge/API-Gateway-6f42c1)](veil-api/)
[![CLI](https://img.shields.io/badge/CLI-Terminal-2ea44f)](veil-cli/)
[![Dashboard](https://img.shields.io/badge/Web-React-61dafb)](veil-dashboard/)

Veil is a universal LLM gateway that lets you keep using your existing AI coding
tools — Claude CLI, Cursor, Aider, Codex — while routing their traffic to more
cost-effective models in the background. It exposes the **Anthropic** and
**OpenAI** API surfaces, translates incoming requests to a configurable provider
(DeepSeek by default), and returns responses in the exact format your tool
expects. The tool is unaware that a translation is happening.

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

The project is split into three independently versioned repositories, kept
together in this monorepo:

| Component | Description | Stack |
|-----------|-------------|-------|
| [`veil-api`](veil-api/) | The gateway: request routing, format translation, quotas, billing | Go (Fiber, pgx, Redis, Stripe, Clerk) |
| [`veil-cli`](veil-cli/) | The command-line client: authentication, tool configuration, terminal dashboard | Go (Cobra, Bubble Tea, Lipgloss) |
| [`veil-dashboard`](veil-dashboard/) | The web dashboard: usage monitoring, API key management, billing | React, Vite, TypeScript, Tailwind CSS |

---

## Table of contents

- [Motivation](#motivation)
- [Architecture](#architecture)
- [Repository structure](#repository-structure)
- [Getting started](#getting-started)
- [Usage](#usage)
- [API reference](#api-reference)
- [Plans and billing](#plans-and-billing)
- [Configuration](#configuration)
- [Testing](#testing)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

---

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
- **Cost reduction.** By default, requests are forwarded to DeepSeek, which can
  cut API spend by up to 90% depending on usage patterns.
- **Provider isolation.** The upstream client, payment processor, auth provider,
  and mailer are each isolated behind an interface. Swapping one out is a
  one-line change in the composition root.
- **Self-service platform.** A web dashboard and a CLI allow users to manage
  API keys, monitor usage, and upgrade plans without contacting an operator.

---

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

### Request flow

1. A local tool sends a request to `veil-api` using its native format
   (`/v1/messages` for Anthropic, `/v1/chat/completions` or `/v1/responses` for
   OpenAI), authenticated with an API key (`Bearer vl_live_xxx`).
2. The gateway validates the key, enforces the user's quota, and meters the
   request.
3. The `translator` adapters convert the incoming format to the upstream
   provider's format.
4. The `provider` client forwards the request to DeepSeek.
5. The response is translated back to the tool's expected format and returned.
6. Token usage is recorded and attributed to the calling user's plan.

### Design decisions

- **Providers behind interfaces.** `provider/`, `billing/stripe/`,
  `auth/clerk/`, and `mailer/resend/` each implement a small interface. The
  concrete implementation is selected once in `veil-api/cmd/server/main.go`.
- **Format translators.** The `translator/` package contains adapters for the
  Anthropic Messages API, the OpenAI Chat Completions API, and the OpenAI
  Responses API.
- **Token metering and quotas.** The `billing/` package tracks monthly usage per
  user and enforces plan limits; overage is billed at the metered rate.
- **Automatic migrations.** The database schema is versioned under
  `veil-api/migrations/` and applied automatically at server startup.
- **Two authentication surfaces.** LLM endpoints use hashed API keys
  (`vl_live_xxx`); dashboard endpoints use Clerk JWTs resolved to Veil user
  records via email.

---

## Repository structure

```
veil/
├── veil-api/          Go gateway — routing, translation, quotas, billing
├── veil-cli/          Go CLI — auth, tool configuration, terminal dashboard
└── veil-dashboard/    React web app — usage, API keys, billing
```

### `veil-api`

```
cmd/server/           Server entry point (composition root)
cmd/seed/             Development seeder (test user + API key)
internal/
  auth/               API key validation and quota enforcement
  auth/clerk/         Clerk JWT provider (swappable)
  billing/            Token metering and quota management
  billing/stripe/     Stripe webhook provider (swappable)
  gateway/            Request routing and response translation
  provider/           Upstream LLM client (DeepSeek)
  translator/         Anthropic / OpenAI / Responses format adapters
  api/keys/           Dashboard — API key management
  api/usage/          Dashboard — usage statistics
  api/billing/        Dashboard — plan and billing
  api/v1handler/      CLI-facing endpoints (/v1/usage, /v1/billing/plan)
  mailer/             Transactional email interface
  mailer/resend/      Resend provider (swappable)
  migrate/            Auto-migration at startup
migrations/           Versioned SQL schema
pkg/models/           Shared domain models
docs/                 Generated Swagger documentation
```

### `veil-cli`

```
cmd/                  Composition root
internal/
  adapter/            Config storage, API client, tool configurator
  delivery/           Cobra commands and interactive REPL (Bubble Tea)
  domain/             Auth, config, and gateway domain logic
  ports/              Port interfaces (hexagonal architecture)
```

### `veil-dashboard`

```
src/
  pages/              Dashboard, Usage, APIKeys, Billing, Login, Activate
  components/         Layout and UI components (Radix, Tailwind)
  hooks/              Shared React hooks
  lib/                API client and utilities
  types/              TypeScript types
```

---

## Getting started

### Prerequisites

- Go 1.22 or later (1.24 recommended)
- Node.js 18+ and npm (for the dashboard)
- PostgreSQL 16
- Redis 7+
- A DeepSeek API key (or any upstream provider)

### 1. Start the gateway (`veil-api`)

```bash
cd veil-api

# 1. Configure the environment
cp .env.example .env

# 2. Start PostgreSQL and Redis (optional if you run them locally)
make docker-up

# 3. Start the server (migrations run automatically)
make run

# 4. Seed a test user and API key for local development
make seed
```

The API and Swagger UI are available at `http://localhost:3000`.

### 2. Install the CLI (`veil-cli`)

```bash
cd veil-cli
make install        # installs `veil` into /usr/local/bin

veil auth login     # authenticate this machine (device authorization flow)
veil use claude     # configure a local tool to route through Veil
veil up             # show server status and monthly usage
```

### 3. Run the dashboard (`veil-dashboard`)

```bash
cd veil-dashboard
npm install

# Create a local environment file
echo "VITE_CLERK_PUBLISHABLE_KEY=pk_test_xxx" > .env.local
echo "VITE_API_URL=http://localhost:8080" >> .env.local

npm run dev         # http://localhost:5173
```

---

## Usage

### CLI commands

| Command | Description |
|---------|-------------|
| `veil auth login` | Authenticate via the device authorization flow |
| `veil auth logout` | Remove the local API key |
| `veil auth status` | Show authentication state |
| `veil use [tool]` | Configure a local tool to route through Veil |
| `veil up` | Show server status and monthly token usage |
| `veil status` | Show local configuration |
| `veil stats` | Show monthly usage and savings |
| `veil logs` | Stream live server logs (Ctrl+C to stop) |
| `veil doctor` | Run a connectivity diagnostic |
| `veil down` | Delete the local session |
| `veil version` | Print the current version |
| `veil update` | Check for available updates |

Running `veil` with no arguments launches an interactive REPL with slash
commands: `/status`, `/stats`, `/billing`, `/logs`, `/config`, `/use`,
`/doctor`, `/login`, `/logout`, `/help`, `/exit`.

### Supported tools

| Tool | Config file | Keys written |
|------|-------------|--------------|
| Claude Code | `~/.claude/settings.json` | `apiBaseUrl`, `apiKey` |
| Cursor | `~/.cursor/settings.json` | `openai.apiBase`, `openai.apiKey` (nested) |
| Codex CLI | `~/.codex/config.toml` | `api_base_url`, `api_key` |
| Aider | `~/.aider.conf.yml` | `openai-api-base`, `openai-api-key` |

Each `veil use` invocation creates a backup at `<config>.veil.bak` before
writing.

---

## API reference

### LLM endpoints — `Bearer vl_live_xxx`

| Method | Path | Format |
|--------|------|--------|
| POST | `/v1/messages` | Anthropic Messages API |
| POST | `/v1/chat/completions` | OpenAI Chat Completions |
| POST | `/v1/responses` | OpenAI Responses API |
| GET | `/v1/models` | Model list |
| GET | `/v1/usage` | Current month usage (CLI) |
| GET | `/v1/billing/plan` | Active plan (CLI) |

### Dashboard endpoints — `Bearer <Clerk JWT>`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/keys` | Create an API key |
| GET | `/api/keys` | List API keys |
| DELETE | `/api/keys/:id` | Revoke an API key |
| GET | `/api/usage` | Current month usage |
| GET | `/api/usage/history` | Request history |
| GET | `/api/billing/plan` | Active plan |
| POST | `/api/billing/upgrade` | Upgrade plan |

### Public

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Liveness check |
| POST | `/webhooks/payment` | Payment provider webhooks |

---

## Plans and billing

| Plan | Price | Tokens / month | API keys |
|------|-------|----------------|----------|
| Free | $0 | 100k | 1 |
| Starter | $9 | 2M | 3 |
| Pro | $29 | 10M | 10 |
| Team | $99 | 50M | 50 |

Overage is billed at $0.50 per 1M tokens. Payments are processed through Stripe;
the `POST /webhooks/payment` endpoint keeps the metered usage and plan state in
sync with the payment provider.

---

## Configuration

Configuration is read from `.env` (via `viper`), with environment variables
taking precedence.

| Variable | Description |
|----------|-------------|
| `PORT` | HTTP port (default: `3000`) |
| `HOST` | Swagger UI host (default: `localhost:3000`) |
| `BASE_URL` | Public base URL (default: `http://localhost:3000`) |
| `ENV` | `development` or `production` |
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `DEEPSEEK_API_KEY` | DeepSeek secret key |
| `RESEND_API_KEY` | Resend email key (optional, noop if absent) |
| `STRIPE_SECRET_KEY` | Stripe secret key |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret |
| `CLERK_SECRET_KEY` | Clerk secret key (dashboard authentication) |
| `API_KEY_SECRET` | Salt used to hash API keys |

---

## Testing

```bash
# Gateway — unit tests, race detector, and coverage
cd veil-api
make test
make test-race
make test-cover

# CLI
cd veil-cli
go test ./... -v -cover

# Dashboard — type checking and linting
cd veil-dashboard
npm run typecheck
npm run lint
```

---

## Development

### Common tasks (`veil-api`)

| Command | Description |
|---------|-------------|
| `make run` | Start the server |
| `make build` | Compile to `bin/veil` |
| `make test` / `make test-race` | Run tests (with race detector) |
| `make lint` | Run golangci-lint |
| `make migrate-up` / `migrate-down` | Apply / roll back migrations |
| `make sqlc` | Regenerate SQLC code from `sqlc/sqlc.yaml` |
| `make swag` | Regenerate Swagger docs from annotations |
| `make seed` | Insert a test user and API key |

### Common tasks (`veil-cli`)

| Command | Description |
|---------|-------------|
| `make build` | Compile to `bin/veil` |
| `make install` | Install to `/usr/local/bin` |
| `make dev` | Run from source |
| `make test` | Run tests with coverage |
| `make lint` | Run golangci-lint |

### Docker

```bash
cd veil-api
docker-compose up -d        # PostgreSQL + Redis
docker build -t veil .      # build the gateway image
docker run -p 3000:3000 --env-file .env veil
```

### Regenerating documentation

The OpenAPI/Swagger documentation is generated from Go annotations in
`cmd/server/main.go`:

```bash
cd veil-api
make swag-install   # one-time: install the swag CLI
make swag           # regenerate docs/
```

---

## Contributing

- Each component lives in its own repository and has its own README with
  component-specific instructions.
- Follow the existing code conventions: small interfaces, dependency
  injection, and isolated providers behind ports.
- Run the linter and the test suite before opening a pull request.
- Keep documentation in sync with code changes (Swagger annotations, READMEs).

---

## License

MIT. See the license file in each component repository for details.
