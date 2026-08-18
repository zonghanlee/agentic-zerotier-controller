---
name: zerotier-vps-controller-genesis
description: Bootstrap a 100% self-hosted ZeroTier controller + ZeroUI (dec0dos/zero-ui) from scratch on a raw Linux VPS with ZERO public web UI exposure throughout the entire lifecycle.
version: 1.4.0
platforms: [linux]
metadata:
  hermes:
    tags: [zerotier, zeroui, self-hosted, vps, zero-trust, security]
    category: DEPLOYMENT
---

# Skill: zerotier-vps-controller-genesis

## Trigger Phrases
- "bootstrap self-hosted zerotier controller"
- "deploy zerotier controller with zero public UI exposure"
- "genesis deployment of ZeroUI"
- "spin up private zerotier mesh from scratch"

## What This Skill Does
Executes the zero-trust genesis deployment of a self-hosted ZeroTier Controller and ZeroUI management UI on a fresh Linux VPS:
1. Hardens SSH (Key-Only Auth).
2. Deploys the ZeroTier Controller daemon (`zerotier-controller`) via Docker Compose.
3. Bootstraps Network Zone 1 (`qLab`) silently via local REST API (no public web UI).
4. Joins & authorizes the operator client node (e.g. MacBook).
5. Deploys ZeroUI Web UI bound **exclusively** to port 4444 over ZeroTier.

---

## Procedure

### Step 1: Confirm key access, THEN harden SSH

**Never disable password auth before confirming key-based login actually works — doing so locks you out of the box with no recovery path short of a console/rescue-mode reboot.**

From a second session, confirm the key-based login succeeds:
```bash
ssh -o PasswordAuthentication=no -o PreferredAuthentications=publickey user@vps "echo key-auth-ok"
```
Only proceed once that prints `key-auth-ok`. Then disable password authentication:
```bash
sudo sed -i 's/^#\?PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### Step 2: Deploy Controller Engine (`zerotier-controller`)
Create the workspace directory and boot the ZeroTier controller daemon:

```bash
mkdir -p ~/zerotier-genesis/data
cd ~/zerotier-genesis

cat << 'EOF' > docker-compose.controller.yml
services:
  zerotier-controller:
    image: zerotier/zerotier:latest
    container_name: zerotier-controller
    network_mode: "host"
    devices:
      - /dev/net/tun
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
    volumes:
      - ./data:/var/lib/zerotier-one
    restart: unless-stopped
EOF

docker compose -f docker-compose.controller.yml up -d
```

### Step 3: Silent CLI Bootstrap (Create Zone 1 + Authorize Operator Node)
Run local REST API calls inside the VPS using `authtoken.secret` (no web UI needed).

**Use `curl`, not `wget`** — the `zerotier/zerotier` image does not include `wget` (confirmed live 2026-08-16, `wget: executable file not found`); it does have `curl`.

```bash
# 1. Fetch local authtoken
TOKEN=$(docker exec zerotier-controller cat /var/lib/zerotier-one/authtoken.secret)

# 2. Create Network Zone 1 ("qLab")
CREATE_RESP=$(docker exec zerotier-controller curl -s -X POST \
  -d '{"name":"qLab","private":true,"v4AssignMode":{"zt":true},"routes":[{"target":"10.246.231.0/24"}],"ipAssignmentPools":[{"ipRangeStart":"10.246.231.1","ipRangeEnd":"10.246.231.254"}]}' \
  -H "X-ZT1-Auth: $TOKEN" \
  "http://localhost:9993/controller/network")

# Extract generated NWID
NWID=$(echo $CREATE_RESP | grep -o '"id":"[^"]*' | cut -d'"' -f4)
echo "Zone 1 Created successfully! NWID: $NWID"

# 3. On Operator Laptop: Join the network
# zerotier-cli join $NWID
# Fetch Operator Laptop Node ID (e.g., 28ea0395f1)

OPERATOR_NODE_ID="<OPERATOR_LAPTOP_NODE_ID>"

