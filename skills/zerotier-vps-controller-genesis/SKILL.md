---
name: zerotier-vps-controller-genesis
description: Bootstrap a 100% self-hosted ZeroTier controller + ZeroUI (dec0dos/zero-ui) from scratch on a raw Linux VPS with ZERO public web UI exposure throughout the entire lifecycle.
version: 1.2.0
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

Deploy **ZeroUI** (`dec0dos/zero-ui:latest`) using `network_mode: "host"`. ZeroUI connects directly to `http://127.0.0.1:9993`, mounts `/var/lib/zerotier-one` read-only, and listens on port 4444:

```bash
cat << 'EOF' > docker-compose.zeroui.yml
services:
  zeroui:
    image: dec0dos/zero-ui:latest
    container_name: zeroui
    network_mode: "host"
    volumes:
      - /var/lib/zerotier-one:/var/lib/zerotier-one:ro
      - zeroui-data:/app/backend/data
    environment:
      - ZT_ADDR=127.0.0.1:9993
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

*ZeroUI is now accessible over your ZeroTier mesh at `http://10.246.231.x:4444`.*

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

## Relations
- guide: [[../../assets/zerotier-genesis-sovereign-mesh-guide.md]]
- knowledge: [[../../knowledge/zerotier-contabo1.md]]
- knowledge: [[../../knowledge/jym-sg-articles.md]]

## Changelog
### 2026-08-17
- Standardized on **ZeroUI** (`dec0dos/zero-ui:latest`) across all documentation and deployment recipes.
- Cleaned all legacy ZTNET references.
