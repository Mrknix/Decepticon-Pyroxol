# Local README

This file is for local-only notes about configs, modifications, and operational
changes made in this working copy. It is excluded from git via
`.git/info/exclude`.

## 2026-04-30 - Safe Pentest Setup

Added repo files for running Decepticon in a safer pentest posture:

- `docker-compose.safe.yml`
  - Safe Docker Compose override for VM use.
  - Replaces exposed ports with `127.0.0.1` bindings.
  - Uses VM-local workspace paths under `${DECEPTICON_HOME}/workspace`.
  - Adds `no-new-privileges:true` to the sandbox container.
- `.env.safe.example`
  - Safe environment template.
  - Uses `DECEPTICON_MODEL_PROFILE=test`.
  - Keeps `COMPOSE_PROFILES=` so C2 and victim services do not start by default.
  - Includes placeholder provider keys and local infrastructure secrets.
- `scripts/decepticon-vpn-killswitch.nft`
  - nftables VPN kill-switch template.
  - Allows outbound traffic only through the approved VPN interface after
    replacing `VPN_IFACE`, `VPN_SERVER`, and `VPN_PORT`.
- `docs/safe-pentest-setup.md`
  - Step-by-step VirtualBox VM setup.
  - VPN, kill-switch, start, validation, containment, and teardown procedure.

Validation performed:

```bash
docker compose -f docker-compose.yml -f docker-compose.safe.yml config --quiet
docker compose -f docker-compose.yml -f docker-compose.safe.yml config --services
```

Rendered enabled services:

```text
postgres
neo4j
sandbox
litellm
langgraph
web
```

Notes:

- The stack was not started on the host.
- Firewall rules were not applied on the host.
- `local-dev-mcp-full` was reachable but could not access this repo path, so
  native tools were used for repo work.

## 2026-04-30 - VM Resource Decision

Confirmed the original safe VM baseline will stay:

- VirtualBox VM.
- Ubuntu Server 24.04 LTS.
- 4 vCPU.
- 12 GB RAM.
- 80 GB disk.
- NAT networking only.

This is not the minimum requirement. It is the chosen safer operating baseline
for avoiding resource pressure while running the full Decepticon stack.

## 2026-04-30 - Dedicated Laptop API-First Setup Handoff

Intent:

- Run Decepticon on a dedicated laptop.
- Use an LLM API first for initial setup.
- Do not connect the pentest VM back to the Windows/WSL local model.
- Keep the local GPU/model setup separate until a later local-only design is
  planned.

Setup target:

- Host: dedicated laptop.
- VM platform: VirtualBox.
- Guest OS: Ubuntu Server 24.04 LTS.
- VM resources: 4 vCPU, 12 GB RAM, 80 GB disk.
- Network: one NAT adapter only.
- VirtualBox sharing:
  - Shared clipboard: disabled.
  - Drag and drop: disabled.
  - USB passthrough: disabled.
  - Shared folders: none.

VirtualBox management:

- Add NAT SSH port forwarding:
  - Host IP: `127.0.0.1`
  - Host port: `2222`
  - Guest IP: blank or VM IP
  - Guest port: `22`
- Manage VM from the laptop host with:

```bash
ssh -p 2222 decepticon@127.0.0.1
```

- Access the web dashboard through SSH local forwarding:

```bash
ssh -p 2222 -L 3000:127.0.0.1:3000 decepticon@127.0.0.1
```

- Then browse from the laptop host to:

```text
http://127.0.0.1:3000
```