# 4. Authorize Operator Laptop & Assign Static IP
docker exec zerotier-controller curl -s -X POST \
  -d '{"authorized":true,"ipAssignments":["10.246.231.203"]}' \
  -H "X-ZT1-Auth: $TOKEN" \
  "http://localhost:9993/controller/network/$NWID/member/$OPERATOR_NODE_ID"
```

*At this stage, the Operator Laptop and VPS are connected over the private ZeroTier mesh (`10.246.231.x`).*

**Testing tip**: for a quick connectivity test (not production), set `"private":false` on the network instead — public networks auto-accept any joining node, skipping step 4's per-member authorize call entirely.

**A controller does not automatically join its own network.** `zerotier-cli listnetworks` will correctly show empty right after Step 2 — verify network creation via the controller API (`GET /controller/network` lists the nwid) instead.

### Step 4: Deploy ZeroUI Management Interface (`dec0dos/zero-ui:latest`)

Deploy **ZeroUI** (`dec0dos/zero-ui:latest`) using `network_mode: "host"`. ZeroUI connects directly to `http://127.0.0.1:9993`, mounts the controller's data directory read-only, and listens on port 4444:

**Mount the controller's *actual* data path** — i.e. whatever `./data` maps to in Step 2's compose file (usually `~/zerotier-genesis/data`), NOT the generic `/var/lib/zerotier-one` host path. That system path may not exist on the host at all if the controller uses a local bind mount instead — mounting the wrong path makes ZeroUI crash-loop on a missing `authtoken.secret`.

**Use `ZU_CONTROLLER_ENDPOINT`, not `ZT_ADDR`** — `ZT_ADDR` is not read by this image. Set the endpoint explicitly to `http://127.0.0.1:9993/` (not `localhost`) — under Node 18, `localhost` resolves to `::1` first, and the controller isn't listening on the IPv6 loopback, so the connection fails silently with only a generic "Couldn't connect to the controller" log line.

```bash
cat << 'EOF' > docker-compose.zeroui.yml
services:
  zeroui:
    image: dec0dos/zero-ui:latest
    container_name: zeroui
    network_mode: "host"
    volumes:
      - ./data:/var/lib/zerotier-one:ro
      - zeroui-data:/app/backend/data
    environment:
      - ZU_CONTROLLER_ENDPOINT=http://127.0.0.1:9993/
      - ZU_DEFAULT_USERNAME=admin
      - ZU_DEFAULT_PASSWORD=YourSecurePassword123!
      - ZU_SECURE_HEADERS=false
      - PORT=4444
    restart: unless-stopped

volumes:
  zeroui-data:
EOF

docker compose -f docker-compose.zeroui.yml up -d
```

**Compose project-name footgun**: if the controller (Step 2) and ZeroUI (Step 4) are defined in separate compose files in the same directory, they share the same default Compose project name (the directory name) — Compose treats them as one project. Running `docker compose -f docker-compose.zeroui.yml down` will also stop and remove `zerotier-controller`. Prefer `stop`/`rm` scoped to a single service (never bare `down`) when only one service needs to be touched, or give each file a distinct `-p`/`COMPOSE_PROJECT_NAME`.

*ZeroUI is now accessible over your ZeroTier mesh at `http://10.246.231.x:4444`.*

### Step 5: Join the controller to its own network (required for a reachable ZT mesh IP)
Creating the network via the REST API does **not** make the controller a member of it — its own `zerotier-cli listnetworks` will stay empty and, since ZeroUI runs in `network_mode: "host"`, ZeroUI has no ZT mesh IP to be reached at until this step runs:

```bash
docker exec zerotier-controller zerotier-cli join <NWID>
TOKEN=$(docker exec zerotier-controller cat /var/lib/zerotier-one/authtoken.secret)
docker exec zerotier-controller curl -s -X POST \
  -d '{"authorized":true,"ipAssignments":["10.246.231.1"]}' \
  -H "X-ZT1-Auth: $TOKEN" \
  "http://localhost:9993/controller/network/<NWID>/member/<CONTROLLER_NODE_ID>"
docker exec zerotier-controller zerotier-cli listnetworks   # should now show OK with a 10.246.231.x IP
```

