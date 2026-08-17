# ZeroTier VPS Controller Genesis — Handover — 2026-08-16

## Start Here
Read this file top to bottom. Follow each section in order.

## WHAT
A unified Docker Compose recipe (and companion skill) for standing up a 100% self-hosted ZeroTier controller + ZeroUI web interface (`dec0dos/zero-ui:latest`) on a raw Linux VPS, with zero public web-UI exposure at any point in the setup. The controller creates its own network purely via local REST API calls — no signup, no SaaS dependency on `my.zerotier.com`, no dashboard needed for setup. Designed to be agent-operable end to end: an AI agent with SSH access to a fresh VPS can run the entire flow via a single `docker-compose.yml`.

Live-tested & verified — ZeroUI connects directly to `127.0.0.1:9993` using `network_mode: "host"` with zero database server requirements.

## Dependencies
Check each before proceeding:
- [ ] Docker + docker compose on the target VPS: `docker --version && docker compose version`
- [ ] SSH key-based access to the target VPS already working — **confirm this BEFORE disabling password auth**, see skill Step 1 for the exact safety check
- [ ] `curl` (not `wget` — the `zerotier/zerotier` image doesn't ship it)

## Setup
1. Load `skills/zerotier-vps-controller-genesis/SKILL.md` — full step-by-step procedure lives there
2. Step 1: confirm key-based SSH access works, THEN harden (disable password auth)
3. Step 2: deploy unified `docker-compose.yml` running both `zerotier-controller` and `zeroui` in `network_mode: "host"`
4. Step 3: create the network + authorize nodes via local REST API (`http://localhost:9993/controller/network`)
5. Step 4: access ZeroUI management UI securely at `http://<zt-private-ip>:4444` (`admin` / `adminPassword123!`)

## Verify
`docker exec zerotier-controller zerotier-cli info` → expect `ONLINE`
`curl http://<public-ip>:4444` → expect refused/timeout (zero public exposure on firewall)
`curl -I http://<zt-private-ip>:4444` → expect `HTTP/1.1 302 Found` (ZeroUI reachable over ZeroTier mesh)

## Key Files
- `skills/zerotier-vps-controller-genesis/SKILL.md` — the actual procedure, live-tested and corrected, start here for any task
- `knowledge/zerotier-controller-genesis.md` — self-contained reference for this whole design, including every gotcha found during live testing
- `assets/zerotier-vps-controller-genesis/README.md` — original design draft (article-facing prose)

## Common Tasks
- Bootstrap a new self-hosted controller → load `skills/zerotier-vps-controller-genesis/SKILL.md`, follow Steps 1-4

## Operating Loop
See `CLAUDE.md` / `AGENT.md` — recall → actuate → persist.