Guest base packages:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl git gnupg nftables openssh-server wireguard
```

Docker:

- Install Docker Engine and Docker Compose inside the VM using Docker's official
  Ubuntu instructions.
- Add the VM user to the `docker` group if desired, then log out/in:

```bash
sudo usermod -aG docker "$USER"
docker --version
docker compose version
```

Repo setup inside VM:

```bash
git clone <repo-url> Decepticon-Pyroxol
cd Decepticon-Pyroxol
cp .env.safe.example .env
mkdir -p /home/decepticon/.decepticon/workspace
chmod 700 /home/decepticon/.decepticon
```

API-first `.env` configuration:

- Pick one API provider for first setup.
- Do not paste API keys into chat or committed files.
- Edit `.env` only inside the VM.

OpenAI example:

```env
OPENAI_API_KEY=sk-...
DECEPTICON_AUTH_PRIORITY=openai_api
DECEPTICON_MODEL_PROFILE=test
COMPOSE_PROFILES=
DECEPTICON_HOME=/home/decepticon/.decepticon
AUTO_UPDATE=false
```

Anthropic example:

```env
ANTHROPIC_API_KEY=sk-ant-...
DECEPTICON_AUTH_PRIORITY=anthropic_api
DECEPTICON_MODEL_PROFILE=test
COMPOSE_PROFILES=
DECEPTICON_HOME=/home/decepticon/.decepticon
AUTO_UPDATE=false
```

Required secret changes before real client work:

- Replace `LITELLM_MASTER_KEY`.
- Replace `LITELLM_SALT_KEY`.
- Replace `POSTGRES_PASSWORD`.
- Replace `NEO4J_PASSWORD`.

VPN and egress:

- Use a client-approved VPN/static egress IP.
- Preferred interface: WireGuard `wg0`.
- Do not implement arbitrary source-IP spoofing.
- Render and apply the nftables template only after replacing:
  - `VPN_IFACE`
  - `VPN_SERVER`
  - `VPN_PORT`

Example:

```bash
sudo sed -e 's/VPN_IFACE/wg0/g' \
         -e 's/VPN_SERVER/<approved-vpn-endpoint-ip>/g' \
         -e 's/VPN_PORT/<approved-vpn-port>/g' \
         scripts/decepticon-vpn-killswitch.nft | sudo nft -f -
sudo nft list ruleset
```

VPN validation:

```bash
curl -4 https://ifconfig.me
sudo systemctl stop wg-quick@wg0
curl -4 --max-time 5 https://ifconfig.me || echo "blocked as expected"
sudo systemctl start wg-quick@wg0
```

The visible IP must be the approved egress IP. With VPN down, internet access
must fail.

Compose validation:

```bash
docker compose -f docker-compose.yml -f docker-compose.safe.yml config --quiet
docker compose -f docker-compose.yml -f docker-compose.safe.yml config --services
```

Expected enabled services:

```text
postgres
neo4j
sandbox
litellm
langgraph
web
```

Start core safe stack:

```bash
docker compose -f docker-compose.yml -f docker-compose.safe.yml up -d --build \
  postgres neo4j litellm langgraph sandbox web
