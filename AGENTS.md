# Mobile Command Agent Instructions

This repository branch is for the safe dedicated-laptop setup of Decepticon.
Agents must treat this as offensive tooling that is only safe when isolated,
authorized, and scoped.

## Tool Policy

Prefer the Docker-backed `local-dev-mcp-full` MCP server before native Codex
tools when it can do the work.

At the start of repository work, verify that `local-dev-mcp-full` is available
before using native Codex tools for overlapping repository tasks. If it is
unavailable, disconnected, missing the needed capability, or rejects the repo
path, report that briefly and continue with the smallest necessary native
fallback.

Use MCP first for:

- repository inspection, file reads, search, symbol/reference lookup, and repo trees
- git status, diffs, and logs
- tests, lint, builds, typechecks, package scripts, diagnostics, logs, and artifacts
- file creation, edits, moves, deletes, and patch application
- Docker and Docker Compose inspection or workflow commands
- browser page reads/workflows, security scans, AWS, Terraform, Kubernetes, Helm,
  database migration checks, release checks, and local memory tools
- shell commands through `local-dev-mcp-full.run_command` when the command fits
  MCP policy and does not require an interactive terminal

Use native Codex tools only when:

- `local-dev-mcp-full` is unavailable, disconnected, missing the needed
  capability, or returns a failure that cannot be resolved through MCP
- an interactive or long-running terminal session is required
- native `apply_patch` is needed for precise source edits or MCP patching fails
- `update_plan`, sub-agents, image viewing/perception, tool discovery, app
  connectors, or web search are required
- the user explicitly asks for native Codex tooling

When falling back from MCP to native Codex tools for overlapping work, briefly
state the reason in the working update or final answer.

## Safety And Scope

- Do not run Decepticon on the main host.
- Finish setup on the dedicated laptop inside the VirtualBox VM.
- Use Ubuntu Server 24.04 LTS, 4 vCPU, 12 GB RAM, 80 GB disk, NAT networking only.
- Disable shared folders, shared clipboard, drag/drop, USB passthrough, and
  bridged networking.
- Use API-first LLM configuration for initial setup.
- Do not bridge the VM to the Windows/WSL local model.
- Keep `COMPOSE_PROFILES=` unless C2 or local victim labs are explicitly approved.
- Do not enable `c2-sliver` during baseline setup.
- Do not enable `victims` unless intentionally running a local lab/demo.
- Do not start any real engagement without written authorization, target scope,
  testing window, exclusions, approved technique level, and emergency stop contact.
- Do not use arbitrary source-IP spoofing. Use client-approved VPN/static egress.

## Required Docs To Read

Before making setup changes, read:

- `LOCAL_README.md`
- `docs/safe-pentest-setup.md`
- `docs/mobile-command-agent-tooling.md`
- `docker-compose.safe.yml`
- `.env.safe.example`

For web dashboard changes under `clients/web`, also read `clients/web/AGENTS.md`
and the relevant Next.js docs in `node_modules/next/dist/docs/` after installing
dependencies. This project uses a newer Next.js version with breaking changes.

## Branch Discipline

- Keep this work on the `mobile-command` branch.
- Do not mix in Aegis-Sentry changes or files.
- Do not commit `.env`, credentials, VPN configs, VM secrets, or client scope docs.
- Commit only tracked setup docs/config/templates intended for the dedicated
  workstation handoff.
- Do not use Copilot Chat, Roo/Cline, or any second-agent VS Code extension to
  modify this branch. Use the current Codex session, repository scripts, and
  approved CLI tools only.
