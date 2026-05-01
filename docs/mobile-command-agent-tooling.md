# Mobile Command Agent Tooling

This document is the allowed tooling guide for the agent that finishes the
dedicated-laptop setup. Use these tools so setup is repeatable and does not
depend on guessing or unrelated extensions.

## VS Code Extensions

No project-level VS Code extension list existed before this branch. The branch
now recommends only extensions relevant to this repo and the safe setup.

Relevant installed extensions to use for this branch:

- `anthropic.claude-code` — optional API/OAuth workflow support if the operator
  uses Claude tooling.
- `openai.chatgpt` — optional OpenAI workflow support.
- `github.vscode-pull-request-github` — review branch/PR state.
- `github.vscode-github-actions` — inspect workflow definitions and run status.
- `ms-azuretools.vscode-docker` — inspect Docker files, images, containers.
- `ms-azuretools.vscode-containers` — container/dev-container inspection.
- `bradlc.vscode-tailwindcss` — web dashboard CSS/Tailwind support.
- `dbaeumer.vscode-eslint` — web dashboard linting.
- `esbenp.prettier-vscode` — formatting for frontend files only when project
  scripts/configs call for it.
- `yoavbls.pretty-ts-errors` — TypeScript diagnostics.
- `ms-playwright.playwright` — browser validation if web dashboard changes are made.
- `hashicorp.terraform` — only if Terraform/cloud docs or future infra files are touched.

Available but not approved for this setup branch:

- `github.copilot-chat` — do not use as a second agent or source of code changes.
- `rooveterinaryinc.roo-cline` — do not use separate agent automation for this setup.
- `amazonwebservices.aws-toolkit-vscode` — not needed for the safe VM/API-first setup.
- `ms-dotnettools.vscode-dotnet-runtime` — not needed by this repo.

Do not install random offensive/security VS Code extensions. Use repository
scripts and Dockerized tools instead.

## Required System Tools In The VM

Install inside the dedicated Ubuntu Server VM:

- `git`
- `curl`
- `ca-certificates`
- `gnupg`
- `openssh-server`
- `wireguard`
- `nftables`
- Docker Engine
- Docker Compose v2
- Node.js 20+
- npm 10+
- Python 3.13 where available
- `uv` for local Python test/lint workflows

Optional but useful:

- `gh` for GitHub branch/PR/repo work.
- `ss`/`iproute2` for port and network validation.
- `jq` for JSON inspection.
- `psql` only for direct Postgres debugging.
- Neo4j Browser via `http://127.0.0.1:7474` only after the safe stack is running.

## Repository Tools To Use

Use these commands rather than inventing new workflows:

```bash
docker compose -f docker-compose.yml -f docker-compose.safe.yml config --quiet
docker compose -f docker-compose.yml -f docker-compose.safe.yml config --services
docker compose -f docker-compose.yml -f docker-compose.safe.yml up -d --build \
  postgres neo4j litellm langgraph sandbox web
docker compose -f docker-compose.yml -f docker-compose.safe.yml ps
docker compose -f docker-compose.yml -f docker-compose.safe.yml down
```

Python checks:

```bash
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run basedpyright
```

Web checks:

```bash
npm install
npm run build --workspace=clients/web
npm run dev --workspace=clients/web
```

Launcher checks, only when touching `clients/launcher`:

```bash
cd clients/launcher
go test ./...
```

Database/web setup tools already used by this repo:

- Prisma through `npx prisma ...` in `clients/web`.
- PostgreSQL in Docker Compose.
- Neo4j in Docker Compose.
- LangGraph/LiteLLM through Docker Compose.

Do not install alternative package managers or app frameworks unless the repo
already uses them or the operator explicitly asks.

## Safe Setup Files

Use these files directly:

- `.env.safe.example` — copy to `.env` inside the VM and fill in local-only secrets.
- `docker-compose.safe.yml` — safe Compose override for loopback bindings.
- `scripts/decepticon-vpn-killswitch.nft` — VPN kill-switch template.
- `docs/safe-pentest-setup.md` — operator setup guide.
- `LOCAL_README.md` — full dedicated-laptop handoff.

## Tools And Behaviors To Avoid

- Do not use bridged VM networking.
- Do not mount host folders into the VM or Decepticon sandbox.
- Do not paste API keys, VPN configs, or client scope into tracked files.
- Do not run `docker compose up` without `docker-compose.safe.yml` for the safe
  setup path.
- Do not set `COMPOSE_PROFILES=c2-sliver` unless C2 is explicitly authorized.
- Do not set `COMPOSE_PROFILES=victims` unless running a local lab/demo.
- Do not use the Windows/WSL local model from the VM during API-first setup.
- Do not run destructive git commands such as `git reset --hard` unless the
  operator explicitly asks for that operation.