```

Health checks:

```bash
docker compose -f docker-compose.yml -f docker-compose.safe.yml ps
curl -fsS http://127.0.0.1:2024/ok
curl -fsS http://127.0.0.1:3000 >/dev/null
docker exec decepticon-sandbox curl -4 https://ifconfig.me
```

Containment checks:

```bash
docker ps --format '{{.Names}}' | grep -E 'c2|dvwa|msf2' && echo "unexpected profile running"
ss -ltnp | grep -E ':(2024|3000|3003|4000|5432|7474|7687)\b'
docker exec decepticon-sandbox find /workspace -maxdepth 2 -type f | head
```

Operational defaults:

- Keep `COMPOSE_PROFILES=` for first setup.
- Do not enable `c2-sliver` unless the engagement explicitly authorizes C2.
- Do not enable `victims` unless running a local lab/demo.
- Take a clean VM snapshot before the first real engagement.
- Revert or destroy the VM after high-risk work.

Stop commands:

```bash
docker compose -f docker-compose.yml -f docker-compose.safe.yml down
docker compose -f docker-compose.yml -f docker-compose.safe.yml down --volumes --remove-orphans
```

Instruction for the next agent:

- Read `docs/safe-pentest-setup.md` and this `LOCAL_README.md`.
- Finish setup on the dedicated laptop/VM only.
- Do not start Decepticon on the main host.
- Use API-first LLM configuration.
- Do not bridge to the Windows/WSL local model.

## 2026-04-30 - Next Agent Handoff Checklist

The next agent should have enough repo context to perform the setup, but must
ask for a few environment-specific inputs before executing.

Repo files already prepared:

- `docker-compose.safe.yml`
  - Compose override for safe VM operation.
  - Ensures exposed services bind to `127.0.0.1`.
  - Keeps runtime workspace VM-local.
- `.env.safe.example`
  - Template for API-first setup.
  - Defaults to `DECEPTICON_MODEL_PROFILE=test`.
  - Defaults to `COMPOSE_PROFILES=` so C2/victim services stay disabled.
- `scripts/decepticon-vpn-killswitch.nft`
  - nftables kill-switch template.
  - Requires concrete VPN interface, endpoint IP, and endpoint port before use.
- `docs/safe-pentest-setup.md`
  - Public/tracked setup guide.
- `LOCAL_README.md`
  - Local-only handoff notes.
  - This file is ignored by git; copy it manually to the dedicated laptop if
    the next agent starts from a fresh clone.

What the next agent should ask for before setup:

- Dedicated laptop access and confirmation that setup should happen only there.
- Permission to install VirtualBox or confirmation it is already installed.
- Permission to download Ubuntu Server 24.04 LTS ISO, or the local ISO path.
- VM username. Default assumption: `decepticon`.
- VM password or preferred SSH key setup method.
- Confirmation for VirtualBox NAT SSH forwarding:
  - Host IP: `127.0.0.1`
  - Host port: `2222`
  - Guest port: `22`
- Repo source:
  - Git remote URL, or
  - local archive/copy method.
- LLM provider choice for first setup:
  - OpenAI API, or
  - Anthropic API.
- The actual LLM API key. It must be entered only into the VM-local `.env`;
  do not paste it into chat or commit it.
- VPN details:
  - VPN type: WireGuard preferred, OpenVPN acceptable.
  - VPN client config file.
  - Approved VPN endpoint IP.
  - Approved VPN endpoint port.
  - VPN interface name. Default assumption: `wg0`.
  - Expected public egress IP after connection.
- Whether a real engagement is planned immediately. If yes, ask for:
  - written authorization,
  - target CIDRs/domains/repos,
  - testing window and timezone,
  - exclusions,
  - approved technique level,
  - emergency stop contact.

What the next agent should set up:

- VirtualBox VM:
  - Ubuntu Server 24.04 LTS.
  - 4 vCPU.
  - 12 GB RAM.
  - 80 GB disk.
  - One NAT adapter only.
  - No bridged adapter.
  - No shared folders.
  - Shared clipboard disabled.
  - Drag/drop disabled.
  - USB passthrough disabled.
- NAT SSH forwarding:
  - Host `127.0.0.1:2222` to guest port `22`.
- Guest packages:
  - `ca-certificates`
  - `curl`
  - `git`
  - `gnupg`
  - `nftables`
  - `openssh-server`
  - `wireguard`
- Docker Engine and Docker Compose inside the VM.
- Repo clone inside the VM only.
- Safe env:
  - `cp .env.safe.example .env`
  - set one real LLM API key,
  - set `DECEPTICON_AUTH_PRIORITY` to the matching provider,
  - replace LiteLLM/Postgres/Neo4j local secrets,
  - keep `COMPOSE_PROFILES=`,
  - keep `AUTO_UPDATE=false`.
- VPN:
  - install VPN config inside the VM,
  - bring up VPN,
  - render/apply nftables kill switch with real VPN values,
  - verify VPN-down traffic is blocked.
- Safe Compose stack:

```bash
docker compose -f docker-compose.yml -f docker-compose.safe.yml up -d --build \
  postgres neo4j litellm langgraph sandbox web
```

What the next agent should validate:

```bash
docker compose -f docker-compose.yml -f docker-compose.safe.yml config --quiet
docker compose -f docker-compose.yml -f docker-compose.safe.yml config --services
docker compose -f docker-compose.yml -f docker-compose.safe.yml ps
curl -fsS http://127.0.0.1:2024/ok
curl -fsS http://127.0.0.1:3000 >/dev/null
docker exec decepticon-sandbox curl -4 https://ifconfig.me
ss -ltnp | grep -E ':(2024|3000|3003|4000|5432|7474|7687)\b'
docker ps --format '{{.Names}}' | grep -E 'c2|dvwa|msf2' && echo "unexpected profile running"
```

Expected `config --services` output:

```text
postgres
neo4j
sandbox
litellm
langgraph
web
```

What the next agent must not do:

- Do not run Decepticon on the main host.
- Do not use bridged networking.
- Do not enable shared folders, shared clipboard, drag/drop, or USB passthrough.
- Do not connect the VM to the Windows/WSL local model.
- Do not put API keys in git or chat.
- Do not enable `c2-sliver` unless explicitly authorized.
- Do not enable `victims` unless intentionally running a local lab/demo.
- Do not start a real engagement without written authorization and scope.
- Do not use arbitrary source-IP spoofing; use approved VPN/static egress only.