### Step 6 (optional but recommended): Interface-scoped `ufw` hardening
Enforces the zero-exposure guarantee at the host firewall itself, rather than relying on network topology (NAT/no port-forward) to keep port 4444 unreachable. Find the ZT interface name via `zerotier-cli listnetworks` (the `<dev>` column), then:

```bash
sudo ufw allow 22/tcp                                     # SSH stays open everywhere — don't lock yourself out
sudo ufw allow 9993/udp                                   # ZeroTier's own P2P port, keep open everywhere
sudo ufw allow in on <zt-iface> to any port 4444           # 4444 allowed ONLY via the ZeroTier interface
sudo ufw deny 4444                                         # 4444 blocked everywhere else (LAN + internet)
sudo ufw enable                                            # rules above must exist first, or you may lock yourself out
```

Order matters — ufw evaluates rules top-to-bottom, so the interface-specific `allow` must precede the general `deny`. A single SSH attempt timing out right after `enable` is often just an IPv6-then-IPv4 client fallback delay, not a lockout — retry with `ssh -4` before assuming the worst; an already-open SSH session survives `enable` regardless.

---

## Verification & Safety Check

1. **Test Public Access (Should FAIL)**:
   ```bash
   curl -I http://<vps-public-ip>:4444
   # Connection Refused / Timeout (0% public exposure)
   ```

2. **Test ZeroTier VPN Access (Should SUCCEED)**:
   - On Operator Laptop connected to ZeroTier:
   - Navigate to `http://10.246.231.240:4444`
   - Log in with default credentials (`admin` / `YourSecurePassword123!`). All networks and members will auto-populate.

---

## Gotchas & Hardening Notes
- **Back up `./data/identity.secret` immediately**: This file defines your 10-character Controller Node ID. Losing it destroys all network IDs.
- **Port 9993 UDP**: Must remain open on host firewall for P2P peer communications. **Do not remap the external port** (e.g. `19993:9993/udp`) unless truly unavoidable.
- **Port 4444 TCP**: Must NEVER be published to `0.0.0.0` on the public interface firewall.
- **`wget` is not in the `zerotier/zerotier` image** — use `curl` for all local REST API calls.
- **A controller is not automatically a member of its own network** — `zerotier-cli listnetworks` on the controller shows nothing until it explicitly joins; verify network creation via the controller API instead.
- **`ufw` inactive does not mean port 4444 is publicly reachable** — a public curl timeout can just as easily come from NAT/no port-forwarding on the network path, not host firewall enforcement. Don't conflate the two. Enabling `ufw` with explicit allow rules (SSH, `9993/udp`, default-deny) is a good future hardening step if the VPS ever gets a directly routable public IP — not required for the zero-exposure guarantee to hold today, but not equivalent to it either.

## Relations
- guide: [[../../assets/zerotier-genesis-sovereign-mesh-guide.md]]
- knowledge: [[../../knowledge/zerotier-contabo1.md]]
- knowledge: [[../../knowledge/jym-sg-articles.md]]

## Changelog
### 2026-08-18 (2)
- Added Step 5 (join controller to its own network — required for ZeroUI to have any reachable ZT mesh IP at all, previously undocumented) and Step 6 (optional interface-scoped `ufw` hardening) after a live end-to-end operator-node authorization + firewall lockdown run.

### 2026-08-18
- Fixed live deployment bugs found on a real VPS run: ZeroUI data-volume mount now points at the controller's actual `./data` path (not the generic `/var/lib/zerotier-one` host path, which may not exist); replaced non-functional `ZT_ADDR` with `ZU_CONTROLLER_ENDPOINT=http://127.0.0.1:9993/` to dodge a Node 18 `localhost`→`::1` resolution failure; documented the shared-project-name footgun where `docker compose down` on one compose file also tears down the other service's container.
- Added a gotcha noting `ufw` inactive ≠ port publicly reachable — don't conflate NAT/no-port-forward with host firewall enforcement.

### 2026-08-17
- Standardized on **ZeroUI** (`dec0dos/zero-ui:latest`) across all documentation and deployment recipes.
- Cleaned all legacy ZTNET references.
